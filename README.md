# Debezium CDC Sandbox

A local sandbox for experimenting with Change Data Capture (CDC) using Debezium Server, with PostgreSQL, Oracle, and SQL Server as data sources and Redis as the message sink.

## Architecture

```
[PostgreSQL] ──┐
               ├──> [Debezier Server #1] ──> [Redis Streams] ──> [Python Consumer]
[Oracle DB] ───┤                                                ──> [MinIO / S3]
[SQL Server] ──┘
```

## Services

| Service | Image | Port | Purpose |
|---|---|---|---|
| `postgres` | `postgres:18.0-alpine` | 5432 | Source database (PostgreSQL) |
| `oracle` | `gvenzl/oracle-free:23.8-full` | 1521 | Source database (Oracle) |
| `mssql` | `mcr.microsoft.com/mssql/server:2022-latest` | 1433 | Source database (SQL Server) |
| `redis` | `redis:7.2-alpine` | 6379 | Message sink (Redis Streams) |
| `redis-insight` | `redislabs/redisinsight:2.66` | 5540 | Redis GUI dashboard |
| `debezium-server` | `quay.io/debezium/server:2.4` | — | CDC connector for PostgreSQL |
| `debezium-server-oracle` | `quay.io/debezium/server:2.7` | — | CDC connector for Oracle |
| `debezium-server-mssql` | `quay.io/debezium/server:2.7` | — | CDC connector for SQL Server |
| `minio` | `quay.io/minio/minio` | 9000, 9001 | S3-compatible object storage |

## Quick Start

### 1. Start all services

```bash
docker compose up -d
```

### 2. Verify services are running

```bash
docker compose ps
```

### 3. Explore CDC events

Connect to Redis to see streamed CDC events:

```bash
redis-cli
> XREAD STREAMS cdc.public.<table_name> COUNT 10 $
> XREAD STREAMS cdc-oracle.<schema>.<table_name> COUNT 10 $
> XREAD STREAMS cdc-mssql.<schema>.<table_name> COUNT 10 $
```

Or use **Redis Insight** at http://localhost:5540

### 4. (Optional) Python consumer

Enable the `python-consumer` service in `docker-compose.yaml` by uncommenting it, then restart:

```bash
docker compose up -d python-consumer
```

Configure the `STREAM_KEY` in `python-consumer/consumer.py` to match your table, e.g.:

```python
STREAM_KEY = 'cdc.public.users'       # PostgreSQL
STREAM_KEY = 'cdc-oracle.c##appuser.customers'  # Oracle
STREAM_KEY = 'cdc-mssql.dbo.customers'  # SQL Server
```

## Oracle Setup

The Oracle container auto-initializes on first run via scripts in `oracle-init/`:

- `01-enable-archivelog.sh` — Enables ARCHIVELOG mode and supplemental log data
- `02-setup-logminer.sql` — Creates Debezium user (`c##dbzuser`), application user (`c##appuser`), and a sample `customers` table

## SQL Server Setup

The SQL Server container requires initialization via `mssql-init/init-db.py`:

1. Ensure SQL Server is running:
   ```bash
   docker compose up -d mssql
   ```

2. Run the initialization script (requires `pymssql`):
   ```bash
   pip install pymssql
   python mssql-init/init-db.py
   ```

This creates the `sourcedb` database, `customers` table, and enables CDC.

## Configuration

### PostgreSQL CDC

