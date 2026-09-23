# TLS와 HTTPS

## HTTPS와 TLS Handshake

### HTTPS 개요

HTTPS는 HTTP에 TLS를 더한 통신 방식이다. 아래에서 다루는 계층 구조에서는 IP와 TCP 위에 TLS가 있고, 그 위에서 HTTP의 GET, POST 같은 메시지를 주고받는다. TLS는 이 HTTP 통신에 암호화, 인증, 무결성 보호를 제공한다.

- TLS가 제공하는 보안:
- 기밀성 (Confidentiality): 대칭키 암호화로 데이터 보호
- 무결성 (Integrity): MAC (Message Authentication Code)로 변조 감지
- 인증 (Authentication): 인증서로 서버(선택적으로 클라이언트) 신원 확인

### TLS 1.2 Handshake

- TLS 1.2 Full Handshake (2-RTT)
- Client, Server
- ClientHello, →
- (TLS version, cipher suites,, RTT 1
- random, session_id, extensions)
- ←, ServerHello
- (selected cipher, random)
- Server → Certificate → Client
- ←, ServerKeyExchange (ECDHE params)
- ←, ServerHelloDone
- ClientKeyExchange, →
- (ECDHE public key), RTT 2
- Client → ChangeCipherSpec → Server
- Client → Finished (encrypted) → Server
- Server → ChangeCipherSpec → Client
- Server → Finished (encrypted) → Client
- Client → Application Data (encrypted) → Server
- Server → Application Data (encrypted) → Client

### TLS 1.3 Handshake (RFC 8446)

- TLS 1.3의 주요 개선사항:
- 1-RTT Handshake (0-RTT 재연결 지원)
- 불안전한 암호 스위트 제거 (RSA 키 교환, SHA-1, RC4, DES 등)
- 핸드셰이크 메시지 암호화 (ServerHello 이후)
- Forward Secrecy 필수화

- TLS 1.3 Full Handshake (1-RTT)
- Client, Server
- ClientHello, →
- key_share (ECDHE public key)
- supported_versions (TLS 1.3)
- signature_algorithms
- psk_key_exchange_modes (optional), RTT 1
- ←, ServerHello
- key_share (ECDHE public key)
- supported_versions
- Server → {EncryptedExtensions} → Client
- Server → {Certificate} → Client
- Server → {CertificateVerify} → Client
- Server → {Finished} → Client
- Client → {Finished} → Server
- Client → [Application Data] → Server
- { } = 암호화된 핸드셰이크 메시지
- [ ] = 암호화된 애플리케이션 데이터

- TLS 1.3 0-RTT Resumption:
- TLS 1.3 0-RTT Resumption
- Client, Server
- ClientHello, →
- early_data
- pre_shared_key
- Client → [0-RTT Application Data], , 0-RTT! → Server
- ←, ServerHello
- Server → {EncryptedExtensions} → Client
- early_data (accepted)
- Server → {Finished} → Client
- Client → {EndOfEarlyData} → Server
- Client → {Finished} → Server
- 주의: 0-RTT 데이터는 Replay 공격에 취약할 수 있음
- 멱등하지 않은 요청에는 사용 주의

### 인증서 체인 검증

인증서 체인은 Root CA에서 중간 CA를 거쳐 서버 인증서로 이어진다. 예를 들어 브라우저나 OS에 신뢰 앵커로 내장된 자체 서명 Root CA 인증서가 중간 CA 인증서를 발급하고, 중간 CA가 `www.example.com`의 End-Entity 인증서를 발급한다.

클라이언트는 발급 방향과 반대로 `End-Entity → Intermediate → Root` 순서로 체인을 확인한다. 다음 코드는 유효 기간, 서명 연결, 신뢰하는 루트, 호스트 이름과 폐기 여부 등을 확인하는 과정을 개념적으로 보여준다.

- 인증서 검증 과정:

