# LSM 트리

LSM-Tree는 랜덤 쓰기를 순차 쓰기로 바꿔 쓰기 처리량을 높이는 자료구조다. RocksDB, LevelDB, Cassandra, HBase 등 여러 NoSQL과 NewSQL 시스템에서 이 구조를 사용한다. 쓰기를 모아서 처리하는 대신 읽을 때 여러 파일을 확인하거나 중복 데이터를 보관해야 하므로, 파일을 합치는 Compaction 전략이 읽기 성능과 공간 효율에도 영향을 준다.

## LSM-Tree 구조

- LSM-Tree Architecture
- Memory
- MemTable (활성), ← 새 쓰기
- Sorted by Key (Red-Black Tree)
- Immutable MemTable, ← 플러시 대기
- ↓ Flush (순차 쓰기)
- Disk (SSTables)
- Level 0: [SST] [SST] [SST], ← 오버랩 가능
- ↓ Compaction
- Level 1: [SST] [SST] [SST] [SST], ← 비오버랩
- ↓ Compaction
- Level 2: [SST][SST][SST][SST]...[SST], ← 10배 크기
- Level N: ...

## 쓰기 경로 (Write Path)

쓰기 요청은 먼저 WAL(Write-Ahead Log)에 순차적으로 기록해 내구성을 확보한다. 이어서 Skip List나 Red-Black Tree 같은 메모리 내 정렬 자료구조인 MemTable에 삽입한 뒤 완료를 응답한다.

MemTable이 가득 차면 더 이상 수정하지 않는 Immutable MemTable로 바꾸고 새 MemTable에서 쓰기를 받는다. 백그라운드 스레드는 기존 MemTable을 SSTable로 플러시한다. 디스크 저장이 끝나면 해당 데이터를 복구하기 위한 WAL도 삭제할 수 있다.

MemTable 크기를 M이라 할 때 쓰기 성능은 O(1) 또는 O(log M)이다. 이 쓰기 경로는 디스크 I/O를 순차적으로 수행하므로 HDD에서도 높은 처리량을 얻을 수 있다.

## SSTable (Sorted String Table)

- SSTable Structure
- Data Blocks
- Block 1: [k1:v1] [k2:v2] [k3:v3] ...
- Block 2: [k100:v100] [k101:v101] ...
- Index Block
- [Block1_key → offset] [Block2_key → offset] ...
- Bloom Filter
- 키 존재 여부 빠른 확인 (false positive 가능)
- Footer
- Index offset, Filter offset, Magic number
- 특징:
- 불변(Immutable): 한 번 쓰면 변경 불가
- 정렬됨: 키 순서로 정렬
- 블록 단위 압축

## 읽기 경로 (Read Path)

- Read Path
- Point Query (key = 'X')
- MemTable 검색, → 있으면 반환
- Immutable MemTable 검색, → 있으면 반환
- Level 0 SSTable들 검색, → 모두 확인 (오버랩)
- Level 1 SSTable 검색, → 1개만 확인
- Level 2 SSTable 검색, → 1개만 확인
- 각 SSTable에서:
- a. Bloom Filter 확인 → negative면 스킵
- b. Index Block으로 Data Block 위치 찾기
- c. Data Block에서 이진 탐색
- 최악의 경우: 모든 레벨의 SSTable 확인
- 읽기 증폭 (Read Amplification)

## Compaction 전략

