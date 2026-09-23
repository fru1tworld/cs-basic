# Kafka

Kafka의 exactly-once 처리를 이해하려면 소비 오프셋과 출력 레코드가 어떤 트랜잭션으로 묶이는지 살펴봐야 한다. 이 보장은 둘을 같은 Kafka 트랜잭션으로 관리하고 커밋된 결과를 읽는 범위를 전제로 한다. 외부 데이터베이스 저장이나 API 호출까지 포함하려면 해당 시스템과도 협력해야 한다. [Kafka 전달 의미론](https://kafka.apache.org/41/design/design/#message-delivery-semantics)

## 아키텍처

### Kafka 핵심 구성요소

- Kafka Cluster
- Broker 1, Broker 2, Broker 3
- Topic A, Topic A, Topic A
- P0(L), P0(F), P1(L)
- P1(F), P1(F), P0(F)
- Topic B, Topic B, Topic B
- P0(F), P0(L), P0(F)
- KRaft Controller Quorum
- (Kafka 4.0+ : ZooKeeper 완전 제거)
- Producer, Consumer
- Group
- L = Leader, F = Follower

### Broker

- Broker는 Kafka 클러스터의 개별 서버로, 메시지를 저장하고 클라이언트 요청을 처리함.

```yaml
# server.properties (KRaft 모드)
# Kafka 4.0+에서는 ZooKeeper가 완전히 제거됨

# 브로커 ID
node.id=1

# KRaft 역할: broker, controller, 또는 둘 다
process.roles=broker,controller

# Controller Quorum 설정
controller.quorum.voters=1@localhost:9093,2@localhost:9094,3@localhost:9095

# 리스너 설정
listeners=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER

# 로그 저장 경로
log.dirs=/var/kafka-logs

# 기본 파티션 수 및 복제 팩터
num.partitions=3
default.replication.factor=3

# 로그 보존 정책
log.retention.hours=168  # 7일
log.retention.bytes=-1   # 무제한
log.segment.bytes=1073741824  # 1GB
```

### Topic과 Partition

Topic은 메시지를 카테고리나 피드처럼 묶는 논리적 단위다. 하나의 Topic을 여러 Partition으로 나누면 데이터를 분산해 저장하고 병렬로 처리할 수 있다.

아래 예제는 `orders` Topic을 3개 Partition으로 나누고 복제 팩터를 3으로 설정한다. 세 Broker에 리더를 하나씩 배치한 경우, 각 Partition의 메시지와 복제본은 다음처럼 구성할 수 있다.

| Partition | Offset 0 | Offset 1 | Leader | Followers |
| --- | --- | --- | --- | --- |
| 0 | Msg A | Msg D | Broker 1 | Broker 2, 3 |
| 1 | Msg B | Msg E | Broker 2 | Broker 1, 3 |
| 2 | Msg C | Msg F | Broker 3 | Broker 1, 2 |

```java
// Topic 생성 (AdminClient API)
import org.apache.kafka.clients.admin.*;

Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

try (AdminClient admin = AdminClient.create(props)) {
    NewTopic newTopic = new NewTopic("orders", 3, (short) 3)
        .configs(Map.of(
            "cleanup.policy", "delete",
            "retention.ms", "604800000",  // 7일
            "min.insync.replicas", "2"
        ));

    admin.createTopics(List.of(newTopic)).all().get();
}
```

### Segment

Partition의 로그는 여러 Segment로 나뉜다. 각 Segment에는 메시지를 저장하는 로그 파일(`.log`), offset으로 위치를 찾는 인덱스(`.index`), 타임스탬프로 offset을 찾는 인덱스(`.timeindex`)가 있다. 다음은 offset 0과 156789에서 시작하는 두 Segment의 파일 구성이다.

- Partition 0 디렉터리 구조:
- 00000000000000000000.log, # 첫 번째 세그먼트 (offset 0부터)
- 00000000000000000000.index, # offset → position 매핑
- 00000000000000000000.timeindex, # timestamp → offset 매핑
- 00000000000000156789.log, # 두 번째 세그먼트 (offset 156789부터)
- 00000000000000156789.index
- 00000000000000156789.timeindex
- leader-epoch-checkpoint, # 리더 에포크 정보

- Segment 롤오버 조건:
- `log.segment.bytes` 크기 초과 (기본 1GB)
- `log.roll.ms` 또는 `log.roll.hours` 시간 경과
- 인덱스/타임인덱스 파일 크기 제한 초과

### Offset

- Offset은 Partition 내 각 메시지의 고유 식별자임. Consumer가 읽은 위치를 추적하는 데 사용됨.

- Partition 0:
- Offset: 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11
- [A], [B], [C], [D], [E], [F], [G], [H], [I], [J], [K], [L]
- Consumer Position, High Watermark
- (Committed: 3), (Committed to ISR)
- Log Start Offset: 0 (가장 오래된 메시지)
- Log End Offset: 12 (다음 메시지가 쓰일 위치)
- High Watermark: 10 (Consumer가 읽을 수 있는 최대 위치)

## Producer와 Consumer

### Producer 기본 설정

```java
import org.apache.kafka.clients.producer.*;

Properties props = new Properties();

// 필수 설정
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// 신뢰성 설정 (Kafka 3.0+ 기본값)
props.put("acks", "all");                    // 모든 ISR 복제 확인
props.put("enable.idempotence", "true");     // 중복 방지
props.put("max.in.flight.requests.per.connection", "5");  // 순서 보장

// 성능 설정
props.put("batch.size", "16384");            // 배치 크기 (16KB)
props.put("linger.ms", "5");                 // 배치 대기 시간
props.put("buffer.memory", "33554432");      // 버퍼 메모리 (32MB)
props.put("compression.type", "lz4");        // 압축 방식

// 재시도 설정
props.put("retries", "2147483647");          // 무한 재시도 (기본값)
props.put("retry.backoff.ms", "100");        // 재시도 간격
props.put("delivery.timeout.ms", "120000");  // 최대 전송 시간 (2분)

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

### Producer 메시지 전송

```java
// 동기 전송 (블로킹)
try {
    RecordMetadata metadata = producer.send(
        new ProducerRecord<>("orders", "key1", "order data")
    ).get();  // 블로킹

    System.out.printf("Sent to partition %d, offset %d%n",
        metadata.partition(), metadata.offset());
} catch (Exception e) {
    e.printStackTrace();
}

// 비동기 전송 (논블로킹, 권장)
producer.send(
    new ProducerRecord<>("orders", "key2", "order data"),
    (metadata, exception) -> {
        if (exception != null) {
            // 전송 실패 처리
            exception.printStackTrace();
        } else {
            System.out.printf("Sent to partition %d, offset %d%n",
                metadata.partition(), metadata.offset());
        }
    }
);

// 배치 전송 후 플러시
for (int i = 0; i < 100; i++) {
    producer.send(new ProducerRecord<>("orders", "key" + i, "data" + i));
}
producer.flush();  // 버퍼의 모든 메시지 전송 완료 대기
```

### Partitioner

- 메시지가 어느 Partition으로 전송될지 결정하는 컴포넌트

```java
// 기본 파티셔닝 전략
// 1. Key가 있는 경우: murmur2(key) % numPartitions
// 2. Key가 없는 경우: Sticky Partitioner (배치 단위로 같은 파티션)

// 커스텀 파티셔너
public class RegionPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {

        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key == null) {
            // Sticky Partitioner 동작
            return RecordMetadata.UNKNOWN_PARTITION;
        }

        String keyStr = (String) key;

        // 지역 기반 파티셔닝
        if (keyStr.startsWith("KR-")) {
            return 0;  // 한국 데이터는 파티션 0
        } else if (keyStr.startsWith("US-")) {
            return 1;  // 미국 데이터는 파티션 1
        }

        // 기본: 해시 기반
        return Math.abs(keyStr.hashCode()) % numPartitions;
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}

