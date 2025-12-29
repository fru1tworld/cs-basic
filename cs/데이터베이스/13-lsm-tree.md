# LSM-Tree (Log-Structured Merge-Tree)

## 개요

LSM-Tree는 **쓰기 최적화된** 자료구조로, 랜덤 쓰기를 순차 쓰기로 변환하여 높은 쓰기 처리량을 달성한다. RocksDB, LevelDB, Cassandra, HBase 등 많은 NoSQL과 NewSQL 시스템에서 사용된다. 반면 읽기 성능과 공간 효율성에서 트레이드오프가 있으며, **Compaction** 전략이 성능에 큰 영향을 미친다.

## 핵심 개념

### 1. LSM-Tree 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    LSM-Tree Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─────────────────────────────────────────┐                │
│ │            Memory                        │                │
│ │ ┌─────────────────────────────────────┐ │                │
│ │ │         MemTable (활성)              │ │ ← 새 쓰기     │
│ │ │    Sorted by Key (Red-Black Tree)   │ │                │
│ │ └─────────────────────────────────────┘ │                │
│ │ ┌─────────────────────────────────────┐ │                │
│ │ │      Immutable MemTable             │ │ ← 플러시 대기  │
│ │ └─────────────────────────────────────┘ │                │
│ └─────────────────────────────────────────┘                │
│                      │                                      │
│                      ▼ Flush (순차 쓰기)                    │
│ ┌─────────────────────────────────────────┐                │
│ │            Disk (SSTables)              │                │
│ │                                         │                │
│ │ Level 0: [SST] [SST] [SST]             │ ← 오버랩 가능  │
│ │            │                           │                │
│ │            ▼ Compaction                │                │
│ │ Level 1: [SST] [SST] [SST] [SST]       │ ← 비오버랩     │
│ │            │                           │                │
│ │            ▼ Compaction                │                │
│ │ Level 2: [SST][SST][SST][SST]...[SST]  │ ← 10배 크기   │
│ │            │                           │                │
│ │            ▼                           │                │
│ │ Level N: ...                           │                │
│ └─────────────────────────────────────────┘                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. 쓰기 경로 (Write Path)

```
┌─────────────────────────────────────────────────────────────┐
│                    Write Path                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. WAL (Write-Ahead Log)에 기록                             │
│    - 순차 쓰기, 내구성 보장                                  │
│                                                             │
│ 2. MemTable에 삽입                                          │
│    - 메모리 내 정렬된 자료구조 (Skip List, Red-Black Tree)   │
│    - 쓰기 완료 응답                                          │
│                                                             │
│ 3. MemTable이 가득 차면 Immutable로 변환                    │
│    - 새 MemTable 생성                                        │
│    - 기존 MemTable은 더 이상 쓰기 불가                       │
│                                                             │
│ 4. Immutable MemTable을 SSTable로 플러시                    │
│    - 백그라운드 스레드                                       │
│    - 순차 쓰기로 디스크에 저장                               │
│    - 완료 후 WAL 삭제 가능                                   │
│                                                             │
│ 쓰기 성능: O(1) 또는 O(log M) (M = MemTable 크기)           │
│ 모든 디스크 I/O가 순차 → HDD에서도 높은 처리량              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. SSTable (Sorted String Table)

```
┌─────────────────────────────────────────────────────────────┐
│                    SSTable Structure                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐│
│ │                    Data Blocks                          ││
│ │ ┌───────────────────────────────────────────────────┐   ││
│ │ │ Block 1: [k1:v1] [k2:v2] [k3:v3] ...              │   ││
│ │ │ Block 2: [k100:v100] [k101:v101] ...              │   ││
│ │ │ ...                                                │   ││
│ │ └───────────────────────────────────────────────────┘   ││
│ └─────────────────────────────────────────────────────────┘│
│ ┌─────────────────────────────────────────────────────────┐│
│ │                    Index Block                          ││
│ │ [Block1_key → offset] [Block2_key → offset] ...        ││
│ └─────────────────────────────────────────────────────────┘│
│ ┌─────────────────────────────────────────────────────────┐│
│ │                    Bloom Filter                         ││
│ │ 키 존재 여부 빠른 확인 (false positive 가능)            ││
│ └─────────────────────────────────────────────────────────┘│
│ ┌─────────────────────────────────────────────────────────┐│
│ │                    Footer                               ││
│ │ Index offset, Filter offset, Magic number              ││
│ └─────────────────────────────────────────────────────────┘│
│                                                             │
│ 특징:                                                        │
│ - 불변(Immutable): 한 번 쓰면 변경 불가                     │
│ - 정렬됨: 키 순서로 정렬                                    │
│ - 블록 단위 압축                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4. 읽기 경로 (Read Path)

