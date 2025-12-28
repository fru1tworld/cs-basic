# RabbitMQ

## 목차
1. [AMQP 프로토콜](#1-amqp-프로토콜)
2. [Exchange 타입](#2-exchange-타입)
3. [Queue와 Binding](#3-queue와-binding)
4. [메시지 보장](#4-메시지-보장)
5. [Dead Letter Queue](#5-dead-letter-queue)
6. [클러스터링과 고가용성](#6-클러스터링과-고가용성)
---

## 1. AMQP 프로토콜

### 1.1 AMQP란?

AMQP(Advanced Message Queuing Protocol)는 메시지 지향 미들웨어를 위한 개방형 표준 애플리케이션 계층 프로토콜이다. RabbitMQ는 AMQP 0-9-1을 기본 프로토콜로 사용하며, AMQP 1.0, MQTT, STOMP도 지원한다.

### 1.2 AMQP 0-9-1 vs AMQP 1.0

| 구분 | AMQP 0-9-1 | AMQP 1.0 |
|------|-----------|----------|
| **표준화** | RabbitMQ 자체 스펙 | ISO/IEC 19464, OASIS 표준 |
| **토폴로지** | Broker 중심 (Exchange, Queue, Binding) | 피어-투-피어 기반 |
| **라우팅** | 브로커가 라우팅 담당 | 애플리케이션이 라우팅 담당 |
| **기본 포트** | 5672 | 5672 (동일) |

### 1.3 AMQP 0-9-1 모델 핵심 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                         AMQP Broker                             │
│  ┌──────────┐    Binding    ┌──────────┐                       │
│  │ Exchange │──────────────▶│  Queue   │──────▶ Consumer       │
│  └──────────┘               └──────────┘                       │
│       ▲                                                         │
│       │                                                         │
│    Publish                                                      │
└───────┼─────────────────────────────────────────────────────────┘
        │
   Producer
```

**핵심 흐름:**
1. Producer가 메시지를 Exchange에 발행
2. Exchange는 Binding 규칙에 따라 Queue로 메시지 라우팅
3. Consumer가 Queue에서 메시지 소비

### 1.4 AMQP 프레임 구조

```
┌────────────────────────────────────────────────────┐
│  Frame = Frame Header + Frame Payload + Frame End  │
├────────────────────────────────────────────────────┤
│  Type (1B) │ Channel (2B) │ Size (4B) │ Payload    │
└────────────────────────────────────────────────────┘

Frame Types:
- Method Frame (1): AMQP 명령
- Content Header Frame (2): 메시지 속성
- Body Frame (3): 메시지 본문
- Heartbeat Frame (8): 연결 상태 확인
```

### 1.5 Connection과 Channel

```python
import pika

# Connection: TCP 연결 (비용이 큼)
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)

# Channel: Connection 내의 가상 연결 (경량)
# 하나의 Connection에서 여러 Channel 생성 가능
channel = connection.channel()

# 권장: 스레드당 하나의 Channel 사용
# Connection은 애플리케이션당 1-2개로 제한
```

**Connection vs Channel 사용 가이드:**

| 구분 | Connection | Channel |
|------|-----------|---------|
| **생성 비용** | 높음 (TCP 핸드셰이크) | 낮음 (가상 연결) |
| **권장 개수** | 애플리케이션당 1-2개 | 스레드당 1개 |
| **스레드 안전** | 스레드 안전 | 스레드 안전하지 않음 |

---

## 2. Exchange 타입

Exchange는 Producer로부터 받은 메시지를 적절한 Queue로 라우팅하는 역할을 한다.

### 2.1 Direct Exchange

가장 단순한 형태로, **routing key가 정확히 일치**하는 Queue로 메시지를 전달한다.

```
Producer ──[routing_key: "error"]──▶ Direct Exchange
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
            Queue A                  Queue B                Queue C
         [binding: error]         [binding: info]        [binding: error]
              ✓                        ✗                       ✓
```

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Direct Exchange 선언
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

# Queue 선언 및 바인딩
channel.queue_declare(queue='error_queue')
channel.queue_bind(
    exchange='direct_logs',
    queue='error_queue',
    routing_key='error'  # 정확히 'error' routing key만 수신
)

# 메시지 발행
channel.basic_publish(
    exchange='direct_logs',
    routing_key='error',  # 이 routing key로 라우팅
    body='This is an error message'
)
```

### 2.2 Fanout Exchange

**모든 바인딩된 Queue에 메시지를 브로드캐스트**한다. Routing key를 무시한다.

```
Producer ──[any routing_key]──▶ Fanout Exchange
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
               Queue A            Queue B           Queue C
                 ✓                  ✓                 ✓

※ 모든 바인딩된 큐에 메시지 전달 (브로드캐스트)
```

```python
# Fanout Exchange 선언
channel.exchange_declare(exchange='broadcast', exchange_type='fanout')

# 여러 Queue 바인딩 - routing_key는 무시됨
channel.queue_bind(exchange='broadcast', queue='queue_1')
channel.queue_bind(exchange='broadcast', queue='queue_2')
channel.queue_bind(exchange='broadcast', queue='queue_3')

# 메시지 발행 - routing_key는 무시됨
channel.basic_publish(
    exchange='broadcast',
    routing_key='',  # Fanout에서는 의미 없음
    body='Broadcast message to all queues'
)
```

**사용 사례:**
- 실시간 알림 시스템
- 로그 분산 처리
- 캐시 무효화 브로드캐스트

### 2.3 Topic Exchange

**패턴 매칭**을 사용하여 routing key를 기반으로 메시지를 라우팅한다.

**패턴 문법:**
- `*` (star): 정확히 하나의 단어 매칭
- `#` (hash): 0개 이상의 단어 매칭
- `.` (dot): 단어 구분자

```
Producer                                Topic Exchange
    │                                         │
    ├──[stock.us.nyse]────────────────────────┤
    │                                         │
    ├──[stock.eu.lse]─────────────────────────┤
    │                                         │
    └──[weather.us.east]──────────────────────┤
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
            Queue US Stocks            Queue All EU              Queue Weather
          [stock.us.*] ✓            [*.eu.#] ✓               [weather.#] ✓
```

```python
# Topic Exchange 선언
channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

# 패턴 바인딩 예시
# 모든 에러 로그 수신
channel.queue_bind(
    exchange='topic_logs',
    queue='all_errors',
    routing_key='*.error'  # auth.error, payment.error 등
)

# auth 서비스의 모든 로그 수신
channel.queue_bind(
    exchange='topic_logs',
    queue='auth_logs',
    routing_key='auth.#'  # auth.info, auth.error, auth.debug.trace 등
)

# 정확한 패턴 매칭
channel.queue_bind(
    exchange='topic_logs',
    queue='critical_payment',
    routing_key='payment.error.critical'
)

# 메시지 발행
channel.basic_publish(
    exchange='topic_logs',
    routing_key='auth.error',
    body='Authentication failed'
)
```

### 2.4 Headers Exchange

**메시지 헤더 속성**을 기반으로 라우팅한다. Routing key를 사용하지 않고 헤더 값으로 매칭한다.

```python
# Headers Exchange 선언
channel.exchange_declare(exchange='headers_exchange', exchange_type='headers')

# 헤더 기반 바인딩
channel.queue_bind(
    exchange='headers_exchange',
    queue='pdf_queue',
    arguments={
        'x-match': 'all',  # 'all': 모든 헤더 일치, 'any': 하나라도 일치
        'format': 'pdf',
        'type': 'report'
    }
)

# 메시지 발행 (헤더 포함)
properties = pika.BasicProperties(
    headers={
        'format': 'pdf',
        'type': 'report'
    }
)
channel.basic_publish(
    exchange='headers_exchange',
    routing_key='',  # Headers Exchange에서는 사용 안 함
    body='PDF Report Content',
    properties=properties
)
```

### 2.5 Exchange 타입 비교

| Exchange 타입 | 라우팅 기준 | 사용 사례 |
|--------------|-----------|----------|
| **Direct** | 정확한 routing key 일치 | 작업 분배, 로그 레벨별 처리 |
| **Fanout** | 모든 바인딩 Queue로 전달 | 브로드캐스트, 알림 |
| **Topic** | 패턴 매칭 (*, #) | 멀티테넌트, 지역별 라우팅 |
| **Headers** | 헤더 속성 매칭 | 복잡한 라우팅 규칙 |

---

## 3. Queue와 Binding

### 3.1 Queue 선언

```python
# 기본 Queue 선언
channel.queue_declare(
    queue='task_queue',
    durable=True,           # 브로커 재시작 후에도 Queue 유지
    exclusive=False,        # 다른 연결에서도 접근 가능
    auto_delete=False,      # 모든 Consumer 연결 해제 시 자동 삭제 안 함
    arguments={
        'x-message-ttl': 60000,           # 메시지 TTL (60초)
        'x-max-length': 10000,            # 최대 메시지 수
        'x-max-length-bytes': 1048576,    # 최대 바이트 (1MB)
        'x-overflow': 'reject-publish',   # 초과 시 동작
        'x-queue-type': 'quorum'          # Quorum Queue 사용
    }
)
```

### 3.2 Queue 타입 비교

RabbitMQ 4.0부터 Classic Mirrored Queue가 제거되고, Quorum Queue가 고가용성의 표준이 되었다.

| 구분 | Classic Queue | Quorum Queue | Stream |
|------|--------------|--------------|--------|
| **복제** | 미지원 (4.0+) | Raft 기반 복제 | 복제 지원 |
| **성능** | 높음 | 중간 | 매우 높음 |
| **메시지 순서** | 보장 | 보장 | 보장 |
| **재시도** | 지원 | 지원 | 제한적 |
| **사용 사례** | 단일 노드, 임시 데이터 | 중요 비즈니스 데이터 | 대용량 로그 스트리밍 |

### 3.3 Binding 설정

```python
# 기본 바인딩
channel.queue_bind(
    exchange='orders',
    queue='order_processing',
    routing_key='order.created'
)

# 다중 바인딩 (하나의 Queue에 여러 routing key)
routing_keys = ['order.created', 'order.updated', 'order.cancelled']
for key in routing_keys:
    channel.queue_bind(
        exchange='orders',
        queue='order_processing',
        routing_key=key
    )

# 바인딩 해제
channel.queue_unbind(
    exchange='orders',
    queue='order_processing',
    routing_key='order.cancelled'
)
```

### 3.4 메시지 속성 (Properties)

```python
from datetime import datetime

properties = pika.BasicProperties(
    content_type='application/json',
    content_encoding='utf-8',
    delivery_mode=2,            # 2 = persistent (디스크 저장)
    priority=5,                 # 0-9, 높을수록 우선순위 높음
    correlation_id='abc123',    # 요청-응답 매칭용
    reply_to='response_queue',  # 응답을 받을 Queue
    expiration='60000',         # 메시지 TTL (밀리초)
    message_id='msg-001',       # 고유 메시지 ID
    timestamp=int(datetime.now().timestamp()),
    type='order.created',       # 메시지 타입
    user_id='guest',            # 발행자 ID
    app_id='order-service',     # 애플리케이션 ID
    headers={                   # 커스텀 헤더
        'version': '1.0',
        'source': 'api'
    }
)

channel.basic_publish(
    exchange='orders',
    routing_key='order.created',
    body=json.dumps(order_data),
    properties=properties
)
```

---

## 4. 메시지 보장

### 4.1 Consumer Acknowledgement (ACK)

Consumer가 메시지를 성공적으로 처리했음을 브로커에 알리는 메커니즘이다.

```python
def callback(ch, method, properties, body):
    try:
        # 메시지 처리
        process_message(body)

        # 수동 ACK - 처리 성공
        ch.basic_ack(delivery_tag=method.delivery_tag)

    except RecoverableError:
        # NACK with requeue - 재처리 가능한 오류
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=True  # Queue에 다시 넣기
        )

    except UnrecoverableError:
        # NACK without requeue - DLQ로 이동
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=False  # DLQ로 이동 (DLX 설정 시)
        )

# auto_ack=False로 수동 ACK 모드 설정
channel.basic_consume(
    queue='task_queue',
    on_message_callback=callback,
    auto_ack=False  # 중요: 반드시 False로 설정
)
```

**ACK 모드 비교:**

| 모드 | 설정 | 장점 | 단점 |
|-----|------|-----|------|
| **Auto ACK** | `auto_ack=True` | 간단, 높은 처리량 | 메시지 손실 위험 |
| **Manual ACK** | `auto_ack=False` | 안정성, 재처리 가능 | 구현 복잡도 증가 |

### 4.2 Publisher Confirms

네트워크 장애나 브로커 문제로 인해 메시지 발행 실패를 감지하고 처리하는 메커니즘이다.

```python
# Publisher Confirms 활성화
channel.confirm_delivery()

# 동기 방식 - 개별 확인
try:
    channel.basic_publish(
        exchange='orders',
        routing_key='order.created',
        body=message,
        properties=pika.BasicProperties(delivery_mode=2),
        mandatory=True  # 라우팅 불가 시 반환
    )
    # publish가 성공적으로 확인되면 계속 진행
    print("Message published successfully")
except pika.exceptions.UnroutableError:
    print("Message was returned - no matching queue")
except pika.exceptions.NackError:
    print("Message was NACKed by broker")
```

**비동기 Publisher Confirms (고성능):**

```python
import asyncio
from aio_pika import connect_robust, Message, DeliveryMode

async def publish_with_confirms():
    connection = await connect_robust("amqp://guest:guest@localhost/")
    channel = await connection.channel()

    # Publisher Confirms 활성화
    await channel.set_qos(prefetch_count=100)

    exchange = await channel.declare_exchange('orders', 'direct')

    # 배치 발행
    confirmations = []
    for i in range(1000):
        message = Message(
            body=f"Order {i}".encode(),
            delivery_mode=DeliveryMode.PERSISTENT
        )
        # 발행과 동시에 확인 Future 수집
        confirmation = await exchange.publish(
            message,
            routing_key='order.created'
        )
        confirmations.append(confirmation)

    # 모든 확인 대기
    await asyncio.gather(*confirmations)
    print("All messages confirmed")
```

### 4.3 메시지 지속성 (Persistence)

완전한 메시지 보장을 위한 3가지 조건:

```python
# 1. Queue를 durable로 선언
channel.queue_declare(queue='persistent_queue', durable=True)

# 2. 메시지를 persistent로 발행
channel.basic_publish(
    exchange='',
    routing_key='persistent_queue',
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=2  # persistent
    )
)

# 3. Publisher Confirms 사용
channel.confirm_delivery()
```

### 4.4 Prefetch와 Fair Dispatch

```python
# Consumer별 prefetch 설정
channel.basic_qos(
    prefetch_count=10,   # Consumer당 미확인 메시지 최대 개수
    prefetch_size=0,     # 바이트 기반 제한 (0 = 무제한)
    global_qos=False     # False: Consumer별, True: 채널 전체
)
```

**Prefetch 설정 가이드:**

| 시나리오 | 권장 prefetch_count | 이유 |
|---------|-------------------|------|
| 빠른 처리, 균등 분배 | 1 | 공정한 분배, 느린 Consumer 보호 |
| 중간 처리량 | 10-50 | 처리량과 공정성 균형 |
| 높은 처리량 | 100-250 | 최대 처리량, 네트워크 오버헤드 감소 |
| 배치 처리 | 배치 크기 | 배치 단위로 처리 |

---

## 5. Dead Letter Queue

### 5.1 Dead Letter Exchange (DLX) 개념

메시지가 "죽은 편지"가 되는 경우:
1. Consumer가 `basic.reject` 또는 `basic.nack`으로 메시지 거부 (requeue=false)
2. 메시지 TTL 만료
3. Queue 최대 길이 초과

```
┌─────────────┐    reject/expire    ┌─────────────┐    ┌─────────────┐
│ Main Queue  │───────────────────▶│     DLX     │───▶│    DLQ      │
└─────────────┘                     └─────────────┘    └─────────────┘
```

### 5.2 DLX 설정

```python
# 1. Dead Letter Exchange 선언
channel.exchange_declare(exchange='dlx', exchange_type='direct')

# 2. Dead Letter Queue 선언
channel.queue_declare(queue='dead_letter_queue', durable=True)

# 3. DLQ를 DLX에 바인딩
channel.queue_bind(
    exchange='dlx',
    queue='dead_letter_queue',
    routing_key='dead-letter'
)

# 4. 메인 Queue 선언 (DLX 연결)
channel.queue_declare(
    queue='main_queue',
    durable=True,
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'dead-letter',
        'x-message-ttl': 30000  # 30초 후 DLQ로 이동
    }
)
```

### 5.3 재시도 패턴 구현

```python
import json
import time

def process_with_retry(ch, method, properties, body):
    headers = properties.headers or {}
    retry_count = headers.get('x-retry-count', 0)
    max_retries = 3

    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)

    except RecoverableError as e:
        if retry_count < max_retries:
            # 재시도 큐로 발행 (지연 후 재처리)
            retry_headers = {
                'x-retry-count': retry_count + 1,
                'x-original-error': str(e)
            }

            # 지수 백오프 지연
            delay = (2 ** retry_count) * 1000  # 1s, 2s, 4s

            ch.basic_publish(
                exchange='retry_exchange',
                routing_key='retry',
                body=body,
                properties=pika.BasicProperties(
                    headers=retry_headers,
                    expiration=str(delay)  # 지연 시간
                )
            )
            ch.basic_ack(delivery_tag=method.delivery_tag)
        else:
            # 최대 재시도 초과 - DLQ로 이동
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=False
            )
```

### 5.4 At-Least-Once Dead Lettering (Quorum Queue)

RabbitMQ 3.10+에서 Quorum Queue를 사용할 때 DLX의 신뢰성을 보장한다.

```python
# Quorum Queue with at-least-once dead lettering
channel.queue_declare(
    queue='reliable_queue',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'dead-letter',
        'x-dead-letter-strategy': 'at-least-once'  # 핵심 설정
    }
)
```

---

## 6. 클러스터링과 고가용성

### 6.1 RabbitMQ 클러스터 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                     RabbitMQ Cluster                            │
│                                                                 │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│  │  Node 1  │◀────▶│  Node 2  │◀────▶│  Node 3  │             │
│  │ (Leader) │      │(Follower)│      │(Follower)│             │
│  └──────────┘      └──────────┘      └──────────┘             │
│       │                 │                 │                    │
│       └─────────────────┼─────────────────┘                    │
│                         │                                      │
│              Erlang Distribution Protocol                      │
│                   (포트: 25672)                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Quorum Queue (권장)

RabbitMQ 4.0부터 Classic Mirrored Queue가 제거되고, Quorum Queue가 고가용성의 표준이 되었다.

```python
# Quorum Queue 선언
channel.queue_declare(
    queue='ha_queue',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',
        'x-quorum-initial-group-size': 3,  # 복제본 수
        'x-delivery-limit': 5               # 최대 재전송 횟수
    }
)
```

**Quorum Queue 특징:**

| 특성 | 설명 |
|-----|------|
| **복제 방식** | Raft 합의 알고리즘 |
| **리더 선출** | 자동 (과반수 기반) |
| **데이터 안전성** | 과반수 노드에 쓰기 완료 후 ACK |
| **장애 허용** | (N/2) - 1 노드 장애 허용 |

### 6.3 클러스터 노드 타입

```yaml
# rabbitmq.conf
# Disc 노드 (기본) - 메타데이터를 디스크에 저장
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@node1
cluster_formation.classic_config.nodes.2 = rabbit@node2
cluster_formation.classic_config.nodes.3 = rabbit@node3

# RAM 노드 - 메타데이터를 메모리에만 저장 (빠르지만 재시작 시 손실)
# 주의: 최소 1개의 Disc 노드 필요
```

### 6.4 로드 밸런싱

```nginx
# HAProxy 설정 예시
frontend rabbitmq_frontend
    bind *:5672
    mode tcp
    default_backend rabbitmq_backend

backend rabbitmq_backend
    mode tcp
    balance roundrobin

    # 헬스체크
    option tcp-check

    server rabbit1 192.168.1.1:5672 check inter 5s rise 2 fall 3
    server rabbit2 192.168.1.2:5672 check inter 5s rise 2 fall 3
    server rabbit3 192.168.1.3:5672 check inter 5s rise 2 fall 3
```

### 6.5 네트워크 파티션 처리

```yaml
# rabbitmq.conf
# 파티션 처리 전략

# pause_minority: 소수 파티션 일시 중지 (권장)
cluster_partition_handling = pause_minority

# autoheal: 자동 복구 (데이터 손실 가능)
# cluster_partition_handling = autoheal

# ignore: 수동 처리 필요
# cluster_partition_handling = ignore
```

### 6.6 Federation과 Shovel

**Federation** - 지리적으로 분산된 클러스터 간 메시지 복제:

```bash
# Federation 플러그인 활성화
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management
```

```python
# Federation upstream 설정 (HTTP API)
import requests

federation_policy = {
    "pattern": "^federated\\.",
    "definition": {
        "federation-upstream-set": "all"
    },
    "apply-to": "exchanges"
}

requests.put(
    "http://localhost:15672/api/policies/%2F/federation",
    json=federation_policy,
    auth=('guest', 'guest')
)
```

---

## 참고 자료

- [RabbitMQ 공식 문서](https://www.rabbitmq.com/docs)
- [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [Clustering Guide](https://www.rabbitmq.com/docs/clustering)