- Leveled Compaction
- L0: [SST1] [SST2] [SST3], (오버랩 허용)
- L1: [A-E] [F-K] [L-P] [Q-Z], (비오버랩, 10MB)
- L2: 10배 더 큰 영역 (100MB)
- Compaction 과정:
- L0 SSTable이 임계값 초과
- L0의 SSTable과 L1의 오버랩 SSTable 선택
- Merge Sort로 합치고 새 SSTable 생성
- 원본 SSTable 삭제
- 장점: 읽기 증폭 낮음 (레벨당 1개 SSTable만 확인)
- 단점: 쓰기 증폭 높음 (같은 데이터 여러 번 재작성)
- Write Amplification = ~10 × (레벨 수)
- Tiered (Size-Tiered) Compaction
- Tier 0: [Small] [Small] [Small] [Small]
- ↓ (비슷한 크기끼리 합침)
- Tier 1: [Medium] [Medium]
- Tier 2: [Large]
- Compaction 과정:
- 같은 티어에 SSTable이 N개(보통 4개) 쌓임
- N개를 합쳐서 다음 티어로
- 장점: 쓰기 증폭 낮음
- 단점: 읽기 증폭 높음, 공간 증폭 높음
- Cassandra 기본 전략
- FIFO Compaction
- 시계열 데이터에 적합
- [Oldest] [Old] [Recent] [New]
- 삭제 (TTL 만료)
- Compaction 없이 오래된 파일만 삭제
- 쓰기 증폭 = 1 (최소)

## Write/Read/Space Amplification

- Amplification Factors
- Write Amplification (WA):
- = 디스크에 실제 쓴 바이트 / 애플리케이션이 쓴 바이트
- Leveled: 높음 (~10-30)
- Tiered: 낮음 (~2-4)
- Read Amplification (RA):
- = 읽기 당 디스크 I/O 횟수
- Leveled: 낮음 (레벨당 1개)
- Tiered: 높음 (티어당 여러 개)
- Space Amplification (SA):
- = 디스크 사용량 / 실제 데이터 크기
- Leveled: 낮음 (~1.1)
- Tiered: 높음 (중복 키 존재)
- 트레이드오프: WA ↓ ↔ RA ↑ ↔ SA ↑

## Bloom Filter 활용

Bloom Filter는 키에 대해 "아마도 있음" 또는 "확실히 없음"을 답하는 확률적 자료구조다. False Positive는 가능하지만 False Negative는 없으므로, 없다는 결과가 나오면 해당 SSTable을 읽지 않아도 된다. LSM-Tree는 각 SSTable에 이 필터를 두어 특히 존재하지 않는 키를 검색할 때 디스크 I/O를 줄인다.

필터는 m비트 배열과 k개의 해시 함수로 구성된다. 삽입할 때는 각 해시 결과가 가리키는 비트를 1로 설정한다. 예를 들어 `h1(key)=3`, `h2(key)=7`이면 `bits[3]=1`, `bits[7]=1`로 기록한다.

조회할 때 같은 위치를 확인해 하나라도 0이면 "확실히 없음"을 반환한다. 모두 1이어도 다른 키가 만든 비트일 수 있으므로 "아마도 있음"에 그친다. False Positive Rate는 `≈ (1 - e^(-kn/m))^k`이며 일반적으로 1% 미만으로 설정한다.

## RocksDB/LevelDB 분석

- RocksDB Architecture
- LevelDB 기반 + Facebook 최적화
- 주요 기능:
- Column Families: 논리적 DB 분리
- Compaction 스레드 풀
- Rate Limiter: I/O 제한
- Block Cache: 읽기 캐시
- Write Buffer Manager: 메모리 관리
- Universal Compaction: Tiered와 Leveled 혼합
- 튜닝 옵션:
- write_buffer_size: MemTable 크기
- max_write_buffer_number: MemTable 개수
- level0_file_num_compaction_trigger: L0 compaction 트리거
- max_bytes_for_level_base: L1 크기
- max_bytes_for_level_multiplier: 레벨 간 비율

## 참고 자료

- O'Neil et al. "The Log-Structured Merge-Tree (LSM-Tree)" (1996)
- RocksDB Wiki: https://github.com/facebook/rocksdb/wiki
- "Database Internals" (Petrov) - Chapter 7
- CMU 15-445: Storage Models & Compression

## 관련 학습

- [힙 파일과 빈 공간 관리](03-힙-파일과-빈-공간-관리.md)
