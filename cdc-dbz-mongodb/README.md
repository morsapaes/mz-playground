# MongoDB CDC (Debezium)

#### Getting the setup up and running
```bash
export DEBEZIUM_VERSION=1.4

# Start the setup
docker-compose up -d

# Is everything really up and running?
docker-compose ps
```

## MongoDB

```bash
# Initialize a MongoDB replica set
docker-compose exec mongodb bash -c '/usr/local/bin/init-inventory.sh'

# Check the sample data
docker-compose exec mongodb bash -c 'mongo -u $MONGODB_USER -p $MONGODB_PASSWORD --authenticationDatabase admin inventory --eval "db.customers.find()"'
```

## Kafka + Kafka Connect

### Debezium MongoDB CDC connector

```bash
export CURRENT_HOST='<your-host>'

# Start the connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://$CURRENT_HOST:8083/connectors/ -d @register-mongodb.json

# Check that the connector is running
curl http://$CURRENT_HOST:8083/connectors/inventory-connector/status # | jq

# Did our MongoDB records land in Kafka?
docker-compose exec schema-registry kafka-avro-console-consumer \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --topic dbserver1.inventory.customers
```

## Materialize

```bash
# Start the Materialize CLI
docker-compose run mzcli
```

### Kafka+Avro (Debezium Envelope)

Starting from the assumption that the Debezium MongoDB connector will emit an envelope with a similar structure to that of e.g. Postgres, let's try to create a source consuming Avro-formatted events from Kafka using the [Debezium envelope](https://materialize.com/docs/sql/create-source/avro-kafka/#debezium-envelope-details):

```sql
CREATE SOURCE customers
FROM KAFKA BROKER 'kafka:9092' TOPIC 'dbserver1.inventory.customers'
FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://schema-registry:8081'
ENVELOPE DEBEZIUM;
```

This leads to:

`ERROR:  validating avro value schema: source schema is missing 'before' field`

Looking into the [Debezium docs](https://debezium.io/documentation/reference/connectors/mongodb.html):

> In MongoDB’s oplog, update events do not contain the before or after states of the changed document. Consequently, it is not possible for a Debezium connector to provide this information. (…) Downstream consumers of the stream can reconstruct document state by keeping the latest state for each document and comparing the state in a new event with the saved state.

Because Debezium uses a different format for MongoDB's oplog (that omits the `before` field), it's not possible to consume it with Materialize out-of-the-box. 

<hr>

If you're interested in MongoDB CDC support, please leave a comment with your use case in this GitHub issue: https://github.com/MaterializeInc/materialize/issues/7289
