# Postgres CDC

#### Getting the setup up and running

```bash
export DEBEZIUM_VERSION=1.6

# Start the setup
docker-compose up -d

# Is everything really up and running?
docker-compose ps
```

There are two ways to connect Materialize to a Postgres database for CDC:

* [Direct Postgres Source](#direct-postgres-source)
* [Kafka + Debezium](#kafka--debezium)

## Direct Postgres Source

### Database configuration

Start the Postgres client, connecting to the sample `sportsdb` database:

```bash
docker-compose exec postgres bash -c 'psql sportsdb'
```

Logical replication is enabled on startup by setting `wal_level=logical`. To check:

```sql
SHOW wal_level;
```

#### Create a user

It's recommended to create a dedicated user for replication (that doesn't have _superuser_ privileges), so let's set up a `repl` user:

```sql
CREATE ROLE repl REPLICATION LOGIN PASSWORD 'postgres';
```

To avoid:

`ERROR:  Source error: materialize.public.mz_source: failed during initialization, must be dropped and recreated: db error: ERROR: permission denied for table display_names`

, the replication user also needs `SELECT` privileges on the table to copy the initial data:

```sql
GRANT SELECT ON display_names TO repl;
```

#### Set tables up for replication

```sql
ALTER TABLE display_names REPLICA IDENTITY FULL;
```

```sql
CREATE PUBLICATION mz_source FOR TABLE display_names;

-- Check that the publication has been created
SELECT * FROM pg_catalog.pg_publication; 
```

### Materialize

```bash
# Start the Materialize CLI
docker-compose run mzcli
```

#### Direct Postgres source

```sql
CREATE MATERIALIZED SOURCE "mz_source" FROM POSTGRES
CONNECTION 'host=postgres port=5432 user=repl dbname=sportsdb password=postgres'
PUBLICATION 'mz_source';

CREATE VIEWS FROM SOURCE mz_source (display_names);
```

We can check that a replication slot was created by running in Postgres:

```sql
SELECT * FROM pg_replication_slots;
```

## Kafka + Debezium

### Database configuration

Start the Postgres client, connecting to the sample `sportsdb` database:

```bash
docker-compose exec postgres bash -c 'psql sportsdb'
```

Logical replication is enabled on startup by setting `wal_level=logical`. To check:

```sql
SHOW wal_level;
```

#### Create a user

It's recommended to create a dedicated user for replication (that doesn't have _superuser_ privileges), so let's set up a `repl` user:

```sql
CREATE ROLE repl REPLICATION LOGIN PASSWORD 'postgres';
```

If we want to allow Debezium to auto-create publications, we also need to grant `CREATE ON DATABASE`:

```sql
GRANT CREATE ON DATABASE sportsdb TO repl;
```

to avoid:

`ERROR: permission denied for database sportsdb`

The `repl` user needs to have ownership of all tables to be added to the publication:

```sql
-- Create a replication group
CREATE ROLE repl_group;

--Add the original owner of the table to the group
GRANT repl_group TO postgres;

--Add the Debezium replication user to the group
GRANT repl_group TO repl;

--Transfer ownership of the table to repl_group
ALTER TABLE display_names OWNER TO repl_group;
```

#### Set tables up for replication

If the tables we want to replicate have **no primary key** defined (gulp!):

```sql
ALTER TABLE display_names REPLICA IDENTITY FULL;
```

Failing to do this will result in no records being sent to Materialize with a silent warning in the logs:

`WARN interchange::avro::decode: Record with unrecognized source coordinates in materialize.public.mv_display_names_primary_idx. You might be using an unsupported upstream database.`


The sample table has a unique index, but not a primary key. Let's create one to test if we can use `UPSERT` and go around `REPLICA IDENTITY FULL` (didn't try to set it to `REPLICA IDENTITY INDEX`. This might work as well!):

```sql
ALTER TABLE display_names ADD PRIMARY KEY (id);
```

### Debezium Postgres CDC connector