```python
# 인증서 검증 개념적 구현
class CertificateValidator:
    def __init__(self, trusted_roots):
        self.trusted_roots = trusted_roots  # 신뢰하는 Root CA 목록

    def validate_chain(self, cert_chain, hostname):
        """
        인증서 체인 검증

        Args:
            cert_chain: [서버 인증서, 중간 CA, ...]
            hostname: 접속하려는 도메인

        Returns:
            bool: 검증 성공 여부
        """
        checks = [
            self.check_not_expired(cert_chain),
            self.check_signature_chain(cert_chain),
            self.check_trusted_root(cert_chain),
            self.check_hostname(cert_chain[0], hostname),
            self.check_revocation(cert_chain),  # OCSP, CRL
            self.check_key_usage(cert_chain[0]),
        ]
        return all(checks)

    def check_not_expired(self, cert_chain):
        """모든 인증서의 유효 기간 확인"""
        for cert in cert_chain:
            if cert.not_before > now or cert.not_after < now:
                return False
        return True

    def check_signature_chain(self, cert_chain):
        """각 인증서가 상위 CA에 의해 서명되었는지 확인"""
        for i in range(len(cert_chain) - 1):
            if not cert_chain[i].verify_signature(cert_chain[i + 1].public_key):
                return False
        return True

    def check_hostname(self, server_cert, hostname):
        """
        인증서의 CN 또는 SAN이 hostname과 일치하는지 확인

        Subject Alternative Name (SAN):
        - DNS:www.example.com
        - DNS:*.example.com (와일드카드)
        """
        return hostname in server_cert.get_subject_alt_names()
```

### TLS 1.2 vs TLS 1.3 비교

- 특성: Handshake RTT:
  - TLS 1.2: 2-RTT
  - TLS 1.3: 1-RTT (0-RTT 재연결)
- 특성: 암호화 시작 시점:
  - TLS 1.2: Finished 후
  - TLS 1.3: ServerHello 직후
- 특성: Forward Secrecy:
  - TLS 1.2: 선택적
  - TLS 1.3: 필수 (ECDHE만)
- 특성: RSA 키 교환:
  - TLS 1.2: 지원
  - TLS 1.3: 제거됨
- 특성: 지원 암호 스위트:
  - TLS 1.2: 수백 개
  - TLS 1.3: 5개 (AEAD만)
- 특성: 메시지 암호화:
  - TLS 1.2: 일부
  - TLS 1.3: 대부분 (ServerHello 제외)
- 특성: 0-RTT:
  - TLS 1.2: 미지원
  - TLS 1.3: 지원 (재연결 시)

- TLS 1.3 지원 암호 스위트 (RFC 8446):
- TLS_AES_128_GCM_SHA256
- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256
- TLS_AES_128_CCM_SHA256
- TLS_AES_128_CCM_8_SHA256

## HTTPS란?

- HTTPS(Hypertext Transfer Protocol Secure)는 HTTP에 TLS(Transport Layer Security) 암호화 계층을 추가한 프로토콜임.

- HTTP + TLS = HTTPS

## TLS의 주요 목표

- 목표: 기밀성 (Confidentiality):
  - 설명: 데이터를 암호화하여 도청 방지
- 목표: 무결성 (Integrity):
  - 설명: 데이터 변조 감지 (MAC 사용)
- 목표: 인증 (Authentication):
  - 설명: 인증서를 통한 서버 신원 확인

## SSL vs TLS 역사

- SSL 1.0 (1994) - 공개되지 않음
- SSL 2.0 (1995) - 심각한 보안 취약점
- SSL 3.0 (1996) - POODLE 공격에 취약
- TLS 1.0 (1999) - SSL 3.0 기반, 보안 개선
- TLS 1.1 (2006) - CBC 공격 방어
- TLS 1.2 (2008) - SHA-256, GCM 지원
- TLS 1.3 (2018) - 대폭 개선, 현재 권장

- 참고: SSL은 공식적으로 사용 중단되었지만, 관례적으로 "SSL"이라는 용어가 계속 사용됨.

## TLS 1.2 vs TLS 1.3 Handshake

### TLS 1.2 Handshake (2-RTT)

- Client, Server
- ClientHello
- TLS 버전
- 지원 암호화 스위트
- 랜덤 값
- ServerHello
- 선택된 암호화 스위트
- 랜덤 값
- Certificate
- 서버 인증서 체인
- ServerKeyExchange (선택적)
- DH 파라미터
- ServerHelloDone
- ClientKeyExchange
- Pre-Master Secret
- ChangeCipherSpec
- Finished (암호화됨)
- ChangeCipherSpec
- Finished (암호화됨)
- 암호화된 통신 시작

### TLS 1.3 Handshake (1-RTT)

