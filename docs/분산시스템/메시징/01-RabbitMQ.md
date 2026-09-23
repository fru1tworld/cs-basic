# RabbitMQ

## AMQP 프로토콜

### AMQP란?

AMQP(Advanced Message Queuing Protocol)는 메시지 지향 미들웨어를 위한 개방형 표준 애플리케이션 계층 프로토콜이다. RabbitMQ는 AMQP 0-9-1을 기본 프로토콜로 사용하며 AMQP 1.0, MQTT, STOMP도 지원한다. 여기서는 AMQP 0-9-1의 Exchange, Queue, Binding을 중심으로 메시지가 전달되는 과정을 살펴본다.

### AMQP 0-9-1 vs AMQP 1.0

- 구분: 표준화:
  - AMQP 0-9-1: RabbitMQ 자체 스펙
  - AMQP 1.0: ISO/IEC 19464, OASIS 표준
- 구분: 토폴로지:
  - AMQP 0-9-1: Broker 중심 (Exchange, Queue, Binding)
  - AMQP 1.0: 피어-투-피어 기반
- 구분: 라우팅:
  - AMQP 0-9-1: 브로커가 라우팅 담당
  - AMQP 1.0: 애플리케이션이 라우팅 담당
- 구분: 기본 포트:
  - AMQP 0-9-1: 5672
  - AMQP 1.0: 5672 (동일)

### AMQP 0-9-1 모델 핵심 개념

Producer가 Exchange에 메시지를 발행하면 Exchange는 Binding 규칙에 맞는 Queue로 메시지를 보낸다. Consumer는 이렇게 Queue에 들어온 메시지를 받아 처리한다. 따라서 어떤 Queue로 메시지를 전달할지는 Exchange 타입과 Binding 규칙으로 결정한다.

### AMQP 프레임 구조

- Frame = Frame Header + Frame Payload + Frame End
- Type (1B), Channel (2B), Size (4B), Payload
- Frame Types:
- Method Frame (1): AMQP 명령
- Content Header Frame (2): 메시지 속성
- Body Frame (3): 메시지 본문
- Heartbeat Frame (8): 연결 상태 확인

### Connection과 Channel

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

- Connection vs Channel 사용 가이드:

- 구분: 생성 비용:
  - Connection: 높음 (TCP 핸드셰이크)
  - Channel: 낮음 (가상 연결)
- 구분: 권장 개수:
  - Connection: 애플리케이션당 1-2개
  - Channel: 스레드당 1개
- 구분: 스레드 안전:
  - Connection: 스레드 안전
  - Channel: 스레드 안전하지 않음

## Exchange 타입

- Exchange는 Producer로부터 받은 메시지를 적절한 Queue로 라우팅하는 역할을 함.

### Direct Exchange

Direct Exchange는 routing key가 정확히 일치하는 Queue로 메시지를 전달한다. Queue A와 C가 `error`, Queue B가 `info`에 바인딩되어 있다면 `error` 메시지는 A와 C에 전달된다. 아래 코드는 같은 방식으로 `error_queue`를 바인딩한다.

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

### Fanout Exchange

Fanout Exchange는 routing key와 관계없이 바인딩된 모든 Queue에 메시지를 보낸다. 아래 예제에서는 `broadcast`에 발행한 메시지를 `queue_1`, `queue_2`, `queue_3`이 모두 받는다.

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

- 사용 사례:
- 실시간 알림 시스템
- 로그 분산 처리
- 캐시 무효화 브로드캐스트

### Topic Exchange

Topic Exchange는 routing key를 패턴과 비교해 전달할 Queue를 선택한다. 패턴에서는 다음 기호를 사용한다.

- `*` (star): 정확히 하나의 단어 매칭
- `#` (hash): 0개 이상의 단어 매칭
- `.` (dot): 단어 구분자

예를 들어 `stock.us.nyse`는 `stock.us.*`에, `stock.eu.lse`는 `*.eu.#`에, `weather.us.east`는 `weather.#`에 매칭된다. 아래 로그 예제도 같은 규칙으로 서비스와 로그 수준을 구분한다.

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

### Headers Exchange

Headers Exchange는 routing key 대신 메시지 헤더를 비교한다. 아래 예제는 `format=pdf`와 `type=report`가 모두 일치하는 메시지를 `pdf_queue`로 보낸다.

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

### Exchange 타입 비교

- Exchange 타입: Direct:
  - 라우팅 기준: 정확한 routing key 일치
  - 사용 사례: 작업 분배, 로그 레벨별 처리
- Exchange 타입: Fanout:
  - 라우팅 기준: 모든 바인딩 Queue로 전달
  - 사용 사례: 브로드캐스트, 알림
