# Memcached

Memcached는 2003년 Brad Fitzpatrick이 LiveJournal을 위해 개발한 분산 메모리 캐시 시스템이다. 복잡한 자료구조보다 단순한 Key-Value 접근과 속도에 초점을 맞췄으며, 대규모 웹 서비스의 캐시 계층에 널리 사용된다.

## 핵심 특징

- 단순성: Key-Value 저장만 지원, 복잡한 자료구조 없음
- 멀티스레드: 여러 CPU 코어를 활용한 병렬 처리
- 비영속성: 순수 인메모리, 재시작 시 데이터 손실
- 분산 아키텍처: 클라이언트 측 샤딩으로 수평 확장
- BSD 라이선스: 완전한 오픈 소스, 상업적 사용 자유

## 라이선스

- Memcached는 BSD 3-Clause 라이선스로, 수정 및 재배포가 자유롭음. Redis의 라이선스 변경(AGPLv3) 이후 Memcached에 대한 관심이 다시 높아지고 있음.

## 아키텍처

### 전체 구조

애플리케이션은 Memcached 클라이언트 라이브러리를 통해 서버에 접근한다. 클라이언트는 Consistent Hashing으로 키를 여러 서버에 분산하고, Connection Pooling으로 연결을 관리한다.

각 Memcached 서버에서는 Worker Thread가 요청을 처리한다. 데이터는 RAM에 저장하며, Slab Allocator가 메모리 할당을 맡는다.

### 서버 내부 구조

- Memcached Server
- Main Thread (Listener)
- Accept new connections
- Distribute to worker threads
- Worker, Worker, Worker
- Thread 1, Thread 2, Thread N
- libevent, libevent, libevent
- Event Loop, Event Loop, Event Loop
- Hash Table (Key Index)
- Fast O(1) lookup
- Dynamic resizing
- Slab Allocator
- Slab 1, Slab 2, Slab 3, Slab N
- 96B, 120B, 152B, 1MB
- LRU Lists (per Slab Class)

### 기본 사용법

```python
import memcache
from pymemcache.client.base import Client
from pymemcache.client.hash import HashClient
import json

# 단일 서버 연결
client = Client(('localhost', 11211))

# 기본 CRUD
client.set('key', 'value', expire=3600)  # 1시간 만료
value = client.get('key')
client.delete('key')

# 숫자 연산 (원자적)
client.set('counter', '0')
client.incr('counter', 1)
client.decr('counter', 1)

# CAS (Compare-And-Swap) - 동시성 제어
result, cas_token = client.gets('key')
# 다른 클라이언트가 수정했는지 확인하며 업데이트
success = client.cas('key', 'new_value', cas_token)

# 멀티 키 연산
client.set_many({'key1': 'val1', 'key2': 'val2'})
values = client.get_many(['key1', 'key2'])

# JSON 직렬화
def json_serializer(key, value):
    if isinstance(value, str):
        return value, 1
    return json.dumps(value), 2

def json_deserializer(key, value, flags):
    if flags == 1:
        return value
    if flags == 2:
        return json.loads(value)
    return value

client = Client(
    ('localhost', 11211),
    serializer=json_serializer,
    deserializer=json_deserializer
)

client.set('user', {'name': 'John', 'age': 30})
user = client.get('user')  # {'name': 'John', 'age': 30}
```

### 분산 클라이언트 설정

```python
from pymemcache.client.hash import HashClient
from pymemcache.client.rendezvous import RendezvousHash

# 여러 서버에 연결 (일관된 해싱 사용)
servers = [
    ('memcache1.example.com', 11211),
    ('memcache2.example.com', 11211),
    ('memcache3.example.com', 11211),
]

# 기본 해시 클라이언트
client = HashClient(servers)

# Rendezvous 해싱 (더 균일한 분산)
client = HashClient(
    servers,
    hasher=RendezvousHash()
)

# 연결 풀링과 함께
from pymemcache.client.hash import HashClient
from pymemcache import PooledClient

client = HashClient(
    servers,
    use_pooling=True,
    max_pool_size=10
)

# 장애 대응
client = HashClient(
    servers,
    retry_attempts=2,
    retry_timeout=1,
    dead_timeout=60,  # 죽은 서버 재시도 간격
    ignore_exc=True   # 예외 무시 (캐시 미스로 처리)
)
```