- Client, Server
- ClientHello
- TLS 버전
- 지원 암호화 스위트
- 랜덤 값
- key_share (DH 공개키)
- supported_versions
- ServerHello
- 선택된 암호화 스위트
- 랜덤 값
- key_share (DH 공개키)
- EncryptedExtensions (암호화됨)
- Certificate (암호화됨)
- CertificateVerify (암호화됨)
- Finished (암호화됨)
- Finished (암호화됨)
- 암호화된 통신 시작

### TLS 1.3 0-RTT Resumption

- Client, Server
- ClientHello
- early_data extension
- pre_shared_key (이전 세션 티켓)
- early_data (암호화된 요청 데이터)
- ServerHello
- early_data 수락/거부
- (이후 일반 1-RTT 핸드셰이크 계속)

### 주요 차이점 비교

- 특성: RTT (Round Trip Time):
  - TLS 1.2: 2-RTT
  - TLS 1.3: 1-RTT (0-RTT 가능)
- 특성: 키 교환:
  - TLS 1.2: RSA, DHE, ECDHE
  - TLS 1.3: ECDHE, DHE만 지원
- 특성: 암호화 스위트:
  - TLS 1.2: 37개
  - TLS 1.3: 5개
- 특성: Perfect Forward Secrecy:
  - TLS 1.2: 선택적
  - TLS 1.3: 필수
- 특성: 인증서 암호화:
  - TLS 1.2: 평문 전송
  - TLS 1.3: 암호화됨
- 특성: 레거시 알고리즘:
  - TLS 1.2: 지원
  - TLS 1.3: 제거

### 제거된 알고리즘 (TLS 1.3)

- 제거된 키 교환:
- RSA (PFS 미지원)
- Static DH
- Static ECDH
- 제거된 암호화:
- RC4
- DES/3DES
- AES-CBC (GCM만 지원)
- 제거된 해시:
- MD5
- SHA-1
- 제거된 기타:
- Compression (CRIME 공격 방지)
- Renegotiation

### TLS 1.3 지원 암호화 스위트

- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256
- TLS_AES_128_GCM_SHA256
- TLS_AES_128_CCM_8_SHA256
- TLS_AES_128_CCM_SHA256

## 인증서 체계 (PKI)

### PKI (Public Key Infrastructure) 개요

- Root CA (최상위 인증기관)
- Intermediate CA, Intermediate CA
- End Entity, End Entity
- (서버 인증서), (서버 인증서)

### 인증서 체인 검증 과정

```java
// 인증서 체인 검증 로직 (개념적)
public class CertificateChainValidator {

    public boolean validateCertificateChain(List<X509Certificate> chain,
                                            Set<TrustAnchor> trustAnchors)
                                            throws Exception {

        // 1. 체인의 각 인증서 검증
        for (int i = 0; i < chain.size() - 1; i++) {
            X509Certificate cert = chain.get(i);
            X509Certificate issuer = chain.get(i + 1);

            // 유효 기간 확인
            cert.checkValidity();

            // 발급자 검증
            if (!cert.getIssuerX500Principal().equals(issuer.getSubjectX500Principal())) {
                throw new CertificateException("발급자 불일치");
            }

            // 서명 검증
            cert.verify(issuer.getPublicKey());
        }

        // 2. 루트 인증서가 신뢰 저장소에 있는지 확인
        X509Certificate rootCert = chain.get(chain.size() - 1);
        for (TrustAnchor anchor : trustAnchors) {
            if (anchor.getTrustedCert().equals(rootCert)) {
                return true;
            }
        }

        return false;
    }
}
```

### 인증서 구조 (X.509 v3)

```text
Certificate ::= SEQUENCE {
    tbsCertificate       TBSCertificate,
    signatureAlgorithm   AlgorithmIdentifier,
    signatureValue       BIT STRING
}

TBSCertificate ::= SEQUENCE {
    version         [0]  Version DEFAULT v1,
    serialNumber         CertificateSerialNumber,
    signature            AlgorithmIdentifier,
    issuer               Name,
    validity             Validity,
    subject              Name,
    subjectPublicKeyInfo SubjectPublicKeyInfo,
    extensions      [3]  Extensions OPTIONAL
}
```

### 주요 인증서 확장 필드

