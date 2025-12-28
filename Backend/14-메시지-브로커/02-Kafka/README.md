# Apache Kafka

## 목차
1. [아키텍처](#1-아키텍처)
2. [Producer와 Consumer](#2-producer와-consumer)
3. [Consumer Group과 Rebalancing](#3-consumer-group과-rebalancing)
4. [정확히 한 번 전송 (Exactly-Once Semantics)](#4-정확히-한-번-전송-exactly-once-semantics)
5. [Kafka Streams](#5-kafka-streams)
6. [Kafka Connect](#6-kafka-connect)
7. [ISR (In-Sync Replicas)](#7-isr-in-sync-replicas)
---

## 1. 아키텍처

### 1.1 Kafka 핵심 구성요소

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Kafka Cluster                                     │
│                                                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                           │
│  │ Broker 1 │     │ Broker 2 │     │ Broker 3 │                           │
│  │          │     │          │     │          │                           │
│  │ Topic A  │     │ Topic A  │     │ Topic A  │                           │
│  │ P0(L)    │     │ P0(F)    │     │ P1(L)    │                           │
│  │ P1(F)    │     │ P1(F)    │     │ P0(F)    │                           │
│  │          │     │          │     │          │                           │
│  │ Topic B  │     │ Topic B  │     │ Topic B  │                           │
│  │ P0(F)    │     │ P0(L)    │     │ P0(F)    │                           │
│  └──────────┘     └──────────┘     └──────────┘                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    KRaft Controller Quorum                          │  │
│  │              (Kafka 4.0+ : ZooKeeper 완전 제거)                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
        ▲                                                       │
        │                                                       ▼
   ┌─────────┐                                           ┌─────────────┐
   │Producer │                                           │Consumer     │
   └─────────┘                                           │Group        │
                                                         └─────────────┘

L = Leader, F = Follower
```

### 1.2 Broker

Broker는 Kafka 클러스터의 개별 서버로, 메시지를 저장하고 클라이언트 요청을 처리한다.

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

### 1.3 Topic과 Partition

**Topic**: 메시지의 논리적 그룹. 카테고리 또는 피드 이름과 유사

**Partition**: Topic을 물리적으로 분할한 단위. 병렬 처리와 확장성의 핵심

```
Topic: orders (3 Partitions, Replication Factor: 3)

Partition 0:                    Partition 1:                    Partition 2:
┌─────────────────────┐        ┌─────────────────────┐        ┌─────────────────────┐
│ Offset 0 │ Offset 1 │        │ Offset 0 │ Offset 1 │        │ Offset 0 │ Offset 1 │
│   Msg A  │   Msg D  │        │   Msg B  │   Msg E  │        │   Msg C  │   Msg F  │
└─────────────────────┘        └─────────────────────┘        └─────────────────────┘
     │                              │                              │
     ▼                              ▼                              ▼
  Leader: Broker 1              Leader: Broker 2              Leader: Broker 3
  Followers: Broker 2,3         Followers: Broker 1,3         Followers: Broker 1,2
```

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

### 1.4 Segment

Partition은 여러 Segment 파일로 구성된다. 각 Segment는 로그 파일(.log), 인덱스 파일(.index), 타임스탬프 인덱스(.timeindex)로 구성된다.

```
Partition 0 디렉토리 구조:
├── 00000000000000000000.log        # 첫 번째 세그먼트 (offset 0부터)
├── 00000000000000000000.index      # offset -> position 매핑
├── 00000000000000000000.timeindex  # timestamp -> offset 매핑
├── 00000000000000156789.log        # 두 번째 세그먼트 (offset 156789부터)
├── 00000000000000156789.index
├── 00000000000000156789.timeindex
└── leader-epoch-checkpoint         # 리더 에포크 정보
```

**Segment 롤오버 조건:**
1. `log.segment.bytes` 크기 초과 (기본 1GB)
2. `log.roll.ms` 또는 `log.roll.hours` 시간 경과
3. 인덱스/타임인덱스 파일 크기 제한 초과

### 1.5 Offset

Offset은 Partition 내 각 메시지의 고유 식별자이다. Consumer가 읽은 위치를 추적하는 데 사용된다.

```
Partition 0:
┌────────────────────────────────────────────────────────────────────┐
│ Offset: 0    1    2    3    4    5    6    7    8    9   10   11  │
│        [A]  [B]  [C]  [D]  [E]  [F]  [G]  [H]  [I]  [J]  [K]  [L]  │
└────────────────────────────────────────────────────────────────────┘
                   ▲                                    ▲
                   │                                    │
            Consumer Position                    High Watermark
              (Committed: 3)                   (Committed to ISR)

Log Start Offset: 0 (가장 오래된 메시지)
Log End Offset: 12 (다음 메시지가 쓰일 위치)
High Watermark: 10 (Consumer가 읽을 수 있는 최대 위치)
```

---

## 2. Producer와 Consumer

### 2.1 Producer 기본 설정

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

### 2.2 Producer 메시지 전송

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

### 2.3 Partitioner

메시지가 어느 Partition으로 전송될지 결정하는 컴포넌트

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

### 2.4 Consumer 기본 설정

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

### 2.5 Consumer 메시지 소비

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

### 2.6 Offset Commit 전략

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

---

## 3. Consumer Group과 Rebalancing

### 3.1 Consumer Group 개념

```
Topic: orders (6 Partitions)

Consumer Group: order-processing
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Consumer 1          Consumer 2          Consumer 3                    │
│  ┌─────────┐         ┌─────────┐         ┌─────────┐                   │
│  │ P0, P1  │         │ P2, P3  │         │ P4, P5  │                   │
│  └─────────┘         └─────────┘         └─────────┘                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

규칙:
- 하나의 파티션은 그룹 내 하나의 Consumer에만 할당
- Consumer 수 > 파티션 수 → 일부 Consumer는 유휴 상태
- Consumer 수 < 파티션 수 → 일부 Consumer가 여러 파티션 담당
```

### 3.2 Rebalancing 트리거

```
Rebalancing 발생 조건:
1. Consumer 그룹에 새 Consumer 참여
2. Consumer가 그룹 탈퇴 (정상 종료 또는 장애)
3. Topic에 새 파티션 추가
4. Consumer가 max.poll.interval.ms 내에 poll() 미호출
5. session.timeout.ms 내에 heartbeat 미전송
```

### 3.3 Partition Assignment 전략

```java
// Kafka 3.0+ 기본값: CooperativeStickyAssignor
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");

// 여러 전략 동시 설정 (점진적 마이그레이션용)
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor," +
    "org.apache.kafka.clients.consumer.RangeAssignor");
```

| 전략 | 설명 | 장점 | 단점 |
|-----|------|-----|------|
| **RangeAssignor** | 토픽별로 연속된 파티션 할당 | 예측 가능한 할당 | 토픽이 많을 때 불균형 |
| **RoundRobinAssignor** | 전체 파티션을 순환 할당 | 균등 분배 | Eager 프로토콜 사용 |
| **StickyAssignor** | 기존 할당 유지 + 균형 재조정 | 재할당 최소화 | Eager 프로토콜 사용 |
| **CooperativeStickyAssignor** | Sticky + Incremental Rebalance | 중단 시간 최소화 | Kafka 2.4+ 필요 |

### 3.4 Eager vs Cooperative Rebalancing

```
=== Eager Protocol (기존) ===

1단계: 모든 Consumer 정지 (Stop-the-World)
    Consumer 1: P0, P1 → 반납
    Consumer 2: P2, P3 → 반납
    Consumer 3: P4, P5 → 반납

2단계: 전체 재할당
    Consumer 1: P0, P2 할당
    Consumer 2: P1, P3 할당
    Consumer 3: P4, P5 할당

문제: 전체 Consumer가 일시적으로 메시지 처리 중단


=== Cooperative Protocol (Incremental) ===

1단계: 필요한 파티션만 반납
    Consumer 1: P1 반납 (P0 유지하며 계속 처리)
    Consumer 2: (변경 없음, 계속 처리)
    Consumer 3: (변경 없음, 계속 처리)

2단계: 반납된 파티션만 재할당
    Consumer 4 (신규): P1 할당

장점: 대부분의 Consumer가 계속 처리 가능
```

### 3.5 KIP-848: 차세대 Consumer Rebalance Protocol

Kafka 4.0부터 서버 사이드 리밸런싱이 도입되었다.

```java
// Kafka 4.0+ 설정
// 서버에서 group.consumer.assignors 설정
// uniform (기본) 또는 range

// 클라이언트는 별도 설정 불필요
// 자동으로 ConsumerGroupHeartbeat API 사용
```

**KIP-848 핵심 변경사항:**
1. 리밸런싱 로직이 클라이언트에서 서버(Group Coordinator)로 이동
2. 전역 동기화 장벽 제거 - 진정한 증분 리밸런싱
3. `ConsumerGroupHeartbeat` API로 멤버십과 할당 통합 관리
4. 할당이 변경되지 않는 Consumer는 리밸런싱 영향 없음

### 3.6 Rebalancing 최적화

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

---

## 4. 정확히 한 번 전송 (Exactly-Once Semantics)

### 4.1 메시지 전달 보장 수준

| 보장 수준 | 설명 | 메시지 손실 | 중복 |
|----------|------|-----------|------|
| **At-most-once** | 최대 한 번 전달 | 가능 | 없음 |
| **At-least-once** | 최소 한 번 전달 | 없음 | 가능 |
| **Exactly-once** | 정확히 한 번 전달 | 없음 | 없음 |

### 4.2 Idempotent Producer

네트워크 장애로 인한 재시도 시 중복 메시지를 방지한다.

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

**동작 원리:**
```
┌──────────┐                              ┌──────────┐
│ Producer │                              │  Broker  │
│          │                              │          │
│ PID: 123 │──── Batch (seq=0) ──────────▶│          │
│          │                              │ seq=0 저장│
│          │◀─── ACK ─────────────────────│          │
│          │                              │          │
│          │──── Batch (seq=1) ──────────▶│          │
│          │    (네트워크 오류)            │ seq=1 저장│
│          │                              │          │
│          │──── Batch (seq=1) 재시도 ────▶│          │
│          │                              │ 중복 감지 │
│          │◀─── ACK (중복이지만 성공) ───│ seq=1 이미│
│          │                              │ 있음     │
└──────────┘                              └──────────┘

- PID (Producer ID): 각 Producer 인스턴스의 고유 ID
- Sequence Number: 파티션별 메시지 순서 번호
- Broker는 (PID, Partition, SeqNum)으로 중복 감지
```

### 4.3 Transactional Producer

여러 파티션에 대한 원자적 쓰기와 Consumer offset 커밋을 보장한다.

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

### 4.4 Consume-Transform-Produce 패턴

읽기, 처리, 쓰기를 하나의 트랜잭션으로 묶는 패턴

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

### 4.5 Transactional Consumer

```java
// read_committed: 트랜잭션이 커밋된 메시지만 읽음
props.put("isolation.level", "read_committed");

// read_uncommitted (기본): 모든 메시지 읽음 (중단된 트랜잭션 포함)
// props.put("isolation.level", "read_uncommitted");
```

**isolation.level 동작:**
```
Producer 트랜잭션:
[Msg A] [Msg B] [Msg C] [COMMIT] [Msg D] [Msg E] [ABORT] [Msg F] [COMMIT]

read_uncommitted Consumer: A, B, C, D, E, F (모두 읽음)
read_committed Consumer:   A, B, C, F (커밋된 것만 읽음)
```

---

## 5. Kafka Streams

### 5.1 Kafka Streams 개요

Kafka Streams는 Kafka에 내장된 클라이언트 라이브러리로, 스트림 처리 애플리케이션을 구축할 수 있다.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kafka Streams Application                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        Stream Processing Topology                    │  │
│  │                                                                      │  │
│  │   Source           Processor           Processor           Sink     │  │
│  │  ┌───────┐        ┌─────────┐        ┌─────────┐        ┌───────┐  │  │
│  │  │ Input │───────▶│  Filter │───────▶│   Map   │───────▶│Output │  │  │
│  │  │ Topic │        └─────────┘        └─────────┘        │ Topic │  │  │
│  │  └───────┘             │                  │              └───────┘  │  │
│  │                        │                  │                         │  │
│  │                        ▼                  ▼                         │  │
│  │                   ┌─────────────────────────────┐                  │  │
│  │                   │        State Store           │                  │  │
│  │                   │     (RocksDB / In-Memory)    │                  │  │
│  │                   └─────────────────────────────┘                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 DSL API 예제

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

### 5.3 Processor API (저수준)

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

### 5.4 Windowed Operations

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

---

## 6. Kafka Connect

### 6.1 Kafka Connect 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kafka Connect Cluster                               │
│                                                                             │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │    Worker 1     │   │    Worker 2     │   │    Worker 3     │          │
│  │                 │   │                 │   │                 │          │
│  │ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────────┐ │          │
│  │ │Source Task 1│ │   │ │Source Task 2│ │   │ │ Sink Task 1 │ │          │
│  │ └─────────────┘ │   │ └─────────────┘ │   │ └─────────────┘ │          │
│  │ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────────┐ │          │
│  │ │ Sink Task 2 │ │   │ │ Sink Task 3 │ │   │ │ Sink Task 4 │ │          │
│  │ └─────────────┘ │   │ └─────────────┘ │   │ └─────────────┘ │          │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
        ▲                                                   │
        │                                                   ▼
┌───────────────┐                                   ┌───────────────┐
│  Source       │                                   │  Sink         │
│  (Database,   │           Kafka Topics            │  (Database,   │
│   Files, etc) │                                   │   S3, etc)    │
└───────────────┘                                   └───────────────┘
```

### 6.2 Connector 유형

| 유형 | 방향 | 예시 |
|-----|------|-----|
| **Source Connector** | 외부 → Kafka | JDBC Source, Debezium, File Source |
| **Sink Connector** | Kafka → 외부 | JDBC Sink, Elasticsearch Sink, S3 Sink |

### 6.3 Source Connector 예제 (Debezium)

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

### 6.4 Sink Connector 예제 (Elasticsearch)

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

### 6.5 Single Message Transforms (SMT)

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

### 6.6 Distributed Mode 설정

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

---

## 7. ISR (In-Sync Replicas)

### 7.1 복제 메커니즘

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Partition 0 Replication (RF=3)                           │
│                                                                             │
│  ┌───────────────────┐                                                     │
│  │   Broker 1        │                                                     │
│  │   (Leader)        │                                                     │
│  │                   │                                                     │
│  │  [0][1][2][3][4]  │◀──── Producer writes here                          │
│  │   Log End: 5      │                                                     │
│  │   HW: 4           │                                                     │
│  └───────────────────┘                                                     │
│          │                                                                  │
│          │ Fetch                                                           │
│          ▼                                                                  │
│  ┌───────────────────┐   ┌───────────────────┐                            │
│  │   Broker 2        │   │   Broker 3        │                            │
│  │   (Follower/ISR)  │   │   (Follower/ISR)  │                            │
│  │                   │   │                   │                            │
│  │  [0][1][2][3][4]  │   │  [0][1][2][3]     │ ◀── replica.lag.max.messages│
│  │   Log End: 5      │   │   Log End: 4      │     초과 시 ISR에서 제거    │
│  └───────────────────┘   └───────────────────┘                            │
│                                                                             │
│  ISR = {Broker 1, Broker 2, Broker 3}                                      │
│  High Watermark (HW) = 4 (모든 ISR에 복제된 최대 offset)                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 ISR 관련 설정

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

### 7.3 ISR 축소/확장 시나리오

```
시나리오: Broker 3 네트워크 지연

시간 T0:
  ISR = {1, 2, 3}
  Leader (Broker 1): offset 0-100
  Follower 2: offset 0-100 (sync)
  Follower 3: offset 0-100 (sync)

시간 T1 (replica.lag.time.max.ms 초과):
  ISR = {1, 2}  # Broker 3 제거
  Leader (Broker 1): offset 0-150
  Follower 2: offset 0-150 (sync)
  Follower 3: offset 0-100 (lagging)

시간 T2 (Broker 3 복구):
  ISR = {1, 2, 3}  # Broker 3 재합류
  모든 브로커: offset 0-200 (sync)
```

### 7.4 Unclean Leader Election

```properties
# 기본값: false (데이터 손실 방지)
# ISR이 비어있을 때 out-of-sync replica를 Leader로 선출하지 않음
unclean.leader.election.enable=false

# true로 설정 시:
# - 가용성 향상 (ISR이 비어도 서비스 계속)
# - 데이터 손실 위험 (동기화되지 않은 replica가 Leader가 됨)
```

### 7.5 Producer acks와 min.insync.replicas 조합

| acks | min.insync.replicas | 동작 | 내구성 | 지연시간 |
|------|---------------------|------|--------|----------|
| 0 | N/A | ACK 없이 전송 | 낮음 | 매우 낮음 |
| 1 | N/A | Leader만 확인 | 중간 | 낮음 |
| all | 1 | Leader만 있어도 성공 | 중간 | 중간 |
| all | 2 | 최소 2개 ISR 필요 | 높음 | 높음 |
| all | 3 | 모든 ISR 복제 필요 | 매우 높음 | 매우 높음 |

```java
// 권장 설정 (높은 내구성)
props.put("acks", "all");
// Topic 생성 시
// min.insync.replicas=2
// replication.factor=3
```

---

## 참고 자료

- [Apache Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [Confluent Developer](https://developer.confluent.io/)
- [Kafka Streams Architecture](https://kafka.apache.org/documentation/streams/architecture)
- [Kafka Connect Design](https://docs.confluent.io/platform/current/connect/design.html)
- [KIP-848: Next Generation Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848)
- [Exactly-Once Semantics in Apache Kafka](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
