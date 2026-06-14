<div align = "center">

# AivenIO Database Manager

[![Org. Home](https://img.shields.io/badge/aivenio-%F0%9F%91%93_DBA-black?logo=github&style=plastic)](https://github.com/aivenio)
![Contributor Covenant](https://img.shields.io/badge/👩‍💻_Contributor%20Covenant-2.1-4baaaa.svg?style=plastic)
![Contributing Guidelines](https://img.shields.io/badge/🤝-Contributing_Guidelines-blue?style=plastic)
[![License](https://img.shields.io/badge/⚖_License-GPL_v3.0-blue.svg?style=plastic)](https://www.gnu.org/licenses/gpl-3.0.en.html)
[![](https://img.shields.io/badge/💭-Community_Discussion-orange?style=plastic)](https://github.com/orgs/aivenio/discussions)
[![sponsors](https://img.shields.io/github/sponsors/aivenio?label=💰%20Active%20Sponsors&style=plastic)](https://github.com/sponsors/aivenio)
[![Members](https://img.shields.io/badge/AivenIO-Members_View-blue?logo=github&style=plastic)](https://github.com/orgs/aivenio/people)

[![GitHub Release](https://img.shields.io/github/v/release/aivenio/macrodb?label=🌎%20MacroDB&style=plastic)](https://github.com/aivenio/macrodb/releases)
[![GitHub Release](https://img.shields.io/github/v/release/aivenio/stocksdb?label=📈%20StocksDB&style=plastic)](https://github.com/aivenio/stocksdb/releases)
[![GitHub Release](https://img.shields.io/github/v/release/aivenio/tradesdb?label=🤖%20TradesDB&style=plastic)](https://github.com/aivenio/tradesdb/releases)

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

We're using logical replication (WAL) to sync data between different database tables, which is the practical way to manage foreign
key constraints across the servers. As an end user, if you are using **two** different servers, then the normal process should be
enough as below:

```pgsql
CREATE PUBICATION ...; -- on the publication server
CREATE SUBSCRIPTION ...; -- on the subscription server
```

However, if you are **using the same server with two different databases** (typically useful for data management, etc.), then
you may need to create a slot replication (in the subscriber server) method as per the detailed debugging steps below:

```pgsql
CREATE SUBSCRIPTION ...
  WITH (
    create_slot = false, enabled = false,
    slot_name = ...
  );

ALTER SUBSCRIPTION <slot-name> ENABLE;
```

A *practical deep-down documentation* is available [here](../docs/logicalReplication.md). This document was created from the
original server logs and steps to fix the issue.

### High-Level ER Diagram

The flowchart provides a high-level overview of the *information* available under different microservices and their
relationship with each other. For a detailed ER diagram, refer to the [`schema.dbml`](https://dbml.dbdiagram.io/home) file
in the respective repository.

```mermaid
flowchart TD

  subgraph MacroDB

    subgraph COUNTRY[Country Information]
      direction LR

      continent_mw --> country_mw
      region_mw --> country_mw
      subregion_mw --> country_mw
      country_mw --> state_mw
      state_mw --> city_mw

      region_mw --> subregion_mw
    end

    subgraph CURRENCY[Currency Information]
      direction LR

      currency_type_mw --> currency_mw
      currency_type_mw --> currency_mw

      currency_type_mw --> currency_subtype_mw

      forex_rate_tx
    end


  style COUNTRY fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
  style CURRENCY fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
  end

  subgraph StocksDB

    subgraph EXCHANGES[Stock Exchanges]
      direction LR

      stock_exchange_mw
    end

    subgraph SECURITIES[Security Information]
      direction LR

      securities_mw
      securities_exchange_symbol_mw
    end

  stock_exchange_mw --> securities_mw
  securities_mw --> securities_exchange_symbol_mw

  style EXCHANGES fill:#D3CFF0,stroke:#7B1FA2,stroke-width:2px
  style SECURITIES fill:#D3CFF0,stroke:#7B1FA2,stroke-width:2px
  end

  country_mw --IN--> stock_exchange_mw
  currency_mw --USING--> stock_exchange_mw

  style MacroDB fill:#D6A0DE,stroke:#7B1FA2,stroke-width:2px
  style StocksDB fill:#ADA0DE,stroke:#7B1FA2,stroke-width:2px
```

The diagram is designed to highlight the relationships between the two independent micro-services using *publication* rules
of the PostgreSQL database. A relationship is maintained in a `PUBLICATION → SUBSCRIPTION` format for understanding. A set
of important cross-functional linkages are as follows:

  1. A country has one/more listed stock exchanges (`country_mw --IN--> stock_exchange_mw`) where a security
    is being traded. Cros-functional link is maintained via the **`macrodb_geography_table`** publication.
  2. A security exchange primarily operates using (`currency_mw --USING--> stock_exchange_mw`) a set of currency
    which is part of the country. Cross-functional link is maintained via the **`macrodb_currency_table`** publication.

## ⚖ Project Licensing

Our projects strictly follow [`GNU GPL v3`](https://www.gnu.org/licenses/gpl-3.0.en.html), a strong copy-left license. Please
refer to the individual `LICENSE` file in each repository for complete information and important terms and conditions.

## ⚠ Project Disclaimer

The service is intended solely to provide a data structure that enables efficient management of databases containing various
data points that can be used effectively for analysis. Certain *non-sensitive data* that is *available in the public domain*
may be distributed with the project. Other data may not be shared, and the source of the same may not be disclosed; the
organization is under no obligation to make such data available to the general public.

In a certain project, there might be information available on tradable securities. Any information or discussions are for
general information and educational purposes only. It does not constitute financial, investment, legal, tax, or accounting
advice, and should not be relied upon as such.

All content, including but not limited to market data, analysis, commentary, and any other materials, is provided in good
faith but without warranty of any kind, express or implied. We make no representation or guarantee regarding the accuracy,
completeness, or timeliness of any information presented.

</div>
