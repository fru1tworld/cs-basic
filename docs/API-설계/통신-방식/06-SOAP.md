# SOAP

## SOAP 개요 및 역사

### SOAP이란?

SOAP(Simple Object Access Protocol)은 분산 환경에서 구조화된 정보를 주고받기 위한 XML 기반 메시지 교환 프로토콜이다. W3C에서 관리하며, HTTP뿐 아니라 SMTP, TCP 등 다양한 하위 프로토콜 위에서 동작할 수 있다.

- W3C 정의: "SOAP Version 1.2는 분산되고 분권화된 환경에서 구조화된 정보를 교환하기 위한 경량 프로토콜임."

### 역사적 배경

#### 탄생 배경 (1997-1998)

- 1997년: Microsoft에서 XML 기반 분산 컴퓨팅 연구 시작
- 1998년 초: DevelopMentor, UserLand와 협력하여 "SOAP"이라는 이름 탄생
- 1998년 6월: Dave Winer가 XML-RPC를 Frontier 5.1의 일부로 먼저 공개
  - Dave Winer, Don Box, Bob Atkinson, Mohsen Al-Ghosein이 초기 개발에 참여

#### XML-RPC와의 분리

Microsoft 내부의 DCOM(Distributed Component Object Model) 진영은 SOAP 대신 HTTP 터널링을 통한 DCOM 와이어 프로토콜을 선호했다. 이 반대로 개발이 지연되자 Dave Winer는 SOAP의 타입 시스템을 간소화한 XML-RPC를 별도로 공개했다.

#### 표준화 과정

- 버전: SOAP 0.9:
  - 날짜: 1999년 9월
  - 주요 내용: IETF 인터넷 드래프트로 공개
- 버전: SOAP 1.0:
  - 날짜: 1999년 12월
  - 주요 내용: 공식 첫 버전
- 버전: SOAP 1.1:
  - 날짜: 2000년 5월
  - 주요 내용: W3C Note로 제출 (IBM과 Microsoft 공동)
- 버전: SOAP 1.2:
  - 날짜: 2003년 6월
  - 주요 내용: W3C Recommendation으로 승인
- 버전: SOAP 1.2 Second Edition:
  - 날짜: 2007년 4월
  - 주요 내용: 현재 표준 (Errata 반영)

#### IBM의 합류

- 2000년, IBM이 SOAP의 공동 저자로 참여하면서 큰 전환점을 맞았음. IBM은 Java SOAP 구현체를 Apache XML 프로젝트에 오픈소스로 기증했으며, 이는 SOAP의 신뢰성을 높이는 데 크게 기여했음.

### SOAP 1.2의 주요 변경 사항

SOAP 1.1이 RPC와 HTTP를 중심으로 했다면, SOAP 1.2는 메시지를 중심으로 다양한 메시지 패턴과 전송 프로토콜을 지원하는 방향으로 바뀌었다. 이와 함께 "Simple Object Access Protocol"이라는 약어 풀이도 사용하지 않게 됐다.

## SOAP 메시지 구조

SOAP 메시지는 XML 문서이며, Envelope 안에 Header와 Body를 배치하는 정해진 구조를 따른다.

### 전체 구조

필수 루트 요소인 Envelope 안에는 선택 요소인 Header와 필수 요소인 Body가 들어간다. Header에는 Header Block 1, Header Block 2처럼 메타데이터를 담는 블록을 배치하고, Body에는 실제 메시지 내용이나 SOAP Fault를 담는다.

### Envelope (봉투)

- Envelope은 SOAP 메시지의 루트 요소로, 메시지의 시작과 끝을 정의함.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema">

    <!-- Header와 Body가 여기에 위치 -->

</soap:Envelope>
```

- 주요 네임스페이스:
- SOAP 1.2 Envelope: `http://www.w3.org/2003/05/soap-envelope`
- SOAP 1.1 Envelope: `http://schemas.xmlsoap.org/soap/envelope/`
- SOAP Encoding: `http://www.w3.org/2003/05/soap-encoding`

### Header (헤더)

- Header는 선택적 요소로, 인증 정보, 트랜잭션 관리, 라우팅 정보 등 메타데이터를 포함함.

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

- 주요 속성:

- 속성: `mustUnderstand`:
  - 설명: `true`인 경우, 수신자가 해당 헤더를 반드시 처리해야 함. 처리하지 못하면 Fault 반환