```
┌─────────────────────────────────────────────────────────────┐
│                    Read Path                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Point Query (key = 'X')                                     │
│                                                             │
│ 1. MemTable 검색                    → 있으면 반환           │
│ 2. Immutable MemTable 검색          → 있으면 반환           │
│ 3. Level 0 SSTable들 검색           → 모두 확인 (오버랩)    │
│ 4. Level 1 SSTable 검색             → 1개만 확인            │
│ 5. Level 2 SSTable 검색             → 1개만 확인            │
│ ...                                                         │
│                                                             │
│ 각 SSTable에서:                                              │
│ a. Bloom Filter 확인 → negative면 스킵                      │
│ b. Index Block으로 Data Block 위치 찾기                     │
│ c. Data Block에서 이진 탐색                                 │
│                                                             │
│ 최악의 경우: 모든 레벨의 SSTable 확인                       │
│ 읽기 증폭 (Read Amplification)                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. Compaction 전략

```
┌─────────────────────────────────────────────────────────────┐
│              Leveled Compaction                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ L0: [SST1] [SST2] [SST3]  (오버랩 허용)                     │
│      ↓                                                      │
│ L1: [A-E] [F-K] [L-P] [Q-Z]  (비오버랩, 10MB)               │
│      ↓                                                      │
│ L2: 10배 더 큰 영역 (100MB)                                 │
│                                                             │
│ Compaction 과정:                                             │
│ 1. L0 SSTable이 임계값 초과                                 │
│ 2. L0의 SSTable과 L1의 오버랩 SSTable 선택                  │
│ 3. Merge Sort로 합치고 새 SSTable 생성                      │
│ 4. 원본 SSTable 삭제                                        │
│                                                             │
│ 장점: 읽기 증폭 낮음 (레벨당 1개 SSTable만 확인)            │
│ 단점: 쓰기 증폭 높음 (같은 데이터 여러 번 재작성)           │
│                                                             │
│ Write Amplification = ~10 × (레벨 수)                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              Tiered (Size-Tiered) Compaction                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Tier 0: [Small] [Small] [Small] [Small]                    │
│                    ↓ (비슷한 크기끼리 합침)                  │
│ Tier 1: [Medium] [Medium]                                   │
│                    ↓                                        │
│ Tier 2: [Large]                                             │
│                                                             │
│ Compaction 과정:                                             │
│ 1. 같은 티어에 SSTable이 N개(보통 4개) 쌓임                 │
│ 2. N개를 합쳐서 다음 티어로                                 │
│                                                             │
│ 장점: 쓰기 증폭 낮음                                        │
│ 단점: 읽기 증폭 높음, 공간 증폭 높음                        │
│                                                             │
│ Cassandra 기본 전략                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              FIFO Compaction                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 시계열 데이터에 적합                                         │
│                                                             │
│ [Oldest] [Old] [Recent] [New]                               │
│    ↓                                                        │
│ 삭제 (TTL 만료)                                             │
│                                                             │
│ Compaction 없이 오래된 파일만 삭제                          │
│ 쓰기 증폭 = 1 (최소)                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6. Write/Read/Space Amplification

