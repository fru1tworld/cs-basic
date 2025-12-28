# WebSocket

## 목차
1. [WebSocket 개요](#websocket-개요)
2. [WebSocket 프로토콜 (RFC 6455)](#websocket-프로토콜-rfc-6455)
3. [연결 관리와 핸드셰이크](#연결-관리와-핸드셰이크)
4. [하트비트와 재연결](#하트비트와-재연결)
5. [Socket.io vs 네이티브 WebSocket](#socketio-vs-네이티브-websocket)
6. [실전 구현 예제](#실전-구현-예제)
---

## WebSocket 개요

WebSocket은 클라이언트와 서버 간에 단일 TCP 연결을 통해 **전이중(Full-Duplex) 양방향 통신**을 제공하는 프로토콜입니다. HTTP와 달리 연결을 유지하면서 실시간 데이터 교환이 가능합니다.

### HTTP vs WebSocket 비교

```
HTTP (Half-Duplex):
┌─────────────────────────────────────────────────────────┐
│ Client                           Server                 │
│   │                                 │                   │
│   │─── Request 1 ──────────────────>│                   │
│   │<── Response 1 ──────────────────│                   │
│   │                                 │                   │
│   │─── Request 2 ──────────────────>│                   │
│   │<── Response 2 ──────────────────│                   │
│   │                                 │                   │
│   (매 요청마다 새 연결 또는 Keep-Alive 사용)              │
└─────────────────────────────────────────────────────────┘

WebSocket (Full-Duplex):
┌─────────────────────────────────────────────────────────┐
│ Client                           Server                 │
│   │                                 │                   │
│   │═══ HTTP Upgrade (Handshake) ═══>│                   │
│   │<══ 101 Switching Protocols ════│                   │
│   │                                 │                   │
│   │←───── Message ─────────────────→│  (양방향)         │
│   │←───── Message ─────────────────→│                   │
│   │←───── Message ─────────────────→│                   │
│   │                                 │                   │
│   (단일 TCP 연결 유지, 언제든 양방향 전송)               │
└─────────────────────────────────────────────────────────┘
```

### WebSocket의 장점

| 장점 | 설명 |
|------|------|
| 실시간 통신 | 서버가 클라이언트에 즉시 데이터 푸시 가능 |
| 낮은 지연시간 | 연결 오버헤드 없이 메시지 전송 |
| 효율적인 대역폭 | HTTP 헤더 오버헤드 없음 |
| 양방향 통신 | 클라이언트/서버 모두 언제든 메시지 전송 |

### 사용 사례

- 실시간 채팅 애플리케이션
- 주식/코인 시세 피드
- 온라인 게임
- 협업 도구 (동시 편집)
- 라이브 스포츠 점수판
- IoT 디바이스 통신
- 실시간 알림

---

## WebSocket 프로토콜 (RFC 6455)

RFC 6455는 2011년 IETF에서 발표한 WebSocket 프로토콜 표준입니다.

### 프로토콜 특징

```
URI 스킴:
- ws://  (비암호화, 포트 80)
- wss:// (TLS 암호화, 포트 443)

예시:
ws://example.com/socket
wss://example.com/socket?token=abc123
```

### 프레임 구조

WebSocket은 프레임 기반 프로토콜입니다:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### 프레임 필드 설명

| 필드 | 크기 | 설명 |
|------|------|------|
| FIN | 1 bit | 최종 프래그먼트인지 여부 |
| RSV1-3 | 3 bits | 확장용 예약 비트 |
| Opcode | 4 bits | 프레임 타입 |
| MASK | 1 bit | 페이로드 마스킹 여부 |
| Payload length | 7+ bits | 페이로드 길이 |
| Masking key | 0 or 4 bytes | 마스킹 키 |
| Payload | 가변 | 실제 데이터 |

### Opcode 종류

```
Opcode 값:
0x0 - Continuation Frame (연속 프레임)
0x1 - Text Frame (텍스트 데이터)
0x2 - Binary Frame (바이너리 데이터)
0x3-0x7 - 예약 (비제어 프레임)
0x8 - Close Frame (연결 종료)
0x9 - Ping Frame (연결 확인 요청)
0xA - Pong Frame (Ping 응답)
0xB-0xF - 예약 (제어 프레임)
```

### 마스킹 (Masking)

클라이언트에서 서버로 보내는 모든 프레임은 마스킹되어야 합니다:

```javascript
// 마스킹 알고리즘 (XOR 연산)
function mask(data, maskingKey) {
  const masked = new Uint8Array(data.length);
  for (let i = 0; i < data.length; i++) {
    masked[i] = data[i] ^ maskingKey[i % 4];
  }
  return masked;
}
```

> **마스킹 목적**: 프록시 캐시 오염 공격(Cache Poisoning) 방지. 클라이언트가 악의적인 프레임을 보내 중간 프록시를 속이는 것을 방지합니다.

### Close 상태 코드

```javascript
// 정상 종료 코드
1000 - Normal Closure          // 정상 종료
1001 - Going Away              // 서버 셧다운, 브라우저 페이지 이동

// 에러 코드
1002 - Protocol Error          // 프로토콜 오류
1003 - Unsupported Data        // 지원하지 않는 데이터 타입
1006 - Abnormal Closure        // 비정상 종료 (Close 프레임 없이 끊김)
1007 - Invalid Payload Data    // 잘못된 데이터 형식
1008 - Policy Violation        // 정책 위반
1009 - Message Too Big         // 메시지 크기 초과
1010 - Missing Extension       // 필요한 확장 누락
1011 - Internal Error          // 서버 내부 오류
1015 - TLS Handshake Failure   // TLS 핸드셰이크 실패

// 애플리케이션 예약 범위
3000-3999 - 라이브러리/프레임워크용
4000-4999 - 애플리케이션용
```

---

## 연결 관리와 핸드셰이크

### 핸드셰이크 과정

WebSocket 연결은 HTTP Upgrade 요청으로 시작됩니다:

```
1. 클라이언트 → 서버: HTTP Upgrade 요청
2. 서버 → 클라이언트: 101 Switching Protocols 응답
3. WebSocket 연결 수립
```

### 클라이언트 요청

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Extensions: permessage-deflate
Origin: http://example.com
```

| 헤더 | 필수 | 설명 |
|------|------|------|
| Upgrade | O | "websocket" 고정 |
| Connection | O | "Upgrade" 고정 |
| Sec-WebSocket-Key | O | Base64 인코딩된 16바이트 랜덤 값 |
| Sec-WebSocket-Version | O | 프로토콜 버전 (13) |
| Sec-WebSocket-Protocol | X | 서브프로토콜 |
| Sec-WebSocket-Extensions | X | 확장 협상 |

### 서버 응답

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### Sec-WebSocket-Accept 계산

```javascript
const crypto = require('crypto');

function generateAcceptKey(clientKey) {
  // RFC 6455에 정의된 GUID
  const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

  // SHA-1 해시 후 Base64 인코딩
  return crypto
    .createHash('sha1')
    .update(clientKey + GUID)
    .digest('base64');
}

// 예시
const clientKey = 'dGhlIHNhbXBsZSBub25jZQ==';
const acceptKey = generateAcceptKey(clientKey);
// 결과: 's3pPLMBiTxaQ9kYGzzhZRbK+xOo='
```

### Node.js 서버 구현 (핸드셰이크)

```javascript
const http = require('http');
const crypto = require('crypto');

const server = http.createServer();

server.on('upgrade', (req, socket, head) => {
  // WebSocket 업그레이드 요청 검증
  if (req.headers['upgrade'] !== 'websocket') {
    socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
    return;
  }

  // Accept 키 생성
  const clientKey = req.headers['sec-websocket-key'];
  const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
  const acceptKey = crypto
    .createHash('sha1')
    .update(clientKey + GUID)
    .digest('base64');

  // 핸드셰이크 응답
  const headers = [
    'HTTP/1.1 101 Switching Protocols',
    'Upgrade: websocket',
    'Connection: Upgrade',
    `Sec-WebSocket-Accept: ${acceptKey}`,
    '',
    ''
  ].join('\r\n');

  socket.write(headers);

  // 이제 WebSocket 프레임으로 통신
  socket.on('data', (buffer) => {
    const frame = parseWebSocketFrame(buffer);
    // 프레임 처리...
  });
});

server.listen(8080);
```

### 클라이언트 연결 (브라우저)

```javascript
// 기본 연결
const ws = new WebSocket('wss://example.com/socket');

// 서브프로토콜 지정
const wsWithProtocol = new WebSocket('wss://example.com/socket', ['chat', 'json']);

// 이벤트 핸들러
ws.onopen = (event) => {
  console.log('Connected');
  ws.send('Hello Server!');
};

ws.onmessage = (event) => {
  console.log('Received:', event.data);
};

ws.onerror = (event) => {
  console.error('Error:', event);
};

ws.onclose = (event) => {
  console.log(`Closed: ${event.code} ${event.reason}`);
};

// 연결 상태 확인
// ws.readyState
// 0 - CONNECTING
// 1 - OPEN
// 2 - CLOSING
// 3 - CLOSED
```

---

## 하트비트와 재연결

### 하트비트 (Keep-Alive)

네트워크 연결 상태를 확인하고 유지하기 위한 메커니즘:

```
┌─────────────────────────────────────────────────────────┐
│                    Heartbeat Flow                        │
├─────────────────────────────────────────────────────────┤
│ Server                           Client                  │
│   │                                 │                    │
│   │──── Ping Frame ─────────────────>│                   │
│   │                                 │                    │
│   │<─── Pong Frame ─────────────────│                    │
│   │                                 │                    │
│   │      (pingTimeout 내에 Pong 없으면 연결 종료)         │
└─────────────────────────────────────────────────────────┘
```

### 서버 측 하트비트 구현

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

// 연결된 클라이언트 관리
const clients = new Map();

wss.on('connection', (ws) => {
  // 클라이언트 상태 초기화
  clients.set(ws, { isAlive: true });

  // Pong 응답 수신 시 상태 업데이트
  ws.on('pong', () => {
    const client = clients.get(ws);
    if (client) {
      client.isAlive = true;
    }
  });

  ws.on('close', () => {
    clients.delete(ws);
  });
});

// 주기적으로 Ping 전송
const HEARTBEAT_INTERVAL = 30000; // 30초

const heartbeatInterval = setInterval(() => {
  wss.clients.forEach((ws) => {
    const client = clients.get(ws);

    if (!client || !client.isAlive) {
      // 이전 Ping에 응답하지 않은 경우 연결 종료
      console.log('Client not responding, terminating');
      clients.delete(ws);
      return ws.terminate();
    }

    // 상태 초기화 후 Ping 전송
    client.isAlive = false;
    ws.ping();
  });
}, HEARTBEAT_INTERVAL);

wss.on('close', () => {
  clearInterval(heartbeatInterval);
});
```

### 클라이언트 측 하트비트 구현

```javascript
class WebSocketClient {
  constructor(url, options = {}) {
    this.url = url;
    this.options = {
      heartbeatInterval: 25000,  // 25초
      heartbeatTimeout: 10000,   // 10초
      reconnectInterval: 1000,   // 1초
      maxReconnectInterval: 30000, // 최대 30초
      reconnectDecay: 1.5,       // 지수 백오프 계수
      maxReconnectAttempts: 10,
      ...options
    };

    this.reconnectAttempts = 0;
    this.heartbeatTimer = null;
    this.heartbeatTimeoutTimer = null;

    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectAttempts = 0;
      this.startHeartbeat();
    };

    this.ws.onmessage = (event) => {
      // Pong 메시지 처리 (애플리케이션 레벨)
      if (event.data === 'pong') {
        this.resetHeartbeatTimeout();
        return;
      }
      this.onMessage(event.data);
    };

    this.ws.onclose = (event) => {
      console.log(`Disconnected: ${event.code}`);
      this.stopHeartbeat();
      this.reconnect();
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }

  startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      if (this.ws.readyState === WebSocket.OPEN) {
        // 애플리케이션 레벨 Ping (텍스트 메시지)
        this.ws.send('ping');

        // 타임아웃 설정
        this.heartbeatTimeoutTimer = setTimeout(() => {
          console.log('Heartbeat timeout, closing connection');
          this.ws.close();
        }, this.options.heartbeatTimeout);
      }
    }, this.options.heartbeatInterval);
  }

  resetHeartbeatTimeout() {
    if (this.heartbeatTimeoutTimer) {
      clearTimeout(this.heartbeatTimeoutTimer);
      this.heartbeatTimeoutTimer = null;
    }
  }

  stopHeartbeat() {
    this.resetHeartbeatTimeout();
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  reconnect() {
    if (this.reconnectAttempts >= this.options.maxReconnectAttempts) {
      console.log('Max reconnect attempts reached');
      return;
    }

    // 지수 백오프
    const delay = Math.min(
      this.options.reconnectInterval * Math.pow(
        this.options.reconnectDecay,
        this.reconnectAttempts
      ),
      this.options.maxReconnectInterval
    );

    console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts + 1})`);

    setTimeout(() => {
      this.reconnectAttempts++;
      this.connect();
    }, delay);
  }

  send(data) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(data);
    }
  }

  onMessage(data) {
    // 오버라이드하여 사용
    console.log('Received:', data);
  }

  close() {
    this.stopHeartbeat();
    this.ws.close(1000, 'Normal closure');
  }
}

// 사용 예시
const client = new WebSocketClient('wss://example.com/socket', {
  heartbeatInterval: 30000,
  maxReconnectAttempts: 5
});
```

### 재연결 전략

```javascript
// 1. 즉시 재연결 (네트워크 일시 끊김)
// 2. 지수 백오프 (서버 과부하 방지)
// 3. 최대 시도 횟수 제한
// 4. Jitter 추가 (동시 재연결 방지)

function calculateReconnectDelay(attempt, options) {
  const { baseDelay, maxDelay, decayFactor } = options;

  // 지수 백오프
  let delay = baseDelay * Math.pow(decayFactor, attempt);

  // 최대 지연 제한
  delay = Math.min(delay, maxDelay);

  // Jitter 추가 (0.5 ~ 1.5 배)
  const jitter = 0.5 + Math.random();
  delay = Math.floor(delay * jitter);

  return delay;
}

// 예시: 1초, 2초, 4초, 8초, 16초... (최대 30초)
```

---

## Socket.io vs 네이티브 WebSocket

### 비교 개요

```
┌─────────────────────────────────────────────────────────┐
│                   Socket.io 아키텍처                     │
├─────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────┐ │
│  │              Socket.io Client                       │ │
│  └────────────────────────────────────────────────────┘ │
│                          ↓                              │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Engine.io Layer                        │ │
│  │  (Transport 추상화: WebSocket, HTTP Long-Polling)    │ │
│  └────────────────────────────────────────────────────┘ │
│                          ↓                              │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Socket.io Server                       │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 상세 비교표

| 기능 | 네이티브 WebSocket | Socket.io |
|------|-------------------|-----------|
| **프로토콜** | 표준 WebSocket (RFC 6455) | 자체 프로토콜 (Engine.io) |
| **폴백** | 없음 | HTTP Long-Polling 자동 폴백 |
| **자동 재연결** | 직접 구현 필요 | 내장 (지수 백오프) |
| **하트비트** | 직접 구현 필요 | 내장 |
| **이벤트 기반** | 기본 이벤트만 | 커스텀 이벤트 지원 |
| **네임스페이스** | 없음 | 지원 |
| **룸** | 없음 | 내장 지원 |
| **브로드캐스팅** | 직접 구현 | 내장 |
| **ACK (응답 확인)** | 없음 | 내장 |
| **바이너리 지원** | 지원 | 지원 |
| **오버헤드** | 최소 | 추가 메타데이터 |
| **호환성** | WebSocket 전용 | Socket.io 클라이언트 필수 |
| **패키지 크기** | 0 KB (브라우저 내장) | ~10 KB (gzip) |

### 네이티브 WebSocket 사용

```javascript
// 클라이언트
const ws = new WebSocket('wss://example.com/socket');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'message',
    data: { text: 'Hello!' }
  }));
};

ws.onmessage = (event) => {
  const { type, data } = JSON.parse(event.data);
  switch (type) {
    case 'message':
      console.log('Received message:', data);
      break;
    case 'notification':
      showNotification(data);
      break;
  }
};
```

```javascript
// 서버 (ws 라이브러리)
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// 클라이언트 관리
const clients = new Set();

wss.on('connection', (ws) => {
  clients.add(ws);

  ws.on('message', (message) => {
    const { type, data } = JSON.parse(message);

    switch (type) {
      case 'message':
        // 브로드캐스트 (직접 구현)
        clients.forEach((client) => {
          if (client !== ws && client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({
              type: 'message',
              data
            }));
          }
        });
        break;
    }
  });

  ws.on('close', () => {
    clients.delete(ws);
  });
});
```

### Socket.io 사용

```javascript
// 클라이언트
import { io } from 'socket.io-client';

const socket = io('wss://example.com', {
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
});

socket.on('connect', () => {
  console.log('Connected:', socket.id);
});

// 커스텀 이벤트 전송
socket.emit('chat:message', { text: 'Hello!' }, (ack) => {
  console.log('Message delivered:', ack);
});

// 커스텀 이벤트 수신
socket.on('chat:message', (data) => {
  console.log('Received:', data);
});

// 네임스페이스 사용
const adminSocket = io('wss://example.com/admin');

// 룸 참여 요청
socket.emit('join:room', 'room-123');

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason);
});
```

```javascript
// 서버
const { Server } = require('socket.io');
const io = new Server(3000, {
  cors: {
    origin: 'http://localhost:3000',
    methods: ['GET', 'POST']
  },
  pingInterval: 25000,
  pingTimeout: 20000
});

io.on('connection', (socket) => {
  console.log('New connection:', socket.id);

  // 커스텀 이벤트 핸들러
  socket.on('chat:message', (data, callback) => {
    console.log('Message from', socket.id, ':', data);

    // 같은 룸의 모든 클라이언트에 브로드캐스트
    socket.to('room-123').emit('chat:message', {
      from: socket.id,
      ...data
    });

    // ACK 응답
    callback({ status: 'delivered' });
  });

  // 룸 관리
  socket.on('join:room', (roomId) => {
    socket.join(roomId);
    io.to(roomId).emit('user:joined', { userId: socket.id });
  });

  socket.on('leave:room', (roomId) => {
    socket.leave(roomId);
    io.to(roomId).emit('user:left', { userId: socket.id });
  });

  // 특정 사용자에게 전송
  socket.on('private:message', ({ to, message }) => {
    io.to(to).emit('private:message', {
      from: socket.id,
      message
    });
  });

  socket.on('disconnect', (reason) => {
    console.log('Disconnected:', socket.id, reason);
  });
});

// 네임스페이스
const adminNamespace = io.of('/admin');
adminNamespace.on('connection', (socket) => {
  console.log('Admin connected:', socket.id);
});
```

### Socket.io 특수 기능

```javascript
// 1. 브로드캐스팅 옵션
io.emit('message', data);                    // 모든 클라이언트
socket.broadcast.emit('message', data);       // 발신자 제외 모두
io.to('room').emit('message', data);         // 특정 룸
socket.to('room').emit('message', data);     // 특정 룸 (발신자 제외)
io.except('room').emit('message', data);     // 특정 룸 제외

// 2. 휘발성 메시지 (연결 끊김 시 버림)
socket.volatile.emit('position', { x: 10, y: 20 });

// 3. 압축 비활성화 (이미 압축된 데이터)
socket.compress(false).emit('binary', buffer);

// 4. 타임아웃 설정
socket.timeout(5000).emit('request', data, (err, response) => {
  if (err) {
    console.log('Timeout');
  }
});

// 5. 미들웨어
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (validateToken(token)) {
    next();
  } else {
    next(new Error('Authentication error'));
  }
});
```

### 언제 무엇을 선택할까?

**네이티브 WebSocket:**
- 최소 오버헤드가 중요한 경우
- WebSocket을 지원하는 환경만 대상
- 간단한 통신 요구사항
- 커스텀 프로토콜 구현 필요
- 패키지 크기 최소화 필요

**Socket.io:**
- 다양한 브라우저/환경 지원 필요
- 자동 재연결, 하트비트 필요
- 룸, 네임스페이스 기능 필요
- ACK(응답 확인) 필요
- 빠른 개발이 중요

---

## 실전 구현 예제

### 채팅 서버 (Node.js + ws)

```javascript
// server.js
const WebSocket = require('ws');
const { v4: uuidv4 } = require('uuid');

const wss = new WebSocket.Server({ port: 8080 });

// 연결된 클라이언트와 룸 관리
const clients = new Map();
const rooms = new Map();

wss.on('connection', (ws) => {
  const clientId = uuidv4();
  clients.set(ws, { id: clientId, rooms: new Set() });

  console.log(`Client connected: ${clientId}`);

  // 연결 성공 알림
  send(ws, {
    type: 'connected',
    payload: { clientId }
  });

  ws.on('message', (message) => {
    try {
      const { type, payload } = JSON.parse(message);
      handleMessage(ws, type, payload);
    } catch (error) {
      send(ws, { type: 'error', payload: { message: 'Invalid message format' } });
    }
  });

  ws.on('close', () => {
    const client = clients.get(ws);
    if (client) {
      // 참여한 룸에서 퇴장 처리
      client.rooms.forEach((roomId) => {
        leaveRoom(ws, roomId);
      });
      clients.delete(ws);
      console.log(`Client disconnected: ${client.id}`);
    }
  });

  ws.on('pong', () => {
    const client = clients.get(ws);
    if (client) {
      client.isAlive = true;
    }
  });
});

function handleMessage(ws, type, payload) {
  const client = clients.get(ws);

  switch (type) {
    case 'join':
      joinRoom(ws, payload.roomId);
      break;

    case 'leave':
      leaveRoom(ws, payload.roomId);
      break;

    case 'message':
      broadcastToRoom(payload.roomId, {
        type: 'message',
        payload: {
          from: client.id,
          roomId: payload.roomId,
          text: payload.text,
          timestamp: Date.now()
        }
      }, ws);
      break;

    case 'typing':
      broadcastToRoom(payload.roomId, {
        type: 'typing',
        payload: { userId: client.id, roomId: payload.roomId }
      }, ws);
      break;
  }
}

function joinRoom(ws, roomId) {
  const client = clients.get(ws);

  if (!rooms.has(roomId)) {
    rooms.set(roomId, new Set());
  }

  rooms.get(roomId).add(ws);
  client.rooms.add(roomId);

  // 본인에게 확인
  send(ws, {
    type: 'joined',
    payload: { roomId, members: rooms.get(roomId).size }
  });

  // 다른 멤버들에게 알림
  broadcastToRoom(roomId, {
    type: 'user:joined',
    payload: { userId: client.id, roomId }
  }, ws);
}

function leaveRoom(ws, roomId) {
  const client = clients.get(ws);
  const room = rooms.get(roomId);

  if (room) {
    room.delete(ws);
    client.rooms.delete(roomId);

    broadcastToRoom(roomId, {
      type: 'user:left',
      payload: { userId: client.id, roomId }
    });

    // 빈 룸 정리
    if (room.size === 0) {
      rooms.delete(roomId);
    }
  }
}

function broadcastToRoom(roomId, message, excludeWs = null) {
  const room = rooms.get(roomId);
  if (!room) return;

  const messageStr = JSON.stringify(message);
  room.forEach((ws) => {
    if (ws !== excludeWs && ws.readyState === WebSocket.OPEN) {
      ws.send(messageStr);
    }
  });
}

function send(ws, message) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify(message));
  }
}

// 하트비트
const HEARTBEAT_INTERVAL = 30000;
const heartbeatInterval = setInterval(() => {
  wss.clients.forEach((ws) => {
    const client = clients.get(ws);
    if (!client || client.isAlive === false) {
      clients.delete(ws);
      return ws.terminate();
    }
    client.isAlive = false;
    ws.ping();
  });
}, HEARTBEAT_INTERVAL);

wss.on('close', () => {
  clearInterval(heartbeatInterval);
});

console.log('WebSocket server running on port 8080');
```

### 채팅 클라이언트 (React)

```jsx
// useWebSocket.js
import { useEffect, useRef, useState, useCallback } from 'react';

export function useWebSocket(url) {
  const [isConnected, setIsConnected] = useState(false);
  const [lastMessage, setLastMessage] = useState(null);
  const wsRef = useRef(null);
  const reconnectTimeoutRef = useRef(null);
  const reconnectAttempts = useRef(0);

  const connect = useCallback(() => {
    const ws = new WebSocket(url);

    ws.onopen = () => {
      console.log('Connected');
      setIsConnected(true);
      reconnectAttempts.current = 0;
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setLastMessage(message);
    };

    ws.onclose = () => {
      console.log('Disconnected');
      setIsConnected(false);

      // 재연결 (지수 백오프)
      const delay = Math.min(1000 * Math.pow(2, reconnectAttempts.current), 30000);
      reconnectTimeoutRef.current = setTimeout(() => {
        reconnectAttempts.current++;
        connect();
      }, delay);
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    wsRef.current = ws;
  }, [url]);

  useEffect(() => {
    connect();

    return () => {
      if (reconnectTimeoutRef.current) {
        clearTimeout(reconnectTimeoutRef.current);
      }
      if (wsRef.current) {
        wsRef.current.close();
      }
    };
  }, [connect]);

  const sendMessage = useCallback((type, payload) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({ type, payload }));
    }
  }, []);

  return { isConnected, lastMessage, sendMessage };
}

// ChatRoom.jsx
import React, { useState, useEffect } from 'react';
import { useWebSocket } from './useWebSocket';

function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);
  const [inputText, setInputText] = useState('');
  const { isConnected, lastMessage, sendMessage } = useWebSocket('ws://localhost:8080');

  // 룸 참여
  useEffect(() => {
    if (isConnected) {
      sendMessage('join', { roomId });
    }

    return () => {
      if (isConnected) {
        sendMessage('leave', { roomId });
      }
    };
  }, [isConnected, roomId, sendMessage]);

  // 메시지 수신 처리
  useEffect(() => {
    if (lastMessage) {
      switch (lastMessage.type) {
        case 'message':
          setMessages((prev) => [...prev, lastMessage.payload]);
          break;
        case 'user:joined':
          console.log('User joined:', lastMessage.payload.userId);
          break;
        case 'user:left':
          console.log('User left:', lastMessage.payload.userId);
          break;
      }
    }
  }, [lastMessage]);

  const handleSend = () => {
    if (inputText.trim()) {
      sendMessage('message', { roomId, text: inputText });
      setInputText('');
    }
  };

  return (
    <div className="chat-room">
      <div className="status">
        {isConnected ? 'Connected' : 'Disconnected'}
      </div>

      <div className="messages">
        {messages.map((msg, index) => (
          <div key={index} className="message">
            <span className="from">{msg.from}:</span>
            <span className="text">{msg.text}</span>
          </div>
        ))}
      </div>

      <div className="input">
        <input
          type="text"
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleSend()}
          disabled={!isConnected}
        />
        <button onClick={handleSend} disabled={!isConnected}>
          Send
        </button>
      </div>
    </div>
  );
}

export default ChatRoom;
```

---

## 참고 자료

- [RFC 6455 - The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [MDN WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [Socket.io Documentation](https://socket.io/docs/v4/)
- [ws - WebSocket library for Node.js](https://github.com/websockets/ws)
- [WebSocket Security](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/11-Client-side_Testing/10-Testing_WebSockets)
