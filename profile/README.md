<div align = "center">

# AivenIO Database Manager

[![Org. Home](https://img.shields.io/badge/aivenio-%F0%9F%91%93_DBA-black?logo=github&style=plastic)](https://github.com/aivenio)
![Contributor Covenant](https://img.shields.io/badge/👩‍💻_Contributor%20Covenant-2.1-4baaaa.svg?style=plastic)
![Contributing Guidelines](https://img.shields.io/badge/🤝-Contributing_Guidelines-blue?style=plastic)
[![License](https://img.shields.io/badge/⚖_License-GPL_v3.0-blue.svg?style=plastic)](https://www.gnu.org/licenses/gpl-3.0.en.html)
[![](https://img.shields.io/badge/💭-Community_Discussion-orange?style=plastic)](https://github.com/orgs/aivenio/discussions)
[![sponsors](https://img.shields.io/github/sponsors/aivenio?label=💰%20Active%20Sponsors&style=plastic)](https://github.com/sponsors/aivenio)
[![Members](https://img.shields.io/badge/AivenIO-Members_View-blue?logo=github&style=plastic)](https://github.com/orgs/aivenio/people)

[![GitHub Release](https://img.shields.io/github/v/release/aivenio/macrodb?label=MacroDB&style=plastic)](https://github.com/aivenio/macrodb/releases)
[![GitHub Release](https://img.shields.io/github/v/release/aivenio/stocksdb?label=StocksDB&style=plastic)](https://github.com/aivenio/stocksdb/releases)

</div>

<div align = "justify">

👋 [**`AivenIO`**](https://github.com/aivenio) is a scalable database platform built for modern data services - fully managed,
cloud-native, and highly adaptable to your specific requirements. The usage is infinite - from a collection of awesome data for
projects or integration with applications that provide data-driven (backed) decision-making for a holistic picture in the fields
of (but not limited to) science and technology, finance, and macroeconomic factors.

## ✨ Getting Started

The repository provides PostgreSQL as the de facto backend database, divided into *microservices* that can work independently or
in coordination with other projects. The different microservices can either be configured in separate servers (clouds, systems)
or can be configured in the same host under a different named schema. Check our [website](https://aivenio.github.io/) or our
community discussion [page](https://github.com/orgs/aivenio/discussions) for more details. Check the individual project's updated
release from the badges.

### PUBLISHER → SUBSCRIPTION Database Tables

We're using logical replication (WAL) to sync data between different database tables which is the practical way to manage foreign
key constraints across the servers. As an end user, if you are using **two** different server, then normal process should be
enough as below:

```pgsql
CREATE PUBICATION ...; -- on the publication server
CREATE SUBSCRIPTION ...; -- on the subscription server
```

However, if you are **using the same server with two different database** (typically useful for data management, etc.) then
you may need to create slot replication (in the subscriber server) method as per detailed debugging steps below:

```pgsql
CREATE SUBSCRIPTION ...
  WITH (
    create_slot = false, enabled = false,
    slot_name = ...
  );

ALTER SUBSCRIPTION <slot-name> ENABLE;
```

A practical deep down documentation is available [here](../docs/logicalReplication.md). This document was created from the
original server logs and steps to fix the issue. 

## ⚖ Project Licensing

Our projects strictly follow [`GNU GPL v3`](https://www.gnu.org/licenses/gpl-3.0.en.html), a strong copy-left license. Please
refer to the individual `LICENSE` file for more information.

## ⚖ Project Disclaimer

The service is intended solely to provide a data structure that enables efficient management of databases containing various
data points that can be used effectively for analysis. Certain *non-sensitive data* that is *available in the public domain*
may be distributed with the project. Other data may not be shared, and the source of the same may not be disclosed; the
organization is under no obligation to make such data available to the general public.

In a certain project, there might be information available of tradeble securities. Any information or discussions are for
general information and educational purposes only. It does not constitute financial, investment, legal, tax, or accounting
advice, and should not be relied upon as such.

All content, including but not limited to market data, analysis, commentary, and any other materials, is provided in good
faith but without warranty of any kind, express or implied. We make no representation or guarantee regarding the accuracy,
completeness, or timeliness of any information presented.

</div>