## Redis와 비교

### 기능 비교

- 특성: 자료구조:
  - Memcached: String만
  - Redis: String, List, Set, Hash, ZSet, Stream 등
- 특성: 영속성:
  - Memcached: 없음
  - Redis: RDB, AOF 지원
- 특성: 복제:
  - Memcached: 없음
  - Redis: Master-Replica 복제
- 특성: 클러스터링:
  - Memcached: 클라이언트 측 샤딩
  - Redis: 서버 측 클러스터링
- 특성: 스레드 모델:
  - Memcached: 멀티스레드
  - Redis: 싱글스레드 (I/O 스레딩 옵션)
- 특성: 메모리 효율:
  - Memcached: 높음 (Slab 할당)
  - Redis: 상대적으로 낮음
- 특성: 최대 키 크기:
  - Memcached: 250 bytes
  - Redis: 512 MB
- 특성: 최대 값 크기:
  - Memcached: 1 MB (기본)
  - Redis: 512 MB
- 특성: Pub/Sub:
  - Memcached: 없음
  - Redis: 지원
- 특성: Lua 스크립팅:
  - Memcached: 없음
  - Redis: 지원
- 특성: 트랜잭션:
  - Memcached: CAS만
  - Redis: MULTI/EXEC
- 특성: 라이선스:
  - Memcached: BSD
  - Redis: AGPLv3 (8.0+) / 상용

### 성능 비교

```python
import time
import memcache
import redis

def benchmark_memcached():
    client = memcache.Client(['localhost:11211'])

    # SET 벤치마크
    start = time.time()
    for i in range(100000):
        client.set(f'key:{i}', f'value:{i}')
    set_time = time.time() - start

    # GET 벤치마크
    start = time.time()
    for i in range(100000):
        client.get(f'key:{i}')
    get_time = time.time() - start

    return set_time, get_time

def benchmark_redis():
    client = redis.Redis(host='localhost', port=6379)

    # SET 벤치마크
    start = time.time()
    for i in range(100000):
        client.set(f'key:{i}', f'value:{i}')
    set_time = time.time() - start

    # GET 벤치마크
    start = time.time()
    for i in range(100000):
        client.get(f'key:{i}')
    get_time = time.time() - start

    return set_time, get_time

def benchmark_redis_pipeline():
    """Redis 파이프라인 사용 시 성능"""
    client = redis.Redis(host='localhost', port=6379)

    # Pipeline SET
    start = time.time()
    pipe = client.pipeline()
    for i in range(100000):
        pipe.set(f'key:{i}', f'value:{i}')
    pipe.execute()
    set_time = time.time() - start

    # Pipeline GET
    start = time.time()
    pipe = client.pipeline()
    for i in range(100000):
        pipe.get(f'key:{i}')
    pipe.execute()
    get_time = time.time() - start

    return set_time, get_time

# 일반적인 결과 (환경에 따라 다름):
# Memcached: SET 5.2s, GET 4.8s
# Redis: SET 6.1s, GET 5.5s
# Redis Pipeline: SET 1.2s, GET 1.0s
```

### 멀티스레드 vs 싱글스레드

- Memcached (멀티스레드)
- Request 1, →, Thread 1
- Request 2, →, Thread 2, → Memory
- Request 3, →, Thread 3
- 장점:
- 여러 CPU 코어 활용
- 개별 요청의 지연이 다른 요청에 영향 안 줌
- 단점:
- 락 경합 가능
- 스레드 동기화 오버헤드
- Redis (싱글스레드)
- Request 1
- Request 2, →, Event Loop, → Memory
- (Single Core)
- Request 3
- 장점:
- 락 없이 원자적 연산
- 예측 가능한 지연 시간
- 구현 단순
- 단점:
- CPU 하나만 사용 (I/O 스레딩으로 일부 완화)
- 긴 작업이 다른 요청 블로킹

### 언제 어떤 것을 선택할까?

- Memcached가 적합한 경우:
- 단순한 Key-Value 캐싱만 필요
- 멀티코어 활용이 중요
- 메모리 효율이 매우 중요
- 데이터 손실이 허용됨 (순수 캐시)
- 라이선스 제약이 중요