- Exchange 타입: Topic:
  - 라우팅 기준: 패턴 매칭 (\*, #)
  - 사용 사례: 멀티테넌트, 지역별 라우팅
- Exchange 타입: Headers:
  - 라우팅 기준: 헤더 속성 매칭
  - 사용 사례: 복잡한 라우팅 규칙

## Queue와 Binding

### Queue 선언

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

### Queue 타입 비교

- RabbitMQ 4.0부터 Classic Mirrored Queue가 제거되고, Quorum Queue가 고가용성의 표준이 됨.

- 구분: 복제:
  - Classic Queue: 미지원 (4.0+)
  - Quorum Queue: Raft 기반 복제
  - Stream: 복제 지원
- 구분: 성능:
  - Classic Queue: 높음
  - Quorum Queue: 중간
  - Stream: 매우 높음
- 구분: 메시지 순서:
  - Classic Queue: 보장
  - Quorum Queue: 보장
  - Stream: 보장
- 구분: 재시도:
  - Classic Queue: 지원
  - Quorum Queue: 지원
  - Stream: 제한적
- 구분: 사용 사례:
  - Classic Queue: 단일 노드, 임시 데이터
  - Quorum Queue: 중요 비즈니스 데이터
  - Stream: 대용량 로그 스트리밍

### Binding 설정

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

### 메시지 속성 (Properties)

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

## 메시지 보장

### Consumer Acknowledgement (ACK)

Consumer Acknowledgement(ACK)는 Consumer가 메시지를 처리했음을 브로커에 알리는 방식이다. 아래 예제는 처리에 성공한 뒤 ACK를 보내고, 오류가 나면 재처리 가능 여부에 따라 Queue에 다시 넣을지 결정한다.

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

- ACK 모드 비교:

- 모드: Auto ACK:
  - 설정: `auto_ack=True`
  - 장점: 간단, 높은 처리량
  - 단점: 메시지 손실 위험
- 모드: Manual ACK:
  - 설정: `auto_ack=False`
  - 장점: 안정성, 재처리 가능
  - 단점: 구현 복잡도 증가

### Publisher Confirms

Consumer의 처리 결과를 확인하는 ACK와 별개로, Producer도 메시지 발행 결과를 확인해야 한다. Publisher Confirms는 네트워크나 브로커 문제로 발행에 실패했는지 감지하는 데 사용한다.

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

- 비동기 Publisher Confirms (고성능):

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

### 메시지 지속성 (Persistence)

- 완전한 메시지 보장을 위한 3가지 조건:

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

### Prefetch와 Fair Dispatch

```python
# Consumer별 prefetch 설정
channel.basic_qos(
    prefetch_count=10,   # Consumer당 미확인 메시지 최대 개수
    prefetch_size=0,     # 바이트 기반 제한 (0 = 무제한)
    global_qos=False     # False: Consumer별, True: 채널 전체
)
```

- Prefetch 설정 가이드:

- 시나리오: 빠른 처리, 균등 분배:
  - 권장 prefetch_count: 1
  - 이유: 공정한 분배, 느린 Consumer 보호
- 시나리오: 중간 처리량:
  - 권장 prefetch_count: 10-50
  - 이유: 처리량과 공정성 균형
- 시나리오: 높은 처리량:
  - 권장 prefetch_count: 100-250
  - 이유: 최대 처리량, 네트워크 오버헤드 감소
- 시나리오: 배치 처리:
  - 권장 prefetch_count: 배치 크기
  - 이유: 배치 단위로 처리

## Dead Letter Queue

### Dead Letter Exchange (DLX) 개념

- 메시지가 "죽은 편지"가 되는 경우:
- Consumer가 `basic.reject` 또는 `basic.nack`으로 메시지 거부 (requeue=false)
- 메시지 TTL 만료
- Queue 최대 길이 초과

- reject/expire
- Main Queue, →, DLX, →, DLQ

### DLX 설정

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

### 재시도 패턴 구현

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

### At-Least-Once Dead Lettering (Quorum Queue)

- RabbitMQ 3.10+에서 Quorum Queue를 사용할 때 DLX의 신뢰성을 보장함.

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

## 클러스터링과 고가용성

### RabbitMQ 클러스터 아키텍처

- RabbitMQ Cluster
- Node 1, ← →, Node 2, ← →, Node 3
- (Leader), (Follower), (Follower)
- Erlang Distribution Protocol
- (포트: 25672)

### Quorum Queue (권장)

- RabbitMQ 4.0부터 Classic Mirrored Queue가 제거되고, Quorum Queue가 고가용성의 표준이 됨.

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

- Quorum Queue 특징:

- 특성: 복제 방식:
  - 설명: Raft 합의 알고리즘
- 특성: 리더 선출:
  - 설명: 자동 (과반수 기반)
- 특성: 데이터 안전성:
  - 설명: 과반수 노드에 쓰기 완료 후 ACK
- 특성: 장애 허용:
  - 설명: (N/2) - 1 노드 장애 허용

### 클러스터 노드 타입

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

### 로드 밸런싱

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

### 네트워크 파티션 처리

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

### Federation과 Shovel

- Federation - 지리적으로 분산된 클러스터 간 메시지 복제:

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

## 참고 자료

- [RabbitMQ 공식 문서](https://www.rabbitmq.com/docs)
- [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [Clustering Guide](https://www.rabbitmq.com/docs/clustering)

## 관련 학습

- [Kafka](02-Kafka.md)
