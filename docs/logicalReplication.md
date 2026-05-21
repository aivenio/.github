# Logical Replication on the Same Cluster: A Debugging Walkthrough

A field guide written from a real incident: setting up PostgreSQL logical
replication where the publisher database and subscriber database lived on the
same PostgreSQL cluster, hitting an invisible deadlock during
`CREATE SUBSCRIPTION`, and working through the diagnosis with pgAdmin as the
client.

> **TL;DR** — When publisher and subscriber are in the same PostgreSQL cluster,
> `CREATE SUBSCRIPTION` can deadlock against its own slot-creation walsender
> because they share the same xid space. PostgreSQL's deadlock detector
> can't see it (one side is a network wait, not a heavyweight lock).
> Fix: split slot creation out of the subscription transaction using
> `WITH (create_slot = false, enabled = false)`, create the slot separately
> on the publisher, then `ALTER SUBSCRIPTION ... ENABLE`.

---

## 1. Context

- **Goal:** allow tables in database `b` (`stocksdb`) to reference tables in
  database `a` (`macrodb`) for FK-like integrity.
- **Important constraint:** PostgreSQL does **not** support foreign keys across
  databases. The chosen workaround was logical replication: mirror the
  reference tables from `macrodb` into a schema in `stocksdb`, then create
  real FKs against the local copy.
- **Crucial detail (discovered late):** both databases were on the **same
  PostgreSQL cluster**. This is the root cause of nearly every problem below.

> **Lesson up front:** if publisher and subscriber share a cluster, use
> **schemas in one database** instead of logical replication. Logical
> replication is designed for *physical* separation (different machines,
> versions, regions). Inside one cluster it gives you all the operational
> cost and none of the benefit.

---

## 2. Symptoms

1. `CREATE SUBSCRIPTION` in pgAdmin sat at "Running" for over an hour for a
   20 MB table.
2. Cancelling in pgAdmin did not clearly release things.
3. Re-running with `WITH (create_slot = false, enabled = false)` *also*
   appeared to hang — but turned out to be a pgAdmin UI artifact.

---

## 3. The Diagnostic Toolbox

These queries solved the case. Keep them bookmarked.

### 3.1 What's happening right now (run on **publisher**, and on subscriber)

```sql
SELECT pid, usename, application_name, state,
       wait_event_type, wait_event,
       backend_start, xact_start,
       now() - COALESCE(xact_start, backend_start) AS age,
       query
FROM pg_stat_activity
WHERE backend_type IN ('walsender', 'client backend')
ORDER BY backend_start;
```

> Note: `pg_stat_activity` is **cluster-wide**, not per-database. When
> publisher and subscriber are on the same cluster, you see everything from
> either side. The duplicate pids across both "sides" of the output were the
> first clue that we had one cluster, not two.

### 3.2 Who is blocking whom

```sql
SELECT pid,
       pg_blocking_pids(pid) AS blocked_by,
       state, wait_event_type, wait_event, backend_xmin,
       application_name, query
FROM pg_stat_activity
WHERE backend_type IN ('walsender', 'client backend')
  AND pid <> pg_backend_pid()
ORDER BY backend_start;
```

`pg_blocking_pids()` is the single most useful function for this class of
problem. If it's empty for every row, you don't have a lock contention
problem — look elsewhere (connection, snapshot, application).

### 3.3 Subscription-side state

On the **subscriber**:

```sql
-- Per-table sync state
SELECT srrelid::regclass AS table_name, srsubstate, srsublsn
FROM pg_subscription_rel
WHERE srsubid = (SELECT oid FROM pg_subscription
                 WHERE subname = 'macrodb_currency_reference');

-- Subscription stats (PG 15+)
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_stat_subscription_stats;
```

`srsubstate` decoder:

| Code | Meaning                                    |
|------|--------------------------------------------|
| `i`  | initializing — hasn't started              |
| `d`  | copying data                               |
| `f`  | finished copy, catching up via WAL         |
| `s`  | synchronized                               |
| `r`  | ready (normal streaming)                   |