- Redis가 적합한 경우:
- 복잡한 자료구조가 필요 (List, Set, Hash 등)
- 데이터 영속성이 필요
- Pub/Sub, Streams 같은 메시징 기능 필요
- Lua 스크립트로 복잡한 원자적 연산 필요
- 복제/클러스터링이 서버 측에서 필요

### 하이브리드 아키텍처

대규모 시스템에서는 역할을 나눠 두 저장소를 함께 사용하기도 한다. 아래 예시는 세션을 Redis에 저장하고, 읽을 때는 Memcached를 먼저 조회하는 구조다. Memcached에 값이 없으면 Redis에서 가져온 뒤 다시 캐시한다.

```python
class HybridCache:
    """Memcached + Redis 하이브리드 캐시"""

    def __init__(self, memcached_servers, redis_client):
        from pymemcache.client.hash import HashClient

        self.memcached = HashClient(memcached_servers)
        self.redis = redis_client

    def get_session(self, session_id: str) -> dict:
        """
        세션 데이터 조회
        - 1차: Memcached (빠른 읽기)
        - 2차: Redis (영속성)
        """
        # Memcached에서 먼저 조회
        cached = self.memcached.get(f"session:{session_id}")
        if cached:
            return json.loads(cached)

        # Redis에서 조회
        session = self.redis.hgetall(f"session:{session_id}")
        if session:
            # Memcached에 캐시
            self.memcached.set(
                f"session:{session_id}",
                json.dumps(session),
                expire=300  # 5분
            )
            return session

        return None

    def set_session(self, session_id: str, data: dict, ttl: int = 3600):
        """
        세션 저장
        - Redis에 영속 저장
        - Memcached에 캐시
        """
        key = f"session:{session_id}"

        # Redis에 저장 (영속성)
        pipe = self.redis.pipeline()
        pipe.hset(key, mapping=data)
        pipe.expire(key, ttl)
        pipe.execute()

        # Memcached에도 캐시
        self.memcached.set(key, json.dumps(data), expire=min(ttl, 300))

    def invalidate_session(self, session_id: str):
        """세션 무효화"""
        key = f"session:{session_id}"
        self.memcached.delete(key)
        self.redis.delete(key)
```

## 일관된 해싱 (Consistent Hashing)

### 기존 모듈러 해싱의 문제

```python
# 단순 모듈러 해싱
def simple_hash(key, num_servers):
    return hash(key) % num_servers

# 서버 3대일 때
servers = ['server1', 'server2', 'server3']
key = 'user:1001'
server_index = hash(key) % 3  # 예: 1 -> server2

# 문제: 서버가 4대로 늘어나면?
server_index = hash(key) % 4  # 예: 2 -> server3 (다른 서버!)

# 거의 모든 키가 다른 서버로 재분배됨 -> 캐시 스탬피드 발생
```

### 일관된 해싱 원리

일관된 해싱은 해시 공간을 원형으로 표현한다. 예를 들어 `0 ~ 2^32-1` 범위를 사용하면 끝에서 다시 0으로 이어지는 링이 된다. 서버 S1, S2, S3와 키 K1, K2를 모두 해시해 이 링에 배치하고, 각 키를 시계 방향으로 가장 가까운 서버에 할당한다.

서버를 추가하거나 제거해도 링 전체의 할당을 다시 계산할 필요는 없다. 변경된 서버와 인접한 구간의 키만 이동하므로, 서버 수가 달라질 때 재분배 범위를 줄일 수 있다.

### 구현