// 사용
props.put("partitioner.class", "com.example.RegionPartitioner");
```

### Consumer 기본 설정

```java
import org.apache.kafka.clients.consumer.*;

Properties props = new Properties();

// 필수 설정
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processing-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

// Offset 설정
props.put("enable.auto.commit", "false");    // 수동 커밋 (권장)
props.put("auto.offset.reset", "earliest");  // earliest | latest | none

// 성능 설정
props.put("fetch.min.bytes", "1");           // 최소 페치 바이트
props.put("fetch.max.wait.ms", "500");       // 최대 대기 시간
props.put("max.poll.records", "500");        // poll당 최대 레코드 수
props.put("max.partition.fetch.bytes", "1048576");  // 파티션당 최대 바이트

// 세션 관리
props.put("session.timeout.ms", "45000");    // 세션 타임아웃
props.put("heartbeat.interval.ms", "3000");  // 하트비트 간격
props.put("max.poll.interval.ms", "300000"); // 최대 poll 간격

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
```

### Consumer 메시지 소비

```java
// Topic 구독
consumer.subscribe(Arrays.asList("orders", "payments"));

// 또는 패턴으로 구독
consumer.subscribe(Pattern.compile("order-.*"));

// 메시지 소비 루프
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("Topic: %s, Partition: %d, Offset: %d, Key: %s, Value: %s%n",
                record.topic(), record.partition(), record.offset(),
                record.key(), record.value());

            // 비즈니스 로직 처리
            processRecord(record);
        }

        // 수동 커밋 (처리 완료 후)
        consumer.commitSync();  // 또는 commitAsync()
    }
} finally {
    consumer.close();
}
```

### Offset Commit 전략

```java
// 1. 자동 커밋 (기본, 권장하지 않음)
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000");