- 속성: `role` (1.2) / `actor` (1.1):
  - 설명: 헤더를 처리할 SOAP 노드를 지정
- 속성: `relay`:
  - 설명: 처리되지 않은 헤더를 다음 노드로 전달할지 여부

### Body (본문)

- Body는 필수 요소로, 실제 메시지 데이터를 포함함.

```xml
<soap:Body>
    <m:GetAccountBalance xmlns:m="http://example.com/banking">
        <m:AccountNumber>1234567890</m:AccountNumber>
        <m:Currency>KRW</m:Currency>
    </m:GetAccountBalance>
</soap:Body>
```

Envelope에는 정확히 하나의 Body가 있어야 하며, Header가 있다면 Body는 그 뒤에 와야 한다. Body 내부의 내용은 애플리케이션별로 정의한다. 위 예제에서는 계좌번호와 통화로 잔액을 조회하는 `GetAccountBalance` 요청을 담았다.

### Fault (오류)

- Fault는 오류 발생 시 Body 내부에 포함되는 요소임.

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

- 코드: `VersionMismatch`:
  - 설명: SOAP 버전 불일치
  - HTTP 상태: 500
- 코드: `MustUnderstand`:
  - 설명: 필수 헤더 처리 실패
  - HTTP 상태: 500
- 코드: `DataEncodingUnknown`:
  - 설명: 알 수 없는 데이터 인코딩
  - HTTP 상태: 500
- 코드: `Sender`:
  - 설명: 클라이언트 측 오류
  - HTTP 상태: 400 계열
- 코드: `Receiver`:
  - 설명: 서버 측 오류
  - HTTP 상태: 500 계열

## WSDL (Web Services Description Language)

### WSDL 개요

SOAP 메시지를 주고받으려면 어떤 연산을 호출할 수 있고 어디로 요청해야 하는지도 알아야 한다. WSDL은 이 인터페이스를 기술하는 XML 기반 언어로, 클라이언트가 서비스를 사용하는 데 필요한 정보를 제공한다.

- W3C 정의: "WSDL은 추상적 모델을 기반으로 웹 서비스를 기술할 수 있는 XML 언어임."

### WSDL 버전 비교

- 항목: 상태:
  - WSDL 1.1: 널리 사용됨 (W3C Note)
  - WSDL 2.0: W3C Recommendation (2007)
- 항목: PortType:
  - WSDL 1.1: `<portType>`
  - WSDL 2.0: `<interface>`
- 항목: Port:
  - WSDL 1.1: `<port>`
  - WSDL 2.0: `<endpoint>`
- 항목: 상속:
  - WSDL 1.1: 미지원
  - WSDL 2.0: Interface 상속 지원
- 항목: HTTP 메서드:
  - WSDL 1.1: GET, POST만 지원
  - WSDL 2.0: 모든 HTTP 메서드 지원
- 항목: RESTful 지원:
  - WSDL 1.1: 제한적
  - WSDL 2.0: 개선됨

### WSDL 2.0 구조

WSDL 2.0은 서비스가 무엇을 제공하는지와 어떻게 호출하는지를 나눠 정의한다. 추상적 정의에서는 Types가 XML Schema로 데이터 타입을, Interface가 연산(operation)을 기술한다. 구체적 정의에서는 Binding이 프로토콜과 데이터 형식을 연결하고, Service가 엔드포인트 위치를 정한다.

### WSDL 예제

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

### Message Exchange Patterns (MEP)

- WSDL 2.0에서는 명시적인 메시지 교환 패턴을 지원함.

- 패턴: In-Only:
  - URI: `http://www.w3.org/ns/wsdl/in-only`
  - 설명: 단방향 메시지 (응답 없음)
- 패턴: Robust In-Only:
  - URI: `http://www.w3.org/ns/wsdl/robust-in-only`
  - 설명: 단방향 + Fault 가능
- 패턴: In-Out:
  - URI: `http://www.w3.org/ns/wsdl/in-out`
  - 설명: 요청-응답 패턴
- 패턴: In-Optional-Out:
  - URI: `http://www.w3.org/ns/wsdl/in-opt-out`
  - 설명: 선택적 응답
- 패턴: Out-Only:
  - URI: `http://www.w3.org/ns/wsdl/out-only`
  - 설명: 서버 → 클라이언트 단방향