```python
import hashlib
import bisect
from typing import List, Dict, Any, Optional

class ConsistentHash:
    """일관된 해싱 구현"""

    def __init__(self, nodes: List[str] = None, replicas: int = 150):
        """
        Args:
            nodes: 서버 노드 목록
            replicas: 각 노드당 가상 노드 수 (분포 균일화)
        """
        self.replicas = replicas
        self.ring: Dict[int, str] = {}  # 해시값 -> 노드
        self.sorted_keys: List[int] = []  # 정렬된 해시값 목록
        self._nodes: set = set()

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key: str) -> int:
        """MD5 해시로 균일한 분포 생성"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """노드 추가"""
        if node in self._nodes:
            return

        self._nodes.add(node)

        # 가상 노드 추가 (replicas 개수만큼)
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = node
            bisect.insort(self.sorted_keys, hash_value)

    def remove_node(self, node: str):
        """노드 제거"""
        if node not in self._nodes:
            return

        self._nodes.discard(node)

        # 가상 노드 제거
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)

    def get_node(self, key: str) -> Optional[str]:
        """키에 해당하는 노드 반환"""
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # 해시값보다 크거나 같은 첫 번째 노드 찾기
        idx = bisect.bisect(self.sorted_keys, hash_value)

        # 링의 끝을 넘어가면 처음으로 (원형)
        if idx >= len(self.sorted_keys):
            idx = 0

        return self.ring[self.sorted_keys[idx]]

    def get_nodes(self, key: str, count: int = 1) -> List[str]:
        """키에 대한 여러 노드 반환 (복제용)"""
        if not self.ring or count <= 0:
            return []

        hash_value = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, hash_value)

        nodes = []
        seen = set()

        for i in range(len(self.sorted_keys)):
            node_idx = (idx + i) % len(self.sorted_keys)
            node = self.ring[self.sorted_keys[node_idx]]

            if node not in seen:
                nodes.append(node)
                seen.add(node)

            if len(nodes) >= count:
                break

        return nodes

# 사용 예제
def demo_consistent_hashing():
    # 초기 서버 3대
    ch = ConsistentHash(['server1', 'server2', 'server3'])

    # 키 분배 확인
    keys = [f'user:{i}' for i in range(10)]
    initial_distribution = {k: ch.get_node(k) for k in keys}
    print("초기 분배:", initial_distribution)

    # 서버 1대 추가
    ch.add_node('server4')

    # 재분배 확인
    new_distribution = {k: ch.get_node(k) for k in keys}
    print("서버 추가 후:", new_distribution)

    # 변경된 키 수 확인
    changed = sum(1 for k in keys if initial_distribution[k] != new_distribution[k])
    print(f"변경된 키: {changed}/{len(keys)} ({changed/len(keys)*100:.1f}%)")
    # 이론적으로 약 25% (1/4)의 키만 이동해야 함

# 부하 분산 테스트
def test_distribution():
    ch = ConsistentHash(['server1', 'server2', 'server3', 'server4'])

    distribution = {'server1': 0, 'server2': 0, 'server3': 0, 'server4': 0}

    for i in range(10000):
        node = ch.get_node(f'key:{i}')
        distribution[node] += 1

    print("키 분배:")
    for server, count in distribution.items():
        print(f"  {server}: {count} ({count/100:.1f}%)")
    # replicas가 충분하면 거의 균등하게 분배됨 (각 ~25%)
```

### 가상 노드 (Virtual Nodes)의 필요성

서버마다 링에 점 하나만 배치하면 담당 구간이 고르지 않을 수 있다. 예를 들어 S1이 해시 공간의 50%, S2가 20%, S3가 30%를 맡으면 S1에 부하가 몰린다.

이를 완화하기 위해 각 서버를 여러 가상 노드로 표현한다. S1을 `S1_0`, `S1_1`, `S1_2`로 나누고 S2와 S3도 같은 방식으로 링에 흩어 놓으면, 서버별 담당 구간을 더 균일하게 분산할 수 있다.

### Memcached 클라이언트에서의 일관된 해싱

```python
from pymemcache.client.hash import HashClient
from pymemcache.client.rendezvous import RendezvousHash

# 기본 Ketama 알고리즘 사용
client = HashClient([
    ('mc1.example.com', 11211),
    ('mc2.example.com', 11211),
    ('mc3.example.com', 11211),
])

# Rendezvous 해싱 (Highest Random Weight)
# - 더 균일한 분배
# - 서버 가중치 지원
client = HashClient(
    [
        ('mc1.example.com', 11211),
        ('mc2.example.com', 11211),
        ('mc3.example.com', 11211),
    ],
    hasher=RendezvousHash()
)

# 장애 대응: 죽은 서버 우회
client = HashClient(
    servers,
    dead_timeout=30,      # 30초 후 재시도
    retry_attempts=2,
    ignore_exc=True       # 예외 무시 (캐시 미스로 처리)
)
```

