# SOAP (Simple Object Access Protocol)

## 목차
1. [SOAP 개요 및 역사](#1-soap-개요-및-역사)
2. [SOAP 메시지 구조](#2-soap-메시지-구조)
3. [WSDL (Web Services Description Language)](#3-wsdl-web-services-description-language)
4. [SOAP vs REST 비교](#4-soap-vs-rest-비교)
5. [WS-* 확장 명세](#5-ws--확장-명세)
6. [실제 SOAP 요청/응답 예제](#6-실제-soap-요청응답-예제)
7. [현대 시스템에서의 SOAP 사용 사례](#7-현대-시스템에서의-soap-사용-사례)
8. [참고 자료](#8-참고-자료)

---

## 1. SOAP 개요 및 역사

### 1.1 SOAP이란?

SOAP(Simple Object Access Protocol)은 **XML 기반의 메시지 교환 프로토콜**로, 분산 환경에서 구조화된 정보를 교환하기 위해 설계되었습니다. W3C에서 관리하는 공식 프로토콜로, 다양한 하위 프로토콜(HTTP, SMTP, TCP 등) 위에서 동작할 수 있습니다.

> **W3C 정의**: "SOAP Version 1.2는 분산되고 분권화된 환경에서 구조화된 정보를 교환하기 위한 경량 프로토콜입니다."

### 1.2 역사적 배경

#### 탄생 배경 (1997-1998)
- **1997년**: Microsoft에서 XML 기반 분산 컴퓨팅 연구 시작
- **1998년 초**: DevelopMentor, UserLand와 협력하여 "SOAP"이라는 이름 탄생
- **1998년 6월**: Dave Winer가 XML-RPC를 Frontier 5.1의 일부로 먼저 공개
  - Dave Winer, Don Box, Bob Atkinson, Mohsen Al-Ghosein이 초기 개발에 참여

#### XML-RPC와의 분리
Microsoft 내부에서 DCOM(Distributed Component Object Model) 진영이 SOAP을 반대했습니다. 이들은 HTTP 터널링을 통한 DCOM 와이어 프로토콜을 선호했습니다. 이런 지연으로 인해 Dave Winer는 SOAP의 타입 시스템을 간소화한 XML-RPC를 별도로 공개했습니다.

#### 표준화 과정

| 버전 | 날짜 | 주요 내용 |
|------|------|----------|
| SOAP 0.9 | 1999년 9월 | IETF 인터넷 드래프트로 공개 |
| SOAP 1.0 | 1999년 12월 | 공식 첫 버전 |
| SOAP 1.1 | 2000년 5월 | W3C Note로 제출 (IBM과 Microsoft 공동) |
| SOAP 1.2 | 2003년 6월 | W3C Recommendation으로 승인 |
| SOAP 1.2 Second Edition | 2007년 4월 | 현재 표준 (Errata 반영) |

#### IBM의 합류
2000년, IBM이 SOAP의 공동 저자로 참여하면서 큰 전환점을 맞았습니다. IBM은 Java SOAP 구현체를 Apache XML 프로젝트에 오픈소스로 기증했으며, 이는 SOAP의 신뢰성을 높이는 데 크게 기여했습니다.

### 1.3 SOAP 1.2의 주요 변경 사항

```
SOAP 1.1 (RPC 중심)     →     SOAP 1.2 (메시지 중심)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- RPC 패턴에 집중           - 다양한 메시지 패턴 지원
- HTTP 중심                 - 다양한 전송 프로토콜 지원
- "Simple Object Access     - 약어 사용 중단
   Protocol"
```

---

## 2. SOAP 메시지 구조

SOAP 메시지는 일반적인 XML 문서로, 엄격하게 정의된 구조를 따릅니다.

### 2.1 전체 구조

```
┌─────────────────────────────────────┐
│           SOAP Envelope            │  ← 루트 요소 (필수)
│  ┌───────────────────────────────┐ │
│  │         SOAP Header           │ │  ← 메타데이터 (선택)
│  │  ┌─────────────────────────┐  │ │
│  │  │     Header Block 1      │  │ │
│  │  ├─────────────────────────┤  │ │
│  │  │     Header Block 2      │  │ │
│  │  └─────────────────────────┘  │ │
│  └───────────────────────────────┘ │
│  ┌───────────────────────────────┐ │
│  │          SOAP Body            │ │  ← 실제 데이터 (필수)
│  │  ┌─────────────────────────┐  │ │
│  │  │     Message Content     │  │ │
│  │  │         또는            │  │ │
│  │  │      SOAP Fault         │  │ │
│  │  └─────────────────────────┘  │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
```

### 2.2 Envelope (봉투)

Envelope은 SOAP 메시지의 **루트 요소**로, 메시지의 시작과 끝을 정의합니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema">

    <!-- Header와 Body가 여기에 위치 -->

</soap:Envelope>
```

**주요 네임스페이스**:
- SOAP 1.2 Envelope: `http://www.w3.org/2003/05/soap-envelope`
- SOAP 1.1 Envelope: `http://schemas.xmlsoap.org/soap/envelope/`
- SOAP Encoding: `http://www.w3.org/2003/05/soap-encoding`

### 2.3 Header (헤더)

Header는 **선택적 요소**로, 인증 정보, 트랜잭션 관리, 라우팅 정보 등 메타데이터를 포함합니다.

```xml
<soap:Header>
    <!-- 인증 정보 -->
    <auth:Credentials xmlns:auth="http://example.com/auth">
        <auth:Username>user123</auth:Username>
        <auth:Token>abc123xyz</auth:Token>
    </auth:Credentials>

    <!-- 트랜잭션 정보 -->
    <tx:Transaction
        xmlns:tx="http://example.com/transaction"
        soap:mustUnderstand="true">
        <tx:Id>TXN-2024-001</tx:Id>
    </tx:Transaction>
</soap:Header>
```

**주요 속성**:

| 속성 | 설명 |
|------|------|
| `mustUnderstand` | `true`인 경우, 수신자가 해당 헤더를 반드시 처리해야 함. 처리하지 못하면 Fault 반환 |
| `role` (1.2) / `actor` (1.1) | 헤더를 처리할 SOAP 노드를 지정 |
| `relay` | 처리되지 않은 헤더를 다음 노드로 전달할지 여부 |

### 2.4 Body (본문)

Body는 **필수 요소**로, 실제 메시지 데이터를 포함합니다.

```xml
<soap:Body>
    <m:GetAccountBalance xmlns:m="http://example.com/banking">
        <m:AccountNumber>1234567890</m:AccountNumber>
        <m:Currency>KRW</m:Currency>
    </m:GetAccountBalance>
</soap:Body>
```

**규칙**:
- Envelope은 정확히 **하나의 Body**만 포함해야 함
- Header가 있는 경우, Body는 Header 다음에 위치해야 함
- Body 내용은 애플리케이션별로 정의됨

### 2.5 Fault (오류)

Fault는 오류 발생 시 **Body 내부**에 포함되는 요소입니다.

#### SOAP 1.2 Fault 구조

```xml
<soap:Body>
    <soap:Fault>
        <!-- 오류 코드 (필수) -->
        <soap:Code>
            <soap:Value>soap:Sender</soap:Value>
            <soap:Subcode>
                <soap:Value>rpc:BadArguments</soap:Value>
            </soap:Subcode>
        </soap:Code>

        <!-- 오류 설명 (필수) -->
        <soap:Reason>
            <soap:Text xml:lang="ko">잘못된 계좌번호입니다.</soap:Text>
            <soap:Text xml:lang="en">Invalid account number.</soap:Text>
        </soap:Reason>

        <!-- 오류 발생 노드 (선택) -->
        <soap:Node>http://example.com/banking/service</soap:Node>

        <!-- 오류 처리 역할 (선택) -->
        <soap:Role>http://www.w3.org/2003/05/soap-envelope/role/ultimateReceiver</soap:Role>

        <!-- 상세 정보 (선택) -->
        <soap:Detail>
            <e:ErrorDetails xmlns:e="http://example.com/errors">
                <e:ErrorCode>ACC-001</e:ErrorCode>
                <e:Timestamp>2024-01-15T10:30:00Z</e:Timestamp>
            </e:ErrorDetails>
        </soap:Detail>
    </soap:Fault>
</soap:Body>
```

#### 표준 Fault 코드 (SOAP 1.2)

| 코드 | 설명 | HTTP 상태 |
|------|------|-----------|
| `VersionMismatch` | SOAP 버전 불일치 | 500 |
| `MustUnderstand` | 필수 헤더 처리 실패 | 500 |
| `DataEncodingUnknown` | 알 수 없는 데이터 인코딩 | 500 |
| `Sender` | 클라이언트 측 오류 | 400 계열 |
| `Receiver` | 서버 측 오류 | 500 계열 |

---

## 3. WSDL (Web Services Description Language)

### 3.1 WSDL 개요

WSDL은 **웹 서비스의 인터페이스를 기술하는 XML 기반 언어**입니다. 클라이언트가 서비스를 사용하기 위해 필요한 모든 정보를 제공합니다.

> **W3C 정의**: "WSDL은 추상적 모델을 기반으로 웹 서비스를 기술할 수 있는 XML 언어입니다."

### 3.2 WSDL 버전 비교

| 항목 | WSDL 1.1 | WSDL 2.0 |
|------|----------|----------|
| 상태 | 널리 사용됨 (W3C Note) | W3C Recommendation (2007) |
| PortType | `<portType>` | `<interface>` |
| Port | `<port>` | `<endpoint>` |
| 상속 | 미지원 | Interface 상속 지원 |
| HTTP 메서드 | GET, POST만 지원 | 모든 HTTP 메서드 지원 |
| RESTful 지원 | 제한적 | 개선됨 |

### 3.3 WSDL 2.0 구조

```
┌──────────────────────────────────────────────────────────────┐
│                        WSDL Document                          │
├──────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    추상적 정의 (무엇을)                      │
│  │   Types     │ ← 데이터 타입 정의 (XML Schema)              │
│  ├─────────────┤                                              │
│  │  Interface  │ ← 연산(operation) 정의                       │
│  └─────────────┘                                              │
├──────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    구체적 정의 (어떻게)                      │
│  │   Binding   │ ← 프로토콜/데이터 형식 바인딩                │
│  ├─────────────┤                                              │
│  │   Service   │ ← 서비스 위치(endpoint) 정의                 │
│  └─────────────┘                                              │
└──────────────────────────────────────────────────────────────┘
```

### 3.4 WSDL 예제

```xml
<?xml version="1.0" encoding="UTF-8"?>
<description
    xmlns="http://www.w3.org/ns/wsdl"
    xmlns:tns="http://example.com/banking"
    xmlns:wsoap="http://www.w3.org/ns/wsdl/soap"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    targetNamespace="http://example.com/banking">

    <!-- 1. Types: 데이터 타입 정의 -->
    <types>
        <xs:schema targetNamespace="http://example.com/banking">
            <xs:element name="GetBalanceRequest">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="accountNumber" type="xs:string"/>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>

            <xs:element name="GetBalanceResponse">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="balance" type="xs:decimal"/>
                        <xs:element name="currency" type="xs:string"/>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>
        </xs:schema>
    </types>

    <!-- 2. Interface: 연산 정의 -->
    <interface name="BankingInterface">
        <operation name="GetBalance"
                   pattern="http://www.w3.org/ns/wsdl/in-out">
            <input messageLabel="In" element="tns:GetBalanceRequest"/>
            <output messageLabel="Out" element="tns:GetBalanceResponse"/>
        </operation>
    </interface>

    <!-- 3. Binding: SOAP 프로토콜 바인딩 -->
    <binding name="BankingSoapBinding"
             interface="tns:BankingInterface"
             type="http://www.w3.org/ns/wsdl/soap"
             wsoap:protocol="http://www.w3.org/2003/05/soap/bindings/HTTP/">

        <operation ref="tns:GetBalance"
                   wsoap:mep="http://www.w3.org/2003/05/soap/mep/request-response"/>
    </binding>

    <!-- 4. Service: 엔드포인트 정의 -->
    <service name="BankingService" interface="tns:BankingInterface">
        <endpoint name="BankingEndpoint"
                  binding="tns:BankingSoapBinding"
                  address="https://api.example.com/banking"/>
    </service>
</description>
```

### 3.5 Message Exchange Patterns (MEP)

WSDL 2.0에서는 **명시적인 메시지 교환 패턴**을 지원합니다.

| 패턴 | URI | 설명 |
|------|-----|------|
| In-Only | `http://www.w3.org/ns/wsdl/in-only` | 단방향 메시지 (응답 없음) |
| Robust In-Only | `http://www.w3.org/ns/wsdl/robust-in-only` | 단방향 + Fault 가능 |
| In-Out | `http://www.w3.org/ns/wsdl/in-out` | 요청-응답 패턴 |
| In-Optional-Out | `http://www.w3.org/ns/wsdl/in-opt-out` | 선택적 응답 |
| Out-Only | `http://www.w3.org/ns/wsdl/out-only` | 서버 → 클라이언트 단방향 |
| Out-In | `http://www.w3.org/ns/wsdl/out-in` | 서버 시작 요청-응답 |

---

## 4. SOAP vs REST 비교

### 4.1 기본 개념 비교

| 구분 | SOAP | REST |
|------|------|------|
| **정의** | W3C 공식 프로토콜 | 아키텍처 스타일 |
| **데이터 형식** | XML만 지원 | JSON, XML, HTML, 텍스트 등 |
| **전송 프로토콜** | HTTP, SMTP, TCP 등 | 주로 HTTP |
| **상태 관리** | Stateful 가능 | Stateless |
| **계약** | WSDL (엄격한 계약) | OpenAPI/Swagger (선택적) |

### 4.2 상세 비교

```
                    SOAP                          REST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
메시지 크기         큼 (XML 오버헤드)              작음 (JSON 기반)

보안               WS-Security 내장              HTTPS + OAuth/JWT

트랜잭션           WS-AtomicTransaction          애플리케이션 수준

신뢰성             WS-ReliableMessaging          애플리케이션 수준

캐싱               복잡함                        HTTP 캐싱 활용

학습 곡선          높음                          낮음

도구 지원          강력한 IDE 지원               광범위한 라이브러리
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 4.3 성능 비교

```
요청 크기 비교 (동일한 데이터)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SOAP 요청:
┌─────────────────────────────────────┐
│ <?xml version="1.0"?>               │
│ <soap:Envelope xmlns:soap="...">    │
│   <soap:Header>...</soap:Header>    │
│   <soap:Body>                       │
│     <GetUser>                       │
│       <userId>123</userId>          │
│     </GetUser>                      │
│   </soap:Body>                      │
│ </soap:Envelope>                    │
└─────────────────────────────────────┘
크기: ~500 bytes

REST 요청:
┌─────────────────────────────────────┐
│ GET /users/123 HTTP/1.1             │
│ Accept: application/json            │
└─────────────────────────────────────┘
크기: ~50 bytes
```

### 4.4 사용 시나리오

#### SOAP을 선택해야 하는 경우

1. **엔터프라이즈 통합**
   - 레거시 시스템과의 호환성 필요
   - 기존 SOAP 기반 인프라 활용

2. **높은 보안 요구사항**
   - WS-Security를 통한 메시지 수준 암호화
   - 디지털 서명 필요

3. **트랜잭션 무결성**
   - ACID 트랜잭션 지원 필요
   - 분산 트랜잭션 처리

4. **신뢰성 있는 메시징**
   - 메시지 전달 보장
   - 순서 보장 및 중복 제거

#### REST를 선택해야 하는 경우

1. **웹/모바일 애플리케이션**
   - 경량 통신 필요
   - 브라우저 친화적

2. **마이크로서비스**
   - 독립적 배포
   - 느슨한 결합

3. **공개 API**
   - 쉬운 접근성
   - 광범위한 클라이언트 지원

4. **실시간 서비스**
   - 낮은 지연시간 요구
   - 높은 처리량

---

## 5. WS-* 확장 명세

SOAP은 다양한 **WS-* (Web Services) 확장 명세**를 통해 엔터프라이즈 기능을 제공합니다.

### 5.1 WS-Security

OASIS에서 관리하는 보안 표준으로, SOAP 메시지의 무결성과 기밀성을 보장합니다.

#### 주요 기능

```
┌─────────────────────────────────────────────────────────────┐
│                      WS-Security                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  인증 토큰    │  │  XML 서명    │  │  XML 암호화  │      │
│  │              │  │              │  │              │      │
│  │ - Username   │  │ - 메시지     │  │ - 메시지     │      │
│  │ - X.509      │  │   무결성     │  │   기밀성     │      │
│  │ - Kerberos   │  │ - 부인방지   │  │ - 부분암호화 │      │
│  │ - SAML       │  │              │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### WS-Security 헤더 예제

```xml
<soap:Header>
    <wsse:Security
        xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd"
        xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">

        <!-- 타임스탬프 -->
        <wsu:Timestamp wsu:Id="TS-1">
            <wsu:Created>2024-01-15T10:00:00Z</wsu:Created>
            <wsu:Expires>2024-01-15T10:05:00Z</wsu:Expires>
        </wsu:Timestamp>

        <!-- UsernameToken -->
        <wsse:UsernameToken wsu:Id="UT-1">
            <wsse:Username>user123</wsse:Username>
            <wsse:Password Type="...#PasswordDigest">
                WeYQnXXy...=
            </wsse:Password>
            <wsse:Nonce EncodingType="...#Base64Binary">
                d3dGVmJh...
            </wsse:Nonce>
            <wsu:Created>2024-01-15T10:00:00Z</wsu:Created>
        </wsse:UsernameToken>

        <!-- X.509 인증서 -->
        <wsse:BinarySecurityToken
            ValueType="...#X509v3"
            EncodingType="...#Base64Binary"
            wsu:Id="X509-1">
            MIICxDCCAa...
        </wsse:BinarySecurityToken>

        <!-- 디지털 서명 -->
        <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
            <ds:SignedInfo>
                <ds:CanonicalizationMethod
                    Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                <ds:SignatureMethod
                    Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1"/>
                <ds:Reference URI="#Body-1">
                    <ds:DigestMethod
                        Algorithm="http://www.w3.org/2000/09/xmldsig#sha1"/>
                    <ds:DigestValue>LyLsF0...</ds:DigestValue>
                </ds:Reference>
            </ds:SignedInfo>
            <ds:SignatureValue>MC0CFF...</ds:SignatureValue>
            <ds:KeyInfo>
                <wsse:SecurityTokenReference>
                    <wsse:Reference URI="#X509-1"/>
                </wsse:SecurityTokenReference>
            </ds:KeyInfo>
        </ds:Signature>
    </wsse:Security>
</soap:Header>
```

### 5.2 WS-ReliableMessaging

OASIS 표준으로, 네트워크 장애 상황에서도 **메시지 전달을 보장**합니다.

#### 핵심 기능

| 기능 | 설명 |
|------|------|
| **AtLeastOnce** | 최소 1회 전달 보장 |
| **AtMostOnce** | 최대 1회 전달 (중복 방지) |
| **ExactlyOnce** | 정확히 1회 전달 |
| **InOrder** | 순서 보장 |

#### 동작 흐름

```
클라이언트                                 서버
    │                                        │
    │────── CreateSequence ─────────────────>│
    │<───── CreateSequenceResponse ─────────│
    │                                        │
    │────── Message 1 (Seq=1) ──────────────>│
    │<───── SequenceAcknowledgement ────────│
    │                                        │
    │────── Message 2 (Seq=2) ──────────────>│
    │       (손실됨)                          │
    │                                        │
    │────── Message 3 (Seq=3) ──────────────>│
    │<───── SequenceAck (Ack: 1,3) ─────────│
    │                                        │
    │────── Message 2 (재전송) ─────────────>│
    │<───── SequenceAck (Ack: 1-3) ─────────│
    │                                        │
    │────── TerminateSequence ──────────────>│
    │<───── TerminateSequenceResponse ──────│
    │                                        │
```

### 5.3 기타 WS-* 명세

| 명세 | 목적 | 관리 기관 |
|------|------|-----------|
| **WS-Policy** | 서비스 정책 정의 | W3C |
| **WS-Addressing** | 전송 독립적 메시지 라우팅 | W3C |
| **WS-AtomicTransaction** | 분산 트랜잭션 조정 | OASIS |
| **WS-Coordination** | 분산 처리 조정 | OASIS |
| **WS-Trust** | 보안 토큰 발급/교환 | OASIS |
| **WS-SecureConversation** | 보안 세션 설정 | OASIS |
| **WS-Federation** | 연합 ID 관리 | OASIS |

### 5.4 WS-* 스택 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    Business Process                         │
│                   (WS-BPEL, WS-CDL)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │WS-Reliable  │  │WS-Atomic    │  │   WS-Security       │ │
│  │Messaging    │  │Transaction  │  │   WS-Trust          │ │
│  │             │  │             │  │   WS-Federation     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│           WS-Policy    │    WS-Addressing                   │
├─────────────────────────────────────────────────────────────┤
│                        SOAP                                 │
├─────────────────────────────────────────────────────────────┤
│                        WSDL                                 │
├─────────────────────────────────────────────────────────────┤
│              HTTP  │  SMTP  │  TCP  │  JMS                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 실제 SOAP 요청/응답 예제

### 6.1 기본 RPC 스타일 요청/응답

#### 요청 (Request)

```xml
POST /banking/service HTTP/1.1
Host: api.example.com
Content-Type: application/soap+xml; charset=utf-8
Content-Length: 512
SOAPAction: "http://example.com/banking/GetAccountBalance"

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:bank="http://example.com/banking">

    <soap:Header>
        <bank:AuthToken>eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...</bank:AuthToken>
        <bank:RequestId>REQ-2024-001</bank:RequestId>
    </soap:Header>

    <soap:Body>
        <bank:GetAccountBalance>
            <bank:AccountNumber>1234-5678-9012</bank:AccountNumber>
            <bank:Currency>KRW</bank:Currency>
        </bank:GetAccountBalance>
    </soap:Body>
</soap:Envelope>
```

#### 성공 응답 (Response)

```xml
HTTP/1.1 200 OK
Content-Type: application/soap+xml; charset=utf-8
Content-Length: 498

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:bank="http://example.com/banking">

    <soap:Header>
        <bank:ResponseId>RES-2024-001</bank:ResponseId>
        <bank:Timestamp>2024-01-15T10:30:00+09:00</bank:Timestamp>
    </soap:Header>

    <soap:Body>
        <bank:GetAccountBalanceResponse>
            <bank:AccountNumber>1234-5678-9012</bank:AccountNumber>
            <bank:Balance>1500000.00</bank:Balance>
            <bank:Currency>KRW</bank:Currency>
            <bank:AvailableBalance>1450000.00</bank:AvailableBalance>
            <bank:LastUpdated>2024-01-15T10:29:55+09:00</bank:LastUpdated>
        </bank:GetAccountBalanceResponse>
    </soap:Body>
</soap:Envelope>
```

### 6.2 오류 응답 (Fault)

```xml
HTTP/1.1 500 Internal Server Error
Content-Type: application/soap+xml; charset=utf-8

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:bank="http://example.com/banking">

    <soap:Body>
        <soap:Fault>
            <soap:Code>
                <soap:Value>soap:Sender</soap:Value>
                <soap:Subcode>
                    <soap:Value>bank:InvalidAccount</soap:Value>
                </soap:Subcode>
            </soap:Code>

            <soap:Reason>
                <soap:Text xml:lang="ko">
                    존재하지 않는 계좌번호입니다.
                </soap:Text>
                <soap:Text xml:lang="en">
                    Account number does not exist.
                </soap:Text>
            </soap:Reason>

            <soap:Detail>
                <bank:ErrorInfo>
                    <bank:ErrorCode>ACC-404</bank:ErrorCode>
                    <bank:RequestedAccount>9999-9999-9999</bank:RequestedAccount>
                    <bank:Timestamp>2024-01-15T10:30:00+09:00</bank:Timestamp>
                    <bank:TraceId>trace-abc123</bank:TraceId>
                </bank:ErrorInfo>
            </soap:Detail>
        </soap:Fault>
    </soap:Body>
</soap:Envelope>
```

### 6.3 WS-Security를 사용한 요청

```xml
POST /secure/banking HTTP/1.1
Host: api.example.com
Content-Type: application/soap+xml; charset=utf-8

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd"
    xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd"
    xmlns:bank="http://example.com/banking">

    <soap:Header>
        <wsse:Security soap:mustUnderstand="true">
            <!-- 타임스탬프 (재생 공격 방지) -->
            <wsu:Timestamp wsu:Id="TS-1">
                <wsu:Created>2024-01-15T10:00:00Z</wsu:Created>
                <wsu:Expires>2024-01-15T10:05:00Z</wsu:Expires>
            </wsu:Timestamp>

            <!-- 사용자 인증 -->
            <wsse:UsernameToken wsu:Id="UsernameToken-1">
                <wsse:Username>banking_user</wsse:Username>
                <wsse:Password
                    Type="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordDigest">
                    nC7xsMjL...
                </wsse:Password>
                <wsse:Nonce
                    EncodingType="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0#Base64Binary">
                    WScqanjC...
                </wsse:Nonce>
                <wsu:Created>2024-01-15T10:00:00Z</wsu:Created>
            </wsse:UsernameToken>
        </wsse:Security>
    </soap:Header>

    <soap:Body wsu:Id="Body-1">
        <bank:TransferFunds>
            <bank:FromAccount>1234-5678-9012</bank:FromAccount>
            <bank:ToAccount>9876-5432-1098</bank:ToAccount>
            <bank:Amount>100000</bank:Amount>
            <bank:Currency>KRW</bank:Currency>
            <bank:Description>월세 이체</bank:Description>
        </bank:TransferFunds>
    </soap:Body>
</soap:Envelope>
```

### 6.4 다양한 언어별 SOAP 클라이언트 구현

#### Java (JAX-WS)

```java
import javax.xml.ws.*;
import javax.xml.namespace.QName;

public class BankingClient {
    public static void main(String[] args) {
        // WSDL에서 서비스 생성
        URL wsdlUrl = new URL("https://api.example.com/banking?wsdl");
        QName serviceName = new QName("http://example.com/banking", "BankingService");

        Service service = Service.create(wsdlUrl, serviceName);
        BankingPortType port = service.getPort(BankingPortType.class);

        // WS-Security 핸들러 설정
        BindingProvider bp = (BindingProvider) port;
        bp.getRequestContext().put(BindingProvider.USERNAME_PROPERTY, "user");
        bp.getRequestContext().put(BindingProvider.PASSWORD_PROPERTY, "password");

        // 서비스 호출
        GetAccountBalanceRequest request = new GetAccountBalanceRequest();
        request.setAccountNumber("1234-5678-9012");
        request.setCurrency("KRW");

        GetAccountBalanceResponse response = port.getAccountBalance(request);
        System.out.println("잔액: " + response.getBalance());
    }
}
```

#### Python (zeep)

```python
from zeep import Client
from zeep.wsse.username import UsernameToken

# WSDL에서 클라이언트 생성
wsdl = 'https://api.example.com/banking?wsdl'
client = Client(
    wsdl,
    wsse=UsernameToken('username', 'password')
)

# 서비스 호출
response = client.service.GetAccountBalance(
    AccountNumber='1234-5678-9012',
    Currency='KRW'
)

print(f"잔액: {response.Balance} {response.Currency}")
```

#### C# (.NET)

```csharp
using System.ServiceModel;

// 서비스 참조 추가 후 자동 생성된 클라이언트 사용
var binding = new BasicHttpsBinding();
binding.Security.Mode = BasicHttpsSecurityMode.Transport;
binding.Security.Transport.ClientCredentialType = HttpClientCredentialType.Basic;

var endpoint = new EndpointAddress("https://api.example.com/banking");
var client = new BankingServiceClient(binding, endpoint);

client.ClientCredentials.UserName.UserName = "username";
client.ClientCredentials.UserName.Password = "password";

var response = await client.GetAccountBalanceAsync(
    new GetAccountBalanceRequest {
        AccountNumber = "1234-5678-9012",
        Currency = "KRW"
    }
);

Console.WriteLine($"잔액: {response.Balance} {response.Currency}");
```

---

## 7. 현대 시스템에서의 SOAP 사용 사례

### 7.1 금융 산업

금융 산업은 SOAP의 **핵심 사용 영역**입니다.

#### 사용 이유

```
┌─────────────────────────────────────────────────────────────┐
│                    금융 시스템 요구사항                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  보안성          ────> WS-Security (메시지 수준 암호화)     │
│  신뢰성          ────> WS-ReliableMessaging                 │
│  트랜잭션        ────> WS-AtomicTransaction (ACID 보장)     │
│  감사 추적       ────> XML 메시지 전체 로깅                  │
│  표준 계약       ────> WSDL (엄격한 인터페이스 정의)         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 실제 사용 사례

| 시스템 | 설명 |
|--------|------|
| **SWIFT** | 국제 은행간 금융 메시징 |
| **FIX Protocol** | 증권 거래 통신 (SOAP over MQ) |
| **코어뱅킹 연동** | 은행 내부 시스템 통합 |
| **결제 게이트웨이** | 카드사-가맹점 통신 |
| **보험 청구** | 보험사 간 청구 처리 |

#### 금융 SOAP 아키텍처 예시

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   모바일     │     │    웹 앱    │     │  ATM/키오스크│
│    앱        │     │             │     │             │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └─────────┬─────────┴─────────┬─────────┘
                 │                   │
                 ▼                   ▼
         ┌──────────────────────────────────┐
         │          API Gateway             │
         │    (REST → SOAP 변환)            │
         └──────────────┬───────────────────┘
                        │
                        ▼
         ┌──────────────────────────────────┐
         │     Enterprise Service Bus       │
         │         (ESB)                    │
         └──────────────┬───────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ 코어뱅킹     │ │  신용평가    │ │  외환 시스템 │
│ (SOAP)      │ │  (SOAP)     │ │  (SOAP)     │
└──────────────┘ └──────────────┘ └──────────────┘
```

### 7.2 기업 시스템 (Enterprise)

#### ERP/CRM 통합

| 시스템 | SOAP 사용 |
|--------|----------|
| **SAP** | RFC, BAPI를 SOAP으로 노출 |
| **Oracle Financials** | SOAP 웹 서비스 기본 제공 |
| **Microsoft Dynamics** | SOAP/OData 하이브리드 |
| **Salesforce** | SOAP API (Enterprise Edition) |

#### B2B 통합

```
┌─────────────┐                           ┌─────────────┐
│  Company A  │                           │  Company B  │
│             │                           │             │
│  ┌───────┐  │      SOAP/WS-Security     │  ┌───────┐  │
│  │  ERP  │──┼───────────────────────────┼──│  ERP  │  │
│  └───────┘  │                           │  └───────┘  │
│             │                           │             │
│  ┌───────┐  │    WS-ReliableMessaging   │  ┌───────┐  │
│  │  SCM  │──┼───────────────────────────┼──│  SCM  │  │
│  └───────┘  │                           │  └───────┘  │
│             │                           │             │
└─────────────┘                           └─────────────┘
```

### 7.3 정부/공공 시스템

#### 전자정부 표준프레임워크

한국의 전자정부 표준프레임워크는 SOAP 기반 연계를 지원합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                  행정정보 공동이용센터                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐              │
│  │ 주민등록   │  │ 건축물대장 │  │  자동차   │  ...        │
│  │   정보    │  │   정보    │  │   정보    │              │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘              │
│        │              │              │                     │
│        └──────────────┴──────────────┘                     │
│                       │                                     │
│                       ▼                                     │
│              ┌─────────────────┐                           │
│              │   SOAP Gateway  │                           │
│              │  (WS-Security)  │                           │
│              └────────┬────────┘                           │
│                       │                                     │
└───────────────────────┼─────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   민원 시스템 │ │  금융기관    │ │  공공기관    │
└──────────────┘ └──────────────┘ └──────────────┘
```

### 7.4 의료 시스템

#### HL7/FHIR과 SOAP

| 표준 | 프로토콜 | 용도 |
|------|----------|------|
| HL7 v2 | 전용 프로토콜 | 레거시 의료 데이터 교환 |
| HL7 v3 | SOAP | 임상 문서 교환 |
| FHIR | REST/SOAP | 차세대 의료 정보 교환 |

### 7.5 레거시 시스템 통합 전략

#### REST-SOAP 게이트웨이 패턴

```
Modern Client                    Legacy System
    │                                 │
    │   REST/JSON                     │   SOAP/XML
    │                                 │
    ▼                                 ▼
┌──────────────────────────────────────────────────┐
│                  API Gateway                      │
│                                                  │
│  ┌──────────────┐      ┌──────────────────────┐ │
│  │ REST Endpoint│ ───> │ SOAP/REST Translator │ │
│  └──────────────┘      └──────────────────────┘ │
│                               │                  │
│                               ▼                  │
│                        ┌──────────────┐         │
│                        │ SOAP Client  │         │
│                        └──────────────┘         │
│                               │                  │
└───────────────────────────────┼──────────────────┘
                                │
                                ▼
                        ┌──────────────┐
                        │ Legacy SOAP  │
                        │   Service    │
                        └──────────────┘
```

#### 변환 예시 (REST → SOAP)

```javascript
// Express.js REST-to-SOAP Gateway
const express = require('express');
const soap = require('soap');

const app = express();

app.get('/api/v1/accounts/:accountId/balance', async (req, res) => {
    const { accountId } = req.params;

    // SOAP 클라이언트 생성
    const client = await soap.createClientAsync(
        'https://legacy.bank.com/banking.wsdl'
    );

    // SOAP 요청
    const [result] = await client.GetAccountBalanceAsync({
        AccountNumber: accountId,
        Currency: 'KRW'
    });

    // REST 응답으로 변환
    res.json({
        accountId: result.AccountNumber,
        balance: parseFloat(result.Balance),
        currency: result.Currency,
        availableBalance: parseFloat(result.AvailableBalance),
        lastUpdated: result.LastUpdated
    });
});
```

---

## 8. 참고 자료

### 8.1 W3C 공식 명세

| 문서 | URL |
|------|-----|
| SOAP 명세 목록 | https://www.w3.org/TR/soap/ |
| SOAP 1.2 Part 0: Primer | https://www.w3.org/TR/2007/REC-soap12-part0-20070427/ |
| SOAP 1.2 Part 1: Messaging Framework | https://www.w3.org/TR/2007/REC-soap12-part1-20070427/ |
| SOAP 1.2 Part 2: Adjuncts | https://www.w3.org/TR/2007/REC-soap12-part2-20070427/ |
| WSDL 2.0 Core Language | https://www.w3.org/TR/2007/REC-wsdl20-20070626/ |
| WSDL 2.0 Adjuncts | https://www.w3.org/TR/wsdl20-adjuncts/ |
| WS-Addressing | https://www.w3.org/TR/ws-addr-core/ |

### 8.2 OASIS 표준

| 문서 | URL |
|------|-----|
| WS-Security 1.1.1 | https://docs.oasis-open.org/wss-m/wss/v1.1.1/os/wss-SOAPMessageSecurity-v1.1.1-os.html |
| WS-Security TC | https://www.oasis-open.org/committees/wss/ |
| WS-ReliableMessaging 1.2 | https://www.oasis-open.org/standard/ws-reliablemessaging/ |
| WS-ReliableMessaging 1.1 Spec | https://docs.oasis-open.org/ws-rx/wsrm/200702/wsrm-1.1-spec-os-01-e1.html |

### 8.3 RFC 문서

| RFC | 제목 | URL |
|-----|------|-----|
| RFC 4227 | Using the Simple Object Access Protocol (SOAP) in Blocks Extensible Exchange Protocol (BEEP) | https://tools.ietf.org/html/rfc4227 |
| RFC 3902 | The "application/soap+xml" media type | https://tools.ietf.org/html/rfc3902 |

### 8.4 도구 및 라이브러리

#### 개발 도구

| 도구 | 설명 |
|------|------|
| SoapUI | SOAP/REST API 테스트 도구 |
| Postman | API 개발 및 테스트 플랫폼 |
| Wireshark | 네트워크 프로토콜 분석 |

#### 언어별 라이브러리

| 언어 | 라이브러리 |
|------|------------|
| Java | JAX-WS, Apache CXF, Apache Axis2 |
| Python | zeep, suds-community |
| JavaScript/Node.js | soap, strong-soap |
| C#/.NET | WCF, System.Web.Services |
| PHP | SoapClient (built-in), nusoap |
| Go | gowsdl |

### 8.5 추가 학습 자료

- [A Brief History of SOAP (XML.com)](https://www.xml.com/pub/a/ws/2001/04/04/soap.html)
- [SOAP and REST At Odds - The History of the Web](https://thehistoryoftheweb.com/soap-rest-odds/)
- [Understanding WS-Security (Oracle)](https://docs.oracle.com/cd/G10809_01/pt861pbr4/eng/pt/tsec/UnderstandingWS-Security-c07766.html)
- [AWS SOAP vs REST 비교](https://aws.amazon.com/compare/the-difference-between-soap-rest/)

---

## 요약

SOAP은 엔터프라이즈 환경에서 **보안**, **신뢰성**, **트랜잭션 무결성**이 중요한 시스템에서 여전히 핵심적인 역할을 합니다. REST가 웹/모바일 애플리케이션에서 주류가 되었지만, 금융, 의료, 정부 시스템에서는 SOAP의 강력한 WS-* 확장이 필수적입니다.

현대 시스템에서는 REST-SOAP 게이트웨이 패턴을 통해 레거시 SOAP 서비스와 현대적인 REST API를 통합하는 하이브리드 접근법이 일반적입니다.
