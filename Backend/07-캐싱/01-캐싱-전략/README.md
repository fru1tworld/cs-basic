# 캐싱 전략 (Caching Strategies)

## 목차
1. [개요](#개요)
2. [Cache-Aside (Lazy Loading)](#cache-aside-lazy-loading)
3. [Read-Through](#read-through)
4. [Write-Through](#write-through)
5. [Write-Behind (Write-Back)](#write-behind-write-back)
6. [캐시 무효화 전략](#캐시-무효화-전략)
7. [캐시 히트율과 미스율](#캐시-히트율과-미스율)
---

## 개요

캐싱은 자주 접근하는 데이터를 빠른 저장소에 임시로 저장하여 시스템 성능을 향상시키는 기술입니다. 적절한 캐싱 전략을 선택하는 것은 백엔드 시스템의 성능과 데이터 일관성에 큰 영향을 미칩니다.

### 캐싱 전략 선택 시 고려사항
- **읽기/쓰기 비율**: 읽기가 많은지, 쓰기가 많은지
- **데이터 일관성 요구사항**: 강한 일관성 vs 최종적 일관성
- **지연 시간 허용 범위**: 실시간성이 중요한지
- **데이터 업데이트 빈도**: 자주 변경되는 데이터인지

---

## Cache-Aside (Lazy Loading)

### 개념

Cache-Aside는 가장 널리 사용되는 캐싱 패턴으로, 애플리케이션이 직접 캐시와 데이터베이스를 관리합니다.

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │
   ┌───┴───┐
   │       │
   ▼       ▼
┌──────┐ ┌──────┐
│Cache │ │  DB  │
└──────┘ └──────┘
```

### 동작 방식

**읽기 (Read)**
1. 애플리케이션이 먼저 캐시에서 데이터 조회
2. Cache Hit: 캐시에 데이터가 있으면 반환
3. Cache Miss: 캐시에 없으면 DB에서 조회 후 캐시에 저장

**쓰기 (Write)**
1. 데이터베이스에 직접 쓰기
2. 관련 캐시 항목 무효화 또는 업데이트

### 코드 예제 (Python + Redis)

```python
import redis
import json
from typing import Optional, Any
from dataclasses import dataclass
from functools import wraps

# Redis 연결
redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

@dataclass
class User:
    id: int
    name: str
    email: str

class UserRepository:
    def __init__(self, db_connection, cache_client: redis.Redis, cache_ttl: int = 3600):
        self.db = db_connection
        self.cache = cache_client
        self.cache_ttl = cache_ttl

    def _cache_key(self, user_id: int) -> str:
        return f"user:{user_id}"

    def get_user(self, user_id: int) -> Optional[User]:
        """Cache-Aside 패턴으로 사용자 조회"""
        cache_key = self._cache_key(user_id)

        # 1. 캐시에서 먼저 조회
        cached_data = self.cache.get(cache_key)
        if cached_data:
            print(f"Cache HIT for user:{user_id}")
            user_dict = json.loads(cached_data)
            return User(**user_dict)

        # 2. Cache Miss - DB에서 조회
        print(f"Cache MISS for user:{user_id}")
        user = self._fetch_from_db(user_id)

        if user:
            # 3. 캐시에 저장
            self.cache.setex(
                cache_key,
                self.cache_ttl,
                json.dumps(user.__dict__)
            )

        return user

    def update_user(self, user: User) -> bool:
        """사용자 정보 업데이트 (캐시 무효화)"""
        # 1. DB 업데이트
        success = self._update_db(user)

        if success:
            # 2. 캐시 무효화 (삭제)
            cache_key = self._cache_key(user.id)
            self.cache.delete(cache_key)

        return success

    def _fetch_from_db(self, user_id: int) -> Optional[User]:
        # 실제 DB 조회 로직
        cursor = self.db.execute("SELECT id, name, email FROM users WHERE id = ?", (user_id,))
        row = cursor.fetchone()
        if row:
            return User(id=row[0], name=row[1], email=row[2])
        return None

    def _update_db(self, user: User) -> bool:
        # 실제 DB 업데이트 로직
        self.db.execute(
            "UPDATE users SET name = ?, email = ? WHERE id = ?",
            (user.name, user.email, user.id)
        )
        self.db.commit()
        return True


# 데코레이터 패턴으로 Cache-Aside 구현
def cache_aside(cache_client: redis.Redis, key_prefix: str, ttl: int = 3600):
    """Cache-Aside 패턴을 데코레이터로 구현"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # 캐시 키 생성 (함수명 + 인자)
            cache_key = f"{key_prefix}:{func.__name__}:{hash(str(args) + str(kwargs))}"

            # 캐시 조회
            cached = cache_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # DB 조회
            result = func(*args, **kwargs)

            # 캐시 저장
            if result is not None:
                cache_client.setex(cache_key, ttl, json.dumps(result))

            return result
        return wrapper
    return decorator


# 사용 예제
@cache_aside(redis_client, "product", ttl=1800)
def get_product_details(product_id: int) -> dict:
    """상품 상세 정보 조회"""
    # 실제로는 DB에서 조회
    return {
        "id": product_id,
        "name": "Sample Product",
        "price": 10000
    }
```

### 장점과 단점

| 장점 | 단점 |
|------|------|
| 구현이 간단하고 유연함 | Cache Miss 시 지연 발생 |
| 필요한 데이터만 캐싱 (메모리 효율적) | 캐시와 DB 간 데이터 불일치 가능 |
| 캐시 장애 시에도 서비스 가능 | 애플리케이션에서 캐시 로직 관리 필요 |
| 읽기 부하가 높은 시스템에 적합 | Cold Start 문제 (초기 캐시 비어있음) |

### 적합한 사용 사례
- 읽기가 쓰기보다 훨씬 많은 경우
- 일부 데이터 손실이 허용되는 경우
- 사용자 프로필, 상품 정보 등 자주 조회되는 데이터

---

## Read-Through

### 개념

Read-Through 캐시는 캐시 자체가 데이터베이스와의 동기화를 담당합니다. 애플리케이션은 항상 캐시만 바라보며, 캐시가 DB에서 데이터를 가져오는 책임을 집니다.

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │
       ▼
┌──────────────┐
│    Cache     │ ◄──── 캐시가 DB 접근 담당
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Database   │
└──────────────┘
```

### 동작 방식

1. 애플리케이션이 캐시에 데이터 요청
2. 캐시에 데이터가 있으면 반환
3. 캐시에 없으면 캐시가 DB에서 직접 로드하여 저장 후 반환

### 코드 예제 (Java + Spring)

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;
import org.springframework.data.redis.core.RedisTemplate;

@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Product> redisTemplate;

    public ProductService(ProductRepository productRepository,
                         RedisTemplate<String, Product> redisTemplate) {
        this.productRepository = productRepository;
        this.redisTemplate = redisTemplate;
    }

    /**
     * Read-Through 패턴
     * - @Cacheable 어노테이션이 캐시 조회/저장을 자동으로 처리
     * - sync=true: 동시에 같은 키 요청 시 하나만 DB 조회 (Cache Stampede 방지)
     */
    @Cacheable(value = "products", key = "#productId", sync = true)
    public Product getProduct(Long productId) {
        // 캐시 미스 시에만 이 메서드가 실행됨
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    /**
     * 캐시 무효화
     */
    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
}

// Spring Cache 설정
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))  // TTL 설정
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
            );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

### Cache-Aside vs Read-Through 비교

| 특성 | Cache-Aside | Read-Through |
|------|-------------|--------------|
| 캐시 접근 주체 | 애플리케이션 | 캐시 라이브러리/프레임워크 |
| 코드 복잡도 | 높음 (직접 구현) | 낮음 (추상화됨) |
| 유연성 | 높음 | 상대적으로 낮음 |
| 캐시 로직 위치 | 비즈니스 로직에 섞임 | 인프라 레이어에 분리 |

---

## Write-Through

### 개념

Write-Through 캐시는 데이터를 쓸 때 캐시와 데이터베이스에 **동시에** 저장합니다. 쓰기 작업이 완료되려면 캐시와 DB 모두 성공해야 합니다.

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │ Write
       ▼
┌──────────────┐     ┌──────────────┐
│    Cache     │────▶│   Database   │
└──────────────┘     └──────────────┘
       │                    │
       └────── 동기 ────────┘
```

### 동작 방식

1. 애플리케이션이 캐시에 쓰기 요청
2. 캐시가 데이터를 저장하고 동시에 DB에도 저장
3. 둘 다 성공하면 완료 응답

### 코드 예제 (Node.js)

```javascript
const Redis = require('ioredis');
const { Pool } = require('pg');

class WriteThroughCache {
    constructor() {
        this.redis = new Redis({
            host: 'localhost',
            port: 6379
        });

        this.db = new Pool({
            host: 'localhost',
            database: 'myapp',
            user: 'postgres',
            password: 'password'
        });

        this.defaultTTL = 3600; // 1시간
    }

    /**
     * Write-Through 패턴으로 데이터 저장
     * @param {string} key - 캐시 키
     * @param {object} data - 저장할 데이터
     */
    async write(key, data) {
        const client = await this.db.connect();

        try {
            // 트랜잭션 시작
            await client.query('BEGIN');

            // 1. 데이터베이스에 저장
            await client.query(
                `INSERT INTO cache_data (key, value, updated_at)
                 VALUES ($1, $2, NOW())
                 ON CONFLICT (key) DO UPDATE SET value = $2, updated_at = NOW()`,
                [key, JSON.stringify(data)]
            );

            // 2. 캐시에 저장 (동기적으로)
            await this.redis.setex(key, this.defaultTTL, JSON.stringify(data));

            // 트랜잭션 커밋
            await client.query('COMMIT');

            return { success: true };

        } catch (error) {
            // 롤백
            await client.query('ROLLBACK');

            // 캐시도 삭제 (일관성 유지)
            await this.redis.del(key);

            throw error;
        } finally {
            client.release();
        }
    }

    /**
     * 데이터 읽기 (Read-Through와 결합)
     */
    async read(key) {
        // 캐시에서 먼저 조회
        const cached = await this.redis.get(key);
        if (cached) {
            return JSON.parse(cached);
        }

        // DB에서 조회
        const result = await this.db.query(
            'SELECT value FROM cache_data WHERE key = $1',
            [key]
        );

        if (result.rows.length > 0) {
            const data = JSON.parse(result.rows[0].value);
            // 캐시에 저장
            await this.redis.setex(key, this.defaultTTL, JSON.stringify(data));
            return data;
        }

        return null;
    }

    /**
     * 배치 쓰기 (성능 최적화)
     */
    async writeBatch(items) {
        const client = await this.db.connect();
        const pipeline = this.redis.pipeline();

        try {
            await client.query('BEGIN');

            for (const { key, data } of items) {
                // DB 저장
                await client.query(
                    `INSERT INTO cache_data (key, value, updated_at)
                     VALUES ($1, $2, NOW())
                     ON CONFLICT (key) DO UPDATE SET value = $2, updated_at = NOW()`,
                    [key, JSON.stringify(data)]
                );

                // Redis 파이프라인에 추가
                pipeline.setex(key, this.defaultTTL, JSON.stringify(data));
            }

            await client.query('COMMIT');
            await pipeline.exec();

            return { success: true, count: items.length };

        } catch (error) {
            await client.query('ROLLBACK');

            // 모든 캐시 키 삭제
            const deletePromises = items.map(({ key }) => this.redis.del(key));
            await Promise.all(deletePromises);

            throw error;
        } finally {
            client.release();
        }
    }
}

// 사용 예제
async function main() {
    const cache = new WriteThroughCache();

    // Write-Through로 사용자 데이터 저장
    await cache.write('user:1001', {
        id: 1001,
        name: 'John Doe',
        email: 'john@example.com'
    });

    // 읽기 (캐시에서 바로 가져옴)
    const user = await cache.read('user:1001');
    console.log('User:', user);
}
```

### 장점과 단점

| 장점 | 단점 |
|------|------|
| 캐시와 DB 간 데이터 일관성 보장 | 쓰기 지연 시간 증가 (두 곳에 저장) |
| 읽기 시 항상 최신 데이터 | 쓰기 처리량 제한 |
| 데이터 손실 위험 낮음 | 캐시 장애 시 쓰기도 실패 가능 |

### 적합한 사용 사례
- 데이터 일관성이 매우 중요한 경우
- 쓰기 빈도가 상대적으로 낮은 경우
- 금융 거래, 재고 관리 등 정확성이 중요한 시스템

---

## Write-Behind (Write-Back)

### 개념

Write-Behind는 데이터를 캐시에만 즉시 저장하고, 데이터베이스에는 비동기적으로 나중에 저장합니다. 쓰기 성능을 극대화하지만 데이터 손실 위험이 있습니다.

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │ Write (즉시 완료)
       ▼
┌──────────────┐     ┌──────────────┐
│    Cache     │- - -│   Database   │
└──────────────┘     └──────────────┘
       │                    ▲
       └──── 비동기 ────────┘
```

### 동작 방식

1. 애플리케이션이 캐시에 쓰기 요청
2. 캐시에 즉시 저장하고 응답 반환
3. 백그라운드에서 배치로 DB에 저장

### 코드 예제 (Python)

```python
import redis
import json
import threading
import time
from queue import Queue
from typing import Dict, Any, List
from dataclasses import dataclass, field
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class WriteOperation:
    key: str
    value: Any
    timestamp: datetime = field(default_factory=datetime.now)

class WriteBehindCache:
    def __init__(self, redis_client: redis.Redis, db_connection,
                 batch_size: int = 100, flush_interval: float = 5.0):
        self.redis = redis_client
        self.db = db_connection
        self.batch_size = batch_size
        self.flush_interval = flush_interval

        # 쓰기 작업 큐
        self.write_queue: Queue[WriteOperation] = Queue()

        # 배경 스레드 시작
        self._running = True
        self._flush_thread = threading.Thread(target=self._flush_loop, daemon=True)
        self._flush_thread.start()

    def write(self, key: str, value: Any, ttl: int = 3600) -> bool:
        """
        Write-Behind 패턴으로 데이터 쓰기
        - 캐시에 즉시 저장
        - DB 쓰기는 큐에 추가 (비동기)
        """
        try:
            # 1. 캐시에 즉시 저장
            self.redis.setex(key, ttl, json.dumps(value))

            # 2. 쓰기 작업 큐에 추가
            operation = WriteOperation(key=key, value=value)
            self.write_queue.put(operation)

            logger.debug(f"Queued write operation for key: {key}")
            return True

        except Exception as e:
            logger.error(f"Write failed for key {key}: {e}")
            return False

    def _flush_loop(self):
        """배경 스레드에서 주기적으로 DB에 저장"""
        while self._running:
            try:
                self._flush_to_db()
            except Exception as e:
                logger.error(f"Flush failed: {e}")

            time.sleep(self.flush_interval)

    def _flush_to_db(self):
        """큐에 쌓인 쓰기 작업을 DB에 배치 저장"""
        operations: List[WriteOperation] = []

        # 배치 크기만큼 큐에서 가져오기
        while len(operations) < self.batch_size and not self.write_queue.empty():
            try:
                op = self.write_queue.get_nowait()
                operations.append(op)
            except:
                break

        if not operations:
            return

        logger.info(f"Flushing {len(operations)} operations to database")

        # 같은 키에 대한 여러 쓰기 중 마지막 것만 유지 (중복 제거)
        latest_operations: Dict[str, WriteOperation] = {}
        for op in operations:
            latest_operations[op.key] = op

        # 배치 저장
        cursor = self.db.cursor()
        try:
            for key, op in latest_operations.items():
                cursor.execute(
                    """
                    INSERT INTO cache_data (key, value, updated_at)
                    VALUES (?, ?, ?)
                    ON CONFLICT(key) DO UPDATE SET
                        value = excluded.value,
                        updated_at = excluded.updated_at
                    """,
                    (key, json.dumps(op.value), op.timestamp.isoformat())
                )

            self.db.commit()
            logger.info(f"Successfully flushed {len(latest_operations)} unique operations")

        except Exception as e:
            self.db.rollback()
            logger.error(f"Database flush failed: {e}")

            # 실패한 작업 다시 큐에 추가
            for op in operations:
                self.write_queue.put(op)

    def read(self, key: str) -> Any:
        """데이터 읽기"""
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    def shutdown(self):
        """안전한 종료 (남은 데이터 모두 저장)"""
        logger.info("Shutting down Write-Behind cache...")
        self._running = False

        # 남은 모든 데이터 저장
        while not self.write_queue.empty():
            self._flush_to_db()

        logger.info("Shutdown complete")


# 고급: Kafka를 활용한 Write-Behind
class KafkaWriteBehindCache:
    """
    Kafka를 활용한 더 안정적인 Write-Behind 구현
    - 메시지 영속성 보장
    - 다중 컨슈머로 처리량 확장
    - 장애 복구 용이
    """

    def __init__(self, redis_client: redis.Redis, kafka_producer):
        self.redis = redis_client
        self.producer = kafka_producer
        self.topic = 'cache-write-behind'

    async def write(self, key: str, value: Any, ttl: int = 3600) -> bool:
        try:
            # 1. 캐시에 즉시 저장
            await self.redis.setex(key, ttl, json.dumps(value))

            # 2. Kafka로 쓰기 이벤트 발행
            message = {
                'key': key,
                'value': value,
                'timestamp': datetime.now().isoformat()
            }
            await self.producer.send(self.topic, json.dumps(message).encode())

            return True
        except Exception as e:
            logger.error(f"Write failed: {e}")
            return False


# 사용 예제
def main():
    import sqlite3

    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    db = sqlite3.connect('cache.db')

    # 테이블 생성
    db.execute('''
        CREATE TABLE IF NOT EXISTS cache_data (
            key TEXT PRIMARY KEY,
            value TEXT,
            updated_at TEXT
        )
    ''')

    # Write-Behind 캐시 생성
    cache = WriteBehindCache(
        redis_client=redis_client,
        db_connection=db,
        batch_size=50,
        flush_interval=3.0  # 3초마다 플러시
    )

    # 빠른 쓰기 작업
    for i in range(1000):
        cache.write(f'item:{i}', {'id': i, 'name': f'Item {i}'})

    # 즉시 읽기 가능 (캐시에서)
    item = cache.read('item:500')
    print(f"Read item: {item}")

    # 잠시 대기 (DB 플러시 확인)
    time.sleep(10)

    # 안전한 종료
    cache.shutdown()

if __name__ == '__main__':
    main()
```

### 장점과 단점

| 장점 | 단점 |
|------|------|
| 매우 빠른 쓰기 성능 | 데이터 손실 위험 (캐시 장애 시) |
| DB 부하 분산 (배치 쓰기) | 데이터 일관성 문제 가능 |
| 쓰기 중복 제거로 효율 향상 | 구현 복잡도 높음 |
| 높은 쓰기 처리량 | 장애 복구 복잡 |

### 적합한 사용 사례
- 쓰기가 매우 빈번한 시스템 (로그, 분석 데이터)
- 일시적 데이터 손실이 허용되는 경우
- 실시간 분석, 세션 데이터 등

---

## 캐시 무효화 전략

캐시 무효화는 "컴퓨터 과학에서 가장 어려운 문제 중 하나"로 알려져 있습니다.

### 1. TTL (Time-To-Live) 기반

```python
# 가장 간단한 방식: 시간이 지나면 자동 만료
redis_client.setex('user:1001', 3600, user_data)  # 1시간 후 만료

# 동적 TTL 설정
def calculate_ttl(data_type: str, data: dict) -> int:
    """데이터 유형과 특성에 따른 동적 TTL 계산"""
    base_ttl = {
        'user_profile': 3600,      # 1시간
        'product_list': 300,       # 5분
        'session': 1800,           # 30분
        'static_config': 86400,    # 24시간
    }

    ttl = base_ttl.get(data_type, 600)

    # 인기 데이터는 TTL 연장
    if data.get('view_count', 0) > 1000:
        ttl *= 2

    return ttl
```

### 2. 이벤트 기반 무효화

```python
import redis
from typing import Callable, List

class EventDrivenCacheInvalidation:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.pubsub = redis_client.pubsub()
        self.handlers: dict[str, List[Callable]] = {}

    def subscribe(self, event_type: str, handler: Callable):
        """이벤트 타입에 핸들러 등록"""
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)

    def publish_invalidation(self, event_type: str, keys: List[str]):
        """캐시 무효화 이벤트 발행"""
        message = json.dumps({
            'event': event_type,
            'keys': keys,
            'timestamp': datetime.now().isoformat()
        })
        self.redis.publish(f'cache:invalidation:{event_type}', message)

    def invalidate_pattern(self, pattern: str):
        """패턴 기반 일괄 무효화"""
        keys = self.redis.keys(pattern)
        if keys:
            self.redis.delete(*keys)
            return len(keys)
        return 0


# 사용 예제: 상품 가격 변경 시 관련 캐시 무효화
class ProductService:
    def __init__(self, cache_invalidator: EventDrivenCacheInvalidation):
        self.invalidator = cache_invalidator

    def update_product_price(self, product_id: int, new_price: float):
        # DB 업데이트
        self._update_db(product_id, new_price)

        # 관련 캐시 모두 무효화
        keys_to_invalidate = [
            f'product:{product_id}',
            f'product:{product_id}:detail',
            f'category:products:*',  # 카테고리 목록도 무효화
        ]

        # 이벤트 발행 (다른 서버 인스턴스에도 전파)
        self.invalidator.publish_invalidation('product_update', keys_to_invalidate)
```

### 3. 버전 기반 무효화

```python
class VersionedCache:
    """버전 번호를 활용한 캐시 무효화"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def _get_version(self, entity: str) -> int:
        """엔티티의 현재 버전 조회"""
        version = self.redis.get(f'version:{entity}')
        return int(version) if version else 0

    def _increment_version(self, entity: str) -> int:
        """버전 증가"""
        return self.redis.incr(f'version:{entity}')

    def cache_key(self, entity: str, id: Any) -> str:
        """버전을 포함한 캐시 키 생성"""
        version = self._get_version(entity)
        return f'{entity}:{id}:v{version}'

    def invalidate(self, entity: str):
        """버전 증가로 모든 관련 캐시 무효화"""
        new_version = self._increment_version(entity)
        print(f"Entity '{entity}' version updated to {new_version}")


# 사용 예제
cache = VersionedCache(redis_client)

# 캐시 저장 (버전 포함 키 사용)
key = cache.cache_key('product', 1001)  # product:1001:v1
redis_client.setex(key, 3600, product_data)

# 상품 정보 업데이트 -> 버전 증가
cache.invalidate('product')  # 이제 product:1001:v2를 사용하게 됨
```

### 4. 태그 기반 무효화

```python
class TaggedCache:
    """태그를 활용한 그룹 캐시 무효화"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def set_with_tags(self, key: str, value: Any, tags: List[str], ttl: int = 3600):
        """태그와 함께 캐시 저장"""
        # 값 저장
        self.redis.setex(key, ttl, json.dumps(value))

        # 각 태그에 키 추가
        for tag in tags:
            self.redis.sadd(f'tag:{tag}', key)

    def invalidate_by_tag(self, tag: str):
        """태그에 속한 모든 캐시 무효화"""
        tag_key = f'tag:{tag}'
        keys = self.redis.smembers(tag_key)

        if keys:
            # 모든 관련 캐시 삭제
            self.redis.delete(*keys)
            # 태그 세트도 삭제
            self.redis.delete(tag_key)

        return len(keys)


# 사용 예제
tagged_cache = TaggedCache(redis_client)

# 상품 캐시 저장 (여러 태그 연결)
tagged_cache.set_with_tags(
    'product:1001',
    {'id': 1001, 'name': 'Laptop', 'category': 'electronics'},
    tags=['category:electronics', 'brand:apple', 'featured']
)

# 'electronics' 카테고리의 모든 캐시 무효화
deleted_count = tagged_cache.invalidate_by_tag('category:electronics')
```

### 캐시 무효화 전략 비교

| 전략 | 장점 | 단점 | 적합한 사용 사례 |
|------|------|------|------------------|
| TTL | 구현 간단, 자동 정리 | 데이터 불일치 가능 | 정적 콘텐츠, 설정 데이터 |
| 이벤트 기반 | 즉시 무효화, 정확함 | 구현 복잡, 인프라 필요 | 실시간성 중요한 데이터 |
| 버전 기반 | 일괄 무효화 간단 | 버전 관리 필요 | 자주 변경되는 엔티티 |
| 태그 기반 | 유연한 그룹 무효화 | 메모리 오버헤드 | 연관된 데이터 그룹 |

---

## 캐시 히트율과 미스율

### 정의

- **캐시 히트율 (Cache Hit Rate)**: 요청 중 캐시에서 데이터를 찾은 비율
- **캐시 미스율 (Cache Miss Rate)**: 요청 중 캐시에서 데이터를 찾지 못한 비율

```
히트율 = (캐시 히트 수) / (총 요청 수) × 100%
미스율 = (캐시 미스 수) / (총 요청 수) × 100%
히트율 + 미스율 = 100%
```

### 모니터링 구현

```python
import time
from dataclasses import dataclass, field
from typing import Dict
import threading

@dataclass
class CacheMetrics:
    hits: int = 0
    misses: int = 0
    total_requests: int = 0
    avg_hit_latency_ms: float = 0.0
    avg_miss_latency_ms: float = 0.0

    @property
    def hit_rate(self) -> float:
        if self.total_requests == 0:
            return 0.0
        return (self.hits / self.total_requests) * 100

    @property
    def miss_rate(self) -> float:
        return 100 - self.hit_rate


class MonitoredCache:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.metrics = CacheMetrics()
        self._lock = threading.Lock()

        # 키별 통계
        self.key_stats: Dict[str, Dict] = {}

    def get(self, key: str) -> Any:
        """통계를 수집하면서 캐시 조회"""
        start_time = time.time()

        value = self.redis.get(key)

        latency_ms = (time.time() - start_time) * 1000

        with self._lock:
            self.metrics.total_requests += 1

            if value is not None:
                self.metrics.hits += 1
                self._update_avg_latency('hit', latency_ms)
                self._update_key_stats(key, 'hit')
            else:
                self.metrics.misses += 1
                self._update_avg_latency('miss', latency_ms)
                self._update_key_stats(key, 'miss')

        return json.loads(value) if value else None

    def _update_avg_latency(self, type: str, latency_ms: float):
        if type == 'hit':
            total = self.metrics.avg_hit_latency_ms * (self.metrics.hits - 1) + latency_ms
            self.metrics.avg_hit_latency_ms = total / self.metrics.hits
        else:
            total = self.metrics.avg_miss_latency_ms * (self.metrics.misses - 1) + latency_ms
            self.metrics.avg_miss_latency_ms = total / self.metrics.misses

    def _update_key_stats(self, key: str, event: str):
        if key not in self.key_stats:
            self.key_stats[key] = {'hits': 0, 'misses': 0}
        self.key_stats[key][f'{event}s'] += 1

    def get_stats(self) -> dict:
        """현재 통계 반환"""
        return {
            'hit_rate': f"{self.metrics.hit_rate:.2f}%",
            'miss_rate': f"{self.metrics.miss_rate:.2f}%",
            'total_requests': self.metrics.total_requests,
            'hits': self.metrics.hits,
            'misses': self.metrics.misses,
            'avg_hit_latency_ms': f"{self.metrics.avg_hit_latency_ms:.2f}ms",
            'avg_miss_latency_ms': f"{self.metrics.avg_miss_latency_ms:.2f}ms"
        }

    def get_cold_keys(self, threshold: float = 0.3) -> list:
        """히트율이 낮은 키 목록 (캐싱 가치 낮음)"""
        cold_keys = []
        for key, stats in self.key_stats.items():
            total = stats['hits'] + stats['misses']
            if total > 10:  # 충분한 샘플이 있는 경우만
                hit_rate = stats['hits'] / total
                if hit_rate < threshold:
                    cold_keys.append({
                        'key': key,
                        'hit_rate': f"{hit_rate * 100:.1f}%",
                        'total': total
                    })
        return sorted(cold_keys, key=lambda x: x['hit_rate'])


# Redis INFO 명령어로 통계 조회
def get_redis_cache_stats(redis_client: redis.Redis) -> dict:
    """Redis 서버의 캐시 통계"""
    info = redis_client.info('stats')

    hits = info.get('keyspace_hits', 0)
    misses = info.get('keyspace_misses', 0)
    total = hits + misses

    return {
        'keyspace_hits': hits,
        'keyspace_misses': misses,
        'hit_rate': f"{(hits / total * 100):.2f}%" if total > 0 else "N/A",
        'expired_keys': info.get('expired_keys', 0),
        'evicted_keys': info.get('evicted_keys', 0)
    }
```

### 히트율 최적화 전략

```python
class CacheOptimizer:
    """캐시 히트율 최적화를 위한 전략들"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    # 1. 캐시 워밍 (Pre-warming)
    def warm_cache(self, popular_keys: List[str], fetch_func: Callable):
        """인기 데이터 미리 캐싱"""
        for key in popular_keys:
            if not self.redis.exists(key):
                data = fetch_func(key)
                if data:
                    self.redis.setex(key, 3600, json.dumps(data))

    # 2. 적응형 TTL
    def set_with_adaptive_ttl(self, key: str, value: Any, base_ttl: int = 3600):
        """접근 빈도에 따른 동적 TTL"""
        access_count = self.redis.incr(f'access_count:{key}')

        # 자주 접근되는 데이터는 TTL 연장
        if access_count > 100:
            ttl = base_ttl * 3
        elif access_count > 50:
            ttl = base_ttl * 2
        else:
            ttl = base_ttl

        self.redis.setex(key, ttl, json.dumps(value))

    # 3. Cache Stampede 방지 (Probabilistic Early Expiration)
    def get_with_pee(self, key: str, fetch_func: Callable,
                     ttl: int = 3600, beta: float = 1.0) -> Any:
        """
        Probabilistic Early Expiration (PEE)
        - TTL 만료 전에 확률적으로 캐시 갱신
        - Cache Stampede 방지
        """
        import random
        import math

        cached = self.redis.get(key)
        remaining_ttl = self.redis.ttl(key)

        if cached and remaining_ttl > 0:
            # 만료가 가까울수록 갱신 확률 증가
            expiry_gap = ttl - remaining_ttl
            random_value = random.random()

            # beta * log(random) 공식으로 조기 만료 확률 계산
            should_refresh = expiry_gap > -beta * math.log(random_value)

            if not should_refresh:
                return json.loads(cached)

        # 캐시 갱신
        data = fetch_func()
        self.redis.setex(key, ttl, json.dumps(data))
        return data

    # 4. 2단계 캐싱 (Local + Distributed)
    def get_two_level(self, key: str, local_cache: dict,
                      fetch_func: Callable) -> Any:
        """
        L1: 로컬 메모리 캐시 (초고속)
        L2: Redis (분산 캐시)
        """
        # L1 확인
        if key in local_cache:
            return local_cache[key]

        # L2 확인
        cached = self.redis.get(key)
        if cached:
            data = json.loads(cached)
            local_cache[key] = data  # L1에도 저장
            return data

        # DB에서 조회
        data = fetch_func()
        if data:
            self.redis.setex(key, 3600, json.dumps(data))
            local_cache[key] = data

        return data
```

### 권장 히트율 기준

| 히트율 | 평가 | 조치 |
|--------|------|------|
| > 95% | 우수 | 현재 전략 유지 |
| 85-95% | 양호 | 최적화 검토 |
| 70-85% | 개선 필요 | TTL, 캐싱 범위 조정 |
| < 70% | 문제 있음 | 전략 전면 재검토 |

---

## 참고 자료

- [AWS - Database Caching Strategies Using Redis](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html)
- [Redis - Caching Solutions](https://redis.io/solutions/caching/)
- [Cache Invalidation Strategies](https://leapcell.io/blog/cache-invalidation-strategies-time-based-vs-event-driven)
- [Microsoft Azure - Cache Best Practices](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices-development)
