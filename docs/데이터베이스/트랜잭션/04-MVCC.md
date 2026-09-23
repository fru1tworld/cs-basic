# MVCC

MVCC(Multi-Version Concurrency Control)는 데이터의 여러 버전을 유지하고 각 트랜잭션에 특정 시점의 스냅샷을 제공한다. 읽는 쪽이 이전 버전을 볼 수 있으므로 읽기와 쓰기가 서로를 기다리는 일을 줄일 수 있다. PostgreSQL과 MySQL InnoDB 모두 이 방식을 사용하지만, 이전 버전을 저장하는 위치와 정리하는 방법은 다르다.

## MVCC 개념

- MVCC는 "Reader는 Writer를 블로킹하지 않고, Writer도 Reader를 블로킹하지 않는" 동시성 제어 방식임.

- Transaction Timeline:
- Time →, →
- T1:, [BEGIN] [UPDATE row] [COMMIT]
- Old Version (xmin=100, xmax=101)
- New Version (xmin=101, xmax=∞)
- T2:, [BEGIN] [SELECT] [COMMIT]
- Sees: depends on isolation level

- MVCC는 데이터의 여러 버전을 유지하여 읽기와 쓰기가 서로 블로킹하지 않도록 함.

- MVCC 동작 방식
- Time →
- T1(xid=100):, [UPDATE row] [COMMIT]
- Row Versions:
- v1: {data: 'old', xmin:50, xmax:100}
- v2: {data: 'new', xmin:100, xmax:∞}
- T2(xid=101):, [SELECT]
- (T1 커밋 전: v1 읽음)
- (T1 커밋 후: v2 읽음) - 격리수준에 따라 다름

## Tuple 구조와 xmin/xmax

```sql
-- 시스템 컬럼 확인 (숨겨진 MVCC 정보)
SELECT xmin, xmax, ctid, * FROM users LIMIT 5;

-- xmin: 튜플을 생성한 트랜잭션 ID
-- xmax: 튜플을 삭제/수정한 트랜잭션 ID (0이면 활성 상태)
-- ctid: 물리적 위치 (page, offset)
```

- MVCC 동작 방식:

- Page (8KB)
- Tuple 1: { xmin=100, xmax=0,, data="Alice" }, ← 활성
- Tuple 2: { xmin=100, xmax=105, data="Bob" }, ← 삭제됨
- Tuple 3: { xmin=105, xmax=0,, data="Bobby" }, ← 새 버전
- UPDATE 시:
- 기존 Tuple의 xmax를 현재 트랜잭션 ID로 설정
- 새로운 Tuple 생성 (xmin = 현재 트랜잭션 ID)
- 기존 Tuple은 그대로 유지 (다른 트랜잭션이 읽을 수 있음)

## Visibility Check

```sql
-- 트랜잭션 가시성 규칙
/*
튜플이 보이려면:
1. xmin이 커밋된 트랜잭션이어야 함
2. xmin이 현재 스냅샷보다 이전이어야 함
3. xmax가 없거나, 아직 커밋되지 않았거나, 현재 스냅샷 이후여야 함
*/

-- 현재 트랜잭션 ID 확인
SELECT txid_current();

-- 스냅샷 정보 확인
SELECT txid_current_snapshot();
-- 결과: xmin:xmax:xip_list
-- 예: 100:105:102,103
-- 의미: 100~104 범위에서 102,103은 아직 진행 중
```

## VACUUM의 중요성

```sql
-- Dead Tuple 정리
VACUUM users;

-- 통계 정보도 함께 업데이트
VACUUM ANALYZE users;

-- 공간 반환까지 수행 (테이블 락 발생)
VACUUM FULL users;

-- Dead Tuple 비율 확인
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;

-- Autovacuum 설정
SHOW autovacuum_vacuum_threshold;        -- 기본 50
SHOW autovacuum_vacuum_scale_factor;     -- 기본 0.2 (20%)
-- VACUUM 트리거 조건: dead tuples > threshold + scale_factor * table_size
```

## Transaction ID Wraparound 방지

```sql
-- XID는 32비트 (약 42억)
-- Wraparound 방지를 위해 VACUUM이 필수

-- 현재 가장 오래된 XID 확인
SELECT
    datname,
    age(datfrozenxid) as xid_age,
    datfrozenxid
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Freeze 상태 확인
-- 20억에 가까워지면 위험
-- autovacuum_freeze_max_age 기본값: 2억
```

## PostgreSQL MVCC

```sql
-- 숨겨진 시스템 컬럼 조회
SELECT xmin, xmax, ctid, * FROM users LIMIT 5;

-- xmin: 이 버전을 생성한 트랜잭션 ID
-- xmax: 이 버전을 삭제/수정한 트랜잭션 ID (0이면 활성)
-- ctid: 물리적 위치 (page, offset)

-- 가시성 판단:
-- 1. xmin이 커밋되었는가?
-- 2. xmin이 현재 스냅샷보다 이전인가?
-- 3. xmax가 없거나 아직 커밋되지 않았는가?
```

## MySQL InnoDB MVCC

