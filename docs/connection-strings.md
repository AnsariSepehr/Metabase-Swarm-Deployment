# Connecting Metabase to a data source

After first-run setup, add data sources from **Admin Settings → Databases →
Add a database**. This document covers how we filled that form for a
Citus/Patroni Postgres cluster, plus general notes for other engines.

## General rule: fill the fields, skip the connection-string box

Metabase's "Connection string (optional)" field only pre-fills
host/port/database — it does **not** carry credentials. It's easier and less
error-prone to just fill in the individual fields below it directly:

| Field | What goes here |
|---|---|
| Display name | Anything descriptive — shown in the UI, doesn't affect the connection |
| Host | Hostname or IP Metabase can reach *on the same Docker network* |
| Port | The database's listening port |
| Database name | The specific database/schema to connect to |
| Username | A database role with (ideally read-only) access |
| Password | That role's password |

If you do want to use the connection-string field, the format is:
```
jdbc:postgresql://<host>:<port>/<database>
```

## Citus / Patroni clusters specifically

**Connect through the load balancer (haproxy), not a coordinator container
directly, if your cluster has one.**

Patroni promotes a new coordinator primary automatically on failover. If you
point Metabase straight at e.g. `coord2`, and `coord2` is later demoted to a
replica, Metabase's connection breaks silently until someone manually
repoints it. An haproxy (or similar) instance in front of the coordinators —
typically listening on a port dedicated to "coordinator primary access" —
handles that redirection automatically, so Metabase keeps working across
failovers with zero manual intervention.

If your cluster doesn't run such a proxy, connecting directly to a
coordinator hostname works, but you take on the responsibility of updating
Metabase's data source if/when that node's role changes.

Example (values are placeholders — substitute your own):

| Field | Value |
|---|---|
| Display name | `Citus - <cluster name>` |
| Host | `<haproxy-service-hostname>` (or a coordinator hostname if no proxy exists) |
| Port | `<haproxy's coordinator-primary port>` (or `5432` if connecting directly) |
| Database name | `citus` (or whatever `PATRONI_CITUS_DATABASE` is set to) |
| Username | see [Least-privilege role](#least-privilege-role-recommended) below |
| Password | that role's password |

**Never connect directly to a worker node.** Workers hold shard data only;
Citus assembles cross-shard query results at the coordinator. Querying a
worker directly will silently return incomplete or wrong results for any
distributed table.

## Least-privilege role (recommended)

Using your cluster's superuser (commonly `postgres`) for Metabase works, but
means anyone with Metabase admin access effectively has full DBA rights on
your production data — including the ability to alter schema. A scoped-down
read-only role is safer and takes two minutes:

```sql
-- Run on the coordinator, connected as an existing superuser
CREATE ROLE metabase_reader WITH LOGIN PASSWORD 'REPLACE_WITH_STRONG_SECRET';

GRANT CONNECT ON DATABASE citus TO metabase_reader;
GRANT USAGE ON SCHEMA public TO metabase_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO metabase_reader;

-- So future tables are covered automatically:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO metabase_reader;
```

Adjust `public` to whatever schema(s) actually hold the tables you want
Metabase to see, and repeat the `GRANT`/`ALTER DEFAULT PRIVILEGES` block per
schema if there are several.

Then use `metabase_reader` / that password as the Username/Password in
Metabase's data source form instead of the superuser.

## Other engines (quick reference)

| Engine | Typical Metabase fields | Notes |
|---|---|---|
| **PostgreSQL / Citus** | Host, Port (`5432`), Database, User, Password | SSL toggle if the server enforces it (`sslmode=require`/`verify-full`) |
| **MySQL / MariaDB** | Host, Port (`3306`), Database, User, Password | Use `useSSL` connection option if the server requires TLS |
| **SQL Server** | Host, Port (`1433`), Database, User, Password | Also supports "Instance name" instead of port for named instances |
| **MongoDB** | Connection string strongly recommended: `mongodb://<user>:<pass>@<host>:27017/<database>` | |
| **BigQuery / Snowflake / Redshift** | Service-account JSON / OAuth or key-pair, no plain user/password | Follow Metabase's per-cloud-provider setup docs; credentials are uploaded as a file/JSON blob, not typed in |

For anything not listed above, Metabase's own
[data source docs](https://www.metabase.com/docs/latest/databases/connecting)
are the authoritative reference — the shape above (host/port/db/user/pass vs.
connection-string vs. cloud credential file) is a good first guess for which
category a new engine falls into.