Stuck at `i` or `d` with no LSN progress = nothing is moving.

### 3.4 Replication slot state (run on **publisher**)

```sql
SELECT slot_name, slot_type, active, active_pid, restart_lsn,
       confirmed_flush_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS retained_wal
FROM pg_replication_slots;
```

What to look for:
- `active = true` with an `active_pid` → walsender is connected and using it.
- `active = false` with non-null `restart_lsn` → orphaned slot from a previous
  attempt; it's holding back WAL until dropped or reattached.
- Growing `retained_wal` over time → a slot has died but isn't dropped;
  publisher disk will fill.

### 3.5 Replication connection state (run on **publisher**)

```sql
SELECT pid, application_name, state, sent_lsn, write_lsn, flush_lsn,
       replay_lsn, sync_state
FROM pg_stat_replication;
```

`state` goes `startup` → `catchup` → `streaming`. Stuck at `startup` for more
than a few seconds usually means a snapshot or lock wait on the publisher.

---

## 4. The Deadlock: What Actually Happened

The `pg_blocking_pids` query revealed this on the publisher:

```
pid    blocked_by    wait_event_type    wait_event         query
87213  {}            Client             LibpqwalreceiverReceive   CREATE SUBSCRIPTION ...
87313  {87213}       Lock               transactionid             CREATE_REPLICATION_SLOT ...
```

A circular wait:

```
  pid 87213 (CREATE SUBSCRIPTION on subscriber db)
      │
      ├─ holds an open transaction that has touched pg_subscription
      │  (a shared catalog visible across the cluster)
      │
      └─ has opened a libpq connection to the publisher db on the
         same cluster, now waiting on the network for the slot
         creation to return
                                │
                                ▼
  pid 87313 (walsender on publisher db, CREATE_REPLICATION_SLOT)
      │
      └─ executing XactLockTableWait(87213) — waiting for transaction
         87213 to commit before it can establish a consistent state
         for the new logical slot
                                │
                                ▼
                       waits for 87213 ────────┐
                                               │
                       (which waits for 87313) ┘
```

**Why the deadlock detector can't see it:** PostgreSQL's deadlock detector only
inspects heavyweight lock waits. One side here is `wait_event_type = Client`
(blocked on a `recv()` from the libpq socket), which the detector ignores.
So the cycle is real but invisible to automatic resolution.

**Why it only happens on the same cluster:** with a remote publisher, the two
backends would live in different xid spaces. The walsender on the publisher
cannot see (and therefore cannot wait on) a transaction on the subscriber,
because they're different clusters. `XactLockTableWait` only works
intra-cluster.

---

## 5. Resolution: Break Slot Creation Out of the Subscription Transaction

The fix is to do the two synchronous steps that `CREATE SUBSCRIPTION` normally
combines — create the slot, and register the subscription — in two
**separate** transactions, neither of which has the deadlock condition.

### 5.1 Clean up the stuck attempt

```sql
-- On SUBSCRIBER: terminate the deadlocked CREATE SUBSCRIPTION backend
SELECT pg_cancel_backend(<pid>);     -- try cancel first
SELECT pg_terminate_backend(<pid>);  -- if cancel doesn't take

-- On PUBLISHER: check for and drop any half-created slot
SELECT slot_name, active FROM pg_replication_slots
WHERE slot_name = 'macrodb_currency_reference_stocksdb';

-- If present and inactive:
SELECT pg_drop_replication_slot('macrodb_currency_reference_stocksdb');
```

Note: in the actual incident, the first attempt *did* successfully create the
slot before deadlocking. We chose to **reuse** that slot in step 5.2 rather
than drop and recreate, by using its existing name with `create_slot = false`.

### 5.2 Create the subscription without inline slot creation