```sql
-- InnoDB는 Undo Log를 사용하여 MVCC 구현

-- Undo Log 조회
SELECT * FROM information_schema.INNODB_TRX;

-- Read View (스냅샷) 생성 시점:
-- READ COMMITTED: 각 쿼리마다
-- REPEATABLE READ: 트랜잭션 시작 시

-- Undo Log 정리
-- Purge Thread가 더 이상 필요 없는 버전 정리
```

## MVCC 장단점

- 장점:
- 읽기가 쓰기를 블로킹하지 않음
- 쓰기가 읽기를 블로킹하지 않음
- 높은 동시성 처리 가능

- 단점:
- 여러 버전 저장으로 저장공간 증가
- VACUUM (PostgreSQL) 또는 Purge (MySQL) 필요
- 트랜잭션 ID Wraparound 관리 필요

## MVCC (Multi-Version Concurrency Control)

- 락 없이 동시성을 보장하는 방식임.

- 원리:
- 데이터를 수정할 때 이전 버전을 유지
- 읽기 요청은 자신의 시작 시점 버전을 읽음
- 쓰기만 락 필요

- 시간 →
- T1 시작 (시점: 100)
- T2 시작 (시점: 101)
- T2: UPDATE row (버전 101 생성)
- T1: SELECT row, T2: COMMIT
- (버전 100 읽음)

- 장점: 읽기와 쓰기가 서로 블로킹하지 않음

## MVCC 기본 원리

- MVCC 개념
- 핵심 아이디어:
- 데이터의 여러 버전(version)을 유지
- 각 트랜잭션은 시작 시점의 스냅샷을 봄
- 읽기는 잠금 없이 진행
- 쓰기는 새 버전 생성
- 기존 버전:
- Row: id=1, name="Kim", version=T10
- T20이 수정 후:
- Row v1: id=1, name="Kim", version=T10, (이전 버전)
- Row v2: id=1, name="Lee", version=T20, (새 버전)
- T15(T10 이후, T20 이전 시작)는 v1을 봄
- T25(T20 이후 시작)는 v2를 봄

## PostgreSQL MVCC 구현

- PostgreSQL 튜플 헤더
- HeapTupleHeaderData:
- t_xmin (4B), : 이 튜플을 생성한 트랜잭션 ID
- t_xmax (4B), : 이 튜플을 삭제/수정한 트랜잭션 ID
- t_cid (4B), : 트랜잭션 내 명령 ID
- t_ctid (6B), : 현재 또는 새 버전의 위치
- t_infomask (2B): 상태 플래그
- t_infomask2(2B): 추가 플래그, 컬럼 수
- t_hoff (1B), : 데이터 시작 오프셋
- [NULL bitmap], : NULL 컬럼 비트맵
- [User Data], : 실제 컬럼 데이터
- 가시성 규칙:
- 튜플이 보이려면:
- xmin이 커밋됨 AND 현재 스냅샷 이전
- xmax가 없거나 (0), 커밋 안 됨, 또는 스냅샷 이후

- PostgreSQL 버전 체인:

- PostgreSQL 버전 관리
- UPDATE는 새 튜플을 생성하고 이전 튜플에 xmax 설정
- 초기 상태:
- Page 5, Offset 10:
- xmin=100, xmax=0, data="Kim"
- T200이 UPDATE 실행:
- Page 5, Offset 10: (이전 버전)
- xmin=100, xmax=200, ctid→(5,15), data="Kim"
- Page 5, Offset 15: (새 버전)
- xmin=200, xmax=0, data="Lee"
- ctid는 새 버전 위치를 가리킴 (UPDATE 체인)

## MySQL InnoDB MVCC 구현

- InnoDB MVCC 구조
- 행 구조:
- DB_TRX_ID (6B), : 마지막 수정 트랜잭션 ID
- DB_ROLL_PTR(7B) : Undo 로그 포인터
- DB_ROW_ID (6B), : 숨겨진 Row ID (PK 없을 때)
- [User Data], : 실제 컬럼 데이터
- Undo Log (Rollback Segment):
- 이전 버전 데이터
- 이전 DB_TRX_ID
- 이전 버전 Undo Log 포인터
- 버전 체인 (Undo Log에 저장):
- 현재 행, → Undo1, → Undo2, → Undo3 (오래된 버전)
- (T300), (T200), (T100), (T50)

## 가시성 체크 (Visibility Check)

- PostgreSQL:

- PostgreSQL 가시성 판단
- 스냅샷 구조:
- xmin: 가장 작은 활성 트랜잭션 ID
- xmax: 다음 할당될 트랜잭션 ID
- xip[]: 현재 진행 중인 트랜잭션 ID 목록
- 가시성 규칙 (의사 코드):
- is_visible(tuple, snapshot):
- # xmin 체크
- if tuple.xmin >= snapshot.xmax:
- return FALSE, # 아직 시작 안 한 트랜잭션
- if tuple.xmin in snapshot.xip:
- return FALSE, # 진행 중인 트랜잭션
- if not is_committed(tuple.xmin):
- return FALSE, # 커밋 안 됨
- # xmax 체크
- if tuple.xmax == 0:
- return TRUE, # 삭제 안 됨
- if tuple.xmax >= snapshot.xmax:
- return TRUE, # 삭제자가 나중
- if tuple.xmax in snapshot.xip:
- return TRUE, # 삭제자가 진행 중
- if not is_committed(tuple.xmax):
- return TRUE, # 삭제자가 커밋 안 함
- return FALSE, # 삭제됨

