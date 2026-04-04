# kcat UBI9-micro Image Verification

Image built from `kcat/Dockerfile.ubi9` on branch `micro-migration-ubi9`.

## Build

```bash
cd kafkacat-images

docker build \
  --build-arg UBI9_VERSION=latest \
  --build-arg UBI_MICRO_VERSION=latest \
  --build-arg DOCKER_UPSTREAM_REGISTRY="" \
  --build-arg DOCKER_UPSTREAM_TAG=ubi9-latest \
  -t kcat-micro-test \
  -f kcat/Dockerfile.ubi9 kcat/
```

## Runtime Dependency Verification

Ran `ldd` against the compiled kcat binary and cross-referenced each shared
library with what `ubi9-micro` ships. Libraries already in ubi9-micro are
marked "built-in"; all others must be installed in the runtime-deps stage.

| Shared Library           | RPM Package        | In ubi9-micro? |
|--------------------------|--------------------|----------------|
| `libm.so.6`             | glibc              | YES (built-in) |
| `libc.so.6`             | glibc              | YES (built-in) |
| `libresolv.so.2`        | glibc              | YES (built-in) |
| `ld-linux-aarch64.so.1` | glibc              | YES (built-in) |
| `libselinux.so.1`       | libselinux         | YES (built-in) |
| `libpcre2-8.so.0`       | pcre2              | YES (built-in) |
| `libz.so.1`             | zlib               | installed      |
| `libcrypto.so.3`        | openssl-libs       | installed      |
| `libssl.so.3`           | openssl-libs       | installed      |
| `libsasl2.so.3`         | cyrus-sasl-lib     | installed      |
| `libcrypt.so.2`         | libxcrypt          | installed      |
| `libgssapi_krb5.so.2`   | krb5-libs          | installed      |
| `libkrb5.so.3`          | krb5-libs          | installed      |
| `libk5crypto.so.3`      | krb5-libs          | installed      |
| `libkrb5support.so.0`   | krb5-libs          | installed      |
| `libcom_err.so.2`       | libcom_err         | installed      |
| `libcurl.so.4`          | libcurl-minimal    | installed      |
| `libkeyutils.so.1`      | keyutils-libs      | installed      |
| `libnghttp2.so.14`      | libnghttp2         | installed      |

Additionally `ca-certificates` is installed for TLS trust store (`/etc/pki/`).

## Test Setup

```bash
# Create a Docker network and start a Kafka broker
docker network create kcat-test-net

docker run -d --name kafka-kcat-test --network kcat-test-net \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka-kcat-test:9092 \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT \
  -e KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  confluentinc/cp-kafka:latest
```

> **Note:** kcat 1.7.0 (librdkafka 1.7.0) has a known compatibility issue with
> the latest Kafka's ListOffsets API. Consume tests use
> `-X broker.version.fallback=2.6.0` as a workaround. This is a pre-existing
> version mismatch, not related to this image change.

## Test Results (13/13 passed)

### Test 1: kcat -V (version check)
```bash
docker run --rm --network kcat-test-net kcat-micro-test -V
```
```
kcat - Apache Kafka producer and consumer tool
https://github.com/edenhill/kcat
Copyright (c) 2014-2021, Magnus Edenhill
Version 1.7.0 (JSON, Avro, Transactions, IncrementalAssign, JSONVerbatim,
  librdkafka 1.7.0 builtin.features=gzip,snappy,ssl,sasl,regex,lz4,
  sasl_gssapi,sasl_plain,sasl_scram,plugins,zstd,sasl_oauthbearer)
```
**[PASS]**

### Test 2: kafkacat symlink
```bash
docker run --rm --network kcat-test-net --entrypoint kafkacat kcat-micro-test -V
```
**[PASS]** — `kafkacat` symlink resolves to `kcat`

### Test 3: Metadata listing (-L)
```bash
docker run --rm --network kcat-test-net kcat-micro-test -L -b kafka-kcat-test:9092
```
```
Metadata for all topics (from broker 1: kafka-kcat-test:9092/1):
 1 brokers:
  broker 1 at kafka-kcat-test:9092 (controller)
```
**[PASS]**

### Test 4: Metadata listing as JSON (-L -J)
```bash
docker run --rm --network kcat-test-net kcat-micro-test -L -b kafka-kcat-test:9092 -J
```
```json
{"originating_broker":{"id":1,"name":"kafka-kcat-test:9092/1"},"query":{"topic":"*"},
 "controllerid":1,"brokers":[{"id":1,"name":"kafka-kcat-test:9092"}],"topics":[...]}
```
**[PASS]**