```bash
# OpenSSL로 인증서 내용 확인
openssl x509 -in certificate.pem -text -noout

# 주요 확장 필드
X509v3 extensions:
    X509v3 Key Usage: critical
        Digital Signature, Key Encipherment

    X509v3 Extended Key Usage:
        TLS Web Server Authentication, TLS Web Client Authentication

    X509v3 Subject Alternative Name:
        DNS:example.com, DNS:*.example.com

    X509v3 Certificate Policies:
        Policy: 2.23.140.1.2.2  # Domain Validated (DV)

    Authority Information Access:
        OCSP - URI:http://ocsp.example.com
        CA Issuers - URI:http://certs.example.com/intermediate.crt

    X509v3 CRL Distribution Points:
        URI:http://crl.example.com/crl.pem
```

### 인증서 유형

- 유형: DV (Domain Validated):
  - 검증 수준: 도메인 소유권만 확인
  - 발급 시간: 몇 분
  - 비용: 무료~저가
- 유형: OV (Organization Validated):
  - 검증 수준: 조직 실재 확인
  - 발급 시간: 1-3일
  - 비용: 중간
- 유형: EV (Extended Validation):
  - 검증 수준: 철저한 조직 검증
  - 발급 시간: 1-2주
  - 비용: 고가

### 인증서 폐기 확인

#### OCSP (Online Certificate Status Protocol)

```java
// OCSP 응답 확인
public boolean checkOCSP(X509Certificate cert, X509Certificate issuerCert)
        throws Exception {

    // OCSP Responder URL 추출
    String ocspUrl = getOCSPUrl(cert);

    // OCSP 요청 생성
    OCSPReqBuilder builder = new OCSPReqBuilder();
    CertificateID certId = new CertificateID(
        new SHA1DigestCalculator(),
        new X509CertificateHolder(issuerCert.getEncoded()),
        cert.getSerialNumber()
    );
    builder.addRequest(certId);
    OCSPReq request = builder.build();

    // OCSP 요청 전송
    byte[] responseBytes = sendOCSPRequest(ocspUrl, request.getEncoded());

    // 응답 파싱
    OCSPResp response = new OCSPResp(responseBytes);
    BasicOCSPResp basicResponse = (BasicOCSPResp) response.getResponseObject();

    // 인증서 상태 확인
    SingleResp[] responses = basicResponse.getResponses();
    CertificateStatus status = responses[0].getCertStatus();

    return status == CertificateStatus.GOOD;
}
```

#### OCSP Stapling

```nginx
# Nginx OCSP Stapling 설정
server {
    listen 443 ssl http2;

    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # OCSP Stapling 활성화
    ssl_stapling on;
    ssl_stapling_verify on;

    # 신뢰 체인 (Intermediate + Root)
    ssl_trusted_certificate /etc/ssl/certs/chain.pem;

    # OCSP Responder 쿼리용 리졸버
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
}
```

### Let's Encrypt를 이용한 무료 인증서

```bash
# Certbot으로 인증서 발급
sudo certbot certonly --webroot \
    -w /var/www/html \
    -d example.com \
    -d www.example.com

# 자동 갱신 (cron)
0 0 1 * * /usr/bin/certbot renew --quiet
```

## Certificate Pinning

### 개념

- Certificate Pinning은 예상되는 인증서나 공개키를 애플리케이션에 미리 저장하여, 중간자 공격(MITM)을 방지하는 기술임.

### Pinning 유형

- 유형: Certificate Pinning:
  - 설명: 전체 인증서 고정
  - 유연성: 낮음
  - 보안성: 높음
- 유형: Public Key Pinning:
  - 설명: 공개키만 고정
  - 유연성: 중간
  - 보안성: 높음
- 유형: CA Pinning:
  - 설명: 발급 CA 고정
  - 유연성: 높음
  - 보안성: 중간

### Android Certificate Pinning

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2025-01-01">
            <!-- 기본 핀 -->
            <pin digest="SHA-256">base64encodedSHA256hash==</pin>
            <!-- 백업 핀 (인증서 교체 시 사용) -->
            <pin digest="SHA-256">backupBase64EncodedHash==</pin>
        </pin-set>
        <trust-anchors>
            <!-- 시스템 CA 신뢰 (선택적) -->
            <certificates src="system"/>
        </trust-anchors>
    </domain-config>
</network-security-config>
```

```kotlin
// AndroidManifest.xml에 설정 적용
// android:networkSecurityConfig="@xml/network_security_config"