## Slab Allocator

### 메모리 단편화 문제

- 일반적인 메모리 할당 (malloc/free)
- 시간 경과에 따른 메모리 상태:
- 초기:
- Free Memory
- 여러 할당 후:
- 100B, Free, 500B, Free, 50B, Free
- 많은 할당/해제 후 (단편화):
- 50 F 30 80 F 100 F 40 120 F 60 50 F, Fragmented Free
- 문제: 전체 여유 공간은 충분하지만 연속된 큰 블록 할당 불가

### Slab Allocator 동작 원리

Slab Allocator는 메모리 풀을 크기별 Slab Class로 나눈다. 각 Slab은 1 MB 페이지를 고정 크기 청크로 분할하며, 예를 들어 Class 1은 96 bytes, Class 2는 120 bytes 청크를 관리한다. 요청한 데이터 크기에 맞는 클래스를 선택해 청크를 할당하는 방식이다.

### Slab Class 구조

```python
# Memcached 기본 Slab 클래스 (growth factor 1.25)
SLAB_CLASSES = [
    # (class_id, chunk_size)
    (1, 96),
    (2, 120),
    (3, 152),
    (4, 192),
    (5, 240),
    (6, 304),
    (7, 384),
    (8, 480),
    (9, 600),
    (10, 752),
    # ... 계속
    (42, 1048576),  # 1MB (최대)
]

def get_slab_class(item_size: int) -> int:
    """아이템 크기에 맞는 Slab 클래스 찾기"""
    for class_id, chunk_size in SLAB_CLASSES:
        if item_size <= chunk_size:
            return class_id, chunk_size
    return None, None  # 너무 큰 아이템

# 예시
item_size = 105  # 105 바이트 아이템
class_id, chunk_size = get_slab_class(item_size)
# class_id: 3, chunk_size: 152

# 내부 낭비: 152 - 105 = 47 바이트
# 이 낭비는 단편화 방지를 위한 트레이드오프
```

### Slab 통계 확인

```bash
# Memcached stats 명령으로 Slab 정보 확인
$ echo "stats slabs" | nc localhost 11211

STAT 1:chunk_size 96
STAT 1:chunks_per_page 10922
STAT 1:total_pages 1
STAT 1:total_chunks 10922
STAT 1:used_chunks 523
STAT 1:free_chunks 10399
STAT 1:free_chunks_end 0
STAT 2:chunk_size 120
STAT 2:chunks_per_page 8738
...
```

```python
import memcache

def analyze_slab_stats(client):
    """Slab 통계 분석"""
    stats = client.get_stats('slabs')[0][1]

    slab_info = {}
    for key, value in stats.items():
        if ':' in key:
            slab_id, metric = key.split(':')
            if slab_id.isdigit():
                slab_id = int(slab_id)
                if slab_id not in slab_info:
                    slab_info[slab_id] = {}
                slab_info[slab_id][metric] = int(value)

    print("Slab 분석:")
    print("-" * 60)

    total_memory = 0
    total_wasted = 0

    for slab_id in sorted(slab_info.keys()):
        info = slab_info[slab_id]
        chunk_size = info.get('chunk_size', 0)
        used_chunks = info.get('used_chunks', 0)
        total_chunks = info.get('total_chunks', 0)
        total_pages = info.get('total_pages', 0)

        memory_used = total_pages * 1024 * 1024  # 1MB per page
        utilization = (used_chunks / total_chunks * 100) if total_chunks > 0 else 0

        print(f"Slab {slab_id}: chunk_size={chunk_size}B, "
              f"used={used_chunks}/{total_chunks}, "
              f"utilization={utilization:.1f}%")

        total_memory += memory_used

    print("-" * 60)
    print(f"Total memory: {total_memory / 1024 / 1024:.1f} MB")

def get_item_stats(client):
    """아이템 통계 분석"""
    stats = client.get_stats('items')[0][1]

    item_info = {}
    for key, value in stats.items():
        if ':' in key:
            parts = key.split(':')
            if len(parts) >= 2 and parts[1].isdigit():
                slab_id = int(parts[1])
                metric = parts[2] if len(parts) > 2 else parts[0]
                if slab_id not in item_info:
                    item_info[slab_id] = {}
                item_info[slab_id][metric] = value

    print("\n아이템 통계:")
    print("-" * 60)

    for slab_id in sorted(item_info.keys()):
        info = item_info[slab_id]
        number = info.get('number', 0)
        evicted = info.get('evicted', 0)
        age = info.get('age', 0)

        print(f"Slab {slab_id}: items={number}, evicted={evicted}, age={age}s")
```