### Test 5: Produce messages (-P)
```bash
echo -e "msg-1\nmsg-2\nmsg-3\nmsg-4\nmsg-5" | \
  docker run --rm -i --network kcat-test-net kcat-micro-test \
  -P -b kafka-kcat-test:9092 -t test-topic
```
**[PASS]** — Produced 5 messages

### Test 6: Consume messages (-C)
```bash
docker run --rm --network kcat-test-net kcat-micro-test \
  -C -b kafka-kcat-test:9092 -t test-topic -o 0 -c 5 \
  -X broker.version.fallback=2.6.0
```
```
msg-1
msg-2
msg-3
msg-4
msg-5
```
**[PASS]** — Consumed 5 messages

### Test 7: Consume with format string
```bash
docker run --rm --network kcat-test-net kcat-micro-test \
  -C -b kafka-kcat-test:9092 -t test-topic -o 0 -c 5 \
  -X broker.version.fallback=2.6.0 \
  -f 'Value: %s | Partition: %p | Offset: %o\n'
```
```
Value: msg-1 | Partition: 0 | Offset: 0
Value: msg-2 | Partition: 0 | Offset: 1
Value: msg-3 | Partition: 0 | Offset: 2
Value: msg-4 | Partition: 0 | Offset: 3
Value: msg-5 | Partition: 0 | Offset: 4
```
**[PASS]**

### Test 8: Produce with keys (-K:)
```bash
echo -e "key1:value1\nkey2:value2\nkey3:value3" | \
  docker run --rm -i --network kcat-test-net kcat-micro-test \
  -P -b kafka-kcat-test:9092 -t keyed-topic -K:
```
**[PASS]** — Produced 3 keyed messages

### Test 9: Consume keyed messages
```bash
docker run --rm --network kcat-test-net kcat-micro-test \
  -C -b kafka-kcat-test:9092 -t keyed-topic -o 0 -c 3 \
  -X broker.version.fallback=2.6.0 -K\\t
```
```
key1	value1
key2	value2
key3	value3
```
**[PASS]**

### Test 10: Consume with full metadata format
```bash
docker run --rm --network kcat-test-net kcat-micro-test \
  -C -b kafka-kcat-test:9092 -t keyed-topic -o 0 -c 2 \
  -X broker.version.fallback=2.6.0 \
  -f '\nKey (%K bytes): %k\tValue (%S bytes): %s\nTimestamp: %T\tPartition: %p\tOffset: %o\n--\n'
```
```
Key (4 bytes): key1	Value (6 bytes): value1
Timestamp: 1771763265375	Partition: 0	Offset: 0
--
Key (4 bytes): key2	Value (6 bytes): value2
Timestamp: 1771763265375	Partition: 0	Offset: 1
--
```
**[PASS]**

### Test 11: Produce to specific partition (-p)
```bash
# Create 3-partition topic
docker exec kafka-kcat-test kafka-topics --bootstrap-server localhost:9092 \
  --create --topic partitioned-topic --partitions 3 --replication-factor 1

# Produce to partition 1
echo -e "p1-msg-1\np1-msg-2" | \
  docker run --rm -i --network kcat-test-net kcat-micro-test \
  -P -b kafka-kcat-test:9092 -t partitioned-topic -p 1

# Consume from partition 1
docker run --rm --network kcat-test-net kcat-micro-test \
  -C -b kafka-kcat-test:9092 -t partitioned-topic -p 1 -o 0 -c 2 \
  -X broker.version.fallback=2.6.0 \
  -f 'Value: %s | Partition: %p | Offset: %o\n'
```
```
Value: p1-msg-1 | Partition: 1 | Offset: 0
Value: p1-msg-2 | Partition: 1 | Offset: 1
```
**[PASS]**

### Test 12: Metadata for specific topic (-L -t)
```bash
docker run --rm --network kcat-test-net kcat-micro-test \
  -L -b kafka-kcat-test:9092 -t test-topic
```
```
Metadata for test-topic (from broker 1: kafka-kcat-test:9092/1):
 1 brokers:
  broker 1 at kafka-kcat-test:9092 (controller)
 1 topics:
  topic "test-topic" with 1 partitions:
    partition 0, leader 1, replicas: 1, isrs: 1
```
**[PASS]**

### Test 13: Non-root user verification
```bash
docker run --rm --entrypoint /bin/sh --network kcat-test-net kcat-micro-test -c 'id; whoami'
```
```
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
appuser
```
**[PASS]**

## Cleanup

```bash
docker rm -f kafka-kcat-test
docker network rm kcat-test-net
docker rmi kcat-micro-test kcat-build
```
