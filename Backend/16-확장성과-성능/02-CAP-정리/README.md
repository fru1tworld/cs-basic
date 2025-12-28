# CAP 정리와 분산 시스템 트레이드오프

## 목차
1. [개요](#개요)
2. [CAP 정리 상세](#cap-정리-상세)
3. [PACELC 정리](#pacelc-정리)
4. [일관성 모델](#일관성-모델)
5. [CP vs AP 시스템 예시](#cp-vs-ap-시스템-예시)
6. [실전 적용 가이드](#실전-적용-가이드)
---

## 개요

분산 시스템 설계에서 가장 중요한 이론적 기반인 CAP 정리와 PACELC 정리를 이해하면, 시스템 아키텍처 결정의 근본적인 트레이드오프를 파악할 수 있습니다.

> "분산 시스템에서 완벽함은 없다. 트레이드오프를 이해하고 비즈니스 요구사항에 맞는 선택을 해야 한다." - Release It!

---

## CAP 정리 상세

### CAP 정리란?

CAP 정리는 2000년 Eric Brewer가 제안하고 2002년 Gilbert와 Lynch가 증명한 정리로, 분산 시스템은 다음 세 가지 속성 중 **최대 두 가지만** 동시에 보장할 수 있다고 말합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                      CAP Theorem                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    Consistency (C)                           │
│                         /\                                   │
│                        /  \                                  │
│                       /    \                                 │
│                      /  CA  \     ← 네트워크 파티션 없을 때  │
│                     /        \      가능 (단일 노드 시스템)  │
│                    /──────────\                              │
│                   /            \                             │
│                  /   CP    AP   \                            │
│                 /                \                           │
│                ────────────────────                          │
│           Partition           Availability                   │
│           Tolerance (P)            (A)                       │
│                                                              │
│   분산 시스템에서 P는 필수 → C와 A 중 선택해야 함            │
└─────────────────────────────────────────────────────────────┘
```

### 세 가지 속성

#### 1. Consistency (일관성)
- **정의**: 모든 노드가 동시에 같은 데이터를 볼 수 있음
- **의미**: 쓰기 작업 후 모든 읽기 작업은 최신 값을 반환
- **기술적 정의**: Linearizability (선형화 가능성)

```java
/**
 * 일관성 예시
 *
 * Node A에 쓰기 → 즉시 Node B에서 읽으면 같은 값이 보여야 함
 */
public class ConsistencyExample {

    // 일관성이 보장되는 경우
    public void consistentWrite() {
        // Node A에서 쓰기
        nodeA.write("balance", 100);

        // 일관성 보장 시: Node B에서 즉시 읽어도 100
        int value = nodeB.read("balance");
        assert value == 100; // 항상 참
    }

    // 일관성이 보장되지 않는 경우 (Eventually Consistent)
    public void eventuallyConsistentWrite() {
        nodeA.write("balance", 100);

        // Eventually Consistent: 잠시 후에야 100
        int value = nodeB.read("balance");
        // value가 이전 값일 수 있음!
    }
}
```

#### 2. Availability (가용성)
- **정의**: 모든 요청에 대해 (성공/실패) 응답을 받을 수 있음
- **의미**: 시스템의 일부가 장애여도 나머지가 동작
- **기술적 정의**: 모든 비장애 노드가 합리적인 시간 내에 응답

```java
/**
 * 가용성 예시
 */
public class AvailabilityExample {

    private final List<Node> nodes;
    private final LoadBalancer loadBalancer;

    /**
     * 고가용성 읽기
     * 일부 노드 장애에도 응답 가능
     */
    public Data highAvailabilityRead(String key) {
        for (int i = 0; i < nodes.size(); i++) {
            Node node = loadBalancer.getNextNode();
            try {
                return node.read(key);
            } catch (NodeUnavailableException e) {
                log.warn("Node {} unavailable, trying next", node.getId());
                continue;
            }
        }
        throw new AllNodesUnavailableException();
    }

    /**
     * Quorum 기반 읽기
     * N개 노드 중 R개 이상 응답하면 성공
     */
    public Data quorumRead(String key, int quorum) {
        List<CompletableFuture<Data>> futures = nodes.stream()
            .map(node -> CompletableFuture.supplyAsync(() -> node.read(key)))
            .collect(Collectors.toList());

        List<Data> responses = new ArrayList<>();
        for (CompletableFuture<Data> future : futures) {
            try {
                responses.add(future.get(TIMEOUT_MS, TimeUnit.MILLISECONDS));
                if (responses.size() >= quorum) {
                    return resolveConflicts(responses);
                }
            } catch (Exception e) {
                // 타임아웃 또는 실패 - 다음 노드 시도
            }
        }
        throw new QuorumNotReachedException();
    }
}
```

#### 3. Partition Tolerance (분할 허용성)
- **정의**: 네트워크 파티션(노드 간 통신 불가)이 발생해도 시스템이 동작
- **의미**: 분산 시스템에서는 **필수적**으로 보장해야 함
- **현실**: 네트워크 장애는 언제든 발생할 수 있음

```
┌─────────────────────────────────────────────────────────────┐
│                   Network Partition                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   정상 상태:                                                 │
│   ┌────────┐ ←──────→ ┌────────┐ ←──────→ ┌────────┐       │
│   │ Node 1 │          │ Node 2 │          │ Node 3 │       │
│   └────────┘          └────────┘          └────────┘       │
│                                                              │
│   파티션 발생:                                               │
│   ┌────────┐ ←──────→ ┌────────┐    ╳    ┌────────┐       │
│   │ Node 1 │          │ Node 2 │    ╳    │ Node 3 │       │
│   └────────┘          └────────┘    ╳    └────────┘       │
│                                                              │
│   Partition A: Node 1, 2    Partition B: Node 3             │
│                                                              │
│   이 상황에서:                                               │
│   - CP 시스템: 일관성을 위해 일부 요청 거부                  │
│   - AP 시스템: 가용성을 위해 불일치 허용                     │
└─────────────────────────────────────────────────────────────┘
```

### CAP 정리의 현실적 해석

```java
/**
 * CAP 정리의 현실적 적용
 *
 * 핵심: P(Partition Tolerance)는 선택이 아닌 필수
 * 분산 시스템에서는 CP 또는 AP 중 선택해야 함
 */
public class CAPRealWorldExample {

    /**
     * CP 시스템 예시: 은행 이체
     * 일관성이 중요하므로 파티션 시 가용성 포기
     */
    public TransferResult cpStyleTransfer(String from, String to, BigDecimal amount) {
        try {
            // 분산 락 획득 시도
            if (!distributedLock.tryLock(from, to, Duration.ofSeconds(5))) {
                throw new ServiceUnavailableException("Cannot acquire lock");
            }

            // 2PC로 일관성 보장
            return twoPhaseCommit.execute(() -> {
                accountService.debit(from, amount);
                accountService.credit(to, amount);
            });
        } catch (PartitionException e) {
            // 파티션 시 작업 거부 (일관성 우선)
            throw new ServiceUnavailableException("Partition detected, try later");
        }
    }

    /**
     * AP 시스템 예시: 소셜 미디어 좋아요
     * 가용성이 중요하므로 일시적 불일치 허용
     */
    public LikeResult apStyleLike(String postId, String userId) {
        try {
            // 로컬 노드에 즉시 쓰기 (가용성 우선)
            localNode.incrementLikeCount(postId);

            // 비동기로 다른 노드에 전파 (최종 일관성)
            eventBus.publishAsync(new LikeEvent(postId, userId));

            return LikeResult.success();
        } catch (Exception e) {
            // 실패해도 다른 노드에서 서비스 가능
            return LikeResult.queued();
        }
    }
}
```

---

## PACELC 정리

### PACELC란?

PACELC는 Daniel Abadi가 2010년에 제안한 CAP의 확장 정리입니다. CAP이 파티션 상황만 다루는 것과 달리, **정상 상황에서의 트레이드오프**도 설명합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                      PACELC Theorem                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   if Partition (P):                                          │
│       choose between Availability (A) and Consistency (C)    │
│   Else (E):                                                  │
│       choose between Latency (L) and Consistency (C)         │
│                                                              │
│   ┌─────────────────────┐    ┌─────────────────────┐        │
│   │  파티션 발생 시 (P) │    │   정상 상태 (E)     │        │
│   ├─────────────────────┤    ├─────────────────────┤        │
│   │                     │    │                     │        │
│   │   A ←────→ C       │    │   L ←────→ C       │        │
│   │   가용성   일관성   │    │   지연시간  일관성   │        │
│   │                     │    │                     │        │
│   └─────────────────────┘    └─────────────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### PACELC 분류

| 시스템 | 파티션 시 (P) | 정상 시 (E) | 분류 |
|--------|---------------|-------------|------|
| Cassandra | A (가용성) | L (지연시간) | PA/EL |
| DynamoDB | A (가용성) | L (지연시간) | PA/EL |
| MongoDB | A (가용성) | C (일관성) | PA/EC |
| HBase | C (일관성) | C (일관성) | PC/EC |
| MySQL Cluster | C (일관성) | L (지연시간) | PC/EL |
| PostgreSQL | C (일관성) | C (일관성) | PC/EC |

### PACELC 구현 예시

```java
/**
 * PA/EL 시스템 예시 (Cassandra 스타일)
 * 파티션 시: 가용성 우선
 * 정상 시: 지연시간 우선
 */
@Configuration
public class PAELSystemConfig {

    @Bean
    public CassandraTemplate cassandraTemplate(CqlSession session) {
        CassandraTemplate template = new CassandraTemplate(session);
        return template;
    }

    /**
     * Consistency Level 설정으로 트레이드오프 제어
     */
    public enum ConsistencyStrategy {
        // 빠른 응답, 약한 일관성
        LOW_LATENCY(ConsistencyLevel.ONE),

        // 중간 (Quorum)
        BALANCED(ConsistencyLevel.QUORUM),

        // 강한 일관성, 높은 지연
        STRONG_CONSISTENCY(ConsistencyLevel.ALL);

        private final ConsistencyLevel level;

        ConsistencyStrategy(ConsistencyLevel level) {
            this.level = level;
        }
    }
}

@Service
public class PAELProductService {

    private final CassandraTemplate cassandraTemplate;

    /**
     * 읽기: 지연시간 우선 (EL)
     * ONE 일관성 레벨로 가장 빠른 노드에서 읽기
     */
    public Product getProduct(String productId) {
        Statement<?> statement = SimpleStatement.builder(
            "SELECT * FROM products WHERE id = ?")
            .addPositionalValue(productId)
            .setConsistencyLevel(ConsistencyLevel.ONE)  // 빠른 응답
            .build();

        return cassandraTemplate.selectOne(statement, Product.class);
    }

    /**
     * 쓰기: 일관성과 지연시간 균형 (Quorum)
     */
    public void saveProduct(Product product) {
        Statement<?> statement = SimpleStatement.builder(
            "INSERT INTO products (id, name, price) VALUES (?, ?, ?)")
            .addPositionalValues(product.getId(), product.getName(), product.getPrice())
            .setConsistencyLevel(ConsistencyLevel.QUORUM)  // 과반수 확인
            .build();

        cassandraTemplate.execute(statement);
    }

    /**
     * 중요한 쓰기: 강한 일관성 (EC 선택)
     */
    public void saveInventory(Inventory inventory) {
        Statement<?> statement = SimpleStatement.builder(
            "UPDATE products SET stock = ? WHERE id = ?")
            .addPositionalValues(inventory.getStock(), inventory.getProductId())
            .setConsistencyLevel(ConsistencyLevel.ALL)  // 모든 노드 확인
            .build();

        cassandraTemplate.execute(statement);
    }
}

/**
 * PC/EC 시스템 예시 (HBase/PostgreSQL 스타일)
 * 항상 일관성 우선
 */
@Service
public class PCECAccountService {

    private final JdbcTemplate jdbcTemplate;
    private final TransactionTemplate transactionTemplate;

    /**
     * 계좌 이체: 강한 일관성 필수
     * 파티션 시에도 일관성 보장 (가용성 희생)
     */
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public TransferResult transfer(String fromAccount, String toAccount, BigDecimal amount) {
        // 비관적 락으로 일관성 보장
        Account from = accountRepository.findByIdForUpdate(fromAccount)
            .orElseThrow(() -> new AccountNotFoundException(fromAccount));
        Account to = accountRepository.findByIdForUpdate(toAccount)
            .orElseThrow(() -> new AccountNotFoundException(toAccount));

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException();
        }

        from.debit(amount);
        to.credit(amount);

        accountRepository.save(from);
        accountRepository.save(to);

        return TransferResult.success();
    }
}
```

---

## 일관성 모델

### 일관성 스펙트럼

```
┌─────────────────────────────────────────────────────────────┐
│                  Consistency Spectrum                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Strong                                           Weak      │
│   ◀────────────────────────────────────────────────────▶    │
│   │                    │              │            │        │
│   │                    │              │            │        │
│   Linearizable    Sequential     Causal      Eventual       │
│   (선형화)        (순차적)       (인과적)    (최종적)        │
│                                                              │
│   - 성능 낮음                            - 성능 높음         │
│   - 구현 복잡                            - 구현 단순         │
│   - 가용성 낮음                          - 가용성 높음       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1. Strong Consistency (강한 일관성)

```java
/**
 * Strong Consistency 구현 예시
 * 모든 읽기는 가장 최근 쓰기 결과를 반환
 */
@Service
public class StrongConsistencyService {

    private final DistributedLock distributedLock;
    private final ReplicatedDatabase database;

    /**
     * 분산 락을 사용한 강한 일관성
     */
    public Data readWithStrongConsistency(String key) {
        // 읽기도 락 필요 (성능 희생)
        try (var lock = distributedLock.acquire(key)) {
            return database.readFromLeader(key);  // 리더에서만 읽기
        }
    }

    /**
     * 2PC(Two-Phase Commit)를 통한 강한 일관성 쓰기
     */
    public void writeWithStrongConsistency(String key, Data data) {
        // Phase 1: Prepare
        List<PrepareResult> prepareResults = nodes.stream()
            .map(node -> node.prepare(key, data))
            .collect(Collectors.toList());

        boolean allPrepared = prepareResults.stream()
            .allMatch(PrepareResult::isSuccess);

        // Phase 2: Commit or Abort
        if (allPrepared) {
            nodes.forEach(node -> node.commit(key));
        } else {
            nodes.forEach(node -> node.abort(key));
            throw new ConsistencyException("Failed to achieve consensus");
        }
    }
}

/**
 * Raft 합의 알고리즘을 사용한 강한 일관성
 */
public class RaftBasedConsistency {

    private final RaftNode raftNode;

    public Data read(String key) {
        // 리더만 읽기 처리 (읽기 일관성)
        if (!raftNode.isLeader()) {
            return raftNode.forwardToLeader(new ReadCommand(key));
        }

        // 리더: 과반수 확인 후 응답
        return raftNode.readWithQuorum(key);
    }

    public void write(String key, Data data) {
        if (!raftNode.isLeader()) {
            raftNode.forwardToLeader(new WriteCommand(key, data));
            return;
        }

        // 로그에 기록하고 과반수 복제 후 커밋
        LogEntry entry = raftNode.appendLog(new WriteCommand(key, data));
        raftNode.replicateToMajority(entry);
        raftNode.commit(entry);
    }
}
```

### 2. Eventual Consistency (최종 일관성)

```java
/**
 * Eventual Consistency 구현 예시
 * 업데이트가 결국 모든 노드에 전파됨
 */
@Service
public class EventualConsistencyService {

    private final LocalStore localStore;
    private final EventBus eventBus;
    private final ConflictResolver conflictResolver;

    /**
     * 로컬 우선 쓰기 + 비동기 전파
     */
    public void write(String key, Data data) {
        // 1. 로컬에 즉시 쓰기 (빠른 응답)
        VersionedData versionedData = new VersionedData(
            data,
            VectorClock.increment(localStore.getVersion(key)),
            System.currentTimeMillis()
        );
        localStore.put(key, versionedData);

        // 2. 비동기로 다른 노드에 전파
        eventBus.publishAsync(new ReplicationEvent(key, versionedData));
    }

    /**
     * 로컬에서 읽기 (최신 아닐 수 있음)
     */
    public Data read(String key) {
        VersionedData data = localStore.get(key);
        // 클라이언트가 stale read 가능성 인지해야 함
        return data.getData();
    }

    /**
     * 복제 이벤트 수신 및 충돌 해결
     */
    @EventListener
    public void onReplicationEvent(ReplicationEvent event) {
        String key = event.getKey();
        VersionedData remote = event.getData();
        VersionedData local = localStore.get(key);

        if (local == null) {
            localStore.put(key, remote);
            return;
        }

        // Vector Clock으로 인과 관계 확인
        VectorClockComparison comparison = VectorClock.compare(
            local.getVersion(), remote.getVersion()
        );

        switch (comparison) {
            case BEFORE:
                // 로컬이 이전 버전 → 원격으로 업데이트
                localStore.put(key, remote);
                break;
            case AFTER:
                // 로컬이 최신 → 무시
                break;
            case CONCURRENT:
                // 동시 업데이트 → 충돌 해결
                VersionedData resolved = conflictResolver.resolve(local, remote);
                localStore.put(key, resolved);
                break;
        }
    }
}

/**
 * 충돌 해결 전략
 */
public interface ConflictResolver {
    VersionedData resolve(VersionedData v1, VersionedData v2);
}

// Last-Write-Wins (LWW) 전략
public class LastWriteWinsResolver implements ConflictResolver {
    @Override
    public VersionedData resolve(VersionedData v1, VersionedData v2) {
        return v1.getTimestamp() > v2.getTimestamp() ? v1 : v2;
    }
}

// 병합 전략 (CRDT)
public class MergeResolver implements ConflictResolver {
    @Override
    public VersionedData resolve(VersionedData v1, VersionedData v2) {
        // G-Counter 예시: 두 값을 합침
        return new VersionedData(
            mergeData(v1.getData(), v2.getData()),
            VectorClock.merge(v1.getVersion(), v2.getVersion()),
            Math.max(v1.getTimestamp(), v2.getTimestamp())
        );
    }
}
```

### 3. Causal Consistency (인과적 일관성)

```java
/**
 * Causal Consistency 구현 예시
 * 인과 관계가 있는 연산은 순서 보장
 */
@Service
public class CausalConsistencyService {

    /**
     * Vector Clock을 사용한 인과적 일관성
     */
    private static class CausalStore {
        private final Map<String, VersionedData> store = new ConcurrentHashMap<>();
        private final VectorClock localClock;
        private final String nodeId;

        public CausalStore(String nodeId, int nodeCount) {
            this.nodeId = nodeId;
            this.localClock = new VectorClock(nodeCount);
        }

        /**
         * 쓰기: 의존성 포함
         */
        public VectorClock write(String key, Data data, VectorClock dependencies) {
            // 의존성이 충족될 때까지 대기
            waitForDependencies(dependencies);

            // 로컬 클럭 증가
            VectorClock newVersion = localClock.increment(nodeId);

            VersionedData versionedData = new VersionedData(data, newVersion);
            store.put(key, versionedData);

            return newVersion;
        }

        /**
         * 읽기: 버전 반환하여 의존성 추적
         */
        public Pair<Data, VectorClock> read(String key) {
            VersionedData data = store.get(key);
            if (data == null) {
                return null;
            }

            // 로컬 클럭 업데이트 (읽은 데이터의 클럭 반영)
            localClock.merge(data.getVersion());

            return Pair.of(data.getData(), data.getVersion());
        }

        private void waitForDependencies(VectorClock dependencies) {
            while (!localClock.dominates(dependencies)) {
                // 의존성이 도착할 때까지 대기
                Thread.sleep(10);
            }
        }
    }

    /**
     * 인과적 일관성 사용 예시: 소셜 미디어 댓글
     */
    public void causalCommentExample() {
        CausalStore store = new CausalStore("node1", 3);

        // 사용자 A가 게시물 작성
        VectorClock postVersion = store.write("post:1",
            new Data("Hello World"),
            VectorClock.empty());

        // 사용자 B가 게시물을 읽고 댓글 작성
        Pair<Data, VectorClock> post = store.read("post:1");

        // 댓글은 게시물에 의존 → 게시물 버전을 의존성으로 전달
        VectorClock commentVersion = store.write("comment:1",
            new Data("Nice post!"),
            post.getRight());  // 게시물 버전을 의존성으로

        // 다른 노드에서: 댓글이 먼저 도착해도 게시물이 올 때까지 대기
        // → 게시물 없이 댓글만 보이는 상황 방지
    }
}
```

### 일관성 모델 비교

| 모델 | 보장 수준 | 성능 | 가용성 | 사용 사례 |
|------|-----------|------|--------|-----------|
| Linearizable | 매우 강함 | 낮음 | 낮음 | 분산 락, 리더 선출 |
| Sequential | 강함 | 중간 | 중간 | 트랜잭션 DB |
| Causal | 중간 | 중간 | 중간 | 소셜 미디어, 협업 도구 |
| Eventual | 약함 | 높음 | 높음 | 캐시, 좋아요, 조회수 |

---

## CP vs AP 시스템 예시

### CP 시스템 (일관성 우선)

```java
/**
 * CP 시스템 예시: ZooKeeper 기반 분산 설정 관리
 */
@Service
public class CPConfigService {

    private final CuratorFramework zkClient;

    /**
     * ZooKeeper: 강한 일관성 보장
     * - Zab 프로토콜로 모든 노드 동기화
     * - 과반수 노드 장애 시 쓰기 불가
     */
    public void updateConfig(String key, String value) throws KeeperException {
        String path = "/config/" + key;

        try {
            // ZK의 setData는 강한 일관성 보장
            zkClient.setData()
                .forPath(path, value.getBytes(StandardCharsets.UTF_8));

            log.info("Config updated: {} = {}", key, value);
        } catch (KeeperException.NoNodeException e) {
            // 노드가 없으면 생성
            zkClient.create()
                .creatingParentsIfNeeded()
                .forPath(path, value.getBytes(StandardCharsets.UTF_8));
        }
    }

    /**
     * 읽기도 일관성 보장 (sync 후 읽기)
     */
    public String getConfig(String key) throws Exception {
        String path = "/config/" + key;

        // sync로 최신 상태 동기화
        zkClient.sync().forPath(path);

        byte[] data = zkClient.getData().forPath(path);
        return new String(data, StandardCharsets.UTF_8);
    }
}

/**
 * CP 시스템 예시: Redis Cluster (동기 복제 모드)
 */
@Service
public class CPCacheService {

    private final RedisClusterClient redisClient;

    /**
     * WAIT 명령으로 동기 복제 보장
     */
    public void setWithSyncReplication(String key, String value) {
        RedisClusterCommands<String, String> commands = redisClient.connect().sync();

        // 값 설정
        commands.set(key, value);

        // 최소 1개 레플리카에 복제될 때까지 대기 (최대 1초)
        long replicasAcked = commands.waitForReplication(1, 1000);

        if (replicasAcked < 1) {
            throw new ReplicationFailedException("Failed to replicate to any replica");
        }
    }
}

/**
 * CP 시스템 예시: etcd
 */
@Service
public class CPServiceDiscovery {

    private final Client etcdClient;

    /**
     * etcd: Raft 기반 강한 일관성
     * 서비스 등록
     */
    public void registerService(String serviceName, String endpoint, int ttlSeconds) {
        KV kvClient = etcdClient.getKVClient();
        Lease leaseClient = etcdClient.getLeaseClient();

        // 리스 생성 (TTL)
        long leaseId = leaseClient.grant(ttlSeconds).get().getID();

        // 서비스 등록 (리스와 함께)
        String key = "/services/" + serviceName + "/" + endpoint;
        kvClient.put(
            ByteSequence.from(key, StandardCharsets.UTF_8),
            ByteSequence.from(endpoint, StandardCharsets.UTF_8),
            PutOption.newBuilder().withLeaseId(leaseId).build()
        ).get();

        // 리스 갱신 (헬스체크)
        leaseClient.keepAlive(leaseId, new StreamObserver<>() {
            @Override
            public void onNext(LeaseKeepAliveResponse response) {
                log.debug("Lease renewed: {}", response.getID());
            }

            @Override
            public void onError(Throwable t) {
                log.error("Lease renewal failed", t);
            }

            @Override
            public void onCompleted() {}
        });
    }
}
```

### AP 시스템 (가용성 우선)

```java
/**
 * AP 시스템 예시: Cassandra
 */
@Service
public class APProductCatalogService {

    private final CassandraTemplate cassandraTemplate;

    /**
     * Cassandra: 가용성 우선
     * - 파티션 시에도 쓰기 가능
     * - 나중에 충돌 해결
     */
    public void updateProductPrice(String productId, BigDecimal newPrice) {
        // ONE: 하나의 노드에만 쓰면 성공
        Statement<?> statement = SimpleStatement.builder(
            "UPDATE products SET price = ?, updated_at = ? WHERE id = ?")
            .addPositionalValues(newPrice, Instant.now(), productId)
            .setConsistencyLevel(ConsistencyLevel.ONE)
            .build();

        cassandraTemplate.execute(statement);
        // 나중에 다른 노드로 비동기 복제
    }

    /**
     * Read Repair로 일관성 복구
     */
    public Product getProduct(String productId) {
        Statement<?> statement = SimpleStatement.builder(
            "SELECT * FROM products WHERE id = ?")
            .addPositionalValue(productId)
            .setConsistencyLevel(ConsistencyLevel.QUORUM)
            .build();

        // Quorum 읽기 시 불일치 감지되면 자동 복구
        return cassandraTemplate.selectOne(statement, Product.class);
    }
}

/**
 * AP 시스템 예시: DynamoDB
 */
@Service
public class APSessionService {

    private final DynamoDbClient dynamoDbClient;
    private final String tableName = "user_sessions";

    /**
     * DynamoDB: Eventually Consistent Read (기본)
     * 빠른 응답, 최신 아닐 수 있음
     */
    public UserSession getSession(String sessionId) {
        GetItemRequest request = GetItemRequest.builder()
            .tableName(tableName)
            .key(Map.of("sessionId", AttributeValue.builder().s(sessionId).build()))
            .consistentRead(false)  // Eventually Consistent (빠름)
            .build();

        GetItemResponse response = dynamoDbClient.getItem(request);
        return mapToSession(response.item());
    }

    /**
     * Strongly Consistent Read (필요시)
     * 느리지만 최신 데이터 보장
     */
    public UserSession getSessionConsistent(String sessionId) {
        GetItemRequest request = GetItemRequest.builder()
            .tableName(tableName)
            .key(Map.of("sessionId", AttributeValue.builder().s(sessionId).build()))
            .consistentRead(true)  // Strongly Consistent (느림)
            .build();

        GetItemResponse response = dynamoDbClient.getItem(request);
        return mapToSession(response.item());
    }
}

/**
 * AP 시스템 예시: CouchDB
 */
@Service
public class APDocumentService {

    private final CouchDbClient couchDbClient;

    /**
     * CouchDB: Multi-Master Replication
     * 충돌 시 Revision 기반 해결
     */
    public void saveDocument(Document doc) {
        // 저장 시 revision 포함
        String revision = couchDbClient.save(doc);
        doc.setRevision(revision);
    }

    /**
     * 충돌 문서 조회 및 해결
     */
    public Document getDocumentWithConflictResolution(String docId) {
        Document doc = couchDbClient.find(Document.class, docId);

        // 충돌 확인
        List<String> conflicts = couchDbClient.getConflicts(docId);

        if (!conflicts.isEmpty()) {
            // 충돌 해결: 각 버전 가져와서 병합
            List<Document> conflictDocs = conflicts.stream()
                .map(rev -> couchDbClient.find(Document.class, docId, rev))
                .collect(Collectors.toList());

            Document merged = mergeDocuments(doc, conflictDocs);

            // 병합된 문서 저장, 충돌 버전 삭제
            couchDbClient.save(merged);
            conflicts.forEach(rev -> couchDbClient.delete(docId, rev));

            return merged;
        }

        return doc;
    }
}
```

### 실제 시스템의 CAP 분류

```
┌─────────────────────────────────────────────────────────────┐
│              Real-World System CAP Classification            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   CP Systems (Consistency + Partition Tolerance):           │
│   ┌─────────────────────────────────────────────┐          │
│   │ • ZooKeeper - 분산 코디네이션              │          │
│   │ • etcd - 서비스 디스커버리                 │          │
│   │ • HBase - 빅데이터 저장소                  │          │
│   │ • MongoDB (default) - 문서 DB              │          │
│   │ • Redis Cluster - 캐시/세션               │          │
│   │ • CockroachDB - 분산 SQL                  │          │
│   └─────────────────────────────────────────────┘          │
│                                                              │
│   AP Systems (Availability + Partition Tolerance):          │
│   ┌─────────────────────────────────────────────┐          │
│   │ • Cassandra - 와이드 컬럼 스토어           │          │
│   │ • DynamoDB - 관리형 NoSQL                  │          │
│   │ • CouchDB - 문서 DB                        │          │
│   │ • Riak - 분산 KV 스토어                   │          │
│   │ • Voldemort - KV 스토어                   │          │
│   └─────────────────────────────────────────────┘          │
│                                                              │
│   Tunable (설정에 따라 변경 가능):                          │
│   ┌─────────────────────────────────────────────┐          │
│   │ • Cassandra (CL 조정)                      │          │
│   │ • DynamoDB (Strong/Eventual Read)          │          │
│   │ • MongoDB (WriteConcern/ReadPreference)    │          │
│   └─────────────────────────────────────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 실전 적용 가이드

### 시스템 선택 의사결정 트리

```
비즈니스 요구사항 분석
        │
        ▼
데이터 정합성이 금전적 손실을 야기하나?
        │
        ├── Yes ──→ CP 시스템 선택
        │           (예: 결제, 재고, 예약)
        │
        └── No
            │
            ▼
        시스템 다운타임이 수용 가능한가?
            │
            ├── Yes ──→ CP 시스템 선택
            │
            └── No
                │
                ▼
            일시적 데이터 불일치 허용?
                │
                ├── Yes ──→ AP 시스템 선택
                │           (예: 소셜미디어, 로그)
                │
                └── No ──→ 강한 일관성 + 고가용성
                           (비용 높음, 복잡도 높음)
                           2PC, Paxos, Raft 고려
```

### 실전 아키텍처 예시

```java
/**
 * 이커머스 시스템 아키텍처
 * 서비스별로 다른 일관성 모델 적용
 */
@Configuration
public class EcommerceArchitecture {

    /**
     * 결제 서비스: CP (강한 일관성)
     * - 금전 거래는 정확해야 함
     * - 파티션 시 실패가 불일치보다 나음
     */
    @Bean
    public PaymentService paymentService(PostgresRepository repo) {
        return new CPPaymentService(repo);
    }

    /**
     * 재고 서비스: CP (강한 일관성)
     * - 오버셀링 방지
     * - 동시성 제어 필수
     */
    @Bean
    public InventoryService inventoryService(RedissonClient redisson) {
        return new CPInventoryService(redisson);
    }

    /**
     * 상품 카탈로그: AP (가용성 우선)
     * - 상품 정보는 약간 지연되어도 OK
     * - 고가용성이 중요
     */
    @Bean
    public ProductCatalogService catalogService(CassandraTemplate template) {
        return new APProductCatalogService(template);
    }

    /**
     * 검색 서비스: AP (가용성 우선)
     * - 검색 결과 약간 오래되어도 OK
     * - 빠른 응답이 중요
     */
    @Bean
    public SearchService searchService(ElasticsearchClient client) {
        return new APSearchService(client);
    }

    /**
     * 세션/캐시: AP (가용성 우선)
     * - 세션 유실 시 재로그인
     * - 고가용성이 중요
     */
    @Bean
    public SessionService sessionService(RedisTemplate<String, Object> template) {
        return new APSessionService(template);
    }
}
```

---

## 참고 자료

- [CAP Theorem - Wikipedia](https://en.wikipedia.org/wiki/CAP_theorem)
- [PACELC Theorem - Wikipedia](https://en.wikipedia.org/wiki/PACELC_design_principle)
- [CAP, PACELC, ACID, BASE - ByteByteGo](https://blog.bytebytego.com/p/cap-pacelc-acid-base-essential-concepts)
- [System Design Interview Basics: CAP vs PACELC - DesignGurus](https://www.designgurus.io/blog/system-design-interview-basics-cap-vs-pacelc)
- [Release It!: Design and Deploy Production-Ready Software](https://pragprog.com/titles/mnee2/release-it-second-edition/) - Michael Nygard
- [Designing Data-Intensive Applications](https://dataintensive.net/) - Martin Kleppmann
