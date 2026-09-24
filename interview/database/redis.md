# Redis / 레디스

## Redis 기본 개념

### REDIS-001
Redis의 기본 개념과 주요 특징은 무엇인가요?

### REDIS-002
Redis가 메모리 기반 데이터 저장소로서 제공하는 장점은 무엇이며, 이로 인한 단점은 무엇인가요?

### REDIS-003
Redis에서 제공하는 데이터 타입(스트링, 리스트, 셋, 정렬된 셋, 해시 등)을 설명해 주세요.

### REDIS-004
Redis의 키-값 구조와 다른 NoSQL 데이터베이스와의 차이점은 무엇인가요?

## Redis Persistence

### REDIS-005
Redis에서 Persistence를 위해 지원하는 RDB와 AOF 방식의 차이점과 각각의 장단점은 무엇인가요?

## Redis Pub/Sub

### REDIS-006
Redis의 Pub/Sub 기능은 어떻게 동작하며, 이를 활용한 메시징 시스템 구현 사례를 설명해 주세요.

## Redis 클러스터와 고가용성

### REDIS-007
Redis Cluster의 기본 아키텍처와 데이터 샤딩(sharding) 방식을 설명해 주세요.

### REDIS-008
Redis Sentinel의 역할은 무엇이며, Sentinel로 어떻게 고가용성을 보장할 수 있나요?

## Redis 캐시 관리

### REDIS-009
Redis의 캐시 만료(expiration) 정책 설정 방법과, 실제 운영 시 고려해야 할 점은 무엇인가요?

### REDIS-010
Redis의 캐시 eviction 정책(LRU, LFU, TTL 등) 간의 차이점과 선택 기준을 설명해 주세요.

## Redis 트랜잭션

### REDIS-011
Redis의 트랜잭션 기능(MULTI, EXEC, WATCH 등)을 활용하여 동시성 문제를 어떻게 해결할 수 있는지 설명해 주세요.

### REDIS-012
Redis에서 Lua 스크립트를 사용하는 이유와, 스크립팅 기능이 주는 이점은 무엇인가요?

## Redis 메모리 관리

### REDIS-013
Redis의 메모리 관리 전략과, 메모리 부족 시 발생할 수 있는 문제 및 해결 방법을 설명해 주세요.

### REDIS-014
Redis에서 Key 네임스페이스(예: Key prefix)를 사용하는 이유와 장점은 무엇인가요?

## Redis 세션 관리

### REDIS-015
Redis를 활용한 세션 관리 구현의 장점과 고려해야 할 단점은 무엇인가요?

## Redis 복제(Replication)

### REDIS-016
Redis의 데이터 복제(replication) 메커니즘과 이를 이용한 데이터 가용성 확보 방법을 설명해 주세요.

### REDIS-017
Redis에서 데이터 정합성을 보장하기 위한 방법에는 어떤 것이 있으며, 각각의 한계는 무엇인가요?

## Redis 성능 최적화

### REDIS-018
Redis 성능 최적화를 위해 고려해야 할 주요 설정과 모니터링 도구에는 어떤 것이 있나요?

## Redis vs Memcached

### REDIS-019
Redis와 Memcached의 차이점 및 각 솔루션의 장단점을 설명해 주세요.

## Redis 보안

### REDIS-020
Redis를 운영할 때 데이터 보안 및 접근 제어는 어떻게 구현할 수 있나요?

## Redis 메모리 단편화

### REDIS-021
Redis 사용 시 발생할 수 있는 메모리 단편화 문제와 이를 완화하기 위한 전략은 무엇인가요?

## Redis 키 만료

### REDIS-022
Redis의 키 만료(expire) 기능이 내부적으로 어떻게 구현되는지, 그리고 만료된 데이터를 효율적으로 처리하는 방법은 무엇인가요?

## Redis 동기/비동기 복제

### REDIS-023
Redis의 비동기 복제와 동기 복제 방식의 차이점, 그리고 각 방식의 선택 기준을 설명해 주세요.

## Redis Optimistic Locking

### REDIS-024
Redis의 WATCH 명령어를 활용한 Optimistic Locking 메커니즘은 어떻게 동작하나요?

## Redis Cluster Resharding

### REDIS-025
Redis Cluster에서 데이터 재분배(Resharding)를 수행할 때의 절차와 주의할 점은 무엇인가요?

## Redis ACID

### REDIS-026
Redis의 Multi/Exec 트랜잭션이 ACID 특성을 어떻게 보장하는지 설명해 주세요.

## Redis Sorted Set

### REDIS-027
정렬된 셋(sorted set)의 내부 구현 방식과, 이를 활용한 대표적인 사례를 설명해 주세요.

## Redis Hash

### REDIS-028
Redis의 해시(Hash) 자료구조를 활용하여 메모리 사용량을 최적화하는 방법은 무엇인가요?

## Redis 모니터링

### REDIS-029
Redis에서 메모리 사용 현황을 모니터링하기 위한 주요 명령어(redis-cli info memory 등)와 그 활용법을 설명해 주세요.

## Redis 최신 기능

### REDIS-030
Redis Modules나 Redis Streams와 같은 최신 기능이 백엔드 시스템에서 어떻게 활용될 수 있는지, 그리고 기존 기능과 비교해 갖는 장점은 무엇인지 설명해 주세요.

## Redis 운영 및 장애 진단

### REDIS-031
Redis 응답은 느린데 SLOWLOG에 기록이 없다면 무엇을 확인해야 하나요?

### REDIS-032
캐시 적중률이 떨어졌을 때 메모리 부족과 TTL 설정 문제를 어떤 지표로 구분하나요?

### REDIS-033
Redis의 `used_memory`와 RSS가 다른 이유는 무엇이며, 백그라운드 저장 중 메모리 사용량은 왜 늘어날 수 있나요?

### REDIS-034
여러 서비스가 Redis를 공유할 때 ACL로 명령과 키 접근 권한을 어떻게 제한하나요?