// OkHttp를 사용한 프로그래매틱 Pinning
val certificatePinner = CertificatePinner.Builder()
    .add(
        "api.example.com",
        "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="  // 백업
    )
    .build()

val client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()
```

### iOS Certificate Pinning

```swift
// URLSession을 사용한 Pinning
class PinningDelegate: NSObject, URLSessionDelegate {

    let pinnedCertificates: [SecCertificate]

    init(certificates: [SecCertificate]) {
        self.pinnedCertificates = certificates
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod ==
              NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 서버 인증서 추출
        let serverCertificateCount = SecTrustGetCertificateCount(serverTrust)
        guard serverCertificateCount > 0,
              let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 핀된 인증서와 비교
        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data

        for pinnedCertificate in pinnedCertificates {
            let pinnedCertificateData = SecCertificateCopyData(pinnedCertificate) as Data
            if serverCertificateData == pinnedCertificateData {
                let credential = URLCredential(trust: serverTrust)
                completionHandler(.useCredential, credential)
                return
            }
        }

        completionHandler(.cancelAuthenticationChallenge, nil)
    }
}
```

### Public Key Pinning (더 유연한 방식)

```java
// Java에서 공개키 핀닝
public class PublicKeyPinningTrustManager implements X509TrustManager {

    private static final Set<String> PINNED_PUBLIC_KEY_HASHES = Set.of(
        "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
    );

    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType)
            throws CertificateException {

        // 기본 인증서 체인 검증
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm()
        );
        tmf.init((KeyStore) null);

        for (TrustManager tm : tmf.getTrustManagers()) {
            if (tm instanceof X509TrustManager) {
                ((X509TrustManager) tm).checkServerTrusted(chain, authType);
            }
        }

        // 공개키 해시 검증
        boolean pinMatched = false;
        for (X509Certificate cert : chain) {
            String publicKeyHash = getPublicKeyHash(cert);
            if (PINNED_PUBLIC_KEY_HASHES.contains("sha256/" + publicKeyHash)) {
                pinMatched = true;
                break;
            }
        }

        if (!pinMatched) {
            throw new CertificateException("공개키 핀 검증 실패");
        }
    }

    private String getPublicKeyHash(X509Certificate cert) throws CertificateException {
        try {
            byte[] publicKeyBytes = cert.getPublicKey().getEncoded();
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(publicKeyBytes);
            return Base64.getEncoder().encodeToString(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new CertificateException(e);
        }
    }

    @Override
    public void checkClientTrusted(X509Certificate[] chain, String authType) {}

    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return new X509Certificate[0];
    }
}
```

### Pinning의 위험성과 대안

인증서를 고정하면 인증서 교체가 애플리케이션 동작에도 영향을 준다. 백업 핀이 없으면 교체 직후 서비스에 연결하지 못할 수 있고, 앱 업데이트가 늦게 배포되는 환경에서는 문제가 더 오래 지속된다.

이를 고려해 다음 항목을 함께 검토한다.

- 최소 2개 이상의 백업 핀 설정
- 만료일로 핀 무효화 시점 명시
- Intermediate CA 핀닝 (Root보다 유연, Leaf보다 안전)
- 점진적 롤아웃으로 문제 사전 감지

- 참고: HTTP Public Key Pinning (HPKP) 헤더는 위험성으로 인해 deprecated 되었음 (Chrome 72에서 제거).

## SSL/TLS 취약점

### POODLE (Padding Oracle On Downgraded Legacy Encryption)

- 발견: 2014년 | 영향: SSL 3.0

- 공격 원리:
- 공격자가 TLS 연결을 SSL 3.0으로 다운그레이드 유도
- SSL 3.0 CBC 모드의 패딩 검증 취약점 악용
- 암호화된 쿠키나 토큰 추출 가능
- 방어:
- SSL 3.0 완전 비활성화
- TLS_FALLBACK_SCSV 활성화

```nginx
# Nginx에서 SSL 3.0 비활성화
ssl_protocols TLSv1.2 TLSv1.3;
```

### BEAST (Browser Exploit Against SSL/TLS)

- 발견: 2011년 | 영향: TLS 1.0 CBC 모드

- 공격 원리:
- CBC 모드에서 Initialization Vector(IV) 예측 가능
- 선택 평문 공격(Chosen Plaintext Attack)으로
- 암호화된 쿠키 추출
- 방어:
- TLS 1.1 이상 사용 (랜덤 IV)
- RC4 사용 (하지만 RC4도 취약)
- 1/n-1 record splitting

### Heartbleed (CVE-2014-0160)

- 발견: 2014년 | 영향: OpenSSL 1.0.1 ~ 1.0.1f

```c
공격 원리:
1. TLS Heartbeat 확장의 버퍼 오버리드 버그
2. 서버 메모리에서 최대 64KB 데이터 유출
3. 개인키, 세션키, 사용자 데이터 노출 가능

