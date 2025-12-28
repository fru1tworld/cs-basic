# Redis 심화

## 목차
1. [Redis 개요](#redis-개요)
2. [자료구조](#자료구조)
3. [영속성 (Persistence)](#영속성-persistence)
4. [복제와 고가용성](#복제와-고가용성)
5. [클러스터](#클러스터)
6. [성능 최적화](#성능-최적화)
---

## Redis 개요

### Redis란?

Redis(Remote Dictionary Server)는 인메모리 데이터 구조 저장소입니다. 캐시, 메시지 브로커, 세션 스토어 등 다양한 용도로 사용됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Redis 아키텍처                            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Single Thread                         │    │
│  │                   Event Loop (I/O)                       │    │
│  │              (6.0+: I/O Multi-threading)                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     In-Memory Data                       │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │    │
│  │  │ Strings │ │  Lists  │ │  Hashes │ │  Sets   │        │    │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘        │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐                    │    │
│  │  │Sorted   │ │ Streams │ │HyperLog │                    │    │
│  │  │  Sets   │ │         │ │  Log    │                    │    │
│  │  └─────────┘ └─────────┘ └─────────┘                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      Persistence                         │    │
│  │              RDB (Snapshots) + AOF (Logs)               │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 특징

- **인메모리**: 모든 데이터를 메모리에 저장, 매우 빠른 응답
- **싱글 스레드**: 명령어 순차 처리, 원자성 보장
- **다양한 자료구조**: String, List, Set, Hash, Sorted Set 등
- **Pub/Sub**: 메시지 브로커 기능
- **Lua 스크립팅**: 복잡한 연산의 원자적 실행

---

## 자료구조

### String

가장 기본적인 자료형. 최대 512MB.

```bash
# 기본 명령어
SET user:1:name "Kim"
GET user:1:name

# 숫자 연산 (Atomic)
SET counter 0
INCR counter          # 1
INCRBY counter 10     # 11
DECR counter          # 10

# 만료 시간 설정
SET session:abc123 "user_data" EX 3600  # 1시간
SETEX session:abc123 3600 "user_data"   # 동일

# 조건부 설정
SETNX lock:resource "locked"  # 키가 없을 때만 설정 (분산 락)
SET lock:resource "locked" NX EX 30  # NX + 만료 시간

# 비트 연산 (일별 활성 사용자 추적)
SETBIT user:active:2024-01-01 1001 1  # 사용자 1001 활성
SETBIT user:active:2024-01-01 1002 1
BITCOUNT user:active:2024-01-01        # 활성 사용자 수
```

### List

양방향 연결 리스트. 메시지 큐로 활용.

```bash
# 기본 명령어
LPUSH queue:tasks "task1"     # 왼쪽 삽입
RPUSH queue:tasks "task2"     # 오른쪽 삽입
LPOP queue:tasks              # 왼쪽에서 꺼내기
RPOP queue:tasks              # 오른쪽에서 꺼내기

# 범위 조회
LRANGE queue:tasks 0 -1       # 전체 조회
LRANGE queue:tasks 0 9        # 처음 10개

# 블로킹 (메시지 큐)
BLPOP queue:tasks 30          # 30초 대기 후 꺼내기
BRPOP queue:tasks 30

# 최근 N개 유지 (타임라인)
LPUSH timeline:user1 "post1"
LTRIM timeline:user1 0 99     # 최근 100개만 유지

# Producer-Consumer 패턴
# Producer:
LPUSH work:queue '{"task": "send_email", "to": "user@example.com"}'

# Consumer:
BRPOP work:queue 0  # 무한 대기
```

### Hash

필드-값 쌍의 집합. 객체 저장에 적합.

```bash
# 사용자 정보 저장
HSET user:1001 name "Kim" email "kim@example.com" age 30
HGET user:1001 name
HGETALL user:1001

# 개별 필드 조작
HINCRBY user:1001 age 1       # 나이 증가
HSETNX user:1001 created_at "2024-01-01"  # 필드가 없을 때만

# 다중 필드
HMSET user:1001 city "Seoul" country "Korea"
HMGET user:1001 name email

# 필드 존재 확인
HEXISTS user:1001 email
HDEL user:1001 age

# 메모리 효율적 (작은 Hash는 ziplist로 인코딩)
# hash-max-ziplist-entries 512
# hash-max-ziplist-value 64
```

### Set

중복 없는 문자열 집합.

```bash
# 태그 시스템
SADD post:1:tags "redis" "database" "cache"
SMEMBERS post:1:tags

# 팔로우/팔로워
SADD user:1:following 2 3 4
SADD user:2:followers 1

# 집합 연산
SINTER user:1:following user:2:following    # 공통 팔로잉 (교집합)
SUNION user:1:following user:2:following    # 합집합
SDIFF user:1:following user:2:following     # 차집합

# 랜덤 추출 (추천 시스템)
SRANDMEMBER users:active 5    # 랜덤 5명

# 중복 체크
SISMEMBER post:1:tags "redis"  # O(1)
SCARD post:1:tags              # 요소 개수
```

### Sorted Set (ZSet)

점수로 정렬되는 Set. 랭킹 시스템에 최적.

```bash
# 게임 리더보드
ZADD leaderboard 1000 "player1"
ZADD leaderboard 2000 "player2"
ZADD leaderboard 1500 "player3"

# 랭킹 조회
ZREVRANGE leaderboard 0 9 WITHSCORES     # Top 10
ZREVRANK leaderboard "player1"            # 순위 (0-based)
ZSCORE leaderboard "player1"              # 점수

# 점수 증가
ZINCRBY leaderboard 500 "player1"

# 범위 조회
ZRANGEBYSCORE leaderboard 1000 2000      # 점수 범위
ZCOUNT leaderboard 1000 2000              # 범위 내 개수

# 실시간 인기 검색어 (시간 가중치)
ZINCRBY search:trending 1 "redis"
# 주기적으로 점수 감소
ZINTERSTORE search:trending 1 search:trending WEIGHTS 0.9

# 시간 기반 정렬 (타임라인)
ZADD timeline:user1 1704067200 "post1"    # Unix timestamp
ZREVRANGEBYSCORE timeline:user1 +inf -inf LIMIT 0 10
```

### Stream

로그 형태의 데이터 구조. 이벤트 소싱, 메시지 큐에 적합.

```bash
# 이벤트 추가
XADD events * type "click" user_id 1001 page "/home"
# 1704067200000-0 (자동 생성 ID)

# 스트림 읽기
XREAD STREAMS events 0-0           # 처음부터
XREAD BLOCK 5000 STREAMS events $  # 새 이벤트 대기

# 소비자 그룹
XGROUP CREATE events mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 STREAMS events >

# 처리 완료 확인
XACK events mygroup 1704067200000-0

# 미처리 이벤트 확인
XPENDING events mygroup

# 범위 조회
XRANGE events 1704067200000-0 1704067300000-0 COUNT 100
```

### HyperLogLog

확률적 카디널리티 추정. 고유 방문자 수 등에 사용.

```bash
# 일별 고유 방문자
PFADD visitors:2024-01-01 "user1" "user2" "user3"
PFADD visitors:2024-01-01 "user1"  # 중복 무시

PFCOUNT visitors:2024-01-01        # 약 3 (오차 0.81%)

# 여러 기간 합치기
PFMERGE visitors:week1 visitors:2024-01-01 visitors:2024-01-02

# 메모리: 최대 12KB로 수십억 개 추정 가능
```

---

## 영속성 (Persistence)

### RDB (Redis Database)

특정 시점의 스냅샷을 저장.

```bash
# redis.conf 설정
save 900 1      # 900초 내 1개 이상 변경 시
save 300 10     # 300초 내 10개 이상 변경 시
save 60 10000   # 60초 내 10000개 이상 변경 시

rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# 수동 스냅샷
BGSAVE          # 백그라운드 저장
SAVE            # 블로킹 저장 (사용 자제)
LASTSAVE        # 마지막 저장 시간
```

**RDB 장단점:**
| 장점 | 단점 |
|------|------|
| 컴팩트한 바이너리 파일 | 데이터 손실 가능 (스냅샷 간격) |
| 빠른 복구 | fork() 시 메모리 사용량 증가 |
| 재해 복구에 적합 | 대용량 데이터 시 저장 시간 증가 |

### AOF (Append Only File)

모든 쓰기 명령을 로그로 기록.

```bash
# redis.conf 설정
appendonly yes
appendfilename "appendonly.aof"

# fsync 정책
appendfsync always    # 매 명령마다 (가장 안전, 가장 느림)
appendfsync everysec  # 1초마다 (권장)
appendfsync no        # OS에 위임 (가장 빠름, 위험)

# AOF 재작성 (파일 크기 최적화)
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 수동 재작성
BGREWRITEAOF
```

**AOF 장단점:**
| 장점 | 단점 |
|------|------|
| 거의 데이터 손실 없음 | RDB보다 큰 파일 크기 |
| 사람이 읽을 수 있는 형식 | 복구 속도 느림 |
| 재작성으로 최적화 가능 | 쓰기 성능 약간 저하 |

### 권장 설정

```bash
# 프로덕션 권장: RDB + AOF 함께 사용
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec

# 복구 시 AOF가 우선됨 (더 완전한 데이터)
```

---

## 복제와 고가용성

### Master-Replica 복제

```bash
# Replica 설정
replicaof 192.168.1.100 6379
masterauth <master-password>

# 복제 상태 확인
INFO replication

# 복제 구조
#       Master (Read/Write)
#          │
#    ┌─────┴─────┐
#    ▼           ▼
# Replica 1   Replica 2  (Read Only)
```

### Sentinel (고가용성)

```bash
# sentinel.conf
sentinel monitor mymaster 192.168.1.100 6379 2
sentinel auth-pass mymaster <password>
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1

# 동작 방식
#                    ┌──────────────┐
#                    │   Sentinel   │
#                    │   Cluster    │
#                    │  (3+ nodes)  │
#                    └──────┬───────┘
#                           │ 모니터링
#            ┌──────────────┼──────────────┐
#            ▼              ▼              ▼
#       ┌────────┐    ┌────────┐    ┌────────┐
#       │ Master │───►│Replica1│───►│Replica2│
#       └────────┘    └────────┘    └────────┘
#
# Master 장애 시:
# 1. Sentinel이 장애 감지
# 2. Quorum(과반수) 합의
# 3. Replica 중 하나를 Master로 승격
# 4. 다른 Replica들이 새 Master에 연결
```

```python
# Python 클라이언트 (redis-py)
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel1', 26379),
    ('sentinel2', 26379),
    ('sentinel3', 26379)
], socket_timeout=0.1)

# Master 연결
master = sentinel.master_for('mymaster', socket_timeout=0.1)
master.set('key', 'value')

# Replica 연결 (읽기)
replica = sentinel.slave_for('mymaster', socket_timeout=0.1)
value = replica.get('key')
```

---

## 클러스터

### Redis Cluster 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                      Redis Cluster                               │
│                                                                  │
│  Hash Slots: 0 ─────────────────────────────────────── 16383    │
│              │                                              │    │
│              │          16384 Hash Slots                    │    │
│              │                                              │    │
│  ┌───────────┴───────────┬───────────────────┬─────────────┴─┐  │
│  │     Slots 0-5460      │  Slots 5461-10922 │Slots 10923-16383│  │
│  │                       │                   │                │  │
│  │  ┌─────────────────┐  │ ┌───────────────┐ │┌─────────────┐ │  │
│  │  │ Master A        │  │ │ Master B      │ ││ Master C    │ │  │
│  │  │ (Primary)       │  │ │ (Primary)     │ ││ (Primary)   │ │  │
│  │  └────────┬────────┘  │ └───────┬───────┘ │└──────┬──────┘ │  │
│  │           │           │         │         │       │        │  │
│  │  ┌────────▼────────┐  │ ┌───────▼───────┐ │┌──────▼──────┐ │  │
│  │  │ Replica A1      │  │ │ Replica B1    │ ││ Replica C1  │ │  │
│  │  └─────────────────┘  │ └───────────────┘ │└─────────────┘ │  │
│  └───────────────────────┴───────────────────┴────────────────┘  │
│                                                                  │
│  Key → CRC16(key) mod 16384 → Slot → Node                       │
└─────────────────────────────────────────────────────────────────┘
```

### 클러스터 설정

```bash
# redis.conf
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000

# 클러스터 생성 (6노드: 3 Master + 3 Replica)
redis-cli --cluster create \
    192.168.1.1:6379 192.168.1.2:6379 192.168.1.3:6379 \
    192.168.1.4:6379 192.168.1.5:6379 192.168.1.6:6379 \
    --cluster-replicas 1

# 클러스터 상태 확인
redis-cli -c cluster info
redis-cli -c cluster nodes
redis-cli -c cluster slots

# 리샤딩
redis-cli --cluster reshard 192.168.1.1:6379

# 노드 추가
redis-cli --cluster add-node 192.168.1.7:6379 192.168.1.1:6379
```

### 클러스터 클라이언트

```python
from redis.cluster import RedisCluster

# 클러스터 연결
rc = RedisCluster(
    host='192.168.1.1',
    port=6379,
    decode_responses=True
)

# 기본 명령어
rc.set('key', 'value')
rc.get('key')

# Hash Tag로 같은 슬롯에 저장
rc.set('{user:1}:name', 'Kim')
rc.set('{user:1}:email', 'kim@example.com')
# MGET 가능 (같은 슬롯)
rc.mget('{user:1}:name', '{user:1}:email')

# 파이프라인 (같은 슬롯 키만)
pipe = rc.pipeline()
pipe.set('{user:1}:age', 30)
pipe.incr('{user:1}:visit_count')
pipe.execute()
```

---

## 성능 최적화

### 메모리 최적화

```bash
# 메모리 정책 설정
maxmemory 4gb
maxmemory-policy allkeys-lru  # 가장 적게 사용된 키 삭제

# 정책 옵션:
# noeviction: 메모리 초과 시 에러
# allkeys-lru: 모든 키 중 LRU 삭제
# volatile-lru: TTL 설정된 키 중 LRU 삭제
# allkeys-random: 모든 키 중 랜덤 삭제
# volatile-ttl: TTL이 짧은 키 우선 삭제

# 메모리 사용량 분석
INFO memory
MEMORY USAGE key
MEMORY DOCTOR

# 작은 데이터 구조 최적화
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
set-max-intset-entries 512
zset-max-ziplist-entries 128
```

### 성능 모니터링

```bash
# 슬로우 로그
slowlog-log-slower-than 10000  # 10ms 이상
slowlog-max-len 128

SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET

# 실시간 명령어 모니터링
MONITOR  # 프로덕션에서 사용 자제 (성능 영향)

# 통계
INFO stats
INFO commandstats

# 클라이언트 연결
CLIENT LIST
CLIENT KILL ID <client-id>

# 메모리 상세 분석
redis-cli --memkeys

# 빅 키 찾기
redis-cli --bigkeys
```

### 파이프라인과 트랜잭션

```python
import redis

r = redis.Redis()

# 파이프라인 (네트워크 RTT 절감)
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()  # 한 번에 전송

# 트랜잭션 (MULTI/EXEC)
pipe = r.pipeline(transaction=True)
pipe.watch('balance:1', 'balance:2')
try:
    balance1 = int(r.get('balance:1'))
    balance2 = int(r.get('balance:2'))

    pipe.multi()
    pipe.set('balance:1', balance1 - 100)
    pipe.set('balance:2', balance2 + 100)
    pipe.execute()
except redis.WatchError:
    # 다른 클라이언트가 값을 변경함
    pass
```

### Lua 스크립팅

```bash
# 원자적 복잡 연산
EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 mykey myvalue

# Rate Limiter 예시
EVAL "
local current = redis.call('incr', KEYS[1])
if current == 1 then
    redis.call('expire', KEYS[1], ARGV[1])
end
if current > tonumber(ARGV[2]) then
    return 0
end
return 1
" 1 rate:user:123 60 100
# KEYS[1] = rate:user:123
# ARGV[1] = 60 (초)
# ARGV[2] = 100 (제한)

# 스크립트 캐싱
SCRIPT LOAD "return redis.call('get', KEYS[1])"
# "a5260dd66ce02462..."
EVALSHA a5260dd66ce02462... 1 mykey
```

---

## 참고 자료

- [Redis 공식 문서](https://redis.io/docs/)
- [Redis Cluster Specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- Designing Data-Intensive Applications by Martin Kleppmann