```
┌─────────────────────────────────────────────────────────────┐
│              Amplification Factors                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Write Amplification (WA):                                   │
│ = 디스크에 실제 쓴 바이트 / 애플리케이션이 쓴 바이트        │
│ - Leveled: 높음 (~10-30)                                   │
│ - Tiered: 낮음 (~2-4)                                       │
│                                                             │
│ Read Amplification (RA):                                    │
│ = 읽기 당 디스크 I/O 횟수                                   │
│ - Leveled: 낮음 (레벨당 1개)                                │
│ - Tiered: 높음 (티어당 여러 개)                             │
│                                                             │
│ Space Amplification (SA):                                   │
│ = 디스크 사용량 / 실제 데이터 크기                          │
│ - Leveled: 낮음 (~1.1)                                      │
│ - Tiered: 높음 (중복 키 존재)                               │
│                                                             │
│ 트레이드오프: WA ↓ ↔ RA ↑ ↔ SA ↑                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7. Bloom Filter 활용

```
┌─────────────────────────────────────────────────────────────┐
│                    Bloom Filter                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 확률적 자료구조:                                             │
│ - "아마도 있음" 또는 "확실히 없음"                           │
│ - False Positive 가능, False Negative 불가능                │
│                                                             │
│ 구조:                                                        │
│ m비트 배열 + k개의 해시 함수                                │
│                                                             │
│ 삽입: k개 해시 결과 위치를 1로 설정                         │
│ h1(key)=3, h2(key)=7 → bits[3]=1, bits[7]=1                │
│                                                             │
│ 조회: k개 해시 결과 위치가 모두 1인지 확인                  │
│ 모두 1: "아마도 있음" (FP 가능)                              │
│ 하나라도 0: "확실히 없음"                                   │
│                                                             │
│ LSM-Tree에서 활용:                                           │
│ - 각 SSTable에 Bloom Filter 저장                            │
│ - 읽기 시 Bloom Filter로 SSTable 스킵                       │
│ - 없는 키 검색 시 디스크 I/O 대폭 감소                      │
│                                                             │
│ False Positive Rate ≈ (1 - e^(-kn/m))^k                     │
│ 일반적으로 1% 미만으로 설정                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8. RocksDB/LevelDB 분석

```
┌─────────────────────────────────────────────────────────────┐
│                    RocksDB Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ LevelDB 기반 + Facebook 최적화                              │
│                                                             │
│ 주요 기능:                                                   │
│ - Column Families: 논리적 DB 분리                           │
│ - Compaction 스레드 풀                                      │
│ - Rate Limiter: I/O 제한                                    │
│ - Block Cache: 읽기 캐시                                    │
│ - Write Buffer Manager: 메모리 관리                         │
│ - Universal Compaction: Tiered와 Leveled 혼합               │
│                                                             │
│ 튜닝 옵션:                                                   │
│ - write_buffer_size: MemTable 크기                          │
│ - max_write_buffer_number: MemTable 개수                    │
│ - level0_file_num_compaction_trigger: L0 compaction 트리거  │
│ - max_bytes_for_level_base: L1 크기                         │
│ - max_bytes_for_level_multiplier: 레벨 간 비율              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 실무 적용

### RocksDB 설정 예시

```cpp
// C++ RocksDB 설정
rocksdb::Options options;

// MemTable 설정
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB
options.max_write_buffer_number = 3;

// Level 설정
options.level0_file_num_compaction_trigger = 4;
options.max_bytes_for_level_base = 256 * 1024 * 1024;  // 256MB
options.max_bytes_for_level_multiplier = 10;

// Compaction
options.compaction_style = rocksdb::kCompactionStyleLevel;
options.num_levels = 7;

// Bloom Filter
rocksdb::BlockBasedTableOptions table_options;
table_options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10));
options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));
```

### 워크로드별 최적화

```
┌─────────────────────────────────────────────────────────────┐
│ 쓰기 중심 워크로드                                           │
├─────────────────────────────────────────────────────────────┤
│ - Tiered/Universal Compaction                               │
│ - 큰 MemTable (쓰기 버퍼링)                                  │
│ - 높은 L0 compaction trigger                                │
│ - 압축 레벨 낮춤 (CPU 절약)                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 읽기 중심 워크로드                                           │
├─────────────────────────────────────────────────────────────┤
│ - Leveled Compaction                                        │
│ - Block Cache 크게                                          │
│ - Bloom Filter bits 높게 (FP rate 낮춤)                     │
│ - 압축으로 I/O 감소                                         │
└─────────────────────────────────────────────────────────────┘
```

## 참고 자료

- O'Neil et al. "The Log-Structured Merge-Tree (LSM-Tree)" (1996)
- RocksDB Wiki: https://github.com/facebook/rocksdb/wiki
- "Database Internals" (Petrov) - Chapter 7
- CMU 15-445: Storage Models & Compression