- 패턴: Out-In:
  - URI: `http://www.w3.org/ns/wsdl/out-in`
  - 설명: 서버 시작 요청-응답

## SOAP vs REST 비교

### 기본 개념 비교

- 구분: 정의:
  - SOAP: W3C 공식 프로토콜
  - REST: 아키텍처 스타일
- 구분: 데이터 형식:
  - SOAP: XML만 지원
  - REST: JSON, XML, HTML, 텍스트 등
- 구분: 전송 프로토콜:
  - SOAP: HTTP, SMTP, TCP 등
  - REST: 주로 HTTP
- 구분: 상태 관리:
  - SOAP: Stateful 가능
  - REST: Stateless
- 구분: 계약:
  - SOAP: WSDL (엄격한 계약)
  - REST: OpenAPI/Swagger (선택적)

### 상세 비교

- SOAP, REST
- 메시지 크기, 큼 (XML 오버헤드), 작음 (JSON 기반)
- 보안, WS-Security 내장, HTTPS + OAuth/JWT
- 트랜잭션, WS-AtomicTransaction, 애플리케이션 수준
- 신뢰성, WS-ReliableMessaging, 애플리케이션 수준
- 캐싱, 복잡함, HTTP 캐싱 활용
- 학습 곡선, 높음, 낮음
- 도구 지원, 강력한 IDE 지원, 광범위한 라이브러리

### 성능 비교

- 요청 크기 비교 (동일한 데이터)
- SOAP 요청:
- <?xml version="1.0"?>
- `<soap:Envelope xmlns:soap="...">`
- `<soap:Header>`...`</soap:Header>`
- `<soap:Body>`
- `<GetUser>`
- `<userId>`123`</userId>`
- `</GetUser>`
- `</soap:Body>`
- `</soap:Envelope>`
- 크기: ~500 bytes
- REST 요청:
- GET /users/123 HTTP/1.1
- Accept: application/json
- 크기: ~50 bytes

### 사용 시나리오

#### SOAP을 선택해야 하는 경우

- 엔터프라이즈 통합
  - 레거시 시스템과의 호환성 필요
  - 기존 SOAP 기반 인프라 활용

- 높은 보안 요구사항
  - WS-Security를 통한 메시지 수준 암호화
  - 디지털 서명 필요

- 트랜잭션 무결성
  - ACID 트랜잭션 지원 필요
  - 분산 트랜잭션 처리

- 신뢰성 있는 메시징
  - 메시지 전달 보장
  - 순서 보장 및 중복 제거

#### REST를 선택해야 하는 경우

- 웹/모바일 애플리케이션
  - 경량 통신 필요
  - 브라우저 친화적

- 마이크로서비스
  - 독립적 배포
  - 느슨한 결합

- 공개 API
  - 쉬운 접근성
  - 광범위한 클라이언트 지원

- 실시간 서비스
  - 낮은 지연시간 요구
  - 높은 처리량

## WS-* 확장 명세

- SOAP은 다양한 WS-\* (Web Services) 확장 명세를 통해 엔터프라이즈 기능을 제공함.

### WS-Security

- OASIS에서 관리하는 보안 표준으로, SOAP 메시지의 무결성과 기밀성을 보장함.

#### 주요 기능

WS-Security의 기능은 인증 토큰, XML 서명, XML 암호화로 나뉜다. Username, X.509, Kerberos, SAML 토큰으로 인증 정보를 전달하고, XML 서명으로 메시지 무결성과 부인방지를 다룬다. XML 암호화는 메시지 기밀성을 보호하며 메시지 일부만 암호화할 수도 있다.

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

### WS-ReliableMessaging

- OASIS 표준으로, 네트워크 장애 상황에서도 메시지 전달을 보장함.

#### 핵심 기능

- 기능: AtLeastOnce:
  - 설명: 최소 1회 전달 보장
- 기능: AtMostOnce:
  - 설명: 최대 1회 전달 (중복 방지)
- 기능: ExactlyOnce:
  - 설명: 정확히 1회 전달
- 기능: InOrder:
  - 설명: 순서 보장

#### 동작 흐름