```bash
export CURRENT_HOST='<your-host>'

# Start the connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://$CURRENT_HOST:8083/connectors/ -d @register-postgres.json

# Check that the connector is running
curl http://$CURRENT_HOST:8083/connectors/sportsdb-connector/status # | jq

curl -X DELETE http://$CURRENT_HOST:8083/connectors/sportsdb-connector

# Did our Postgres records land in Kafka?
docker-compose exec schema-registry kafka-avro-console-consumer \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --topic pg_sportsdb.public.display_names
```

Some configuration properties to keep in mind:

1. "plugin.name":"pgoutput"

    Failing to include this will error out as:

    `ERROR: could not access file \"decoderbufs\": No such file or directory`

2. "table.include.list": "public.display_names"

3. "publication.autocreate.mode"="filtered"

## Materialize

```bash
# Start the Materialize CLI
docker-compose run mzcli
```

### Kafka+Avro (Debezium Envelope)

Using `UPSERT`:

```sql
CREATE SOURCE display_names
FROM KAFKA BROKER 'kafka:9092' TOPIC 'pg_sportsdb.public.display_names'
KEY FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://schema-registry:8081'
VALUE FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://schema-registry:8081'
ENVELOPE DEBEZIUM UPSERT;
```

Using the regular envelope with `REPLICA IDENTITY FULL` in the original table:

```sql
CREATE SOURCE display_names
FROM KAFKA BROKER 'kafka:9092' TOPIC 'pg_sportsdb.public.display_names'
FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://schema-registry:8081'
ENVELOPE DEBEZIUM;
```

And then creating a materialized view on top of this source:

```sql
CREATE MATERIALIZED VIEW mv_display_names AS 
SELECT * FROM display_names;
```

### Is it really working?

To check that the CDC functionality is _actually_ working with upsert (and without), let's go back to the Postgres database and perform a basic sequence of operations:

```sql
INSERT INTO display_names (language, entity_type, entity_id, full_name) 
SELECT 'en-US', 
       'affiliations', 
       MAX(entity_id)+1, 
       'American Football (the band)' 
FROM display_names 
WHERE entity_type = 'affiliations';

UPDATE display_names 
SET full_name = 'American Football (not the band)' 
WHERE id = (SELECT MAX(id) 
    FROM display_names);

DELETE FROM display_names 
WHERE id = (SELECT MAX(id) 
    FROM display_names);
```

, and then verify that these were reflected in Materialize:

```sql
materialize=> SELECT MAX(id) FROM mv_display_names;

 max
------
 3962

-- INSERT
materialize=> SELECT * FROM mv_display_names WHERE id = 3962;

  id  | language | entity_type  | entity_id |          full_name           | first_name | middle_name | last_name | alias | abbreviation | short_name | prefix | suffix
------+----------+--------------+-----------+------------------------------+------------+-------------+-----------+-------+--------------+------------+--------+--------
 3962 | en-US    | affiliations |        32 | American Football (the band) |            |             |           |       |              |            |        |

-- UPDATE
materialize=> SELECT * FROM mv_display_names WHERE id = 3962;

  id  | language | entity_type  | entity_id |            full_name             | first_name | middle_name | last_name | alias | abbreviation | short_name | prefix | suffix
------+----------+--------------+-----------+----------------------------------+------------+-------------+-----------+-------+--------------+------------+--------+--------
 3962 | en-US    | affiliations |        32 | American Football (not the band) |            |             |           |       |              |            |        |

-- DELETE
materialize=> SELECT * FROM mv_display_names WHERE id = 3962;

 id | language | entity_type | entity_id | full_name | first_name | middle_name | last_name | alias | abbreviation | short_name | prefix | suffix
----+----------+-------------+-----------+-----------+------------+-------------+-----------+-------+--------------+------------+--------+--------
(0 rows)
```

<hr>

If you run into issues with Postgres CDC in Materialize, please [open a GitHub issue](https://github.com/MaterializeInc/materialize/issues/new/choose) so we can look into it!