```sql
-- On SUBSCRIBER
CREATE SUBSCRIPTION macrodb_currency_reference
  CONNECTION '...'
  PUBLICATION macrodb_currency_table
  WITH (
    create_slot = false,    -- do NOT create the slot inline
    enabled     = false,    -- do NOT start workers yet
    slot_name   = 'macrodb_currency_reference_stocksdb'
  );
```

This returns in milliseconds. It only inserts a row in `pg_subscription` and
validates that the publisher is reachable. No slot is touched, no worker is
started, no cross-database transaction wait.

### 5.3 Create the slot in its own session on the publisher

Only needed if the slot doesn't already exist:

```sql
-- On PUBLISHER
SELECT pg_create_logical_replication_slot(
  'macrodb_currency_reference_stocksdb', 'pgoutput'
);
```

Because this runs in its own short transaction, it doesn't deadlock with
anything.

### 5.4 Enable the subscription

```sql
-- On SUBSCRIBER
ALTER SUBSCRIPTION macrodb_currency_reference ENABLE;
```

Apply and tablesync workers start in the background. Watch progress with the
queries from §3.3. For a 20 MB table, you should see all rows reach
`srsubstate = 'r'` within seconds.

---

## 6. The pgAdmin Trap

A separate issue that confused diagnosis: pgAdmin's query tool kept showing
"Running" even when the server had nothing active.

### 6.1 What we saw

After running the new `CREATE SUBSCRIPTION` with `create_slot = false`,
pgAdmin's spinner kept turning. But running the diagnostic query showed:

- Every backend in `state = 'idle'`
- The original cancelled backend showing its *previous* query text in
  `pg_stat_activity` (idle connections still show their last query)
- No `CREATE SUBSCRIPTION` in `state = 'active'` anywhere on the cluster

The query had **already completed**. pgAdmin's UI was stale.

### 6.2 Rule of thumb

`pg_stat_activity` is the source of truth. The pgAdmin spinner is not.

> If no row with your query text shows `state = 'active'` on the cluster,
> your query is not running — full stop, regardless of what pgAdmin says.

### 6.3 pgAdmin-specific guidance

- After a long-running or cancelled query, **verify state directly** via
  `pg_subscription`, `pg_replication_slots`, etc. — don't trust the tab.
- pgAdmin's "stop" button is unreliable for hung queries. Use
  `pg_cancel_backend(pid)` / `pg_terminate_backend(pid)` instead.
- An idle pgAdmin tab can hold a transaction open if you ran `BEGIN`
  manually. Check for `state = 'idle in transaction'` with non-null
  `xact_start`.
- For replication and other long operations, prefer `psql` over pgAdmin —
  the feedback loop is tighter and the state is unambiguous.

---

## 7. Step-by-Step Playbook for Future Replication Issues

When a `CREATE SUBSCRIPTION` or replication operation appears stuck:

### Step 1 — Confirm the topology

Are the publisher and subscriber on the **same cluster**?

```sql
-- On both sides
SELECT inet_server_addr(), inet_server_port(), current_database();
```

If addresses and ports match, they're the same cluster. This single fact
explains many "weird" replication problems and points you back at the
"use schemas instead" decision.

### Step 2 — Look at the subscriber log

The most important step. Apply and tablesync workers report errors **only**
to the server log. The psql/pgAdmin session that ran `CREATE SUBSCRIPTION`
returns success even when the worker is failing immediately afterward.

Common log signatures:

| Log message                                             | Cause                                |
|---------------------------------------------------------|--------------------------------------|
| `could not connect to the publisher`                    | bad host/port/password/pg_hba.conf   |
| `duplicate key value violates unique constraint`        | pre-existing rows on subscriber      |
| `insert or update on table ... violates foreign key`    | FK to non-replicated table           |
| `relation "..." does not exist`                         | target table not created on subscriber |
| `logical replication worker ... exited` (looping)       | apply error retry loop               |

### Step 3 — Check what's active on the server

