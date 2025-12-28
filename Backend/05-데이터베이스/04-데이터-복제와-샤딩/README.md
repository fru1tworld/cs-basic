# 데이터 복제와 샤딩

## 목차
1. [데이터 복제 개요](#데이터-복제-개요)
2. [Master-Slave 복제](#master-slave-복제)
3. [샤딩 전략](#샤딩-전략)
4. [CAP 정리](#cap-정리)
5. [일관성 모델](#일관성-모델)
6. [실무 아키텍처](#실무-아키텍처)
---

## 데이터 복제 개요

### 복제의 목적

```
┌─────────────────────────────────────────────────────────────────┐
│                     복제(Replication)의 목적                      │
│                                                                  │
│  1. 고가용성 (High Availability)                                 │
│     - 노드 장애 시에도 서비스 지속                               │
│     - 자동 페일오버                                              │
│                                                                  │
│  2. 읽기 확장성 (Read Scalability)                               │
│     - 읽기 부하를 여러 노드에 분산                               │
│     - Read Replica 추가로 처리량 증가                            │
│                                                                  │
│  3. 지리적 분산 (Geographic Distribution)                        │
│     - 사용자 근처에 데이터 배치                                  │
│     - 네트워크 지연 감소                                         │
│                                                                  │
│  4. 재해 복구 (Disaster Recovery)                                │
│     - 데이터센터 장애 대비                                       │
│     - 지역 간 복제                                               │
└─────────────────────────────────────────────────────────────────┘
```

### 복제 방식

```
┌─────────────────────────────────────────────────────────────────┐
│                       복제 방식 비교                             │
│                                                                  │
│  동기 복제 (Synchronous)                                        │
│  ────────────────────────────────────────────────────────────   │
│  Client ──Write──► Primary ──Replicate──► Replica ──ACK──►     │
│                                     │                            │
│                          ◄──────────┘                            │
│                         ACK                                      │
│                                                                  │
│  장점: 강한 일관성, 데이터 손실 없음                             │
│  단점: 지연 증가, 가용성 저하 가능                               │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  비동기 복제 (Asynchronous)                                     │
│  ────────────────────────────────────────────────────────────   │
│  Client ──Write──► Primary ──ACK──► Client                      │
│                         │                                        │
│                         └──Replicate──► Replica (나중에)        │
│                                                                  │
│  장점: 낮은 지연, 높은 가용성                                    │
│  단점: 데이터 손실 가능, 일관성 지연                             │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  반동기 복제 (Semi-Synchronous)                                 │
│  ────────────────────────────────────────────────────────────   │
│  최소 1개 Replica에 복제 완료 후 ACK                             │
│                                                                  │
│  장점: 동기와 비동기의 균형                                      │
│  단점: 설정 복잡성                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Master-Slave 복제

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                  Master-Slave (Primary-Replica)                  │
│                                                                  │
│        ┌─────────────────────────────────────────────┐          │
│        │               Application                   │          │
│        └────────────────┬───────────────────────────┘          │
│                         │                                        │
│           ┌─────────────┴─────────────┐                         │
│           │        Load Balancer      │                         │
│           └─────┬───────────────┬─────┘                         │
│                 │               │                                │
│         Writes  ▼        Reads  ▼                               │
│        ┌────────────┐   ┌────────────┐                          │
│        │   Master   │   │   Proxy    │                          │
│        │  (Primary) │   │  (PgBouncer,│                          │
│        │            │   │   ProxySQL) │                          │
│        └──────┬─────┘   └──────┬─────┘                          │
│               │                │                                 │
│     Replication               │                                 │
│               │    ┌──────────┼──────────┐                      │
│               │    │          │          │                       │
│               ▼    ▼          ▼          ▼                       │
│        ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│        │ Replica 1│  │ Replica 2│  │ Replica 3│                 │
│        │ (Slave)  │  │ (Slave)  │  │ (Slave)  │                 │
│        └──────────┘  └──────────┘  └──────────┘                 │
│                                                                  │
│  쓰기: Master로만                                                │
│  읽기: Replica들에 분산                                          │
└─────────────────────────────────────────────────────────────────┘
```

### PostgreSQL 복제 설정

```bash
# postgresql.conf (Primary)
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
synchronous_commit = on
synchronous_standby_names = 'replica1'

# pg_hba.conf (Primary)
host replication repl_user 192.168.1.0/24 md5
```

```bash
# Replica 설정
# 베이스 백업 수행
pg_basebackup -h primary -D /var/lib/postgresql/data -U repl_user -P -R

# postgresql.conf (Replica)
hot_standby = on

# standby.signal 파일 생성 (자동)
```

### MySQL 복제 설정

```sql
-- Master 설정 (my.cnf)
-- server-id = 1
-- log_bin = mysql-bin
-- binlog_format = ROW

-- Master에서 복제 사용자 생성
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

SHOW MASTER STATUS;
-- +------------------+----------+
-- | File             | Position |
-- +------------------+----------+
-- | mysql-bin.000003 | 154      |
-- +------------------+----------+

-- Slave 설정 (my.cnf)
-- server-id = 2
-- relay_log = relay-bin
-- read_only = 1

-- Slave에서 복제 시작
CHANGE MASTER TO
    MASTER_HOST='master_ip',
    MASTER_USER='repl',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000003',
    MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G
```

### 페일오버 처리

```python
# 애플리케이션 레벨 페일오버
import psycopg2
from psycopg2 import OperationalError

class DatabaseConnection:
    def __init__(self, primary_host, replica_hosts):
        self.primary_host = primary_host
        self.replica_hosts = replica_hosts
        self.current_replica_index = 0

    def get_write_connection(self):
        return psycopg2.connect(host=self.primary_host, ...)

    def get_read_connection(self):
        # 라운드 로빈으로 Replica 선택
        host = self.replica_hosts[self.current_replica_index]
        self.current_replica_index = (self.current_replica_index + 1) % len(self.replica_hosts)

        try:
            return psycopg2.connect(host=host, ...)
        except OperationalError:
            # 실패 시 다음 Replica 시도
            return self.get_read_connection()
```

### 복제 지연 (Replication Lag)

```sql
-- PostgreSQL: 복제 지연 확인
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- MySQL: 복제 지연 확인
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 0
```

```python
# 복제 지연 고려한 읽기
class SmartReader:
    def read(self, query, allow_stale=True, max_lag_seconds=5):
        if allow_stale:
            return self.read_from_replica(query)
        else:
            lag = self.get_replica_lag()
            if lag <= max_lag_seconds:
                return self.read_from_replica(query)
            else:
                return self.read_from_primary(query)
```

---

## 샤딩 전략

### 샤딩 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                    샤딩 (Sharding)                               │
│                                                                  │
│  단일 DB의 한계:                                                 │
│  - 저장 용량 한계                                                │
│  - 쓰기 성능 한계                                                │
│  - 연결 수 한계                                                  │
│                                                                  │
│  해결: 데이터를 여러 DB에 분산                                   │
│                                                                  │
│        ┌───────────────────────────────────────────────┐        │
│        │                  Router                        │        │
│        └───────────┬───────────────────┬───────────────┘        │
│                    │                   │                         │
│     ┌──────────────┼───────────────────┼──────────────┐         │
│     │              │                   │              │         │
│     ▼              ▼                   ▼              ▼         │
│ ┌────────┐    ┌────────┐         ┌────────┐    ┌────────┐       │
│ │ Shard1 │    │ Shard2 │   ...   │ ShardN │    │ ShardN │       │
│ │ A-F    │    │ G-L    │         │ W-Z    │    │ W-Z    │       │
│ └────────┘    └────────┘         └────────┘    └────────┘       │
│   (Replica)    (Replica)          (Replica)    (Replica)        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Range Sharding (범위 기반)

```python
# 범위 기반 샤딩
class RangeSharding:
    def __init__(self):
        self.ranges = [
            ('A', 'F', 'shard1'),
            ('G', 'L', 'shard2'),
            ('M', 'R', 'shard3'),
            ('S', 'Z', 'shard4'),
        ]

    def get_shard(self, key):
        first_char = key[0].upper()
        for start, end, shard in self.ranges:
            if start <= first_char <= end:
                return shard
        return 'default'

# 날짜 기반 범위 샤딩
class DateRangeSharding:
    def get_shard(self, date):
        year = date.year
        month = date.month
        return f"shard_{year}_{month:02d}"
```

**장단점:**
| 장점 | 단점 |
|------|------|
| 범위 쿼리 효율적 | 데이터 불균형 (핫스팟) |
| 이해하기 쉬움 | 재분배 어려움 |
| 샤드 추가 간단 | 특정 샤드에 부하 집중 |

### Hash Sharding (해시 기반)

```python
# 해시 기반 샤딩
import hashlib

class HashSharding:
    def __init__(self, num_shards):
        self.num_shards = num_shards

    def get_shard(self, key):
        hash_value = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return f"shard_{hash_value % self.num_shards}"

# 사용
sharding = HashSharding(num_shards=4)
print(sharding.get_shard("user_12345"))  # shard_2
```

**Consistent Hashing (일관된 해싱):**

```
┌─────────────────────────────────────────────────────────────────┐
│                   Consistent Hashing                             │
│                                                                  │
│                        ┌───────────┐                             │
│                   ────►│   Node A  │◄────                        │
│                  │     └───────────┘     │                       │
│                  │           │           │                       │
│            ┌─────┴─────┐     │     ┌─────┴─────┐                │
│            │   Hash    │     │     │   Hash    │                │
│            │   Ring    │◄────┴────►│   Ring    │                │
│            └─────┬─────┘           └─────┬─────┘                │
│                  │                       │                       │
│                  │     ┌───────────┐     │                       │
│                  └────►│   Node B  │◄────┘                       │
│                        └───────────┘                             │
│                             │                                    │
│                        ┌────┴────┐                              │
│                        │  Node C │                              │
│                        └─────────┘                              │
│                                                                  │
│  장점: 노드 추가/제거 시 최소한의 키만 재배치                    │
│  가상 노드(Virtual Node)로 불균형 해소                           │
└─────────────────────────────────────────────────────────────────┘
```

```python
import hashlib
from bisect import bisect_left

class ConsistentHash:
    def __init__(self, nodes, virtual_nodes=100):
        self.ring = {}
        self.sorted_keys = []
        self.virtual_nodes = virtual_nodes

        for node in nodes:
            self.add_node(node)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            self.sorted_keys.append(key)
        self.sorted_keys.sort()

    def remove_node(self, node):
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node}:{i}")
            del self.ring[key]
            self.sorted_keys.remove(key)

    def get_node(self, key):
        if not self.ring:
            return None
        hash_key = self._hash(key)
        idx = bisect_left(self.sorted_keys, hash_key) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

    def _hash(self, key):
        return int(hashlib.md5(str(key).encode()).hexdigest(), 16)
```

### Directory Sharding (디렉토리 기반)

```python
# 디렉토리 기반 샤딩
class DirectorySharding:
    def __init__(self, db_connection):
        self.db = db_connection

    def get_shard(self, key):
        # 디렉토리 조회
        result = self.db.query(
            "SELECT shard_id FROM shard_directory WHERE key = ?",
            [key]
        )
        if result:
            return result.shard_id
        else:
            # 새 키는 라운드 로빈으로 할당
            shard = self.assign_new_key(key)
            return shard

    def assign_new_key(self, key):
        shard = self.get_least_loaded_shard()
        self.db.execute(
            "INSERT INTO shard_directory (key, shard_id) VALUES (?, ?)",
            [key, shard]
        )
        return shard
```

```sql
-- 디렉토리 테이블
CREATE TABLE shard_directory (
    key VARCHAR(255) PRIMARY KEY,
    shard_id VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_shard_id ON shard_directory(shard_id);
```

**장단점:**
| 장점 | 단점 |
|------|------|
| 완전한 유연성 | 디렉토리가 단일 실패점 |
| 어떤 매핑도 가능 | 추가 조회 오버헤드 |
| 동적 재분배 가능 | 디렉토리 확장 필요 |

### 샤딩 키 선택

```sql
-- 좋은 샤딩 키 특성:
-- 1. 높은 카디널리티 (다양한 값)
-- 2. 균등한 분포
-- 3. 자주 쿼리에 사용됨
-- 4. 불변 (변경되지 않음)

-- 예시: 사용자 기반 시스템
-- 좋은 샤딩 키: user_id (해시)
-- 나쁜 샤딩 키: country (불균등), created_at (시간 기반 핫스팟)

-- 멀티테넌트 시스템
-- 좋은 샤딩 키: tenant_id
-- 쿼리가 대부분 테넌트 내로 제한됨

-- 복합 샤딩 키
-- 예: (region, user_id)
-- 지역 내 쿼리 최적화 + 사용자별 분산
```

---

## CAP 정리

### CAP 정리 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                      CAP Theorem                                 │
│                                                                  │
│  분산 시스템은 다음 3가지 중 2가지만 보장할 수 있다:             │
│                                                                  │
│                    Consistency (C)                               │
│                         ▲                                        │
│                        /│\                                       │
│                       / │ \                                      │
│                      /  │  \                                     │
│                     /   │   \                                    │
│                    /    │    \                                   │
│                   /     │     \                                  │
│                  /      │      \                                 │
│  Availability ◄────────┼────────► Partition                     │
│      (A)               │            Tolerance (P)               │
│                        │                                         │
│                                                                  │
│  P는 필수 (네트워크 파티션은 발생함)                             │
│  따라서 실제 선택: CP vs AP                                      │
│                                                                  │
│  CP 시스템: 파티션 시 일관성 유지, 가용성 희생                   │
│  - PostgreSQL, MySQL (단일 마스터)                               │
│  - MongoDB (기본 설정)                                           │
│  - HBase, Zookeeper                                              │
│                                                                  │
│  AP 시스템: 파티션 시 가용성 유지, 일관성 희생                   │
│  - Cassandra, DynamoDB                                           │
│  - CouchDB                                                       │
│  - DNS                                                           │
└─────────────────────────────────────────────────────────────────┘
```

### PACELC 정리

CAP를 확장한 PACELC:

```
┌─────────────────────────────────────────────────────────────────┐
│                      PACELC Theorem                              │
│                                                                  │
│  Partition (P) 상황:                                             │
│    Availability (A) vs Consistency (C)                           │
│                                                                  │
│  Else (E) 정상 상황:                                             │
│    Latency (L) vs Consistency (C)                                │
│                                                                  │
│  예시:                                                           │
│  - Cassandra: PA/EL (항상 가용성과 낮은 지연 선호)               │
│  - MongoDB: PC/EC (일관성 선호)                                  │
│  - MySQL: PC/EC (일관성 선호)                                    │
│  - DynamoDB: PA/EL (가용성 선호, eventual consistency)           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 일관성 모델

### 일관성 수준

```
┌─────────────────────────────────────────────────────────────────┐
│                    일관성 수준 (약함 → 강함)                      │
│                                                                  │
│  Eventual Consistency (최종 일관성)                              │
│  ───────────────────────────────────────────────────────────    │
│  - 결국에는 모든 노드가 같은 값을 가짐                           │
│  - 시간 보장 없음                                                │
│  - 예: DNS, 이메일                                               │
│                                                                  │
│  Read Your Writes                                                │
│  ───────────────────────────────────────────────────────────    │
│  - 자신이 쓴 값은 즉시 읽을 수 있음                              │
│  - 다른 사용자의 쓰기는 지연될 수 있음                           │
│                                                                  │
│  Session Consistency                                             │
│  ───────────────────────────────────────────────────────────    │
│  - 세션 내에서 Read Your Writes 보장                             │
│  - 세션 간에는 보장 없음                                         │
│                                                                  │
│  Monotonic Read                                                  │
│  ───────────────────────────────────────────────────────────    │
│  - 한번 읽은 값보다 과거 값을 읽지 않음                          │
│                                                                  │
│  Strong Consistency (강한 일관성)                                │
│  ───────────────────────────────────────────────────────────    │
│  - 쓰기 완료 후 모든 읽기가 최신 값 반환                         │
│  - 선형화 가능성 (Linearizability)                               │
│  - 성능 오버헤드 높음                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Quorum 기반 일관성

```python
# Quorum 설정
# N = 전체 노드 수
# W = 쓰기 Quorum (응답 필요 노드 수)
# R = 읽기 Quorum (응답 필요 노드 수)

# 강한 일관성: W + R > N
# 예: N=3, W=2, R=2 → W + R = 4 > 3

# 높은 가용성 (쓰기): W=1, R=N
# 높은 가용성 (읽기): W=N, R=1

class QuorumConsistency:
    def __init__(self, nodes, write_quorum, read_quorum):
        self.nodes = nodes
        self.W = write_quorum
        self.R = read_quorum
        self.N = len(nodes)

    def write(self, key, value, version):
        successes = 0
        for node in self.nodes:
            try:
                node.write(key, value, version)
                successes += 1
            except NodeError:
                continue

        if successes >= self.W:
            return True
        else:
            raise QuorumNotReached()

    def read(self, key):
        responses = []
        for node in self.nodes:
            try:
                value, version = node.read(key)
                responses.append((value, version))
            except NodeError:
                continue

        if len(responses) >= self.R:
            # 가장 높은 버전 반환
            return max(responses, key=lambda x: x[1])
        else:
            raise QuorumNotReached()
```

### 충돌 해결

```python
# Last Write Wins (LWW)
class LWWResolver:
    def resolve(self, values):
        # 타임스탬프가 가장 최신인 값 선택
        return max(values, key=lambda x: x.timestamp)

# Vector Clock
class VectorClock:
    def __init__(self, node_id):
        self.clock = {}
        self.node_id = node_id

    def increment(self):
        self.clock[self.node_id] = self.clock.get(self.node_id, 0) + 1

    def update(self, other_clock):
        for node, time in other_clock.items():
            self.clock[node] = max(self.clock.get(node, 0), time)

    def is_concurrent(self, other):
        # 어느 쪽도 다른 쪽보다 이전이 아니면 동시 이벤트
        self_newer = any(
            self.clock.get(n, 0) > other.clock.get(n, 0)
            for n in set(self.clock) | set(other.clock)
        )
        other_newer = any(
            other.clock.get(n, 0) > self.clock.get(n, 0)
            for n in set(self.clock) | set(other.clock)
        )
        return self_newer and other_newer  # 둘 다 더 새로운 부분이 있음

# CRDT (Conflict-free Replicated Data Type)
class GCounter:  # Grow-only Counter
    def __init__(self, node_id):
        self.counts = {}
        self.node_id = node_id

    def increment(self):
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + 1

    def value(self):
        return sum(self.counts.values())

    def merge(self, other):
        for node, count in other.counts.items():
            self.counts[node] = max(self.counts.get(node, 0), count)
```

---

## 실무 아키텍처

### 읽기-쓰기 분리

```python
# Django에서 Database Router
class ReadWriteRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'primary'

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'primary'

# settings.py
DATABASES = {
    'primary': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'primary.db.example.com',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'replica.db.example.com',
    },
}

DATABASE_ROUTERS = ['myapp.routers.ReadWriteRouter']
```

### 샤딩 + 복제 조합

```
┌─────────────────────────────────────────────────────────────────┐
│              Production Architecture                             │
│                                                                  │
│      ┌─────────────────────────────────────────────────┐        │
│      │              Application Layer                   │        │
│      └─────────────────────────────────────────────────┘        │
│                              │                                   │
│      ┌─────────────────────────────────────────────────┐        │
│      │           Shard Router / Proxy                   │        │
│      │         (Vitess, ProxySQL, etc.)                │        │
│      └─────────────────────────────────────────────────┘        │
│                   │                    │                         │
│        ┌─────────┴────────┐  ┌────────┴─────────┐               │
│        │    Shard 1       │  │    Shard 2       │               │
│        │ (user_id % 2 = 0)│  │ (user_id % 2 = 1)│               │
│        │                  │  │                  │               │
│        │ ┌────────────┐   │  │ ┌────────────┐   │               │
│        │ │  Primary   │   │  │ │  Primary   │   │               │
│        │ └──────┬─────┘   │  │ └──────┬─────┘   │               │
│        │        │         │  │        │         │               │
│        │ ┌──────▼─────┐   │  │ ┌──────▼─────┐   │               │
│        │ │  Replica   │   │  │ │  Replica   │   │               │
│        │ └────────────┘   │  │ └────────────┘   │               │
│        │ ┌────────────┐   │  │ ┌────────────┐   │               │
│        │ │  Replica   │   │  │ │  Replica   │   │               │
│        │ └────────────┘   │  │ └────────────┘   │               │
│        └──────────────────┘  └──────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Shard 쿼리

```python
# Cross-shard 쿼리 처리
class CrossShardQuery:
    def __init__(self, shards):
        self.shards = shards

    def scatter_gather(self, query, params):
        """모든 샤드에 쿼리 후 결과 합침"""
        results = []
        for shard in self.shards:
            shard_results = shard.execute(query, params)
            results.extend(shard_results)
        return results

    def ordered_scatter_gather(self, query, order_by, limit):
        """정렬이 필요한 경우"""
        results = []
        for shard in self.shards:
            # 각 샤드에서 limit만큼 가져옴
            shard_results = shard.execute(query + f" ORDER BY {order_by} LIMIT {limit}")
            results.extend(shard_results)

        # 전체 결과 정렬 후 limit 적용
        results.sort(key=lambda x: getattr(x, order_by))
        return results[:limit]
```

---

## 참고 자료

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Martin Kleppmann
- [CAP Theorem Explained](https://www.bmc.com/blogs/cap-theorem/)
- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [Consistent Hashing](https://www.toptal.com/big-data/consistent-hashing)