// 2. 동기 커밋 (안전하지만 느림)
consumer.commitSync();

// 3. 비동기 커밋 (빠르지만 재시도 없음)
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed for offsets: {}", offsets, exception);
    }
});

// 4. 파티션별 정밀 커밋 (권장)
Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

for (ConsumerRecord<String, String> record : records) {
    processRecord(record);

    currentOffsets.put(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1)  // 다음 offset
    );

    // 매 1000건마다 커밋
    if (count % 1000 == 0) {
        consumer.commitAsync(currentOffsets, null);
    }
}

// 마지막에 동기 커밋
consumer.commitSync(currentOffsets);
```

## Consumer Group과 Rebalancing

### Consumer Group 개념

Consumer Group은 Topic의 Partition을 나누어 처리한다. 예를 들어 Partition이 6개인 `orders`를 `order-processing` 그룹의 Consumer 3개가 소비한다면 각 Consumer가 2개씩 맡을 수 있다.

하나의 Partition은 같은 그룹 안에서 하나의 Consumer에만 할당된다. 따라서 Consumer가 Partition보다 많으면 일부는 유휴 상태가 되고, 적으면 일부 Consumer가 여러 Partition을 맡는다.

### Rebalancing 트리거

- Rebalancing 발생 조건:
- Consumer 그룹에 새 Consumer 참여
- Consumer가 그룹 탈퇴 (정상 종료 또는 장애)
- Topic에 새 파티션 추가
- Consumer가 max.poll.interval.ms 내에 poll() 미호출
- session.timeout.ms 내에 heartbeat 미전송

### Partition Assignment 전략

```java
// Kafka 3.0+ 기본값: CooperativeStickyAssignor
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");

// 여러 전략 동시 설정 (점진적 마이그레이션용)
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor," +
    "org.apache.kafka.clients.consumer.RangeAssignor");
```

- 전략: RangeAssignor:
  - 설명: 토픽별로 연속된 파티션 할당
  - 장점: 예측 가능한 할당
  - 단점: 토픽이 많을 때 불균형
- 전략: RoundRobinAssignor:
  - 설명: 전체 파티션을 순환 할당
  - 장점: 균등 분배
  - 단점: Eager 프로토콜 사용
- 전략: StickyAssignor:
  - 설명: 기존 할당 유지 + 균형 재조정
  - 장점: 재할당 최소화
  - 단점: Eager 프로토콜 사용
- 전략: CooperativeStickyAssignor:
  - 설명: Sticky + Incremental Rebalance
  - 장점: 중단 시간 최소화
  - 단점: Kafka 2.4+ 필요

### Eager vs Cooperative Rebalancing

- Eager Protocol (기존)
- 1단계: 모든 Consumer 정지 (Stop-the-World)
- Consumer 1: P0, P1 → 반납
- Consumer 2: P2, P3 → 반납
- Consumer 3: P4, P5 → 반납
- 2단계: 전체 재할당
- Consumer 1: P0, P2 할당
- Consumer 2: P1, P3 할당
- Consumer 3: P4, P5 할당
- 문제: 전체 Consumer가 일시적으로 메시지 처리 중단
- Cooperative Protocol (Incremental)
- 1단계: 필요한 파티션만 반납
- Consumer 1: P1 반납 (P0 유지하며 계속 처리)
- Consumer 2: (변경 없음, 계속 처리)
- Consumer 3: (변경 없음, 계속 처리)
- 2단계: 반납된 파티션만 재할당
- Consumer 4 (신규): P1 할당
- 장점: 대부분의 Consumer가 계속 처리 가능

### KIP-848: 차세대 Consumer Rebalance Protocol

- Kafka 4.0부터 서버 사이드 리밸런싱이 도입됨.

```java
// Kafka 4.0+ 설정
// 서버에서 group.consumer.assignors 설정
// uniform (기본) 또는 range

