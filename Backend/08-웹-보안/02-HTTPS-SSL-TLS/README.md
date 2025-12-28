# HTTPS와 SSL/TLS

## 목차
1. [개요](#개요)
2. [TLS 1.2 vs TLS 1.3 Handshake](#tls-12-vs-tls-13-handshake)
3. [인증서 체계 (PKI)](#인증서-체계-pki)
4. [Certificate Pinning](#certificate-pinning)
5. [SSL/TLS 취약점](#ssltls-취약점)
6. [실무 적용](#실무-적용)
---

## 개요

### HTTPS란?

HTTPS(Hypertext Transfer Protocol Secure)는 HTTP에 TLS(Transport Layer Security) 암호화 계층을 추가한 프로토콜입니다.

```
HTTP + TLS = HTTPS
```

### TLS의 주요 목표

| 목표 | 설명 |
|------|------|
| **기밀성 (Confidentiality)** | 데이터를 암호화하여 도청 방지 |
| **무결성 (Integrity)** | 데이터 변조 감지 (MAC 사용) |
| **인증 (Authentication)** | 인증서를 통한 서버 신원 확인 |

### SSL vs TLS 역사

```
SSL 1.0 (1994) - 공개되지 않음
SSL 2.0 (1995) - 심각한 보안 취약점
SSL 3.0 (1996) - POODLE 공격에 취약
TLS 1.0 (1999) - SSL 3.0 기반, 보안 개선
TLS 1.1 (2006) - CBC 공격 방어
TLS 1.2 (2008) - SHA-256, GCM 지원
TLS 1.3 (2018) - 대폭 개선, 현재 권장
```

> **참고**: SSL은 공식적으로 사용 중단되었지만, 관례적으로 "SSL"이라는 용어가 계속 사용됩니다.

---

## TLS 1.2 vs TLS 1.3 Handshake

### TLS 1.2 Handshake (2-RTT)

```
Client                                              Server
   |                                                   |
   |  1. ClientHello                                   |
   |     - TLS 버전                                     |
   |     - 지원 암호화 스위트                            |
   |     - 랜덤 값                                      |
   |-------------------------------------------------->|
   |                                                   |
   |  2. ServerHello                                   |
   |     - 선택된 암호화 스위트                          |
   |     - 랜덤 값                                      |
   |<--------------------------------------------------|
   |                                                   |
   |  3. Certificate                                   |
   |     - 서버 인증서 체인                              |
   |<--------------------------------------------------|
   |                                                   |
   |  4. ServerKeyExchange (선택적)                     |
   |     - DH 파라미터                                  |
   |<--------------------------------------------------|
   |                                                   |
   |  5. ServerHelloDone                               |
   |<--------------------------------------------------|
   |                                                   |
   |  6. ClientKeyExchange                             |
   |     - Pre-Master Secret                           |
   |-------------------------------------------------->|
   |                                                   |
   |  7. ChangeCipherSpec                              |
   |-------------------------------------------------->|
   |                                                   |
   |  8. Finished (암호화됨)                            |
   |-------------------------------------------------->|
   |                                                   |
   |  9. ChangeCipherSpec                              |
   |<--------------------------------------------------|
   |                                                   |
   | 10. Finished (암호화됨)                            |
   |<--------------------------------------------------|
   |                                                   |
   |         === 암호화된 통신 시작 ===                  |
```

### TLS 1.3 Handshake (1-RTT)

```
Client                                              Server
   |                                                   |
   |  1. ClientHello                                   |
   |     - TLS 버전                                     |
   |     - 지원 암호화 스위트                            |
   |     - 랜덤 값                                      |
   |     - key_share (DH 공개키)                        |
   |     - supported_versions                          |
   |-------------------------------------------------->|
   |                                                   |
   |  2. ServerHello                                   |
   |     - 선택된 암호화 스위트                          |
   |     - 랜덤 값                                      |
   |     - key_share (DH 공개키)                        |
   |<--------------------------------------------------|
   |                                                   |
   |  3. EncryptedExtensions (암호화됨)                 |
   |<--------------------------------------------------|
   |                                                   |
   |  4. Certificate (암호화됨)                         |
   |<--------------------------------------------------|
   |                                                   |
   |  5. CertificateVerify (암호화됨)                   |
   |<--------------------------------------------------|
   |                                                   |
   |  6. Finished (암호화됨)                            |
   |<--------------------------------------------------|
   |                                                   |
   |  7. Finished (암호화됨)                            |
   |-------------------------------------------------->|
   |                                                   |
   |         === 암호화된 통신 시작 ===                  |
```

### TLS 1.3 0-RTT Resumption

```
Client                                              Server
   |                                                   |
   |  1. ClientHello                                   |
   |     + early_data extension                        |
   |     + pre_shared_key (이전 세션 티켓)              |
   |     + early_data (암호화된 요청 데이터)            |
   |-------------------------------------------------->|
   |                                                   |
   |  2. ServerHello                                   |
   |     - early_data 수락/거부                         |
   |<--------------------------------------------------|
   |                                                   |
   |  (이후 일반 1-RTT 핸드셰이크 계속)                  |
```

### 주요 차이점 비교

| 특성 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| **RTT (Round Trip Time)** | 2-RTT | 1-RTT (0-RTT 가능) |
| **키 교환** | RSA, DHE, ECDHE | ECDHE, DHE만 지원 |
| **암호화 스위트** | 37개 | 5개 |
| **Perfect Forward Secrecy** | 선택적 | 필수 |
| **인증서 암호화** | 평문 전송 | 암호화됨 |
| **레거시 알고리즘** | 지원 | 제거 |

### 제거된 알고리즘 (TLS 1.3)

```
제거된 키 교환:
- RSA (PFS 미지원)
- Static DH
- Static ECDH

제거된 암호화:
- RC4
- DES/3DES
- AES-CBC (GCM만 지원)

제거된 해시:
- MD5
- SHA-1

제거된 기타:
- Compression (CRIME 공격 방지)
- Renegotiation
```

### TLS 1.3 지원 암호화 스위트

```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
TLS_AES_128_CCM_8_SHA256
TLS_AES_128_CCM_SHA256
```

---

## 인증서 체계 (PKI)

### PKI (Public Key Infrastructure) 개요

```
                Root CA (최상위 인증기관)
                        |
            +-----------+-----------+
            |                       |
      Intermediate CA         Intermediate CA
            |                       |
    +-------+-------+              ...
    |               |
End Entity      End Entity
(서버 인증서)    (서버 인증서)
```

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

```
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

| 유형 | 검증 수준 | 발급 시간 | 비용 |
|------|----------|----------|------|
| **DV (Domain Validated)** | 도메인 소유권만 확인 | 몇 분 | 무료~저가 |
| **OV (Organization Validated)** | 조직 실재 확인 | 1-3일 | 중간 |
| **EV (Extended Validation)** | 철저한 조직 검증 | 1-2주 | 고가 |

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

---

## Certificate Pinning

### 개념

Certificate Pinning은 예상되는 인증서나 공개키를 애플리케이션에 미리 저장하여, 중간자 공격(MITM)을 방지하는 기술입니다.

### Pinning 유형

| 유형 | 설명 | 유연성 | 보안성 |
|------|------|--------|--------|
| **Certificate Pinning** | 전체 인증서 고정 | 낮음 | 높음 |
| **Public Key Pinning** | 공개키만 고정 | 중간 | 높음 |
| **CA Pinning** | 발급 CA 고정 | 높음 | 중간 |

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

```
위험성:
1. 인증서 교체 시 앱이 동작하지 않을 수 있음
2. 백업 핀이 없으면 서비스 장애 발생
3. 앱 업데이트가 느린 경우 문제 심화

대안 및 권장사항:
1. 최소 2개 이상의 백업 핀 설정
2. 만료일 설정으로 핀 무효화 시점 명시
3. Intermediate CA 핀닝 (Root보다 유연, Leaf보다 안전)
4. 점진적 롤아웃으로 문제 사전 감지
```

> **참고**: HTTP Public Key Pinning (HPKP) 헤더는 위험성으로 인해 deprecated 되었습니다 (Chrome 72에서 제거).

---

## SSL/TLS 취약점

### POODLE (Padding Oracle On Downgraded Legacy Encryption)

**발견**: 2014년 | **영향**: SSL 3.0

```
공격 원리:
1. 공격자가 TLS 연결을 SSL 3.0으로 다운그레이드 유도
2. SSL 3.0 CBC 모드의 패딩 검증 취약점 악용
3. 암호화된 쿠키나 토큰 추출 가능

방어:
- SSL 3.0 완전 비활성화
- TLS_FALLBACK_SCSV 활성화
```

```nginx
# Nginx에서 SSL 3.0 비활성화
ssl_protocols TLSv1.2 TLSv1.3;
```

### BEAST (Browser Exploit Against SSL/TLS)

**발견**: 2011년 | **영향**: TLS 1.0 CBC 모드

```
공격 원리:
1. CBC 모드에서 Initialization Vector(IV) 예측 가능
2. 선택 평문 공격(Chosen Plaintext Attack)으로
   암호화된 쿠키 추출

방어:
- TLS 1.1 이상 사용 (랜덤 IV)
- RC4 사용 (하지만 RC4도 취약)
- 1/n-1 record splitting
```

### Heartbleed (CVE-2014-0160)

**발견**: 2014년 | **영향**: OpenSSL 1.0.1 ~ 1.0.1f

```
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

**CRIME** (2012): TLS 압축 취약점
**BREACH** (2013): HTTP 압축 취약점

```
공격 원리:
1. 압축된 데이터 크기를 관찰
2. 알려진 문자열과 비밀 데이터가 함께 압축될 때
   압축률 차이 발생
3. 반복적 추측으로 비밀 데이터 추출

방어:
- TLS 압축 비활성화 (TLS 1.3에서는 제거됨)
- HTTP 응답에 비밀 토큰과 사용자 입력 함께 포함하지 않기
- CSRF 토큰을 요청마다 변경
```

### DROWN (Decrypting RSA with Obsolete and Weakened eNcryption)

**발견**: 2016년 | **영향**: SSLv2 지원 서버

```
공격 원리:
1. SSLv2를 지원하는 서버의 RSA 키 악용
2. TLS 세션 복호화 가능
3. 같은 인증서를 사용하는 모든 서버에 영향

방어:
- SSLv2 완전 비활성화
- 서버 간 인증서 공유 주의
```

### Lucky Thirteen

**발견**: 2013년 | **영향**: TLS CBC 모드

```
공격 원리:
1. MAC-then-Encrypt 구조의 타이밍 차이 악용
2. 패딩 검증 시간 차이로 정보 유출

방어:
- AEAD (Authenticated Encryption with Associated Data) 사용
- AES-GCM, ChaCha20-Poly1305 권장
```

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

---

## 실무 적용

### Spring Boot TLS 설정

```yaml
# application.yml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: tomcat

    # TLS 버전 제한
    enabled-protocols: TLSv1.2,TLSv1.3

    # 암호화 스위트 (TLS 1.2용)
    ciphers: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

```java
// 프로그래매틱 SSL 설정
@Configuration
public class TlsConfig {

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addConnectorCustomizers(connector -> {
            connector.setSecure(true);
            connector.setScheme("https");
            connector.setProperty("sslProtocol", "TLS");

            // TLS 1.3만 허용
            connector.setProperty("sslEnabledProtocols", "TLSv1.3");
        });
        return tomcat;
    }
}
```

### mTLS (Mutual TLS) 구현

```yaml
# application.yml - mTLS 설정
server:
  ssl:
    client-auth: need  # want = 선택적, need = 필수
    trust-store: classpath:truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
```

```java
// 클라이언트 인증서 정보 추출
@RestController
public class ApiController {

    @GetMapping("/api/data")
    public ResponseEntity<String> getData(
            @RequestHeader("X-SSL-Client-Cert") String clientCert,
            HttpServletRequest request) {

        X509Certificate[] certs = (X509Certificate[])
            request.getAttribute("jakarta.servlet.request.X509Certificate");

        if (certs != null && certs.length > 0) {
            String clientDN = certs[0].getSubjectX500Principal().getName();
            // 클라이언트 인증서 기반 인가 처리
            return ResponseEntity.ok("Authenticated as: " + clientDN);
        }

        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }
}
```

### SSL Labs 테스트

```bash
# ssllabs.com/ssltest 또는 CLI 도구 사용
# A+ 등급 획득을 위한 체크리스트:

# 1. TLS 1.2/1.3만 지원
# 2. PFS(Perfect Forward Secrecy) 지원
# 3. HSTS 헤더 설정
# 4. 안전한 암호화 스위트만 사용
# 5. OCSP Stapling 활성화
# 6. CAA DNS 레코드 설정
```

---

## 참고 자료

- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5246 - TLS 1.2](https://datatracker.ietf.org/doc/html/rfc5246)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)
- [OWASP TLS Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html)