- MySQL InnoDB:

- InnoDB 가시성 판단 (Read View)
- Read View 구조:
- m_low_limit_id: 다음 할당될 트랜잭션 ID
- m_up_limit_id: 가장 작은 활성 트랜잭션 ID
- m_ids[]: 활성 트랜잭션 ID 목록
- m_creator_trx_id: Read View 생성자 트랜잭션
- 가시성 규칙:
- if trx_id < m_up_limit_id:
- return TRUE, # 모든 활성 트랜잭션보다 이전
- if trx_id >= m_low_limit_id:
- return FALSE, # Read View 생성 후 시작
- if trx_id in m_ids:
- return FALSE, # 생성 시점에 활성이었던 트랜잭션
- return TRUE, # 그 외 (이미 커밋됨)
- 안 보이면 → Undo Log에서 이전 버전 찾아서 재확인

## PostgreSQL vs MySQL MVCC 비교

- MVCC 구현 비교
- PostgreSQL, MySQL InnoDB
- 버전 저장 위치, 같은 테이블, Undo Log (별도)
- (힙에 새 튜플), (롤백 세그먼트)
- UPDATE 동작, 새 튜플 삽입 +, In-place 수정 +
- 이전 튜플에 xmax, Undo에 이전값
- 인덱스 영향, 모든 인덱스 갱신, PK만 갱신 (보조
- (HOT 예외), 인덱스는 그대로)
- 정리 메커니즘, VACUUM, Purge Thread
- 공간 재사용, VACUUM 후 가능, 즉시 가능
- (Undo만 정리)
- 장점, 구현 단순,, 공간 효율
- 복구 빠름, 인덱스 효율
- 단점, 테이블 bloat,, Undo Log 유지
- VACUUM 필요, 긴 트랜잭션 문제

## 트랜잭션 ID와 랩어라운드

- PostgreSQL 트랜잭션 ID 관리
- 트랜잭션 ID: 32비트 부호 없는 정수 (약 42억)
- 랩어라운드 문제:
- ID가 2^32에 도달하면 0으로 순환
- "과거"와 "미래" 판단이 뒤집힐 수 있음
- 해결: Freezing
- 오래된 튜플의 xmin을 FrozenTransactionId로 변경
- FrozenXID는 모든 트랜잭션에게 "과거"로 인식
- VACUUM 역할:
- Dead 튜플 정리
- 오래된 xmin을 Freeze
- pg_database.datfrozenxid 갱신
- 경고: 20억 트랜잭션 안에 VACUUM 필요
- autovacuum_freeze_max_age 설정

## Read View 생성 시점

READ COMMITTED에서는 각 SQL 문마다 새 스냅샷 또는 Read View를 만든다. 따라서 같은 트랜잭션 안에서도 문장 사이에 다른 데이터가 보일 수 있다. REPEATABLE READ는 트랜잭션 시작 시 또는 첫 쿼리 시 만든 스냅샷을 트랜잭션 동안 유지한다.

PostgreSQL의 SERIALIZABLE은 SSI를 사용한다. REPEATABLE READ의 스냅샷에 더해 읽기와 쓰기의 의존성을 추적하고 직렬화 충돌을 탐지한다.

## 참고 자료

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/)
- [PostgreSQL Internals - InterDB](https://www.interdb.jp/pg/)
- [Bruce Momjian Presentations](https://momjian.us/main/presentations/internals.html)
- [PostgreSQL Wiki - MVCC](https://wiki.postgresql.org/wiki/MVCC)
- Designing Data-Intensive Applications by Martin Kleppmann

- [PostgreSQL 공식 문서 - MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [MySQL 공식 문서 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [Vlad Mihalcea - MVCC](https://vladmihalcea.com/how-does-mvcc-multi-version-concurrency-control-work/)
- Designing Data-Intensive Applications by Martin Kleppmann
- High Performance MySQL, 3rd Edition

- [MySQL 공식 문서 - InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
- [PostgreSQL 공식 문서 - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)

- PostgreSQL Documentation: MVCC
- MySQL Documentation: InnoDB Multi-Versioning
- "Database Internals" (Petrov) - Chapter 5
- CMU 15-445: Multi-Version Concurrency Control

## 관련 학습

- [격리 수준과 이상 현상](03-격리-수준과-이상-현상.md)
- [페이지와 튜플 배치](../저장-구조/02-페이지와-튜플-배치.md)
- [잠금과 교착 상태](05-잠금과-교착-상태.md)
- [PostgreSQL](../제품과-운영/02-PostgreSQL.md)
- [트랜잭션과 ACID](01-트랜잭션과-ACID.md)
