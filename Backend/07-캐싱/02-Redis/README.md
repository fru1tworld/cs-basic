# Redis

## 목차
1. [개요](#개요)
2. [자료구조](#자료구조)
3. [영속성 (Persistence)](#영속성-persistence)
4. [복제와 Sentinel](#복제와-sentinel)
5. [Redis Cluster](#redis-cluster)
6. [Pub/Sub](#pubsub)
7. [Lua 스크립팅](#lua-스크립팅)
---

## 개요

Redis(Remote Dictionary Server)는 오픈 소스 인메모리 데이터 구조 저장소로, 캐시, 메시지 브로커, 세션 저장소 등 다양한 용도로 사용됩니다.

### 핵심 특징
- **인메모리**: 모든 데이터를 RAM에 저장하여 초고속 읽기/쓰기
- **단일 스레드**: 이벤트 루프 기반으로 동작하여 원자적 연산 보장
- **다양한 자료구조**: String 외에 List, Set, Hash, Sorted Set 등 지원
- **영속성 옵션**: RDB 스냅샷, AOF 로그로 데이터 보존 가능
- **고가용성**: Sentinel, Cluster를 통한 장애 대응

### 라이선스 변경 (2024)
Redis 7.2까지는 BSD 라이선스였으나, Redis 8.0부터 AGPLv3 또는 상용 라이선스가 적용됩니다. 이로 인해 Valkey, Dragonfly 등 대안 프로젝트가 주목받고 있습니다.

---

## 자료구조

Redis는 단순한 Key-Value 저장소가 아닌, 다양한 자료구조를 제공하는 **데이터 구조 서버**입니다.

### 1. String

가장 기본적인 자료형으로, 최대 512MB까지 저장 가능합니다. 바이너리 세이프하여 이미지, 직렬화된 객체 등도 저장할 수 있습니다.

```bash
# 기본 사용법
SET user:1001:name "홍길동"
GET user:1001:name

# 만료 시간 설정
SETEX session:abc123 3600 "user_data"  # 1시간 후 만료
PSETEX temp:key 1500 "value"           # 1.5초 후 만료

# 조건부 설정
SETNX lock:resource "locked"  # 키가 없을 때만 설정 (분산 락)
SET lock:resource "locked" NX EX 30  # NX + 만료시간 조합

# 숫자 연산 (원자적)
INCR page:views                # 1 증가
INCRBY counter 5               # 5 증가
INCRBYFLOAT price 1.5          # 실수 증가
DECR stock:item:1001           # 1 감소

# 비트 연산
SETBIT user:1001:login 0 1     # 0번째 비트를 1로
GETBIT user:1001:login 0
BITCOUNT user:1001:login       # 1인 비트 수 카운트
```

**활용 예제: 분산 카운터**

```python
import redis
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

class RateLimiter:
    """Redis String을 활용한 Rate Limiter"""

    def __init__(self, redis_client: redis.Redis, max_requests: int, window_seconds: int):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window = window_seconds

    def is_allowed(self, user_id: str) -> bool:
        key = f"rate_limit:{user_id}"
        current = self.redis.incr(key)

        if current == 1:
            # 첫 요청이면 만료 시간 설정
            self.redis.expire(key, self.window)

        return current <= self.max_requests

    def get_remaining(self, user_id: str) -> int:
        key = f"rate_limit:{user_id}"
        current = self.redis.get(key)
        if current is None:
            return self.max_requests
        return max(0, self.max_requests - int(current))


# 데코레이터로 사용
def rate_limit(limiter: RateLimiter, user_id_func):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            user_id = user_id_func(*args, **kwargs)
            if not limiter.is_allowed(user_id):
                raise Exception("Rate limit exceeded")
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

### 2. List

양방향 연결 리스트로 구현되어 있으며, 양 끝에서의 삽입/삭제가 O(1)입니다.

```bash
# 기본 사용법
LPUSH queue:tasks "task1"      # 왼쪽에 추가
RPUSH queue:tasks "task2"      # 오른쪽에 추가
LPOP queue:tasks               # 왼쪽에서 제거
RPOP queue:tasks               # 오른쪽에서 제거

# 블로킹 연산 (메시지 큐에 적합)
BLPOP queue:tasks 30           # 30초 동안 대기하며 왼쪽에서 제거
BRPOP queue:tasks 30           # 30초 동안 대기하며 오른쪽에서 제거
BRPOPLPUSH source dest 30      # 원자적 이동

# 범위 조회
LRANGE queue:tasks 0 -1        # 전체 조회
LRANGE queue:tasks 0 9         # 처음 10개

# 인덱스 접근
LINDEX queue:tasks 0           # 첫 번째 요소
LSET queue:tasks 0 "updated"   # 첫 번째 요소 수정
LLEN queue:tasks               # 길이

# 트리밍 (최근 N개만 유지)
LTRIM latest:logs 0 99         # 최근 100개만 유지
```

**활용 예제: 작업 큐**

```python
import redis
import json
import time
from typing import Optional, Dict, Any
from dataclasses import dataclass

@dataclass
class Job:
    id: str
    type: str
    data: Dict[str, Any]

class SimpleJobQueue:
    """Redis List를 활용한 간단한 작업 큐"""

    def __init__(self, redis_client: redis.Redis, queue_name: str):
        self.redis = redis_client
        self.queue_name = f"queue:{queue_name}"
        self.processing_queue = f"queue:{queue_name}:processing"

    def enqueue(self, job: Job) -> bool:
        """작업 추가"""
        job_data = json.dumps({
            'id': job.id,
            'type': job.type,
            'data': job.data,
            'enqueued_at': time.time()
        })
        self.redis.rpush(self.queue_name, job_data)
        return True

    def dequeue(self, timeout: int = 30) -> Optional[Job]:
        """
        작업 가져오기 (안전한 방식)
        - BRPOPLPUSH로 원자적으로 processing 큐로 이동
        - 처리 실패 시 복구 가능
        """
        result = self.redis.brpoplpush(
            self.queue_name,
            self.processing_queue,
            timeout=timeout
        )

        if result:
            job_data = json.loads(result)
            return Job(
                id=job_data['id'],
                type=job_data['type'],
                data=job_data['data']
            )
        return None

    def complete(self, job: Job):
        """작업 완료 처리"""
        job_json = json.dumps({
            'id': job.id,
            'type': job.type,
            'data': job.data
        })
        self.redis.lrem(self.processing_queue, 1, job_json)

    def requeue_stuck_jobs(self, max_processing_time: int = 300):
        """처리 중 실패한 작업 재처리"""
        stuck_jobs = self.redis.lrange(self.processing_queue, 0, -1)
        for job_json in stuck_jobs:
            job_data = json.loads(job_json)
            if time.time() - job_data.get('enqueued_at', 0) > max_processing_time:
                # processing 큐에서 제거하고 원래 큐로 복귀
                self.redis.lrem(self.processing_queue, 1, job_json)
                self.redis.lpush(self.queue_name, job_json)


# 사용 예제
queue = SimpleJobQueue(redis_client, "email")
queue.enqueue(Job(id="1", type="welcome", data={"user": "john@example.com"}))

# 워커
while True:
    job = queue.dequeue(timeout=10)
    if job:
        try:
            process_job(job)
            queue.complete(job)
        except Exception as e:
            # 실패 시 별도 처리
            pass
```

### 3. Set

중복을 허용하지 않는 문자열 집합으로, 집합 연산(합집합, 교집합, 차집합)을 지원합니다.

```bash
# 기본 사용법
SADD tags:post:1001 "python" "redis" "backend"
SMEMBERS tags:post:1001        # 모든 멤버 조회
SISMEMBER tags:post:1001 "redis"  # 멤버 확인
SCARD tags:post:1001           # 멤버 수

# 집합 연산
SINTER tags:post:1001 tags:post:1002      # 교집합
SUNION tags:post:1001 tags:post:1002      # 합집합
SDIFF tags:post:1001 tags:post:1002       # 차집합

# 결과를 새 키에 저장
SINTERSTORE common:tags tags:post:1001 tags:post:1002

# 랜덤 추출
SRANDMEMBER tags:post:1001 2   # 2개 랜덤 추출 (제거 안 함)
SPOP tags:post:1001            # 1개 랜덤 추출 및 제거
```

**활용 예제: 실시간 사용자 추적**

```python
import redis
from datetime import datetime
from typing import Set

class OnlineUserTracker:
    """Redis Set을 활용한 온라인 사용자 추적"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def _get_key(self, minute: int = None) -> str:
        if minute is None:
            minute = datetime.now().minute
        return f"online:users:{datetime.now().strftime('%Y%m%d%H')}:{minute:02d}"

    def mark_online(self, user_id: str):
        """사용자 온라인 표시"""
        key = self._get_key()
        self.redis.sadd(key, user_id)
        self.redis.expire(key, 3600)  # 1시간 후 만료

    def get_online_users(self, minutes: int = 5) -> Set[str]:
        """최근 N분간 활동한 사용자"""
        keys = []
        current_minute = datetime.now().minute

        for i in range(minutes):
            minute = (current_minute - i) % 60
            keys.append(self._get_key(minute))

        # 여러 분의 데이터 합집합
        if keys:
            return self.redis.sunion(*keys)
        return set()

    def count_online_users(self, minutes: int = 5) -> int:
        """온라인 사용자 수"""
        return len(self.get_online_users(minutes))

    def get_common_users(self, *time_ranges: str) -> Set[str]:
        """여러 시간대에 모두 접속한 사용자"""
        keys = [f"online:users:{tr}" for tr in time_ranges]
        return self.redis.sinter(*keys)


# 친구 추천 기능
class FriendRecommender:
    """Set 집합 연산을 활용한 친구 추천"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def add_friend(self, user_id: str, friend_id: str):
        """친구 추가 (양방향)"""
        self.redis.sadd(f"friends:{user_id}", friend_id)
        self.redis.sadd(f"friends:{friend_id}", user_id)

    def get_friends(self, user_id: str) -> Set[str]:
        """친구 목록"""
        return self.redis.smembers(f"friends:{user_id}")

    def get_mutual_friends(self, user1: str, user2: str) -> Set[str]:
        """공통 친구"""
        return self.redis.sinter(f"friends:{user1}", f"friends:{user2}")

    def recommend_friends(self, user_id: str, limit: int = 5) -> list:
        """
        친구의 친구 중 나와 아직 친구가 아닌 사람 추천
        """
        my_friends = self.get_friends(user_id)
        candidates = {}

        for friend in my_friends:
            friend_of_friends = self.redis.sdiff(
                f"friends:{friend}",
                f"friends:{user_id}",
                user_id  # 자기 자신 제외
            )

            for candidate in friend_of_friends:
                if candidate != user_id:
                    candidates[candidate] = candidates.get(candidate, 0) + 1

        # 공통 친구가 많은 순으로 정렬
        sorted_candidates = sorted(
            candidates.items(),
            key=lambda x: x[1],
            reverse=True
        )

        return sorted_candidates[:limit]
```

### 4. Hash

필드-값 쌍의 컬렉션으로, 객체를 저장하기에 적합합니다.

```bash
# 기본 사용법
HSET user:1001 name "홍길동" email "hong@example.com" age 30
HGET user:1001 name
HMGET user:1001 name email
HGETALL user:1001

# 필드 존재 확인
HEXISTS user:1001 email

# 필드 삭제
HDEL user:1001 age

# 숫자 증감
HINCRBY user:1001 age 1
HINCRBYFLOAT user:1001 balance 100.50

# 모든 필드/값 조회
HKEYS user:1001
HVALS user:1001
HLEN user:1001

# 부분 스캔 (대용량에 적합)
HSCAN user:1001 0 COUNT 100
```

**활용 예제: 세션 관리**

```python
import redis
import json
import uuid
from datetime import datetime, timedelta
from typing import Optional, Dict, Any

class SessionManager:
    """Redis Hash를 활용한 세션 관리"""

    def __init__(self, redis_client: redis.Redis, session_ttl: int = 3600):
        self.redis = redis_client
        self.session_ttl = session_ttl

    def create_session(self, user_id: str, data: Dict[str, Any] = None) -> str:
        """새 세션 생성"""
        session_id = str(uuid.uuid4())
        session_key = f"session:{session_id}"

        session_data = {
            'user_id': user_id,
            'created_at': datetime.now().isoformat(),
            'last_accessed': datetime.now().isoformat(),
            **(data or {})
        }

        # Hash로 저장
        self.redis.hset(session_key, mapping=session_data)
        self.redis.expire(session_key, self.session_ttl)

        # 사용자별 세션 추적
        self.redis.sadd(f"user_sessions:{user_id}", session_id)

        return session_id

    def get_session(self, session_id: str) -> Optional[Dict[str, Any]]:
        """세션 조회 및 갱신"""
        session_key = f"session:{session_id}"

        session = self.redis.hgetall(session_key)
        if not session:
            return None

        # 마지막 접근 시간 갱신
        self.redis.hset(session_key, 'last_accessed', datetime.now().isoformat())
        self.redis.expire(session_key, self.session_ttl)  # TTL 갱신

        return session

    def update_session(self, session_id: str, data: Dict[str, Any]):
        """세션 데이터 업데이트"""
        session_key = f"session:{session_id}"
        if self.redis.exists(session_key):
            self.redis.hset(session_key, mapping=data)
            return True
        return False

    def destroy_session(self, session_id: str):
        """세션 삭제"""
        session_key = f"session:{session_id}"
        session = self.redis.hgetall(session_key)

        if session:
            user_id = session.get('user_id')
            self.redis.delete(session_key)
            if user_id:
                self.redis.srem(f"user_sessions:{user_id}", session_id)

    def get_user_sessions(self, user_id: str) -> list:
        """사용자의 모든 활성 세션"""
        session_ids = self.redis.smembers(f"user_sessions:{user_id}")
        sessions = []

        for sid in session_ids:
            session = self.get_session(sid)
            if session:
                sessions.append({'session_id': sid, **session})
            else:
                # 만료된 세션 정리
                self.redis.srem(f"user_sessions:{user_id}", sid)

        return sessions

    def destroy_all_user_sessions(self, user_id: str):
        """사용자의 모든 세션 삭제 (강제 로그아웃)"""
        session_ids = self.redis.smembers(f"user_sessions:{user_id}")

        for sid in session_ids:
            self.redis.delete(f"session:{sid}")

        self.redis.delete(f"user_sessions:{user_id}")
```

### 5. Sorted Set (ZSet)

각 멤버가 점수(score)를 가지며, 점수 순으로 자동 정렬됩니다.

```bash
# 기본 사용법
ZADD leaderboard 1000 "player1" 1500 "player2" 2000 "player3"
ZSCORE leaderboard "player1"
ZRANK leaderboard "player1"      # 순위 (오름차순, 0부터)
ZREVRANK leaderboard "player1"   # 순위 (내림차순)

# 범위 조회
ZRANGE leaderboard 0 9           # 상위 10명 (오름차순)
ZREVRANGE leaderboard 0 9        # 상위 10명 (내림차순)
ZREVRANGE leaderboard 0 9 WITHSCORES  # 점수 포함

# 점수 범위 조회
ZRANGEBYSCORE leaderboard 1000 2000
ZCOUNT leaderboard 1000 2000     # 범위 내 멤버 수

# 점수 증가
ZINCRBY leaderboard 100 "player1"

# 집합 연산
ZUNIONSTORE combined 2 leaderboard1 leaderboard2 WEIGHTS 1 2
ZINTERSTORE common 2 set1 set2
```

**활용 예제: 실시간 랭킹 시스템**

```python
import redis
from typing import List, Tuple, Optional
from dataclasses import dataclass

@dataclass
class RankEntry:
    user_id: str
    score: float
    rank: int

class LeaderboardService:
    """Redis Sorted Set을 활용한 리더보드"""

    def __init__(self, redis_client: redis.Redis, leaderboard_name: str):
        self.redis = redis_client
        self.key = f"leaderboard:{leaderboard_name}"

    def update_score(self, user_id: str, score: float):
        """점수 설정"""
        self.redis.zadd(self.key, {user_id: score})

    def increment_score(self, user_id: str, increment: float) -> float:
        """점수 증가"""
        return self.redis.zincrby(self.key, increment, user_id)

    def get_rank(self, user_id: str) -> Optional[int]:
        """순위 조회 (1부터 시작)"""
        rank = self.redis.zrevrank(self.key, user_id)
        return rank + 1 if rank is not None else None

    def get_score(self, user_id: str) -> Optional[float]:
        """점수 조회"""
        return self.redis.zscore(self.key, user_id)

    def get_top_n(self, n: int = 10) -> List[RankEntry]:
        """상위 N명"""
        results = self.redis.zrevrange(self.key, 0, n - 1, withscores=True)
        return [
            RankEntry(user_id=user_id, score=score, rank=rank + 1)
            for rank, (user_id, score) in enumerate(results)
        ]

    def get_around_user(self, user_id: str, range_size: int = 5) -> List[RankEntry]:
        """사용자 주변 순위"""
        rank = self.redis.zrevrank(self.key, user_id)
        if rank is None:
            return []

        start = max(0, rank - range_size)
        end = rank + range_size

        results = self.redis.zrevrange(self.key, start, end, withscores=True)
        return [
            RankEntry(user_id=uid, score=score, rank=start + i + 1)
            for i, (uid, score) in enumerate(results)
        ]

    def get_percentile(self, user_id: str) -> Optional[float]:
        """백분위 계산"""
        rank = self.redis.zrevrank(self.key, user_id)
        if rank is None:
            return None

        total = self.redis.zcard(self.key)
        return ((total - rank) / total) * 100


# 시간별 리더보드
class TimeWindowLeaderboard:
    """시간 윈도우별 리더보드 (일간, 주간, 월간)"""

    def __init__(self, redis_client: redis.Redis, name: str):
        self.redis = redis_client
        self.name = name

    def _get_keys(self) -> dict:
        from datetime import datetime
        now = datetime.now()
        return {
            'daily': f"lb:{self.name}:daily:{now.strftime('%Y%m%d')}",
            'weekly': f"lb:{self.name}:weekly:{now.strftime('%Y%W')}",
            'monthly': f"lb:{self.name}:monthly:{now.strftime('%Y%m')}",
            'alltime': f"lb:{self.name}:alltime"
        }

    def add_score(self, user_id: str, score: float):
        """모든 시간 윈도우에 점수 추가"""
        keys = self._get_keys()
        pipeline = self.redis.pipeline()

        for key in keys.values():
            pipeline.zincrby(key, score, user_id)

        # 일간, 주간, 월간은 TTL 설정
        pipeline.expire(keys['daily'], 86400 * 2)      # 2일
        pipeline.expire(keys['weekly'], 86400 * 14)    # 2주
        pipeline.expire(keys['monthly'], 86400 * 62)   # 2개월

        pipeline.execute()
```

### 6. Stream

Redis 5.0에서 도입된 로그형 자료구조로, 메시지 큐 기능을 제공합니다.

```bash
# 메시지 추가
XADD events * action "click" page "home" user_id "1001"
XADD events MAXLEN ~1000 * action "view"  # 대략 1000개 유지

# 메시지 조회
XRANGE events - +               # 전체
XRANGE events - + COUNT 10      # 10개만
XREAD STREAMS events 0          # 처음부터
XREAD BLOCK 5000 STREAMS events $  # 새 메시지 대기 (5초)

# Consumer Group
XGROUP CREATE events mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 STREAMS events >
XACK events mygroup 1609459200000-0
XPENDING events mygroup

# 정보 조회
XLEN events
XINFO GROUPS events
XINFO CONSUMERS events mygroup
```

**활용 예제: 이벤트 스트리밍**

```python
import redis
import json
import time
import threading
from typing import Dict, Any, List, Callable
from dataclasses import dataclass

@dataclass
class StreamMessage:
    id: str
    data: Dict[str, Any]

class EventStreamProducer:
    """Redis Stream 프로듀서"""

    def __init__(self, redis_client: redis.Redis, stream_name: str,
                 max_len: int = 10000):
        self.redis = redis_client
        self.stream = stream_name
        self.max_len = max_len

    def publish(self, event_type: str, data: Dict[str, Any]) -> str:
        """이벤트 발행"""
        message = {
            'type': event_type,
            'data': json.dumps(data),
            'timestamp': str(time.time())
        }

        message_id = self.redis.xadd(
            self.stream,
            message,
            maxlen=self.max_len,
            approximate=True  # 정확한 maxlen 대신 근사치 사용 (성능)
        )

        return message_id


class EventStreamConsumer:
    """Redis Stream Consumer Group 컨슈머"""

    def __init__(self, redis_client: redis.Redis, stream_name: str,
                 group_name: str, consumer_name: str):
        self.redis = redis_client
        self.stream = stream_name
        self.group = group_name
        self.consumer = consumer_name
        self._running = False

        # Consumer Group 생성 (없으면)
        try:
            self.redis.xgroup_create(
                self.stream, self.group, id='0', mkstream=True
            )
        except redis.ResponseError as e:
            if "BUSYGROUP" not in str(e):
                raise

    def process_messages(self, handler: Callable[[StreamMessage], bool],
                        batch_size: int = 10, block_ms: int = 5000):
        """메시지 처리 루프"""
        self._running = True

        while self._running:
            try:
                # 새 메시지 읽기
                messages = self.redis.xreadgroup(
                    self.group,
                    self.consumer,
                    {self.stream: '>'},
                    count=batch_size,
                    block=block_ms
                )

                if not messages:
                    continue

                for stream_name, stream_messages in messages:
                    for msg_id, msg_data in stream_messages:
                        message = StreamMessage(
                            id=msg_id,
                            data={
                                'type': msg_data.get('type', ''),
                                'data': json.loads(msg_data.get('data', '{}')),
                                'timestamp': msg_data.get('timestamp', '')
                            }
                        )

                        try:
                            success = handler(message)
                            if success:
                                # 처리 완료 확인
                                self.redis.xack(self.stream, self.group, msg_id)
                        except Exception as e:
                            print(f"Error processing message {msg_id}: {e}")
                            # 실패한 메시지는 ACK하지 않음

            except Exception as e:
                print(f"Consumer error: {e}")
                time.sleep(1)

    def stop(self):
        """컨슈머 중지"""
        self._running = False

    def claim_stale_messages(self, min_idle_time_ms: int = 60000) -> List[str]:
        """
        오래된 Pending 메시지 클레임
        - 다른 컨슈머가 처리 중 실패한 메시지 복구
        """
        pending = self.redis.xpending_range(
            self.stream,
            self.group,
            min='-',
            max='+',
            count=100
        )

        claimed_ids = []
        for entry in pending:
            if entry['time_since_delivered'] > min_idle_time_ms:
                # 오래된 메시지 클레임
                claimed = self.redis.xclaim(
                    self.stream,
                    self.group,
                    self.consumer,
                    min_idle_time_ms,
                    [entry['message_id']]
                )
                if claimed:
                    claimed_ids.extend([msg[0] for msg in claimed])

        return claimed_ids


# 사용 예제
def main():
    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

    # 프로듀서
    producer = EventStreamProducer(redis_client, "user-events")

    # 이벤트 발행
    producer.publish("user.login", {"user_id": "1001", "ip": "192.168.1.1"})
    producer.publish("user.click", {"user_id": "1001", "page": "/dashboard"})

    # 컨슈머
    consumer = EventStreamConsumer(
        redis_client,
        "user-events",
        "analytics-group",
        "consumer-1"
    )

    def handle_message(msg: StreamMessage) -> bool:
        print(f"Processing: {msg.data}")
        # 실제 처리 로직
        return True

    # 백그라운드에서 처리
    thread = threading.Thread(
        target=consumer.process_messages,
        args=(handle_message,)
    )
    thread.start()
```

---

## 영속성 (Persistence)

Redis는 인메모리 데이터베이스이지만, 데이터를 디스크에 저장하여 재시작 후에도 복구할 수 있습니다.

### RDB (Redis Database)

특정 시점의 스냅샷을 바이너리 파일로 저장합니다.

```bash
# redis.conf 설정
save 900 1      # 900초(15분) 동안 1개 이상 키 변경 시 저장
save 300 10     # 300초(5분) 동안 10개 이상 키 변경 시 저장
save 60 10000   # 60초(1분) 동안 10000개 이상 키 변경 시 저장

# 수동 저장
BGSAVE          # 백그라운드 저장 (권장)
SAVE            # 동기 저장 (서비스 중단됨)
LASTSAVE        # 마지막 저장 시간
```

**RDB 장점:**
- 컴팩트한 단일 파일 (백업에 용이)
- 빠른 복구 속도
- 자식 프로세스가 저장하므로 성능 영향 적음

**RDB 단점:**
- 마지막 스냅샷 이후 데이터 손실 가능
- 대용량 데이터셋에서 fork() 비용

### AOF (Append Only File)

모든 쓰기 명령을 로그 파일에 순차적으로 기록합니다.

```bash
# redis.conf 설정
appendonly yes
appendfilename "appendonly.aof"

# fsync 정책
appendfsync always     # 모든 쓰기마다 (가장 안전, 성능 저하)
appendfsync everysec   # 1초마다 (권장, 최대 1초 데이터 손실)
appendfsync no         # OS에 맡김 (가장 빠름, 데이터 손실 가능)

# AOF 재작성 (파일 크기 최적화)
auto-aof-rewrite-percentage 100  # 이전 크기 대비 100% 증가 시
auto-aof-rewrite-min-size 64mb   # 최소 64MB 이상일 때

# 수동 재작성
BGREWRITEAOF
```

**AOF 장점:**
- 최소 1초 이내 데이터 손실만 발생
- append-only라 파일 손상 위험 낮음
- 사람이 읽을 수 있는 형식

**AOF 단점:**
- RDB보다 파일 크기가 큼
- 복구 속도가 느릴 수 있음
- fsync 정책에 따라 성능 영향

### 하이브리드 (RDB + AOF)

Redis 4.0부터 지원하는 방식으로, 두 방법의 장점을 결합합니다.

```bash
# redis.conf
aof-use-rdb-preamble yes
```

- AOF 파일 시작 부분에 RDB 스냅샷 포함
- 이후 명령은 AOF 형식으로 추가
- 빠른 복구 + 최소 데이터 손실

### 영속성 전략 선택 가이드

| 시나리오 | 권장 설정 |
|----------|-----------|
| 캐시만 사용 (데이터 손실 OK) | 영속성 비활성화 |
| 약간의 데이터 손실 허용 | RDB만 사용 |
| 최소한의 데이터 손실 | AOF (everysec) |
| 데이터 손실 불허 | AOF (always) 또는 복제와 함께 |
| 균형 잡힌 선택 | 하이브리드 (RDB + AOF) |

```python
# 영속성 설정 확인 스크립트
import redis

def check_persistence_config(redis_client: redis.Redis):
    """Redis 영속성 설정 확인"""
    config = redis_client.config_get()
    info = redis_client.info('persistence')

    print("=== RDB 설정 ===")
    print(f"save: {config.get('save', 'disabled')}")
    print(f"dbfilename: {config.get('dbfilename')}")
    print(f"rdb_last_save_time: {info.get('rdb_last_save_time')}")
    print(f"rdb_changes_since_last_save: {info.get('rdb_changes_since_last_save')}")

    print("\n=== AOF 설정 ===")
    print(f"appendonly: {config.get('appendonly')}")
    print(f"appendfsync: {config.get('appendfsync')}")
    print(f"aof_enabled: {info.get('aof_enabled')}")
    print(f"aof_current_size: {info.get('aof_current_size', 0) / 1024 / 1024:.2f} MB")

    print("\n=== 권장사항 ===")
    if config.get('appendonly') == 'no' and config.get('save') == '':
        print("경고: 영속성이 비활성화되어 있습니다!")
    elif config.get('appendfsync') == 'no':
        print("경고: AOF fsync가 비활성화되어 데이터 손실 위험이 있습니다.")
```

---

## 복제와 Sentinel

### 기본 복제 (Replication)

Redis 복제는 리더-팔로워(Leader-Follower) 구조를 사용합니다.

```bash
# replica.conf
replicaof 192.168.1.100 6379
masterauth "password"          # 마스터에 인증이 필요한 경우
replica-read-only yes          # 복제본은 읽기 전용

# 복제 상태 확인
INFO replication

# 동적 설정
REPLICAOF 192.168.1.100 6379   # 복제 시작
REPLICAOF NO ONE               # 복제 중단 (독립 마스터로 승격)
```

**복제 동작 원리:**
1. 복제본이 마스터에 연결
2. 마스터가 BGSAVE로 RDB 생성
3. RDB를 복제본에 전송
4. 이후 명령은 실시간 스트리밍

### Redis Sentinel

Sentinel은 고가용성(HA)을 위한 모니터링 및 자동 장애 조치 시스템입니다.

```
┌─────────────────────────────────────────────────┐
│                   Sentinels                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │Sentinel1│  │Sentinel2│  │Sentinel3│         │
│  └────┬────┘  └────┬────┘  └────┬────┘         │
└───────┼────────────┼────────────┼───────────────┘
        │            │            │
        ▼            ▼            ▼
   ┌────────────────────────────────┐
   │         Redis Nodes            │
   │  ┌────────┐    ┌────────┐     │
   │  │ Master │───▶│Replica1│     │
   │  └────────┘    └────────┘     │
   │       │        ┌────────┐     │
   │       └───────▶│Replica2│     │
   │                └────────┘     │
   └────────────────────────────────┘
```

**sentinel.conf 설정:**

```bash
# 모니터링할 마스터 설정
# sentinel monitor <마스터이름> <ip> <port> <쿼럼>
sentinel monitor mymaster 192.168.1.100 6379 2

# 마스터 다운 판단 시간 (밀리초)
sentinel down-after-milliseconds mymaster 5000

# 장애 조치 타임아웃
sentinel failover-timeout mymaster 60000

# 동시에 새 마스터와 동기화할 복제본 수
sentinel parallel-syncs mymaster 1

# 인증
sentinel auth-pass mymaster "password"
```

**Sentinel 클라이언트 연결:**

```python
import redis
from redis.sentinel import Sentinel

# Sentinel 연결
sentinel = Sentinel([
    ('sentinel1.example.com', 26379),
    ('sentinel2.example.com', 26379),
    ('sentinel3.example.com', 26379)
], socket_timeout=0.1)

# 마스터 연결 (쓰기용)
master = sentinel.master_for('mymaster', socket_timeout=0.1)
master.set('key', 'value')

# 복제본 연결 (읽기용)
slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
value = slave.get('key')

# 현재 마스터 정보
master_info = sentinel.discover_master('mymaster')
print(f"Current master: {master_info}")

# 복제본 목록
slaves = sentinel.discover_slaves('mymaster')
print(f"Slaves: {slaves}")
```

**Sentinel 주요 명령어:**

```bash
# Sentinel에 연결 후 실행
SENTINEL master mymaster           # 마스터 정보
SENTINEL replicas mymaster         # 복제본 목록
SENTINEL sentinels mymaster        # 다른 Sentinel 목록
SENTINEL get-master-addr-by-name mymaster  # 마스터 주소
SENTINEL failover mymaster         # 수동 장애 조치
```

---

## Redis Cluster

대규모 데이터를 위한 수평 확장 솔루션으로, 데이터를 여러 노드에 자동 분산합니다.

### 클러스터 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                      Redis Cluster                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Node A     │  │   Node B     │  │   Node C     │      │
│  │ Slots 0-5460 │  │Slots 5461-   │  │Slots 10923-  │      │
│  │              │  │    10922     │  │    16383     │      │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │
│  │  │ Master │  │  │  │ Master │  │  │  │ Master │  │      │
│  │  └───┬────┘  │  │  └───┬────┘  │  │  └───┬────┘  │      │
│  │      │       │  │      │       │  │      │       │      │
│  │  ┌───▼────┐  │  │  ┌───▼────┐  │  │  ┌───▼────┐  │      │
│  │  │Replica │  │  │  │Replica │  │  │  │Replica │  │      │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  Total: 16384 hash slots distributed across nodes           │
└─────────────────────────────────────────────────────────────┘
```

### 해시 슬롯

Redis Cluster는 16384개의 해시 슬롯을 사용합니다.

```python
# 키의 해시 슬롯 계산
def get_slot(key: str) -> int:
    """CRC16 기반 해시 슬롯 계산"""
    # 해시 태그 처리 {tag}
    start = key.find('{')
    if start != -1:
        end = key.find('}', start + 1)
        if end != -1 and end != start + 1:
            key = key[start + 1:end]

    import binascii
    return binascii.crc_hqx(key.encode(), 0) % 16384

# 예시
print(get_slot("user:1001"))      # 5598
print(get_slot("user:1002"))      # 10410
print(get_slot("{user}:1001"))    # 해시 태그로 같은 슬롯
print(get_slot("{user}:1002"))    # 해시 태그로 같은 슬롯
```

### 클러스터 설정 및 생성

```bash
# 각 노드의 redis.conf
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes

# 클러스터 생성 (Redis 5.0+)
redis-cli --cluster create \
    192.168.1.101:7000 192.168.1.102:7000 192.168.1.103:7000 \
    192.168.1.101:7001 192.168.1.102:7001 192.168.1.103:7001 \
    --cluster-replicas 1

# 클러스터 상태 확인
redis-cli -c -p 7000 cluster info
redis-cli -c -p 7000 cluster nodes
redis-cli -c -p 7000 cluster slots
```

### 클러스터 클라이언트

```python
from redis.cluster import RedisCluster

# 클러스터 연결
rc = RedisCluster(
    host='192.168.1.101',
    port=7000,
    decode_responses=True,
    # 읽기 복제본에서 읽기
    read_from_replicas=True
)

# 기본 사용
rc.set('key', 'value')
rc.get('key')

# 해시 태그로 같은 노드에 저장
rc.set('{user:1001}:profile', 'data')
rc.set('{user:1001}:settings', 'data')

# 멀티 키 명령 (같은 슬롯에 있어야 함)
rc.mget('{user:1001}:profile', '{user:1001}:settings')

# 파이프라인 (슬롯별로 분리됨)
pipe = rc.pipeline()
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
results = pipe.execute()
```

### 클러스터 관리 명령어

```bash
# 노드 추가
redis-cli --cluster add-node 192.168.1.104:7000 192.168.1.101:7000

# 노드 제거
redis-cli --cluster del-node 192.168.1.101:7000 <node-id>

# 슬롯 재분배
redis-cli --cluster reshard 192.168.1.101:7000

# 리밸런싱
redis-cli --cluster rebalance 192.168.1.101:7000

# 장애 조치 시뮬레이션
redis-cli -c -p 7000 DEBUG SEGFAULT
redis-cli -c -p 7000 CLUSTER FAILOVER
```

### Sentinel vs Cluster 비교

| 특성 | Sentinel | Cluster |
|------|----------|---------|
| 목적 | 고가용성 (HA) | 수평 확장 + HA |
| 데이터 분산 | 없음 (전체 복제) | 자동 샤딩 |
| 최대 용량 | 단일 노드 메모리 | 노드 수 × 메모리 |
| 장애 조치 | 자동 | 자동 |
| 클라이언트 복잡도 | 낮음 | 높음 |
| 멀티 키 명령 | 제한 없음 | 같은 슬롯만 |
| 적합한 경우 | 중소규모, HA 필요 | 대규모, 확장 필요 |

---

## Pub/Sub

Redis Pub/Sub는 발행-구독 메시징 패턴을 구현합니다.

### 기본 사용법

```bash
# 구독자 (터미널 1)
SUBSCRIBE news sports

# 패턴 구독
PSUBSCRIBE news:*

# 발행자 (터미널 2)
PUBLISH news "Breaking news!"
PUBLISH sports "Game score update"
PUBLISH news:tech "New product launch"

# 채널 정보
PUBSUB CHANNELS           # 활성 채널 목록
PUBSUB NUMSUB news        # 구독자 수
```

### 코드 예제

```python
import redis
import threading
import time
from typing import Callable, Dict

class PubSubManager:
    """Redis Pub/Sub 관리자"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.pubsub = redis_client.pubsub()
        self._handlers: Dict[str, Callable] = {}
        self._running = False

    def subscribe(self, channel: str, handler: Callable):
        """채널 구독 및 핸들러 등록"""
        self._handlers[channel] = handler
        self.pubsub.subscribe(channel)

    def psubscribe(self, pattern: str, handler: Callable):
        """패턴 구독"""
        self._handlers[pattern] = handler
        self.pubsub.psubscribe(pattern)

    def publish(self, channel: str, message: str) -> int:
        """메시지 발행"""
        return self.redis.publish(channel, message)

    def start_listening(self):
        """백그라운드에서 메시지 수신"""
        self._running = True

        def listen():
            for message in self.pubsub.listen():
                if not self._running:
                    break

                if message['type'] in ('message', 'pmessage'):
                    channel = message.get('channel', message.get('pattern'))
                    handler = self._handlers.get(channel)

                    if handler:
                        try:
                            handler(message)
                        except Exception as e:
                            print(f"Handler error: {e}")

        thread = threading.Thread(target=listen, daemon=True)
        thread.start()
        return thread

    def stop(self):
        """수신 중지"""
        self._running = False
        self.pubsub.close()


# 실시간 알림 시스템 예제
class NotificationService:
    """Redis Pub/Sub 기반 실시간 알림"""

    def __init__(self, redis_client: redis.Redis):
        self.pubsub_manager = PubSubManager(redis_client)
        self._subscribers: Dict[str, list] = {}

    def notify_user(self, user_id: str, notification: dict):
        """특정 사용자에게 알림"""
        import json
        channel = f"notifications:{user_id}"
        self.pubsub_manager.publish(channel, json.dumps(notification))

    def broadcast(self, notification: dict):
        """모든 사용자에게 브로드캐스트"""
        import json
        self.pubsub_manager.publish("notifications:broadcast", json.dumps(notification))

    def subscribe_user(self, user_id: str, callback: Callable):
        """사용자 알림 구독"""
        channel = f"notifications:{user_id}"
        self.pubsub_manager.subscribe(channel, callback)

    def start(self):
        """알림 서비스 시작"""
        # 브로드캐스트 채널 구독
        self.pubsub_manager.psubscribe("notifications:*", self._handle_notification)
        return self.pubsub_manager.start_listening()

    def _handle_notification(self, message):
        """알림 처리"""
        import json
        channel = message['channel']
        data = json.loads(message['data'])
        print(f"Notification on {channel}: {data}")


# 채팅 시스템 예제
class ChatRoom:
    """Redis Pub/Sub 기반 채팅"""

    def __init__(self, redis_client: redis.Redis, room_id: str):
        self.redis = redis_client
        self.room_id = room_id
        self.channel = f"chat:room:{room_id}"
        self.pubsub = redis_client.pubsub()

    def send_message(self, user_id: str, message: str):
        """메시지 전송"""
        import json
        payload = json.dumps({
            'user_id': user_id,
            'message': message,
            'timestamp': time.time()
        })
        self.redis.publish(self.channel, payload)

        # 최근 메시지 히스토리 저장 (List 활용)
        self.redis.lpush(f"{self.channel}:history", payload)
        self.redis.ltrim(f"{self.channel}:history", 0, 99)  # 최근 100개만

    def get_history(self, count: int = 50) -> list:
        """최근 메시지 히스토리"""
        import json
        messages = self.redis.lrange(f"{self.channel}:history", 0, count - 1)
        return [json.loads(msg) for msg in messages]

    def join(self, message_handler: Callable):
        """채팅방 참여"""
        self.pubsub.subscribe(**{self.channel: message_handler})
        return self.pubsub.run_in_thread(sleep_time=0.001)

    def leave(self):
        """채팅방 나가기"""
        self.pubsub.unsubscribe(self.channel)
```

### Pub/Sub 한계점

- **메시지 영속성 없음**: 구독자가 없으면 메시지 손실
- **메시지 확인 없음**: 전달 보장 없음
- **재연결 시 메시지 손실**: 네트워크 끊김 시 놓친 메시지 복구 불가

> **참고**: 신뢰성이 필요한 경우 Redis Streams 사용 권장

---

## Lua 스크립팅

Lua 스크립트를 사용하면 여러 명령을 원자적으로 실행할 수 있습니다.

### 기본 사용법

```bash
# EVAL: 스크립트 직접 실행
# EVAL script numkeys key [key ...] arg [arg ...]
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# 여러 명령 조합
EVAL "redis.call('SET', KEYS[1], ARGV[1]); return redis.call('GET', KEYS[1])" 1 mykey "value"

# 조건부 로직
EVAL "
local value = redis.call('GET', KEYS[1])
if value then
    return value
else
    redis.call('SET', KEYS[1], ARGV[1])
    return ARGV[1]
end
" 1 mykey "default"

# SCRIPT LOAD & EVALSHA (성능 최적화)
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# Returns: "sha1_hash"
EVALSHA sha1_hash 1 mykey
```

### 실용적인 Lua 스크립트 예제

```python
import redis
import hashlib

class LuaScripts:
    """자주 사용되는 Lua 스크립트 모음"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self._scripts = {}

    def _register_script(self, name: str, script: str):
        """스크립트 등록 및 캐시"""
        sha = self.redis.script_load(script)
        self._scripts[name] = {
            'sha': sha,
            'script': script
        }
        return sha

    def _run_script(self, name: str, keys: list = None, args: list = None):
        """스크립트 실행"""
        script_info = self._scripts[name]
        try:
            return self.redis.evalsha(
                script_info['sha'],
                len(keys or []),
                *(keys or []),
                *(args or [])
            )
        except redis.exceptions.NoScriptError:
            # 스크립트가 캐시에서 사라진 경우 재등록
            self._register_script(name, script_info['script'])
            return self._run_script(name, keys, args)


# 분산 락 (Redlock 알고리즘 간소화 버전)
class DistributedLock:
    """Lua 스크립트 기반 분산 락"""

    ACQUIRE_SCRIPT = """
    if redis.call('SET', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
        return 1
    else
        return 0
    end
    """

    RELEASE_SCRIPT = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
    """

    EXTEND_SCRIPT = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('PEXPIRE', KEYS[1], ARGV[2])
    else
        return 0
    end
    """

    def __init__(self, redis_client: redis.Redis, resource: str,
                 ttl_ms: int = 10000):
        self.redis = redis_client
        self.resource = f"lock:{resource}"
        self.ttl_ms = ttl_ms
        self._token = None

        # 스크립트 등록
        self._acquire_sha = redis_client.script_load(self.ACQUIRE_SCRIPT)
        self._release_sha = redis_client.script_load(self.RELEASE_SCRIPT)
        self._extend_sha = redis_client.script_load(self.EXTEND_SCRIPT)

    def acquire(self, blocking: bool = True, timeout: float = None) -> bool:
        """락 획득"""
        import uuid
        import time

        self._token = str(uuid.uuid4())
        start_time = time.time()

        while True:
            result = self.redis.evalsha(
                self._acquire_sha,
                1,
                self.resource,
                self._token,
                self.ttl_ms
            )

            if result == 1:
                return True

            if not blocking:
                return False

            if timeout and (time.time() - start_time) >= timeout:
                return False

            time.sleep(0.01)

    def release(self) -> bool:
        """락 해제"""
        if not self._token:
            return False

        result = self.redis.evalsha(
            self._release_sha,
            1,
            self.resource,
            self._token
        )
        self._token = None
        return result == 1

    def extend(self, additional_ms: int = None) -> bool:
        """락 연장"""
        if not self._token:
            return False

        ttl = additional_ms or self.ttl_ms
        result = self.redis.evalsha(
            self._extend_sha,
            1,
            self.resource,
            self._token,
            ttl
        )
        return result == 1

    def __enter__(self):
        if not self.acquire():
            raise Exception("Failed to acquire lock")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()


# Rate Limiter (슬라이딩 윈도우)
class SlidingWindowRateLimiter:
    """Lua 스크립트 기반 슬라이딩 윈도우 Rate Limiter"""

    SCRIPT = """
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local limit = tonumber(ARGV[3])

    -- 윈도우 밖의 오래된 항목 제거
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

    -- 현재 윈도우 내 요청 수
    local count = redis.call('ZCARD', key)

    if count < limit then
        -- 새 요청 추가
        redis.call('ZADD', key, now, now .. ':' .. math.random())
        redis.call('EXPIRE', key, window / 1000 + 1)
        return {1, limit - count - 1}  -- 허용, 남은 횟수
    else
        return {0, 0}  -- 거부
    end
    """

    def __init__(self, redis_client: redis.Redis, limit: int, window_ms: int):
        self.redis = redis_client
        self.limit = limit
        self.window_ms = window_ms
        self._script_sha = redis_client.script_load(self.SCRIPT)

    def is_allowed(self, identifier: str) -> tuple:
        """요청 허용 여부 확인"""
        import time
        key = f"ratelimit:{identifier}"
        now = int(time.time() * 1000)

        result = self.redis.evalsha(
            self._script_sha,
            1,
            key,
            now,
            self.window_ms,
            self.limit
        )

        return bool(result[0]), result[1]


# 원자적 증감 with 한계값
class BoundedCounter:
    """최소/최대값이 있는 원자적 카운터"""

    INCREMENT_SCRIPT = """
    local current = tonumber(redis.call('GET', KEYS[1]) or ARGV[3])
    local increment = tonumber(ARGV[1])
    local max_value = tonumber(ARGV[2])

    local new_value = current + increment

    if new_value > max_value then
        return {0, current}  -- 실패, 현재값
    end

    redis.call('SET', KEYS[1], new_value)
    return {1, new_value}  -- 성공, 새 값
    """

    def __init__(self, redis_client: redis.Redis, key: str,
                 min_value: int = 0, max_value: int = 100):
        self.redis = redis_client
        self.key = key
        self.min_value = min_value
        self.max_value = max_value
        self._incr_sha = redis_client.script_load(self.INCREMENT_SCRIPT)

    def increment(self, amount: int = 1) -> tuple:
        """증가 (최대값 초과 방지)"""
        result = self.redis.evalsha(
            self._incr_sha,
            1,
            self.key,
            amount,
            self.max_value,
            self.min_value
        )
        return bool(result[0]), result[1]


# 사용 예제
def main():
    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

    # 분산 락
    lock = DistributedLock(redis_client, "my-resource")
    with lock:
        print("Critical section with lock")

    # Rate Limiter
    limiter = SlidingWindowRateLimiter(redis_client, limit=10, window_ms=60000)
    allowed, remaining = limiter.is_allowed("user:1001")
    print(f"Allowed: {allowed}, Remaining: {remaining}")

    # 재고 관리 (최대값 제한)
    inventory = BoundedCounter(redis_client, "stock:item:1001", min_value=0, max_value=100)
    success, current = inventory.increment(-1)  # 1개 감소
    print(f"Decrement success: {success}, Current stock: {current}")
```

### Lua 스크립트 베스트 프랙티스

1. **짧게 유지**: 스크립트 실행 중 다른 명령 블로킹
2. **KEYS와 ARGV 구분**: 클러스터 호환성을 위해 모든 키는 KEYS로 전달
3. **멱등성 고려**: 스크립트가 재실행되어도 같은 결과 보장
4. **EVALSHA 사용**: 네트워크 대역폭 절약
5. **시간 함수 사용 금지**: `os.time()` 대신 ARGV로 시간 전달 (복제 일관성)

---

## 참고 자료

- [Redis 공식 문서](https://redis.io/docs/)
- [Redis Data Types](https://redis.io/docs/latest/develop/data-types/)
- [Redis Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
- [Redis Cluster Tutorial](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis Lua Scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
