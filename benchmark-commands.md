# Benchmark commands

## Producer

Setup

```sh
bin/kafka-topics.sh \
  --zookeeper zookeeper.example.com:2181 \
  --create \
  --topic test-rep-one \
  --partitions 6 \
  --replication-factor 1
bin/kafka-topics.sh \
  --zookeeper zookeeper.example.com:2181 \
  --create \
  --topic test \
  --partitions 6 --replication-factor 3
```

Single thread, no replication

```sh
bin/kafka-producer-perf-test.sh \
  --topic test \
  --num-records 50000000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props acks=1 \
  bootstrap.servers=kafka.example.com:9092 \
  buffer.memory=67108864 \
  batch.size=8196
```

Single-thread, async 3x replication

```sh
bin/kafk-topics.sh \
  --zookeeper zookeeper.example.com:2181 \
  --create \
  --topic test \
  --partitions 6 \
  --replication-factor 3
bin/kafka-producer-perf-test.sh \
  --topic test \
  --num-records 50000000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props acks=1 \
  bootstrap.servers=kafka.example.com:9092 \
  buffer.memory=67108864 \
  batch.size=8196
```

Single-thread, sync 3x replication

```sh
bin/kafka-producer-perf-test.sh \
  --topic test \
  --num-records 50000000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props acks=1 \
  bootstrap.servers=kafka.example.com:9092 \
  buffer.memory=67108864 batch.size=64000
```

Three Producers, 3x async replication

```sh
bin/kafka-producer-perf-test.sh \
  --topic test \
  --num-records 50000000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props acks=1 \
  bootstrap.servers=kafka.example.com:9092 \
  buffer.memory=67108864 \
  batch.size=8196
```

Throughput Versus Stored Data

```sh
bin/kafka-producer-perf-test.sh \
  --topic test \
  --num-records 50000000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props acks=1 \
  bootstrap.servers=kafka.example.com:9092 \
  buffer.memory=67108864 batch.size=8196
```

Effect of message size

```sh
for i in 10 100 1000 10000 100000; do
  echo ""
  echo $i
  bin/kafka-producer-perf-test.sh \
    --topic test \
    --num-records $((1000*1024*1024/$i))\
    --record-size $i\
    --throughput -1 \
    --producer-props acks=1 \
    bootstrap.servers=kafka.example.com:9092 \
    buffer.memory=67108864 \
    batch.size=128000
done;
```

## Consumer

Consumer throughput

```sh
bin/kafka-consumer-perf-test.sh \
  --zookeeper zookeeper.example.com:2181 \
  --messages 50000000 \
  --topic test \
  --threads 1
```

3 Consumers

On three servers, run:

```sh
bin/kafka-consumer-perf-test.sh \
  --zookeeper zookeeper.example.com:2181 \
  --messages 50000000 \
  --topic test \
  --threads 1
```

## End-to-end Latency

```
bin/kafka-run-class.sh \
  kafka.tools.TestEndToEndLatency \
  kafka.example.com:9092 \
  zookeeper.example.com:2181 \
  test 5000
```

## Producer and consumer

```sh
bin/kafka-run-class.sh \
  org.apache.kafka.tools.ProducerPerformance \
bin/kafka-producer-perf-test.sh \
  --topic test \
  --num-records 50000000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props acks=1 \
  bootstrap.servers=kafka.example.com:9092 \
  buffer.memory=67108864 \
  batch.size=8196
```

```sh
bin/kafka-consumer-perf-test.sh \
  --zookeeper zookeeper.example.com:2181 \
  --messages 50000000 \
  --topic test \
  --threads 1
```