- 클라이언트, 서버
- 클라이언트 → CreateSequence → 서버
- 서버 → CreateSequenceResponse → 클라이언트
- 클라이언트 → Message 1 (Seq=1) → 서버
- 서버 → SequenceAcknowledgement → 클라이언트
- 클라이언트 → Message 2 (Seq=2) → 서버
- (손실됨)
- 클라이언트 → Message 3 (Seq=3) → 서버
- 서버 → SequenceAck (Ack: 1,3) → 클라이언트
- 클라이언트 → Message 2 (재전송) → 서버
- 서버 → SequenceAck (Ack: 1-3) → 클라이언트
- 클라이언트 → TerminateSequence → 서버
- 서버 → TerminateSequenceResponse → 클라이언트

### 기타 WS-* 명세

- 명세: WS-Policy:
  - 목적: 서비스 정책 정의
  - 관리 기관: W3C
- 명세: WS-Addressing:
  - 목적: 전송 독립적 메시지 라우팅
  - 관리 기관: W3C
- 명세: WS-AtomicTransaction:
  - 목적: 분산 트랜잭션 조정
  - 관리 기관: OASIS
- 명세: WS-Coordination:
  - 목적: 분산 처리 조정
  - 관리 기관: OASIS
- 명세: WS-Trust:
  - 목적: 보안 토큰 발급/교환
  - 관리 기관: OASIS
- 명세: WS-SecureConversation:
  - 목적: 보안 세션 설정
  - 관리 기관: OASIS
- 명세: WS-Federation:
  - 목적: 연합 ID 관리
  - 관리 기관: OASIS

### WS-* 스택 구조

- Business Process
- (WS-BPEL, WS-CDL)
- WS-Reliable, WS-Atomic, WS-Security
- Messaging, Transaction, WS-Trust
- WS-Federation
- WS-Policy, WS-Addressing
- SOAP
- WSDL
- HTTP, SMTP, TCP, JMS

## 실제 SOAP 요청/응답 예제

### 기본 RPC 스타일 요청/응답

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

### 오류 응답 (Fault)

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

### WS-Security를 사용한 요청

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

### 다양한 언어별 SOAP 클라이언트 구현

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

## 현대 시스템에서의 SOAP 사용 사례

### 금융 산업

- 금융 산업은 SOAP의 핵심 사용 영역임.

#### 사용 이유

- 금융 시스템 요구사항
- 보안성, → WS-Security (메시지 수준 암호화)
- 신뢰성, → WS-ReliableMessaging
- 트랜잭션, → WS-AtomicTransaction (ACID 보장)
- 감사 추적, → XML 메시지 전체 로깅
- 표준 계약, → WSDL (엄격한 인터페이스 정의)

#### 실제 사용 사례

- 시스템: SWIFT:
  - 설명: 국제 은행간 금융 메시징
- 시스템: FIX Protocol:
  - 설명: 증권 거래 통신 (SOAP over MQ)
- 시스템: 코어뱅킹 연동:
  - 설명: 은행 내부 시스템 통합
- 시스템: 결제 게이트웨이:
  - 설명: 카드사-가맹점 통신
- 시스템: 보험 청구:
  - 설명: 보험사 간 청구 처리

#### 금융 SOAP 아키텍처 예시

- 모바일, 웹 앱, ATM/키오스크
- 앱
- API Gateway
- (REST → SOAP 변환)
- Enterprise Service Bus
- (ESB)
- 코어뱅킹, 신용평가, 외환 시스템
- (SOAP), (SOAP), (SOAP)

### 기업 시스템 (Enterprise)

#### ERP/CRM 통합

- 시스템: SAP:
  - SOAP 사용: RFC, BAPI를 SOAP으로 노출
- 시스템: Oracle Financials:
  - SOAP 사용: SOAP 웹 서비스 기본 제공
- 시스템: Microsoft Dynamics:
  - SOAP 사용: SOAP/OData 하이브리드
- 시스템: Salesforce:
  - SOAP 사용: SOAP API (Enterprise Edition)

#### B2B 통합

- Company A, Company B
- SOAP/WS-Security
- ERP, ERP
- WS-ReliableMessaging
- SCM, SCM

### 정부/공공 시스템

#### 전자정부 표준프레임워크

- 한국의 전자정부 표준프레임워크는 SOAP 기반 연계를 지원함.

- 행정정보 공동이용센터
- 주민등록, 건축물대장, 자동차, ...
- 정보, 정보, 정보
- SOAP Gateway
- (WS-Security)
- 민원 시스템, 금융기관, 공공기관