// 클라이언트는 별도 설정 불필요
// 자동으로 ConsumerGroupHeartbeat API 사용
```

- KIP-848 핵심 변경사항:
- 리밸런싱 로직이 클라이언트에서 서버(Group Coordinator)로 이동
- 전역 동기화 장벽 제거 - 진정한 증분 리밸런싱
- `ConsumerGroupHeartbeat` API로 멤버십과 할당 통합 관리
- 할당이 변경되지 않는 Consumer는 리밸런싱 영향 없음

### Rebalancing 최적화

```java
// 1. Static Membership 사용 (재시작 시 리밸런싱 방지)
props.put("group.instance.id", "consumer-host-1");

// 2. 적절한 타임아웃 설정
props.put("session.timeout.ms", "45000");
props.put("heartbeat.interval.ms", "3000");      // session.timeout의 1/3 이하
props.put("max.poll.interval.ms", "300000");     // 처리 시간에 맞게 조정

// 3. max.poll.records 조정
props.put("max.poll.records", "100");  // 처리량에 맞게 조정

// 4. 처리 시간이 긴 경우 별도 스레드에서 heartbeat
// poll() 루프에서 처리가 오래 걸리면 별도 스레드로 처리 분리
```

## 정확히 한 번 전송 (Exactly-Once Semantics)

### 메시지 전달 보장 수준

- 보장 수준: At-most-once:
  - 설명: 최대 한 번 전달
  - 메시지 손실: 가능
  - 중복: 없음
- 보장 수준: At-least-once:
  - 설명: 최소 한 번 전달
  - 메시지 손실: 없음
  - 중복: 가능
- 보장 수준: Exactly-once:
  - 설명: 정확히 한 번 전달
  - 메시지 손실: 없음
  - 중복: 없음

### Idempotent Producer

- 네트워크 장애로 인한 재시도 시 중복 메시지를 방지함.

```java
// Kafka 3.0+에서는 기본 활성화
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", StringSerializer.class.getName());
props.put("value.serializer", StringSerializer.class.getName());

// Idempotent Producer 설정 (Kafka 3.0+ 기본값)
props.put("enable.idempotence", "true");  // 기본값: true
props.put("acks", "all");                  // 기본값: all
props.put("max.in.flight.requests.per.connection", "5");  // 최대 5

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

- 동작 원리:
- Producer, Broker
- PID: 123, Batch (seq=0), →
- seq=0 저장
- ←, ACK
- Batch (seq=1), →
- (네트워크 오류), seq=1 저장
- Batch (seq=1) 재시도, →
- 중복 감지
- ←, ACK (중복이지만 성공), seq=1 이미
- 있음
- PID (Producer ID): 각 Producer 인스턴스의 고유 ID
- Sequence Number: 파티션별 메시지 순서 번호
- Broker는 (PID, Partition, SeqNum)으로 중복 감지

### Transactional Producer

- 여러 파티션에 대한 원자적 쓰기와 Consumer offset 커밋을 보장함.

```java
// Transactional Producer 설정
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", StringSerializer.class.getName());
props.put("value.serializer", StringSerializer.class.getName());
props.put("transactional.id", "order-service-tx-1");  // 필수: 고유 트랜잭션 ID
// enable.idempotence는 transactional.id 설정 시 자동 활성화

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// 트랜잭션 초기화 (애플리케이션 시작 시 한 번)
producer.initTransactions();

try {
    // 트랜잭션 시작
    producer.beginTransaction();

    // 여러 토픽/파티션에 메시지 발행
    producer.send(new ProducerRecord<>("orders", "order-1", "order data"));
    producer.send(new ProducerRecord<>("payments", "payment-1", "payment data"));
    producer.send(new ProducerRecord<>("inventory", "item-1", "inventory update"));

    // 트랜잭션 커밋
    producer.commitTransaction();

} catch (ProducerFencedException | OutOfOrderSequenceException e) {
    // 치명적 오류 - Producer 재생성 필요
    producer.close();
} catch (KafkaException e) {
    // 일시적 오류 - 트랜잭션 중단
    producer.abortTransaction();
}
```

