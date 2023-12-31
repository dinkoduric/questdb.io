---
title: Grafana
description: Guide for time-series data visualization with QuestDB and Grafana
---

[Grafana](https://grafana.com/) is a popular observability and monitoring
application used to visualize data and help with time-series data analysis. It
has an extensive ecosystem of widgets and plugins. QuestDB supports connecting
to Grafana via the [Postgres](/docs/reference/api/postgres/) endpoint.

## Prerequisites

- [Docker](/docs/get-started/docker/) to run both Grafana and QuestDB

We will use the `--add-host` parameter for both Grafana and QuestDB.

## Start Grafana

Start Grafana using `docker run`:

```shell
docker run --add-host=host.docker.internal:host-gateway \
-p 3000:3000 --name=grafana \
-v grafana-storage:/var/lib/grafana \
grafana/grafana-oss
```

Once the Grafana server has started, you can access it via port 3000
(`http://localhost:3000`). The default login credentials are as follows:

```shell
user:admin
password:admin
```

## Start QuestDB

The Docker version for QuestDB can be run by exposing the port `8812` for the
PostgreSQL connection and port `9000` for the web and REST interface:

```shell
docker run --add-host=host.docker.internal:host-gateway \
-p 9000:9000 -p 9009:9009 -p 8812:8812 -p 9003:9003 \
-e QDB_PG_READONLY_USER_ENABLED=true \
questdb/questdb:latest

```

## Add a data source

1. Open Grafana's UI (by default available at `http://localhost:3000`)
2. Go to the `Configuration` section and click on `Data sources`
3. Click `Add data source`
4. Choose the `PostgreSQL` plugin and configure it with the following settings:

```
host:host.docker.internal:8812
database:qdb
user:user
password:quest
TLS/SSL mode:disable
```

5. When adding a panel, use the "text edit mode" by clicking the pencil icon and
  adding a query

## Global variables

Use
[global variables](https://grafana.com/docs/grafana/latest/variables/variable-types/global-variables/#global-variables)
to simplify queries with dynamic elements such as date range filters.

### `$__timeFilter(timestamp)`

This variable allows filtering results by sending a start-time and end-time to
QuestDB. This expression evaluates to:

```questdb-sql
timestamp BETWEEN
    '2018-02-01T00:00:00Z' AND '2018-02-28T23:59:59Z'
```

### `$__interval`

This variable calculates a dynamic interval based on the time range applied to
the dashboard. By using this function, the sampling interval changes
automatically as the user zooms in and out of the panel.

## Example query

```questdb-sql
SELECT
  pickup_datetime AS time,
  avg(trip_distance) AS distance
FROM taxi_trips
WHERE $__timeFilter(pickup_datetime)
SAMPLE BY $__interval;
```

## Known issues

For alert queries generated by certain Grafana versions, the macro
`$__timeFilter(timestamp)` produces timestamps with nanosecond precision, while
the expected precision is millisecond precision. As a result, the alert queries
are not compatible with QuestDB and lead to an `Invalid date` error. To resolve
this, we recommend the following workaround:

```questdb-sql
SELECT
  pickup_datetime AS time,
  avg(trip_distance) AS distance
FROM taxi_trips
WHERE pickup_datetime BETWEEN cast($__unixEpochFrom()*1000000L as timestamp) and cast($__unixEpochTo()*1000000L as timestamp)

```

See [Grafana issues](https://github.com/grafana/grafana/issues/51611) for more
information.

## See also

- [Build a monitoring dashboard with QuestDB and Grafana](/blog/time-series-monitoring-dashboard-grafana-questdb/)