### Slab 리밸런싱

```bash
# Slab automove 활성화 (Memcached 1.4.11+)
$ memcached -m 1024 -o slab_reassign,slab_automove

# 수동 리밸런싱
$ echo "slabs reassign 1 2" | nc localhost 11211
# Slab 1에서 Slab 2로 메모리 페이지 이동
```

```python
class SlabManager:
    """Slab 관리 유틸리티"""

    def __init__(self, client):
        self.client = client

    def get_slab_efficiency(self):
        """Slab 효율성 분석"""
        slabs = self._get_slab_stats()

        inefficient = []
        for slab_id, info in slabs.items():
            used = info.get('used_chunks', 0)
            total = info.get('total_chunks', 1)
            utilization = used / total

            if utilization < 0.5 and info.get('total_pages', 0) > 1:
                inefficient.append({
                    'slab_id': slab_id,
                    'utilization': utilization,
                    'wasted_pages': int(info['total_pages'] * (1 - utilization))
                })

        return sorted(inefficient, key=lambda x: x['utilization'])

    def identify_hot_slabs(self):
        """자주 eviction 되는 Slab 식별"""
        items = self._get_item_stats()

        hot_slabs = []
        for slab_id, info in items.items():
            evicted = int(info.get('evicted', 0))
            if evicted > 0:
                hot_slabs.append({
                    'slab_id': slab_id,
                    'evicted': evicted,
                    'evicted_time': info.get('evicted_time', 0)
                })

        return sorted(hot_slabs, key=lambda x: x['evicted'], reverse=True)

    def suggest_rebalancing(self):
        """리밸런싱 제안"""
        inefficient = self.get_slab_efficiency()
        hot = self.identify_hot_slabs()

        if not inefficient or not hot:
            return []

        suggestions = []
        for hot_slab in hot[:3]:
            for cold_slab in inefficient[:3]:
                if hot_slab['slab_id'] != cold_slab['slab_id']:
                    suggestions.append({
                        'from': cold_slab['slab_id'],
                        'to': hot_slab['slab_id'],
                        'reason': f"Move page from underutilized slab {cold_slab['slab_id']} "
                                  f"({cold_slab['utilization']:.1%}) to hot slab {hot_slab['slab_id']} "
                                  f"({hot_slab['evicted']} evictions)"
                    })

        return suggestions
```

### LRU 관리

- Memcached는 각 Slab 클래스별로 독립적인 LRU 리스트를 유지함.

- Slab Class 1 (96B)
- LRU List:
- HEAD, TAIL
- [item1] ← → [item2] ← → [item3] ← → ... ← → [itemN]
- (newest), (oldest)
- 새 아이템: HEAD에 추가
- 접근된 아이템: HEAD로 이동
- Eviction: TAIL에서 제거
- 주의: 전역 LRU가 아님!
- Slab 클래스 A가 가득 차면 A의 오래된 아이템만 제거
- Slab 클래스 B에 여유 공간이 있어도 사용 불가
- 이로 인해 비효율 발생 가능 (Slab Calcification)

## 참고 자료

- [Memcached 공식 위키](https://github.com/memcached/memcached/wiki)
- [Memcached Protocol](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)
- [AWS ElastiCache for Memcached](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/WhatIs.html)
- [Understanding The Memcached Source Code - Slab](https://holmeshe.me/understanding-memcached-source-code-I/)
- [Understanding The Memcached Source Code - Consistent Hashing](https://holmeshe.me/understanding-memcached-source-code-X-consistent-hashing/)
- [Consistent Hashing Wikipedia](https://en.wikipedia.org/wiki/Consistent_hashing)
- [Redis vs Memcached - AWS](https://aws.amazon.com/elasticache/redis-vs-memcached/)

## 관련 학습

- [Redis](02-Redis.md)