Run §3.1. Anything with state `active` and an old `xact_start` is interesting.

### Step 4 — Check for lock waits

Run §3.2. Any non-empty `blocked_by` array points directly at the cause.

If a walsender (`application_name` = your subscription name) is blocked by
your own `CREATE SUBSCRIPTION` backend → it's the same-cluster deadlock.
Apply the fix in §5.

### Step 5 — Check subscription and slot state

Run §3.3 and §3.4. Look for:
- `srsubstate` stuck at `i` or `d` for more than a minute (worker is failing
  silently — go back to step 2 and read the log)
- Slot `active = false` with `restart_lsn` set (orphaned from a prior attempt)
- Slot `active = true` with an `active_pid` that doesn't belong to your
  current subscription (a previous attempt's worker never died)

### Step 6 — Check for long-running transactions on the publisher

```sql
-- On publisher
SELECT pid, state, xact_start, now() - xact_start AS age,
       application_name, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Even a transaction on completely unrelated tables can block snapshot
acquisition during slot creation. `idle in transaction` from a forgotten
pgAdmin/psql tab is the usual culprit.

### Step 7 — Decide: kill or wait

- Stale connections, idle-in-transaction sessions, leftover walsenders →
  `pg_terminate_backend(pid)`.
- Legitimate work (`pg_dump`, application transaction) → wait it out.
- Orphan slot blocking a fresh attempt → `pg_drop_replication_slot()`.

### Step 8 — If `CREATE SUBSCRIPTION` keeps deadlocking on same cluster

Use the §5 pattern unconditionally:

```sql
CREATE SUBSCRIPTION ...
  WITH (create_slot = false, enabled = false, slot_name = '...');
-- then on publisher:
SELECT pg_create_logical_replication_slot('...', 'pgoutput');
-- then on subscriber:
ALTER SUBSCRIPTION ... ENABLE;
```

This avoids the same-cluster deadlock by construction.

---

## 8. Quick Reference: Subscription Options That Matter

| Option              | Default | Why you might change it                                  |
|---------------------|---------|----------------------------------------------------------|
| `create_slot`       | `true`  | Set `false` to avoid same-cluster deadlock, or when reusing an existing slot |
| `enabled`           | `true`  | Set `false` to create then start manually with `ALTER SUBSCRIPTION ... ENABLE` |
| `slot_name`         | sub name | Set explicitly when reusing an existing slot or naming for clarity |
| `copy_data`         | `true`  | Set `false` if you've already synced the table manually  |
| `disable_on_error`  | `false` | **Set `true`** (PG 15+) to stop apply-error retry loops  |
| `synchronous_commit`| `off`   | Tune carefully; affects durability on subscriber side    |

---

## 9. Lessons Learned

1. **Same cluster + logical replication = trouble.** Use schemas inside one
   database whenever publisher and subscriber would otherwise be on the same
   PostgreSQL instance. Real FKs, atomic transactions, joins, zero lag,
   simpler ops.

2. **`pg_stat_activity` is cluster-wide.** Duplicate pids on "both sides"
   means there is only one side.

3. **The deadlock detector has a blind spot:** it cannot see cycles that
   cross between heavyweight locks and `Client` waits (libpq network reads).
   Use `pg_blocking_pids()` to surface these.

4. **`CREATE SUBSCRIPTION` is not one thing.** It is at least three:
   register subscription, create slot, start workers. Splitting these via
   `create_slot = false` / `enabled = false` makes debugging dramatically
   easier even when there's no deadlock.

5. **pgAdmin's UI lies sometimes.** Trust the catalog views, not the spinner.
   Long-running replication operations are better driven from `psql`.

6. **Always check the subscriber log.** Most replication problems are obvious
   in one log line and invisible from SQL.

7. **Orphan slots are silent disk killers.** Inactive slots retain WAL
   forever. Whenever a subscription attempt fails, check
   `pg_replication_slots` before retrying.