### Consume-Transform-Produce 패턴

Consume-Transform-Produce 패턴은 읽은 레코드를 변환해 다른 Topic에 쓰면서, 처리한 입력의 offset도 같은 트랜잭션에 포함한다. 아래 코드에서는 `sendOffsetsToTransaction`으로 offset을 전달한 뒤 출력 레코드와 함께 커밋한다.

```java
Properties consumerProps = new Properties();
consumerProps.put("bootstrap.servers", "localhost:9092");
consumerProps.put("group.id", "order-processor");
consumerProps.put("enable.auto.commit", "false");  // 자동 커밋 비활성화
consumerProps.put("isolation.level", "read_committed");  // 커밋된 메시지만 읽기

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps);

consumer.subscribe(List.of("input-topic"));
producer.initTransactions();

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    if (!records.isEmpty()) {
        producer.beginTransaction();

        try {
            Map<TopicPartition, OffsetAndMetadata> offsetsToCommit = new HashMap<>();

            for (ConsumerRecord<String, String> record : records) {
                // 메시지 변환
                String transformed = transform(record.value());

                // 결과 발행
                producer.send(new ProducerRecord<>("output-topic", record.key(), transformed));

                // 오프셋 기록
                offsetsToCommit.put(
                    new TopicPartition(record.topic(), record.partition()),
                    new OffsetAndMetadata(record.offset() + 1)
                );
            }

            // Consumer offset을 트랜잭션의 일부로 커밋
            producer.sendOffsetsToTransaction(offsetsToCommit, consumer.groupMetadata());

            producer.commitTransaction();

        } catch (Exception e) {
            producer.abortTransaction();
            // Consumer 위치 초기화 (롤백)
            consumer.seekToCommitted(consumer.assignment());
        }
    }
}
```

### Transactional Consumer

```java
// read_committed: 트랜잭션이 커밋된 메시지만 읽음
props.put("isolation.level", "read_committed");

// read_uncommitted (기본): 모든 메시지 읽음 (중단된 트랜잭션 포함)
// props.put("isolation.level", "read_uncommitted");
```

- isolation.level 동작:
- Producer 트랜잭션:
- [Msg A] [Msg B] [Msg C] [COMMIT] [Msg D] [Msg E] [ABORT] [Msg F] [COMMIT]
- read_uncommitted Consumer: A, B, C, D, E, F (모두 읽음)
- read_committed Consumer:, A, B, C, F (커밋된 것만 읽음)

## Kafka Streams

### Kafka Streams 개요

Kafka Streams는 스트림 처리 애플리케이션을 만드는 클라이언트 라이브러리다. 입력 Topic에서 읽은 데이터를 Filter, Map 등의 Processor로 처리하고 출력 Topic에 쓰는 흐름을 토폴로지로 정의한다. 처리 중 유지해야 할 상태는 RocksDB나 메모리 기반 State Store에 저장한다.

### DSL API 예제

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;

Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count-app");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

// Exactly-once 처리
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);

StreamsBuilder builder = new StreamsBuilder();

// Word Count 토폴로지 정의
KStream<String, String> textLines = builder.stream("text-input");

KTable<String, Long> wordCounts = textLines
    .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+")))
    .groupBy((key, word) -> word)
    .count(Materialized.as("word-counts-store"));

wordCounts.toStream().to("word-count-output",
    Produced.with(Serdes.String(), Serdes.Long()));

// 애플리케이션 시작
KafkaStreams streams = new KafkaStreams(builder.build(), props);

// Graceful shutdown
Runtime.getRuntime().addShutdownHook(new Thread(streams::close));

streams.start();
```

### Processor API (저수준)

```java
// 커스텀 Processor 정의
public class DeduplicationProcessor implements Processor<String, String, String, String> {

