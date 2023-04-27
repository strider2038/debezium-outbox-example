# debezium-outbox-example

## Скрипт запуска

```shell
# Start the topology as defined in https://debezium.io/documentation/reference/stable/tutorial.html
export DEBEZIUM_VERSION=2.1
docker compose up -d

# Start Postgres connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres.json

# Consume messages from a Debezium topic
docker compose exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic dbserver1.inventory.customers

# Modify records in the database via Postgres client
docker compose exec postgres env PGOPTIONS="--search_path=inventory" bash -c 'psql -U $POSTGRES_USER postgres'

# Shut down the cluster
docker compose down
```

## Настройки

```json5
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "postgres",
    "database.dbname": "postgres",
    "topic.prefix": "dbserver1", // префикс для топиков
    "schema.include.list": "inventory", // схема для отслеживания
    "table.include.list": "inventory.message", // таблицы для отслеживания
    "tombstones.on.delete": "false",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false", // отключение схем в ключах сообщений
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false", // отключение схем в теле сообщений
    "transforms": "outbox", // подключение преобразователя для outbox pattern'а
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.key": "id", // колонка, которая будет подставлена в качестве ключа Kafka-сообщения
    "transforms.outbox.table.field.event.timestamp": "created_at", // колонка, которая будет использована времени создания сообщения 
    "transforms.outbox.table.field.event.payload": "payload", // колонка с телом сообщения
    "transforms.outbox.table.expand.json.payload": true, // декодирования тела сообщения из строки
    "transforms.outbox.route.by.field": "topic", // колонка из которой будет извлечено название топика, будет подставлено как routedByValue ниже
    "transforms.outbox.route.topic.replacement" : "${routedByValue}.events" // шаблон для формирования названия топика
  }
}
```
