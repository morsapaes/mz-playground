# MySQL CDC (Debezium)

#### Getting the setup up and running
```bash
export DEBEZIUM_VERSION=1.6

# Start the setup
docker-compose up -d

# Is everything really up and running?
docker-compose ps
```

## Database Configuration

```bash
docker-compose exec mysql bash -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" employees'
```

Row-based binary replication is enabled on startup by loading an alternative configuration file (`mysql.cnf`) into the MySQL container. To check:

```sql
SHOW VARIABLES
WHERE variable_name IN ('log_bin', 'binlog_format');
```

### Creating a user

It's recommended to create a dedicated user for replication (that doesn't have _superuser_ privileges), so let's set up a `repl` user:

```sql
CREATE USER 'repl' IDENTIFIED BY 'mysqlpwd';

GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl';

FLUSH PRIVILEGES;
```

You can check more information about each permission setting in the [Debezium documentation](https://debezium.io/documentation/reference/1.6/connectors/mysql.html#mysql-creating-user).

## Kafka + Kafka Connect

### Debezium MySQL CDC connector

```bash
export CURRENT_HOST='<your-host>'

# Start the connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://$CURRENT_HOST:8083/connectors/ -d @register-mysql.json


curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://$CURRENT_HOST:8083/connector-plugins/ -d @register-mysql.json /connector-plugins/{connectorType}/config/validate


curl -X DELETE http://$CURRENT_HOST:8083/connectors/employees-connector

# Check that the connector is running
curl http://$CURRENT_HOST:8083/connectors/employees-connector/status # | jq

# Did our MySQL records land in Kafka?
docker-compose exec schema-registry kafka-avro-console-consumer \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --topic dbserver1.employees.salaries
```

## Materialize

```bash
# Start the Materialize CLI
docker-compose run mzcli
```

### Kafka+Avro (Debezium Envelope)

Let's try to create a source consuming Avro-formatted events from Kafka using the [Debezium envelope](https://materialize.com/docs/sql/create-source/avro-kafka/#debezium-envelope-details):

```sql
CREATE SOURCE salaries
FROM KAFKA BROKER 'kafka:9092' TOPIC 'dbserver1.employees.salaries'
FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://schema-registry:8081'
ENVELOPE DEBEZIUM;
```

And then creating a materialized view on top of this source:

```sql
CREATE MATERIALIZED VIEW mv_salaries AS 
SELECT * FROM salaries;
```

### Is it really working?

To check that the CDC functionality is _actually_ working with upsert (and without), let's go back to the MySQL database and perform a basic sequence of operations:

```sql
INSERT INTO salaries
SELECT 35523, 50000, '2021-10-01','9999-01-01';

UPDATE salaries
SET to_date = '2021-10-01'
WHERE emp_no = 35523 AND from_date = '2001-01-21';

DELETE FROM salaries 
WHERE emp_no = 35523 AND from_date = '1997-01-22';
```
, and then verify that these were reflected in Materialize:

```sql
materialize=> SELECT * FROM mv_salaries WHERE emp_no = '35523' ORDER BY to_date ASC;

 emp_no | salary | from_date  |  to_date
--------+--------+------------+------------
  35523 |  40000 | 1997-01-22 | 1998-01-22
  35523 |  43078 | 1998-01-22 | 1999-01-22
  35523 |  42977 | 1999-01-22 | 2000-01-22
  35523 |  44091 | 2000-01-22 | 2001-01-21
  35523 |  46693 | 2001-01-21 | 2001-10-11
(5 rows)

materialize=> SELECT * FROM mv_salaries WHERE emp_no = '35523' ORDER BY to_date ASC;

 emp_no | salary | from_date  |  to_date
--------+--------+------------+------------
  35523 |  40000 | 1997-01-22 | 1998-01-22
  35523 |  43078 | 1998-01-22 | 1999-01-22
  35523 |  42977 | 1999-01-22 | 2000-01-22
  35523 |  44091 | 2000-01-22 | 2001-01-21
  35523 |  46693 | 2001-01-21 | 2001-10-11
  35523 |  50000 | 2021-10-01 | 9999-01-01
(6 rows)

materialize=> SELECT * FROM mv_salaries WHERE emp_no = '35523' ORDER BY to_date ASC;

 emp_no | salary | from_date  |  to_date
--------+--------+------------+------------
  35523 |  40000 | 1997-01-22 | 1998-01-22
  35523 |  43078 | 1998-01-22 | 1999-01-22
  35523 |  42977 | 1999-01-22 | 2000-01-22
  35523 |  44091 | 2000-01-22 | 2001-01-21
  35523 |  46693 | 2001-01-21 | 2021-10-01
  35523 |  50000 | 2021-10-01 | 9999-01-01
(6 rows)

materialize=> SELECT * FROM mv_salaries WHERE emp_no = '35523' ORDER BY to_date ASC;

 emp_no | salary | from_date  |  to_date
--------+--------+------------+------------
  35523 |  43078 | 1998-01-22 | 1999-01-22
  35523 |  42977 | 1999-01-22 | 2000-01-22
  35523 |  44091 | 2000-01-22 | 2001-01-21
  35523 |  46693 | 2001-01-21 | 2021-10-01
  35523 |  50000 | 2021-10-01 | 9999-01-01
(5 rows)
```

<hr>

If you run into issues with MySQL CDC in Materialize, please [open a GitHub issue](https://github.com/MaterializeInc/materialize/issues/new/choose) so we can look into it!