    private KeyValueStore<String, Long> store;
    private ProcessorContext<String, String> context;

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
        this.store = context.getStateStore("dedup-store");
    }

    @Override
    public void process(Record<String, String> record) {
        String key = record.key();
        Long lastSeen = store.get(key);
        long currentTime = record.timestamp();

        // 10분 이내 중복 메시지 필터링
        if (lastSeen == null || currentTime - lastSeen > 600000) {
            store.put(key, currentTime);
            context.forward(record);
        }
    }

    @Override
    public void close() {}
}

// 토폴로지에 Processor 추가
Topology topology = new Topology();

topology.addSource("source", "input-topic")
    .addProcessor("dedup", DeduplicationProcessor::new, "source")
    .addStateStore(
        Stores.keyValueStoreBuilder(
            Stores.persistentKeyValueStore("dedup-store"),
            Serdes.String(),
            Serdes.Long()
        ),
        "dedup"
    )
    .addSink("sink", "output-topic", "dedup");
```

### Windowed Operations

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> events = builder.stream("events");

// Tumbling Window: 고정 크기, 겹침 없음
KTable<Windowed<String>, Long> tumblingCounts = events
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count();

// Hopping Window: 고정 크기, 겹침 있음
KTable<Windowed<String>, Long> hoppingCounts = events
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1))
        .advanceBy(Duration.ofMinutes(1)))
    .count();

// Sliding Window: Join 용도
KStream<String, String> joined = stream1.join(
    stream2,
    (v1, v2) -> v1 + "," + v2,
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
    StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
);

// Session Window: 활동 기반, 동적 크기
KTable<Windowed<String>, Long> sessionCounts = events
    .groupByKey()
    .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
    .count();
```

## Kafka Connect

### Kafka Connect 아키텍처

- Kafka Connect Cluster
- Worker 1, Worker 2, Worker 3
- Source Task 1, Source Task 2, Sink Task 1
- Sink Task 2, Sink Task 3, Sink Task 4
- Source, Sink
- (Database,, Kafka Topics, (Database
- Files, etc), S3, etc)

### Connector 유형

- 유형: Source Connector:
  - 방향: 외부 → Kafka
  - 예시: JDBC Source, Debezium, File Source
- 유형: Sink Connector:
  - 방향: Kafka → 외부
  - 예시: JDBC Sink, Elasticsearch Sink, S3 Sink

### Source Connector 예제 (Debezium)

```json
{
    "name": "mysql-source-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "topic.prefix": "dbserver1",
        "database.include.list": "inventory",
        "table.include.list": "inventory.orders,inventory.products",
        "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
        "schema.history.internal.kafka.topic": "schemahistory.inventory",

        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "false",
        "transforms.unwrap.delete.handling.mode": "rewrite"
    }
}
```

### Sink Connector 예제 (Elasticsearch)

```json
{
    "name": "elasticsearch-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "topics": "orders,products",
        "connection.url": "http://elasticsearch:9200",
        "type.name": "_doc",
        "key.ignore": "false",
        "schema.ignore": "true",

        "behavior.on.null.values": "delete",
        "behavior.on.malformed.documents": "warn",

        "write.method": "upsert",
        "flush.timeout.ms": "10000",
        "batch.size": "2000",
        "max.buffered.records": "20000",

        "errors.tolerance": "all",
        "errors.log.enable": "true",
        "errors.log.include.messages": "true",
        "errors.deadletterqueue.topic.name": "dlq-elasticsearch"
    }
}
```

### Single Message Transforms (SMT)

```json
{
    "name": "jdbc-source-with-transforms",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:postgresql://postgres:5432/mydb",
        "table.whitelist": "users",

        "transforms": "InsertField,MaskField,TimestampRouter",

        "transforms.InsertField.type": "org.apache.kafka.connect.transforms.InsertField$Value",
        "transforms.InsertField.static.field": "source",
        "transforms.InsertField.static.value": "postgres",

        "transforms.MaskField.type": "org.apache.kafka.connect.transforms.MaskField$Value",
        "transforms.MaskField.fields": "ssn,credit_card",

        "transforms.TimestampRouter.type": "org.apache.kafka.connect.transforms.TimestampRouter",
        "transforms.TimestampRouter.topic.format": "${topic}-${timestamp}",
        "transforms.TimestampRouter.timestamp.format": "yyyyMMdd"
    }
}
```

