# MongoDB 심화

## 목차
1. [MongoDB 개요](#mongodb-개요)
2. [아키텍처](#아키텍처)
3. [데이터 모델링](#데이터-모델링)
4. [샤딩 (Sharding)](#샤딩-sharding)
5. [인덱싱](#인덱싱)
6. [집계 파이프라인](#집계-파이프라인)
---

## MongoDB 개요

### MongoDB란?

MongoDB는 문서 지향(Document-Oriented) NoSQL 데이터베이스입니다. JSON과 유사한 BSON(Binary JSON) 형식으로 데이터를 저장합니다.

```
RDBMS                           MongoDB
─────────────────────────────────────────────
Database                        Database
Table                           Collection
Row                             Document
Column                          Field
Primary Key                     _id
Foreign Key                     Reference / Embedded
```

### 핵심 특징

```javascript
// 유연한 스키마
db.users.insertMany([
    {
        name: "Kim",
        email: "kim@example.com",
        age: 25
    },
    {
        name: "Lee",
        email: "lee@example.com",
        // age 필드 없음 - 스키마 유연성
        address: {
            city: "Seoul",
            zip: "12345"
        }
    }
]);

// 중첩 문서 (Embedded Document)
{
    _id: ObjectId("..."),
    title: "MongoDB 가이드",
    author: {
        name: "홍길동",
        email: "hong@example.com"
    },
    tags: ["database", "nosql", "mongodb"],
    comments: [
        { user: "user1", text: "좋은 글입니다", date: ISODate("...") },
        { user: "user2", text: "감사합니다", date: ISODate("...") }
    ]
}
```

---

## 아키텍처

### 단일 노드 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                        MongoDB Server                            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Storage Engine                        │    │
│  │                      (WiredTiger)                        │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐ │    │
│  │  │    Cache      │  │    Journal    │  │  Checkpoint   │ │    │
│  │  │   (Memory)    │  │   (WAL Log)   │  │   (Snapshots) │ │    │
│  │  └───────────────┘  └───────────────┘  └───────────────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Data Files                            │    │
│  │        (.wt files - Collection and Index data)           │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 복제 세트 (Replica Set)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Replica Set                               │
│                                                                  │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│   │   Primary    │────►│  Secondary   │────►│  Secondary   │    │
│   │              │◄────│              │◄────│              │    │
│   │  Read/Write  │     │  Read Only   │     │  Read Only   │    │
│   └──────────────┘     └──────────────┘     └──────────────┘    │
│          │                    │                    │             │
│          │         Oplog Replication               │             │
│          ▼                    ▼                    ▼             │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                      Oplog                                │  │
│   │    (Capped Collection - 복제 로그)                        │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Automatic Failover: Primary 장애 시 Secondary가 Primary 승격   │
└─────────────────────────────────────────────────────────────────┘
```

```javascript
// 복제 세트 상태 확인
rs.status()

// 복제 세트 설정
rs.initiate({
    _id: "myReplicaSet",
    members: [
        { _id: 0, host: "mongo1:27017", priority: 2 },
        { _id: 1, host: "mongo2:27017", priority: 1 },
        { _id: 2, host: "mongo3:27017", priority: 1 }
    ]
});

// Read Preference 설정
// primary: Primary에서만 읽기
// secondary: Secondary에서 읽기
// nearest: 네트워크 지연이 가장 낮은 노드
db.collection.find().readPref("secondary");
```

### 샤딩 클러스터 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                      Sharded Cluster                             │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Application                          │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │                                      │
│  ┌────────────────────────▼────────────────────────────────┐    │
│  │                   mongos (Router)                        │    │
│  │              Query Routing & Distribution                │    │
│  └────────────┬───────────────────────────────┬────────────┘    │
│               │                               │                  │
│  ┌────────────▼────────────┐    ┌─────────────▼────────────┐    │
│  │    Config Servers       │    │     Config Servers       │    │
│  │    (Replica Set)        │    │     (Replica Set)        │    │
│  │  - Cluster Metadata     │    │  - Shard Key Ranges      │    │
│  └─────────────────────────┘    └──────────────────────────┘    │
│               │                               │                  │
│  ┌────────────▼────────────┐    ┌─────────────▼────────────┐    │
│  │       Shard 1           │    │       Shard 2            │    │
│  │    (Replica Set)        │    │    (Replica Set)         │    │
│  │  ┌─────┐┌─────┐┌─────┐ │    │  ┌─────┐┌─────┐┌─────┐  │    │
│  │  │ P   ││ S   ││ S   │ │    │  │ P   ││ S   ││ S   │  │    │
│  │  └─────┘└─────┘└─────┘ │    │  └─────┘└─────┘└─────┘  │    │
│  └─────────────────────────┘    └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 데이터 모델링

### 임베딩 vs 참조

```javascript
// 1. 임베딩 (Embedded) - 1:N 관계에서 N이 작을 때
// 장점: 단일 쿼리로 모든 데이터 조회
// 단점: 문서 크기 제한 (16MB), 중첩 데이터 수정 복잡

// 블로그 포스트와 댓글 (댓글이 적을 때)
{
    _id: ObjectId("..."),
    title: "MongoDB 튜토리얼",
    content: "...",
    author: "kim",
    comments: [
        { user: "lee", text: "좋아요!", date: ISODate("...") },
        { user: "park", text: "감사합니다", date: ISODate("...") }
    ]
}

// 2. 참조 (Reference) - 1:N 관계에서 N이 크거나 독립적일 때
// 장점: 데이터 중복 없음, 대용량 데이터 처리
// 단점: 조인 필요 ($lookup)

// posts 컬렉션
{
    _id: ObjectId("post1"),
    title: "MongoDB 튜토리얼",
    content: "...",
    author_id: ObjectId("user1")
}

// comments 컬렉션
{
    _id: ObjectId("comment1"),
    post_id: ObjectId("post1"),
    user: "lee",
    text: "좋아요!",
    date: ISODate("...")
}

// $lookup으로 조인
db.posts.aggregate([
    {
        $lookup: {
            from: "comments",
            localField: "_id",
            foreignField: "post_id",
            as: "comments"
        }
    }
]);
```

### 모델링 패턴

```javascript
// 1. Extended Reference Pattern - 자주 접근하는 필드 복제
// orders 컬렉션
{
    _id: ObjectId("..."),
    customer_id: ObjectId("customer1"),
    // 자주 필요한 고객 정보만 복제
    customer_name: "김철수",
    customer_email: "kim@example.com",
    items: [...]
}

// 2. Subset Pattern - 일부 데이터만 임베딩
// 최근 리뷰 5개만 제품에 임베딩
{
    _id: ObjectId("..."),
    name: "MacBook Pro",
    price: 2000000,
    recent_reviews: [
        // 최근 5개만
    ]
}
// 전체 리뷰는 별도 컬렉션에 저장

// 3. Bucket Pattern - 시계열 데이터 그룹화
// 1시간 단위로 센서 데이터 버킷팅
{
    sensor_id: "sensor_001",
    start_time: ISODate("2024-01-01T00:00:00Z"),
    end_time: ISODate("2024-01-01T01:00:00Z"),
    measurements: [
        { time: ISODate("2024-01-01T00:00:00Z"), value: 25.1 },
        { time: ISODate("2024-01-01T00:01:00Z"), value: 25.2 },
        // ... 60개의 측정값
    ],
    count: 60,
    sum: 1510.5,
    avg: 25.175
}

// 4. Computed Pattern - 계산 결과 미리 저장
{
    _id: ObjectId("..."),
    title: "MongoDB 입문",
    ratings: [5, 4, 5, 3, 5],
    // 미리 계산된 집계 값
    rating_count: 5,
    rating_sum: 22,
    rating_avg: 4.4
}
```

---

## 샤딩 (Sharding)

### 샤드 키 선택

```javascript
// 샤딩 활성화
sh.enableSharding("mydb");

// 샤드 키로 컬렉션 샤딩
// 1. Range Sharding
sh.shardCollection("mydb.orders", { order_date: 1 });

// 2. Hashed Sharding (균등 분배)
sh.shardCollection("mydb.users", { user_id: "hashed" });

// 3. Compound Shard Key
sh.shardCollection("mydb.logs", { tenant_id: 1, timestamp: 1 });
```

### 좋은 샤드 키 vs 나쁜 샤드 키

```javascript
// 좋은 샤드 키 특성:
// 1. High Cardinality - 다양한 값
// 2. Low Frequency - 각 값이 균등하게 분포
// 3. Non-Monotonic - 단조 증가/감소하지 않음 (Hashed 제외)

// 나쁜 샤드 키 예시
// 1. 단조 증가 (Monotonic)
{ _id: 1 }  // ObjectId는 시간순으로 증가 → 하나의 샤드에 집중

// 2. 낮은 카디널리티
{ status: 1 }  // "active", "inactive" 등 값이 몇 개 없음

// 3. 변경되는 필드
{ address: 1 }  // 사용자 주소는 변경될 수 있음

// 좋은 샤드 키 예시
// 멀티테넌트 시스템
{ tenant_id: 1, _id: 1 }

// 시계열 데이터 (hashed)
{ device_id: "hashed" }

// 복합 키
{ region: 1, user_id: 1 }
```

### 샤딩 관리

```javascript
// 샤드 상태 확인
sh.status()

// 청크 분포 확인
db.orders.getShardDistribution()

// 밸런서 상태
sh.getBalancerState()
sh.isBalancerRunning()

// 밸런서 창 설정 (새벽에만 실행)
db.settings.update(
    { _id: "balancer" },
    { $set: { activeWindow: { start: "02:00", stop: "06:00" } } },
    { upsert: true }
);

// 청크 수동 마이그레이션
sh.moveChunk("mydb.orders", { order_date: ISODate("2024-01-01") }, "shard2")
```

---

## 인덱싱

### 인덱스 유형

```javascript
// 1. Single Field Index
db.users.createIndex({ email: 1 })  // 오름차순
db.users.createIndex({ age: -1 })   // 내림차순

// 2. Compound Index
db.orders.createIndex({ customer_id: 1, order_date: -1 })

// 3. Multikey Index (배열 필드)
db.products.createIndex({ tags: 1 })

// 4. Text Index (전문 검색)
db.articles.createIndex({ title: "text", content: "text" })
db.articles.find({ $text: { $search: "mongodb tutorial" } })

// 5. Geospatial Index
db.places.createIndex({ location: "2dsphere" })
db.places.find({
    location: {
        $near: {
            $geometry: { type: "Point", coordinates: [127.0, 37.5] },
            $maxDistance: 5000  // 5km
        }
    }
})

// 6. Hashed Index
db.users.createIndex({ user_id: "hashed" })

// 7. TTL Index (자동 삭제)
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })

// 8. Partial Index
db.orders.createIndex(
    { customer_id: 1 },
    { partialFilterExpression: { status: "active" } }
)
```

### 인덱스 최적화

```javascript
// Covered Query (인덱스만으로 결과 반환)
db.users.createIndex({ email: 1, name: 1 })
db.users.find(
    { email: "test@example.com" },
    { email: 1, name: 1, _id: 0 }  // Projection
)
// explain의 totalDocsExamined: 0 이면 Covered Query

// 인덱스 사용 분석
db.users.find({ email: "test@example.com" }).explain("executionStats")

// 중요 필드:
// - stage: IXSCAN (인덱스 사용) vs COLLSCAN (전체 스캔)
// - totalDocsExamined: 검사한 문서 수
// - nReturned: 반환된 문서 수
// - executionTimeMillis: 실행 시간

// 인덱스 통계
db.users.aggregate([{ $indexStats: {} }])

// 미사용 인덱스 확인
db.collection.aggregate([
    { $indexStats: {} },
    { $match: { accesses: { ops: 0 } } }
])
```

---

## 집계 파이프라인

### 기본 집계 연산자

```javascript
// 파이프라인 예시: 월별 매출 분석
db.orders.aggregate([
    // 1. 필터링
    { $match: { status: "completed", order_date: { $gte: ISODate("2024-01-01") } } },

    // 2. 필드 추가/변환
    { $addFields: {
        month: { $month: "$order_date" },
        revenue: { $multiply: ["$quantity", "$price"] }
    }},

    // 3. 그룹화
    { $group: {
        _id: "$month",
        totalRevenue: { $sum: "$revenue" },
        orderCount: { $sum: 1 },
        avgOrderValue: { $avg: "$revenue" }
    }},

    // 4. 정렬
    { $sort: { _id: 1 } },

    // 5. 출력 형식
    { $project: {
        month: "$_id",
        totalRevenue: { $round: ["$totalRevenue", 2] },
        orderCount: 1,
        avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }}
]);
```

### 고급 집계 연산

```javascript
// $lookup (조인)
db.orders.aggregate([
    {
        $lookup: {
            from: "customers",
            localField: "customer_id",
            foreignField: "_id",
            as: "customer"
        }
    },
    { $unwind: "$customer" }  // 배열을 개별 문서로
]);

// $facet (다중 집계)
db.products.aggregate([
    {
        $facet: {
            "byCategory": [
                { $group: { _id: "$category", count: { $sum: 1 } } }
            ],
            "priceRanges": [
                { $bucket: {
                    groupBy: "$price",
                    boundaries: [0, 100, 500, 1000, Infinity],
                    output: { count: { $sum: 1 } }
                }}
            ],
            "topProducts": [
                { $sort: { sales: -1 } },
                { $limit: 5 }
            ]
        }
    }
]);

// $graphLookup (재귀적 조회)
db.employees.aggregate([
    { $match: { name: "CEO" } },
    {
        $graphLookup: {
            from: "employees",
            startWith: "$_id",
            connectFromField: "_id",
            connectToField: "manager_id",
            as: "allReports",
            maxDepth: 5
        }
    }
]);

// $merge (결과를 다른 컬렉션에 저장)
db.orders.aggregate([
    { $group: { _id: "$customer_id", totalSpent: { $sum: "$amount" } } },
    { $merge: {
        into: "customer_stats",
        on: "_id",
        whenMatched: "merge",
        whenNotMatched: "insert"
    }}
]);
```

---

## 참고 자료

- [MongoDB 공식 문서](https://www.mongodb.com/docs/manual/)
- [MongoDB University](https://university.mongodb.com/)
- [Performance Best Practices](https://www.mongodb.com/company/blog/mongodb/performance-best-practices-sharding)
- Designing Data-Intensive Applications by Martin Kleppmann
