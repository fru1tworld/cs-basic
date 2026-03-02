# 보안 (Security) 면접 질문

## 인증/인가 (Authentication/Authorization)

### SEC-001
OAuth 2.0의 동작 원리와 주요 Grant Type(Authorization Code, Client Credentials 등)에 대해 설명해주세요.

### SEC-002
OAuth 2.0과 OpenID Connect(OIDC)의 차이점은 무엇인가요?

### SEC-003
JWT(JSON Web Token)의 구조(Header, Payload, Signature)와 보안 고려사항에 대해 설명해주세요.

### SEC-004
Session 기반 인증과 Token 기반 인증의 차이점과 각각의 장단점은 무엇인가요?

### SEC-005
MFA(Multi-Factor Authentication)의 종류와 구현 방법에 대해 설명해주세요.

### SEC-006
Access Token과 Refresh Token의 역할과 보안적 고려사항은 무엇인가요?

## 암호화 (Cryptography)

### SEC-007
대칭키 암호화와 비대칭키 암호화의 차이점과 사용 사례를 설명해주세요.

### SEC-008
해시 함수의 특성과 솔팅(Salting)이 필요한 이유를 설명해주세요.

### SEC-009
bcrypt, scrypt, Argon2 등 패스워드 해싱 알고리즘의 차이점은 무엇인가요?

### SEC-010
TLS/SSL 핸드셰이크 과정을 설명해주세요.

### SEC-011
인증서 체인(Certificate Chain)과 CA(Certificate Authority)의 역할을 설명해주세요.

### SEC-012
HTTPS에서 사용되는 암호화 방식(대칭키 + 비대칭키 조합)을 설명해주세요.

## 웹 보안 (Web Security)

### SEC-013
OWASP Top 10에 대해 설명해주세요.

### SEC-014
XSS(Cross-Site Scripting) 공격의 종류(Stored, Reflected, DOM-based)와 방어 방법을 설명해주세요.

### SEC-015
CSRF(Cross-Site Request Forgery) 공격의 원리와 방어 방법을 설명해주세요.

### SEC-016
SQL Injection 공격의 원리와 방어 방법(Prepared Statement, ORM 등)을 설명해주세요.

### SEC-017
CORS(Cross-Origin Resource Sharing)와 SOP(Same-Origin Policy)에 대해 설명해주세요.

### SEC-018
Content Security Policy(CSP)란 무엇이며 어떻게 설정하나요?

### SEC-019
Clickjacking 공격과 X-Frame-Options 헤더에 대해 설명해주세요.

### SEC-020
HTTP Security Headers(HSTS, X-Content-Type-Options 등)에 대해 설명해주세요.

## 인프라 보안

### SEC-021
방화벽(Firewall)의 종류(Network, Application, WAF)와 역할을 설명해주세요.

### SEC-022
VPN의 동작 원리와 종류(Site-to-Site, Client VPN)에 대해 설명해주세요.

### SEC-023
컨테이너 보안의 주요 고려사항(이미지 스캐닝, 권한 제한 등)에 대해 설명해주세요.

### SEC-024
Kubernetes 보안(RBAC, Network Policy, Pod Security)에 대해 설명해주세요.

### SEC-025
시크릿 관리 도구(HashiCorp Vault, AWS KMS 등)의 역할과 사용 방법을 설명해주세요.

### SEC-026
Zero Trust Architecture란 무엇이며, 기존 보안 모델과 어떻게 다른가요?

## 보안 설계 원칙

### SEC-027
Defense in Depth(심층 방어) 전략에 대해 설명해주세요.

### SEC-028
Principle of Least Privilege(최소 권한 원칙)란 무엇이며 어떻게 적용하나요?

### SEC-029
Secure by Design 원칙에 대해 설명해주세요.

### SEC-030
보안 감사(Security Audit)와 로깅의 중요성 및 구현 방법을 설명해주세요.

### SEC-031
Threat Modeling이란 무엇이며 어떻게 수행하나요?

### SEC-032
보안 취약점 스캐닝(SAST, DAST, SCA)의 차이점과 CI/CD 파이프라인 통합 방법을 설명해주세요.