| Environment Variable | Value | Description |
|---|---|---|
| `DEBEZIUM_SOURCE_CONNECTOR_CLASS` | `PostgresConnector` | Source connector type |
| `DEBEZIUM_SOURCE_DATABASE_HOSTNAME` | `postgres` | Hostname of PostgreSQL |
| `DEBEZIUM_SOURCE_DATABASE_PORT` | `5432` | PostgreSQL port |
| `DEBEZIUM_SOURCE_DATABASE_USER` | `admin` | Database user |
| `DEBEZIUM_SOURCE_DATABASE_PASSWORD` | `secret` | Database password |
| `DEBEZIUM_SOURCE_DATABASE_DBNAME` | `sourcedb` | Database name |
| `DEBEZIUM_SOURCE_TOPIC_PREFIX` | `cdc` | Stream key prefix in Redis |
| `DEBEZIUM_SOURCE_SCHEMA_INCLUDE_LIST` | `public` | Schemas to capture |
| `DEBEZIUM_SOURCE_PLUGIN_NAME` | `pgoutput` | Logical decoding plugin |

### Oracle CDC

| Environment Variable | Value | Description |
|---|---|---|
| `DEBEZIUM_SOURCE_CONNECTOR_CLASS` | `OracleConnector` | Source connector type |
| `DEBEZIUM_SOURCE_DATABASE_HOSTNAME` | `oracle` | Hostname of Oracle |
| `DEBEZIUM_SOURCE_DATABASE_PORT` | `1521` | Oracle port |
| `DEBEZIUM_SOURCE_DATABASE_USER` | `c##dbzuser` | Debezium connector user |
| `DEBEZIUM_SOURCE_DATABASE_PASSWORD` | `dbzpassword` | Connector user password |
| `DEBEZIUM_SOURCE_DATABASE_DBNAME` | `FREE` | Oracle service name |
| `DEBEZIUM_SOURCE_TOPIC_PREFIX` | `cdc-oracle` | Stream key prefix in Redis |
| `DEBEZIUM_SOURCE_SCHEMA_INCLUDE_LIST` | `C##APPUSER` | Schema to capture |

### SQL Server CDC

| Environment Variable | Value | Description |
|---|---|---|
| `DEBEZIUM_SOURCE_CONNECTOR_CLASS` | `SqlServerConnector` | Source connector type |
| `DEBEZIUM_SOURCE_DATABASE_HOSTNAME` | `mssql` | Hostname of SQL Server |
| `DEBEZIUM_SOURCE_DATABASE_PORT` | `1433` | SQL Server port |
| `DEBEZIUM_SOURCE_DATABASE_USER` | `sa` | Database user |
| `DEBEZIUM_SOURCE_DATABASE_PASSWORD` | `YourStrong!Passw0rd` | Database password |
| `DEBEZIUM_SOURCE_DATABASE_NAMES` | `sourcedb` | Database name |
| `DEBEZIUM_SOURCE_TOPIC_PREFIX` | `cdc-mssql` | Stream key prefix in Redis |
| `DEBEZIUM_SOURCE_DATABASE_ENCRYPT` | `false` | Disable SSL cert check for SQL Server 2022 |

To include specific tables in PostgreSQL CDC, uncomment:

```yaml
- DEBEZIUM_SOURCE_TABLE_INCLUDE_LIST=public.users,public.orders
```

## Stream Key Formats

Events are published to Redis Streams with these key patterns:

- **PostgreSQL:** `cdc.<schema>.<table>` (e.g. `cdc.public.users`)
- **Oracle:** `cdc-oracle.<schema>.<table>` (e.g. `cdc-oracle.c##appuser.customers`)
- **SQL Server:** `cdc-mssql.<schema>.<table>` (e.g. `cdc-mssql.dbo.customers`)

## Stopping & Cleanup

```bash
# Stop all services
docker compose down

# Stop and remove volumes (erases CDC offsets and schema history)
docker compose down -v
```

## Project Structure

```
├── docker-compose.yaml        # Service definitions
├── mssql-init/                # SQL Server initialization scripts
│   └── init-db.py
├── oracle-init/               # Oracle initialization scripts
│   ├── 01-enable-archivelog.sh
│   └── 02-setup-logminer.sql
├── python-consumer/           # Python CDC consumer
│   └── consumer.py
├── read.ipynb                 # Jupyter: read CDC events via Redis
├── write.ipynb                # Jupyter: write data to source databases
└── README.md
```