취약한 코드 (개념적):
int tls1_process_heartbeat(SSL *s) {
    unsigned short payload_length;
    unsigned char *payload;

    // 문제: 실제 데이터 길이를 확인하지 않음
    n2s(p, payload_length);  // 클라이언트가 보낸 길이
    payload = p;

    // 클라이언트가 지정한 길이만큼 복사 (버퍼 오버리드)
    memcpy(bp, payload, payload_length);
    ...
}
```

```bash
# Heartbleed 취약점 확인
nmap -p 443 --script ssl-heartbleed example.com

# 취약한 버전 확인
openssl version
# OpenSSL 1.0.1 ~ 1.0.1f는 취약

# 대응
# 1. OpenSSL 1.0.1g 이상으로 업데이트
# 2. 모든 인증서 재발급
# 3. 모든 사용자 비밀번호 변경
```

### CRIME / BREACH

- CRIME (2012): TLS 압축 취약점
- BREACH (2013): HTTP 압축 취약점

두 공격은 알려진 문자열과 비밀 데이터가 함께 압축될 때 나타나는 크기 차이를 이용한다. 공격자가 추측한 문자열을 바꾸며 압축된 데이터의 크기를 반복해서 관찰하면 비밀 데이터를 추출할 수 있다.

방어 방법으로는 TLS 압축 비활성화, HTTP 응답에서 비밀 토큰과 사용자 입력을 함께 포함하지 않기, 요청마다 CSRF 토큰 변경하기가 있다. TLS 1.3에서는 TLS 압축이 제거됐다.

### DROWN (Decrypting RSA with Obsolete and Weakened eNcryption)

- 발견: 2016년 | 영향: SSLv2 지원 서버

- 공격 원리:
- SSLv2를 지원하는 서버의 RSA 키 악용
- TLS 세션 복호화 가능
- 같은 인증서를 사용하는 모든 서버에 영향
- 방어:
- SSLv2 완전 비활성화
- 서버 간 인증서 공유 주의

### Lucky Thirteen

- 발견: 2013년 | 영향: TLS CBC 모드

- 공격 원리:
- MAC-then-Encrypt 구조의 타이밍 차이 악용
- 패딩 검증 시간 차이로 정보 유출
- 방어:
- AEAD (Authenticated Encryption with Associated Data) 사용
- AES-GCM, ChaCha20-Poly1305 권장

### 취약점 대응 종합 설정

```nginx
# Nginx 보안 설정 (2024 Best Practice)
server {
    listen 443 ssl http2;
    server_name example.com;

    # 인증서
    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # 프로토콜 (TLS 1.2, 1.3만 허용)
    ssl_protocols TLSv1.2 TLSv1.3;

    # 암호화 스위트
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;

    # ECDH 곡선
    ssl_ecdh_curve secp384r1;

    # 세션 설정
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # 기타 보안 헤더
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
}
```

## 참고 자료

- [RFC 9110 - HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [RFC 9113 - HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 9114 - HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
- [RFC 9000 - QUIC](https://datatracker.ietf.org/doc/html/rfc9000)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [MDN Web Docs - HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [Cloudflare - A Detailed Look at RFC 8446 (TLS 1.3)](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- [web.dev - HTTP/2](https://web.dev/articles/performance-http2)

- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5246 - TLS 1.2](https://datatracker.ietf.org/doc/html/rfc5246)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)
- [OWASP TLS Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html)

## 관련 학습

- [TCP](../../네트워크/전송과-소켓/01-TCP.md)
- [OAuth 2.0](../인증과-인가/02-OAuth-2.0.md)
- [해싱과 암호화](01-해싱과-암호화.md)
- [HTTP HTTPS](../../네트워크/인터넷과-웹/03-HTTP-HTTPS.md)
- [CORS](../웹-보안/02-CORS.md)