### Distributed Mode 설정

```properties
# connect-distributed.properties

bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092
group.id=connect-cluster

# 내부 토픽 설정
config.storage.topic=connect-configs
config.storage.replication.factor=3

offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=25

status.storage.topic=connect-status
status.storage.replication.factor=3
status.storage.partitions=5

# Exactly-once 지원 (Kafka 3.3+)
exactly.once.source.support=enabled

# Converter 설정
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://schema-registry:8081

# REST API
rest.port=8083
rest.advertised.host.name=connect-worker-1
```

## ISR (In-Sync Replicas)

### 복제 메커니즘

- Partition 0 Replication (RF=3)
- Broker 1
- (Leader)
- [0][1][2][3][4], ←, Producer writes here
- Log End: 5
- HW: 4
- Fetch
- Broker 2, Broker 3
- (Follower/ISR), (Follower/ISR)
- [0][1][2][3][4], [0][1][2][3], ←, replica.lag.max.messages
- Log End: 5, Log End: 4, 초과 시 ISR에서 제거
- ISR = {Broker 1, Broker 2, Broker 3}
- High Watermark (HW) = 4 (모든 ISR에 복제된 최대 offset)

### ISR 관련 설정

```properties
# Broker 설정
# Follower가 ISR에 남아있기 위한 최대 지연 시간
replica.lag.time.max.ms=30000

# Topic 설정
# 메시지를 커밋하기 위한 최소 ISR 수
min.insync.replicas=2

# Producer 설정
# acks=all과 함께 min.insync.replicas로 내구성 보장
acks=all
```

### ISR 축소/확장 시나리오

- 시나리오: Broker 3 네트워크 지연
- 시간 T0:
- ISR = {1, 2, 3}
- Leader (Broker 1): offset 0-100
- Follower 2: offset 0-100 (sync)
- Follower 3: offset 0-100 (sync)
- 시간 T1 (replica.lag.time.max.ms 초과):
- ISR = {1, 2}, # Broker 3 제거
- Leader (Broker 1): offset 0-150
- Follower 2: offset 0-150 (sync)
- Follower 3: offset 0-100 (lagging)
- 시간 T2 (Broker 3 복구):
- ISR = {1, 2, 3}, # Broker 3 재합류
- 모든 브로커: offset 0-200 (sync)

### Unclean Leader Election

```properties
# 기본값: false (데이터 손실 방지)
# ISR이 비어있을 때 out-of-sync replica를 Leader로 선출하지 않음
unclean.leader.election.enable=false

# true로 설정 시:
# - 가용성 향상 (ISR이 비어도 서비스 계속)
# - 데이터 손실 위험 (동기화되지 않은 replica가 Leader가 됨)
```

### Producer acks와 min.insync.replicas 조합

- acks: 0:
  - min.insync.replicas: N/A
  - 동작: ACK 없이 전송
  - 내구성: 낮음
  - 지연시간: 매우 낮음
- acks: 1:
  - min.insync.replicas: N/A
  - 동작: Leader만 확인
  - 내구성: 중간
  - 지연시간: 낮음
- acks: all:
  - min.insync.replicas: 1
  - 동작: Leader만 있어도 성공
  - 내구성: 중간
  - 지연시간: 중간
- acks: all:
  - min.insync.replicas: 2
  - 동작: 최소 2개 ISR 필요
  - 내구성: 높음
  - 지연시간: 높음
- acks: all:
  - min.insync.replicas: 3
  - 동작: 모든 ISR 복제 필요
  - 내구성: 매우 높음
  - 지연시간: 매우 높음

```java
// 권장 설정 (높은 내구성)
props.put("acks", "all");
// Topic 생성 시
// min.insync.replicas=2
// replication.factor=3
```

## 참고 자료

- [Apache Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [Confluent Developer](https://developer.confluent.io/)
- [Kafka Streams Architecture](https://kafka.apache.org/documentation/streams/architecture)
- [Kafka Connect Design](https://docs.confluent.io/platform/current/connect/design.html)
- [KIP-848: Next Generation Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848)
- [Exactly-Once Semantics in Apache Kafka](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)

## 관련 학습

- [RabbitMQ](01-RabbitMQ.md)
- [서비스 메시](03-서비스-메시.md)
