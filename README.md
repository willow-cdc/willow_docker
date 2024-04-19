<p align="center">
  <img src="https://github.com/willow-cdc/willow/blob/main/frontend/src/assets/Willow Logo Transparent.png" width="300" height="auto" />
</p>

# Table Of Contents

- [Prerequisites](#prerequisites)
  - [PosgreSQL](#postgresql)
    - [Configuration](#configuration)
    - [Using a Minimum Priviledged User](#using-a-minimum-privileged-user)
  - [Redis](#redis)
- [Installation & Usage](#installation--usage)
  - [Create a Pipeline](#create-a-pipeline)
  - [View Existing Pipelines](#view-existing-pipelines)
  - [Teardown a Pipeline](#teardown-a-pipeline)

# Prerequisites

In order for Willow to successfully replicate from PostgreSQL to Redis, the following prerequisites must be met:

- PostgreSQL server is set up for logical replication
- Docker is installed on the server running Willow
- Redis server must have RedisJSON and RediSearch modules installed

## PostgreSQL

Instructions are written based on [Debezium's documentation](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#setting-up-postgresql) for configuring PostgreSQL. The following instructions assume that the PostgreSQL user used for replication is named `willow`.

### Configuration

Ensure your PostgreSQL server is configured to support logical replication with the `pgoutput` plugin. This includes:

1. Using PostgreSQL version 10+.
2. Setting the WAL (write-ahead log) level to logical.

```sql
ALTER SYSTEM SET wal_level = logical;
```

3. Adding entries to your PostgreSQL's `pg_hba.conf` file to allow the server running Willow to replicate from the PostgreSQL host.

```
local   replication   willow                      trust
host    replication   willow     127.0.0.1/32     trust
host    replication   willow     ::1/128          trust
```

4. Restart the PostgreSQL server for the changes to take effect.

### Using a Minimum Privileged User

In order to replicate data from PostgreSQL's WAL, you must provide Willow with a PostgreSQL user. This user can either be a `SUPERUSER` or have the minimum required privileges.

The following SQL command can be run in the PostgreSQL terminal to create a `SUPERUSER`. Be sure to provide a unique password.

```sql
CREATE ROLE willow WITH LOGIN SUPERUSER PASSWORD <password>
```

Minimum privileged users must have the following privileges:

- `REPLICATION`
- `LOGIN`
- database level `CREATE`
- table level `SELECT`

The following SQL commands can be run in the PostgreSQL terminal to create a minimum privileged user:

```sql
/* Create a willow user with the REPLICATION and LOGIN permissions */
CREATE ROLE willow REPLICATION LOGIN PASSWORD <password>;


/* Create a willow_replication group for sharing table level privileges with original table owners */
CREATE ROLE willow_replication;


/* Repeat the following command for all databases containing tables you wish to replicate */

/* Provide willow_replication with database level CREATE privileges */
GRANT CREATE ON <database_name> TO willow_replication;


/* Repeat the remaining commands for all tables you wish to replicate. */

/* Place the willow user and the original table owner in the willow_replication group */
GRANT willow_replication TO <original_table_owner>;
GRANT willow_replication TO willow;

/* Make the willow_replication group the table owner and grant SELECT permissions */
GRANT ALTER TABLE <table_name> OWNER TO willow_replication;
GRANT SELECT ON <table_name> TO willow_replication;
```

## Redis

Ensure your Redis cache has both the RedisJSON and RediSearch modules installed. [Redis Stack](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/) by default includes both of these modules and is a simple way to ensure your Redis cache is compatible with Willow.

# Installation & Usage

To install and start up Willow, clone this repo and run:
```
docker compose up
```

Navigate to Willow's home page at `https://localhost:3000`.

## Create a Pipeline

To initialize pipeline setup, click the "CREATE A CDC PIPELINE" button.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/case-study/3.2-1_home.png" alt="Willow's home page." width="80%"/>
</figure>

Enter PostgreSQL database details, including:
- The host and port to access the PostgreSQL server
- The database name containing the tables to replicate
- The username and password for Willow to use when replicating from PostgreSQL

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/case-study/3.2-2_source.png" alt="Willow's form for connecting to a source database." width="80%"/>
</figure>

Select which tables and columns to replicate and provide a unique source connection name. Primary keys (identified with PK) cannot be deselected. Tables without primary keys are not able to be replicated and are not shown.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/case-study/3.2-3_select_data.png" alt="Willow's form for selecting which data should be replicated from the source database." width="80%"/>
</figure>

Enter Redis cache details, including:
- The URL for the Redis cache. Must start with `redis://`.
- The username and password for Willow to use when replicating to Redis.

Click the "VERIFY" button to verify that the Redis cache is accessible.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/case-study/3.2-4_sink.png" alt="Willow's form for selecting which data should be replicated from the source database." width="80%" />
</figure>

Provide a unique sink connection name and click "SUBMIT" to finish CDC pipeline creation.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/case-study/3.2-5_sink_name.png" alt="Willow's form for selecting which data should be replicated from the source database." width="80%" />
</figure>

After pipeline creation, an initial snapshot of the selected tables is taken to prepopulate the cache. Subsequent `INSERT`, `UPDATE`, `DELETE`, and `TRUNCATE` commands that affect the selected tables will be reflected in the cache.

## View Existing Pipelines

All existing pipelines can be viewed at `https://localhost:3000/pipelines`.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/docs/all_pipelines.png" alt="Willow's page for displaying all running pipelines." width="80%" />
</figure>

A single pipeline can be clicked on to view details on the associated source and sink connectors.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/docs/single_pipeline.png" alt="Willow's page for displaying a single pipeline's details." width="80%" />
</figure>

## Teardown a Pipeline

Pipelines can be torn down by clicking the trash can icon.

<figure>
  <img src="https://github.com/willow-cdc/willow-cdc.github.io/blob/main/static/img/docs/delete_pipeline.png" alt="The trash can icon for a single pipeline is boxed in red on Willow's page showing all pipelines.." width="80%" />
</figure>