### 의료 시스템

#### HL7/FHIR과 SOAP

- 표준: HL7 v2:
  - 프로토콜: 전용 프로토콜
  - 용도: 레거시 의료 데이터 교환
- 표준: HL7 v3:
  - 프로토콜: SOAP
  - 용도: 임상 문서 교환
- 표준: FHIR:
  - 프로토콜: REST/SOAP
  - 용도: 차세대 의료 정보 교환

### 레거시 시스템 통합 전략

#### REST-SOAP 게이트웨이 패턴

REST/JSON을 사용하는 클라이언트와 SOAP/XML을 사용하는 레거시 시스템 사이에는 API Gateway를 둘 수 있다. 게이트웨이의 REST 엔드포인트가 요청을 받으면 변환 계층과 SOAP 클라이언트를 거쳐 레거시 SOAP 서비스를 호출한다. 다음 예제는 잔액 조회 요청을 이 방식으로 변환한다.

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

## 참고 자료

- 문서: SOAP 명세 목록:
  - URL: https://www.w3.org/TR/soap/
- 문서: SOAP 1.2 Part 0: Primer:
  - URL: https://www.w3.org/TR/2007/REC-soap12-part0-20070427/
- 문서: SOAP 1.2 Part 1: Messaging Framework:
  - URL: https://www.w3.org/TR/2007/REC-soap12-part1-20070427/
- 문서: SOAP 1.2 Part 2: Adjuncts:
  - URL: https://www.w3.org/TR/2007/REC-soap12-part2-20070427/
- 문서: WSDL 2.0 Core Language:
  - URL: https://www.w3.org/TR/2007/REC-wsdl20-20070626/
- 문서: WSDL 2.0 Adjuncts:
  - URL: https://www.w3.org/TR/wsdl20-adjuncts/
- 문서: WS-Addressing:
  - URL: https://www.w3.org/TR/ws-addr-core/

- 문서: WS-Security 1.1.1:
  - URL: https://docs.oasis-open.org/wss-m/wss/v1.1.1/os/wss-SOAPMessageSecurity-v1.1.1-os.html
- 문서: WS-Security TC:
  - URL: https://www.oasis-open.org/committees/wss/
- 문서: WS-ReliableMessaging 1.2:
  - URL: https://www.oasis-open.org/standard/ws-reliablemessaging/
- 문서: WS-ReliableMessaging 1.1 Spec:
  - URL: https://docs.oasis-open.org/ws-rx/wsrm/200702/wsrm-1.1-spec-os-01-e1.html

- RFC: RFC 4227:
  - 제목: Using the Simple Object Access Protocol (SOAP) in Blocks Extensible Exchange Protocol (BEEP)
  - URL: https://tools.ietf.org/html/rfc4227
- RFC: RFC 3902:
  - 제목: The "application/soap+xml" media type
  - URL: https://tools.ietf.org/html/rfc3902

- 도구: SoapUI:
  - 설명: SOAP/REST API 테스트 도구
- 도구: Postman:
  - 설명: API 개발 및 테스트 플랫폼
- 도구: Wireshark:
  - 설명: 네트워크 프로토콜 분석

- 언어: Java:
  - 라이브러리: JAX-WS, Apache CXF, Apache Axis2
- 언어: Python:
  - 라이브러리: zeep, suds-community
- 언어: JavaScript/Node.js:
  - 라이브러리: soap, strong-soap
- 언어: C#/.NET:
  - 라이브러리: WCF, System.Web.Services
- 언어: PHP:
  - 라이브러리: SoapClient (built-in), nusoap
- 언어: Go:
  - 라이브러리: gowsdl

- [A Brief History of SOAP (XML.com)](https://www.xml.com/pub/a/ws/2001/04/04/soap.html)
- [SOAP and REST At Odds - The History of the Web](https://thehistoryoftheweb.com/soap-rest-odds/)
- [Understanding WS-Security (Oracle)](https://docs.oracle.com/cd/G10809_01/pt861pbr4/eng/pt/tsec/UnderstandingWS-Security-c07766.html)
- [AWS SOAP vs REST 비교](https://aws.amazon.com/compare/the-difference-between-soap-rest/)

## 관련 학습

- [OpenAPI Swagger](05-OpenAPI-Swagger.md)
