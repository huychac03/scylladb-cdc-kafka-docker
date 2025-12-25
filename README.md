# ScyllaDB CDC → Kafka for `bino_post.posts`

This environment provisions ScyllaDB, Kafka, ZooKeeper, and Kafka Connect via Docker Compose, enables CDC on the `bino_post.posts` table, and streams row-level change events into Kafka using the Scylla CDC Source Connector (v1.2.4). Test inserts and updates are captured on the Kafka topic `posts-cdc.bino_post.posts`.

---

## Quick Start

1. **Boot the stack**
   ```bash
   docker compose up -d
   docker compose ps
   ```
2. **Install / update the CDC connector plugin**
   ```bash
   mkdir -p plugins
   curl -sSL https://repo1.maven.org/maven2/com/scylladb/scylla-cdc-source-connector/1.2.7/scylla-cdc-source-connector-1.2.7-jar-with-dependencies.jar \
     -o plugins/scylla-cdc-source-connector.jar
   docker compose restart connect && sleep 30
   curl -s http://localhost:8083/connector-plugins | grep -i scylla
   ```
3. **Create keyspace/table with CDC already enabled**
   ```bash
   docker exec -i scylla-node1 cqlsh -e "
     CREATE KEYSPACE IF NOT EXISTS bino_post
     WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
     USE bino_post;
     CREATE TABLE IF NOT EXISTS posts (
       id BIGINT PRIMARY KEY,
       user_id BIGINT,
       is_anonymous BOOLEAN,
       created_at TIMESTAMP,
       updated_at TIMESTAMP,
       title TEXT,
       description TEXT,
       status TEXT,
       category TEXT,
       video_urls LIST<TEXT>,
       audio_urls LIST<TEXT>,
       image_urls LIST<TEXT>,
       location_lat DOUBLE,
       location_long DOUBLE,
       location_address TEXT,
       is_deleted BOOLEAN
     ) WITH cdc = {'enabled': true};
   "
   docker exec -i scylla-node1 cqlsh -e "DESCRIBE TABLE bino_post.posts;"
   ```
4. **Deploy the Scylla CDC connector**

   ```bash
   curl -s -X POST -H "Content-Type: application/json" \
     --data @connector-config.json \
     http://localhost:8083/connectors

   curl -s http://localhost:8083/connectors/scylla-cdc-connector/status | jq .
   ```

5. **Smoke-test CDC**
   ```bash
   docker exec -i scylla-node1 cqlsh -e "
     INSERT INTO bino_post.posts (
       id, user_id, is_anonymous, created_at, updated_at,
       title, description, status, category, is_deleted
     ) VALUES (
       1, 100, false, toTimestamp(now()), toTimestamp(now()),
       'My First Post', 'This is a test post', 'published', 'general', false
     );
     UPDATE bino_post.posts
     SET title = 'Updated Title', updated_at = toTimestamp(now())
     WHERE id = 1;
   "
   docker exec -i kafka kafka-topics --bootstrap-server localhost:9092 --list
   docker exec -i kafka kafka-console-consumer \
     --bootstrap-server localhost:9092 \
     --topic posts-cdc.bino_post.posts \
     --from-beginning --timeout-ms 1000
   ```

---

## Kafka Topic Naming

- Topics follow `posts-cdc.<keyspace>.<table>`.
- `topic.prefix` in `connector-config.json` is `posts-cdc`, so adding more CDC tables only requires appending additional entries to `scylla.table.names`.
- Heartbeat topic: `__debezium-heartbeat.posts-cdc`.

---

## Consuming CDC Events

- **Ad-hoc debugging**
  ```bash
  docker exec -i kafka kafka-console-consumer \
    --bootstrap-server localhost:9092 \
    --topic posts-cdc.bino_post.posts \
    --from-beginning
  ```
- **Client applications** should deserialize JSON payloads. Each message contains the latest row image per change as produced by the Debezium `ExtractNewRecordState` SMT (tombstones are retained).

---

## Troubleshooting

- **Plugin missing from `/connector-plugins`**
  - Ensure the jar exists inside the container: `docker exec kafka-connect ls /usr/share/java/kafka-connect-scylla-cdc`.
  - Restart Kafka Connect after copying the file.
- **Connector fails validation**
  - Confirm `scylla.cluster.ip.addresses`, `topic.prefix`, and `scylla.table.names` are set in `connector-config.json`.
  - Use `curl -s http://localhost:8083/connector-plugins/com.scylladb.cdc.debezium.connector.ScyllaConnector/config`.
- **No events emitted**
  - Run `DESCRIBE TABLE bino_post.posts;` to confirm `cdc = {'enabled': 'true', ...}`.
  - Check connector status: `curl -s http://localhost:8083/connectors/scylla-cdc-connector/status | jq .`.
  - Insert new rows _after_ the connector is running—the current config starts streaming from the present.
- **Kafka consumer timeout errors**
  - They indicate no more messages; re-run without `--timeout-ms` or insert more rows.
- **Port collisions**
  - Adjust `ports` in `docker-compose.yml` (e.g., map `9043:9042`) and update client commands accordingly.

---

## Running Services & Ports

| Service       | Container       | Ports (host → container)       |
| ------------- | --------------- | ------------------------------ |
| ScyllaDB      | `scylla-node1`  | `9042 → 9042`                  |
| ZooKeeper     | `zookeeper`     | `2181 → 2181`                  |
| Kafka Broker  | `kafka`         | `9092 → 9092`, `29092 → 29092` |
| Kafka Connect | `kafka-connect` | `8083 → 8083`                  |

Check status anytime via `docker compose ps`.

---

## Verification Checklist

- `docker compose ps` shows all containers `Up (healthy)`.
- `curl -s http://localhost:8083/connector-plugins | grep -i scylla` contains `com.scylladb.cdc.debezium.connector.ScyllaConnector`.
- `DESCRIBE TABLE bino_post.posts;` shows `cdc = {'enabled': 'true', ...}`.
- `curl -s http://localhost:8083/connectors/scylla-cdc-connector/status | jq .` reports `RUNNING`.
- `docker exec -i kafka kafka-topics --bootstrap-server localhost:9092 --list` includes `posts-cdc.bino_post.posts`.
- Sample inserts/updates emit events visible via `kafka-console-consumer`.

You now have a reproducible CDC pipeline from ScyllaDB to Kafka for the `posts` table, along with verification and operational guidance for future collaborators.
# scylladb-cdc-kafka-docker
