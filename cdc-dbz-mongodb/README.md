# MongoDB CDC (Debezium)

#### Getting the setup up and running
```bash
export DEBEZIUM_VERSION=1.4

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

```sql
CREATE SOURCE customers
FROM KAFKA BROKER 'kafka:9092' TOPIC 'dbserver1.inventory.customers'
FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://schema-registry:8081'
ENVELOPE DEBEZIUM;
```

`ERROR:  validating avro value schema: source schema is missing 'before' field`