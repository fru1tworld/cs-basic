# 해싱과 암호화

## 목차
1. [개요](#개요)
2. [해시 함수](#해시-함수)
3. [대칭 암호화 vs 비대칭 암호화](#대칭-암호화-vs-비대칭-암호화)
4. [솔팅과 페퍼링](#솔팅과-페퍼링)
5. [Key Derivation Function](#key-derivation-function)
6. [실무 구현](#실무-구현)
---

## 개요

### 해싱 vs 암호화

| 특성 | 해싱 (Hashing) | 암호화 (Encryption) |
|------|---------------|-------------------|
| 방향성 | 단방향 (복호화 불가) | 양방향 (복호화 가능) |
| 키 | 필요 없음 | 키 필요 |
| 결과 길이 | 고정 길이 | 입력에 따라 다름 |
| 용도 | 무결성 검증, 비밀번호 저장 | 데이터 기밀성 보호 |
| 예시 | SHA-256, bcrypt, Argon2 | AES, RSA, ChaCha20 |

```
해싱:
"password123" → SHA-256 → "ef92b778baf..." (고정 64자)
"ef92b778baf..." → ??? → 복원 불가능

암호화:
"password123" → AES(key) → "x7K9mP2n..." (암호문)
"x7K9mP2n..." → AES(key) → "password123" (복원 가능)
```

---

## 해시 함수

### 해시 함수의 특성

```
1. 결정성 (Deterministic)
   - 같은 입력 → 항상 같은 출력

2. 빠른 계산 (일반 해시) / 느린 계산 (비밀번호 해시)
   - 일반: SHA-256은 빠른 계산이 필요
   - 비밀번호: bcrypt, Argon2는 의도적으로 느림

3. 충돌 저항성 (Collision Resistance)
   - 다른 입력이 같은 출력을 생성하기 어려움

4. 역상 저항성 (Preimage Resistance)
   - 출력에서 입력을 추측하기 어려움

5. 눈사태 효과 (Avalanche Effect)
   - 입력의 작은 변화 → 출력의 큰 변화
```

### 일반 해시 함수 (비밀번호에 부적합)

```java
// MD5 - 절대 사용하지 말 것!
// 충돌 발견됨, 레인보우 테이블 공격에 취약
MessageDigest md5 = MessageDigest.getInstance("MD5");
byte[] hash = md5.digest("password".getBytes());
// 128 bits (32자 hex)

// SHA-1 - 보안 용도로 사용하지 말 것!
// 충돌 발견됨 (SHAttered 공격, 2017)
MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
// 160 bits (40자 hex)

// SHA-256 - 일반 용도로는 안전
// 파일 체크섬, 데이터 무결성 검증에 적합
// 하지만 비밀번호 해싱에는 부적합 (너무 빠름)
MessageDigest sha256 = MessageDigest.getInstance("SHA-256");
// 256 bits (64자 hex)
```

```java
// SHA-256이 비밀번호에 부적합한 이유
// GPU로 초당 수십억 개의 해시 계산 가능

public class Sha256BruteForce {
    // 6자리 숫자 비밀번호: 1,000,000 조합
    // SHA-256으로 해싱된 경우: 수 초 내에 크랙 가능

    public static void main(String[] args) throws Exception {
        String targetHash = "..."; // 탈취된 해시
        MessageDigest sha256 = MessageDigest.getInstance("SHA-256");

        for (int i = 0; i < 1000000; i++) {
            String candidate = String.format("%06d", i);
            String hash = bytesToHex(sha256.digest(candidate.getBytes()));
            if (hash.equals(targetHash)) {
                System.out.println("Found: " + candidate);
                break;
            }
        }
    }
}
```

### 비밀번호 해시 함수

#### bcrypt

```java
// bcrypt 특징:
// - 의도적으로 느린 해시 함수
// - 솔트 자동 포함
// - Work Factor로 연산 비용 조절
// - 최대 72 바이트 입력 제한

import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

public class BcryptExample {
    // Work Factor (Cost): 기본값 10
    // Cost 10: ~100ms, Cost 12: ~400ms, Cost 14: ~1600ms
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);

    public String hashPassword(String password) {
        // 결과 형식: $2a$12$salt(22자)hash(31자)
        // 예: $2a$12$N9qo8uLOickgx2ZMRZoMy.MSwEqGq...
        return encoder.encode(password);
    }

    public boolean verifyPassword(String rawPassword, String encodedPassword) {
        return encoder.matches(rawPassword, encodedPassword);
    }
}
```

```java
// bcrypt 72 바이트 제한 처리
public class ExtendedBcrypt {
    private final BCryptPasswordEncoder bcrypt = new BCryptPasswordEncoder(12);

    public String hashPassword(String password) throws Exception {
        // 긴 비밀번호는 SHA-256으로 먼저 해싱
        if (password.length() > 72) {
            MessageDigest sha256 = MessageDigest.getInstance("SHA-256");
            byte[] hash = sha256.digest(password.getBytes(StandardCharsets.UTF_8));
            // Base64로 인코딩 (44자)
            password = Base64.getEncoder().encodeToString(hash);
        }
        return bcrypt.encode(password);
    }
}
```

#### scrypt

```java
// scrypt 특징:
// - 메모리 집약적 (GPU 공격 저항)
// - CPU + 메모리 비용 조절 가능
// - bcrypt보다 메모리 사용량 많음

import org.springframework.security.crypto.scrypt.SCryptPasswordEncoder;

public class ScryptExample {
    // 파라미터: cpuCost, memoryCost, parallelization, keyLength, saltLength
    // cpuCost: 2^N (N은 14~20 권장)
    // memoryCost: 8 권장
    // parallelization: 1 권장
    private final SCryptPasswordEncoder encoder = new SCryptPasswordEncoder(
        16384,  // cpuCost (2^14)
        8,      // memoryCost
        1,      // parallelization
        32,     // keyLength
        16      // saltLength
    );

    public String hashPassword(String password) {
        return encoder.encode(password);
    }

    public boolean verifyPassword(String rawPassword, String encodedPassword) {
        return encoder.matches(rawPassword, encodedPassword);
    }
}
```

#### Argon2 (권장)

```java
// Argon2 특징:
// - Password Hashing Competition 2015 우승
// - 세 가지 변형: Argon2d, Argon2i, Argon2id
// - Argon2id 권장 (GPU + Side-channel 공격 저항)
// - 가장 현대적이고 안전한 선택

import org.springframework.security.crypto.argon2.Argon2PasswordEncoder;

public class Argon2Example {
    // OWASP 권장 설정:
    // - m=47104 (46 MiB), t=1, p=1
    // - 또는 m=19456 (19 MiB), t=2, p=1

    // Spring Security 6+ 기본값
    private final Argon2PasswordEncoder encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();

    // 커스텀 설정
    private final Argon2PasswordEncoder customEncoder = new Argon2PasswordEncoder(
        16,     // saltLength (bytes)
        32,     // hashLength (bytes)
        1,      // parallelism
        47104,  // memory (KiB) = 46 MiB
        1       // iterations
    );

    public String hashPassword(String password) {
        return encoder.encode(password);
    }

    public boolean verifyPassword(String rawPassword, String encodedPassword) {
        return encoder.matches(rawPassword, encodedPassword);
    }
}
```

### 해시 함수 비교

| 알고리즘 | 출력 크기 | 메모리 사용 | GPU 저항성 | 권장 |
|---------|---------|-----------|-----------|------|
| MD5 | 128 bits | 없음 | 없음 | 사용 금지 |
| SHA-256 | 256 bits | 없음 | 없음 | 비밀번호 X |
| bcrypt | 184 bits | 4 KB | 중간 | 레거시 |
| scrypt | 가변 | 높음 | 높음 | 가능 |
| **Argon2id** | 가변 | 가변 | 매우 높음 | **권장** |

---

## 대칭 암호화 vs 비대칭 암호화

### 대칭 암호화 (Symmetric)

```
같은 키로 암호화/복호화

        Key
         │
         ▼
┌─────┐     ┌──────────┐     ┌─────┐
│Plain├────►│ Encrypt  ├────►│Cipher│
│text │     └──────────┘     │text │
└─────┘                      └──┬──┘
                                │
                                │ Key
                                ▼
                          ┌──────────┐     ┌─────┐
                          │ Decrypt  ├────►│Plain│
                          └──────────┘     │text │
                                          └─────┘

알고리즘: AES, ChaCha20, 3DES(레거시)
용도: 대량 데이터 암호화, 데이터베이스 암호화
장점: 빠름
단점: 키 공유 문제
```

#### AES (Advanced Encryption Standard)

```java
import javax.crypto.*;
import javax.crypto.spec.*;
import java.security.*;

public class AesEncryption {

    // AES-GCM (권장) - 인증된 암호화
    // GCM은 무결성 검증 기능 포함

    public EncryptedData encryptAesGcm(String plainText, SecretKey key) throws Exception {
        // 12 바이트 IV (GCM 권장)
        byte[] iv = new byte[12];
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(128, iv);  // 128-bit 태그
        cipher.init(Cipher.ENCRYPT_MODE, key, spec);

        byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

        return new EncryptedData(iv, cipherText);
    }

    public String decryptAesGcm(EncryptedData data, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(128, data.iv());
        cipher.init(Cipher.DECRYPT_MODE, key, spec);

        byte[] plainText = cipher.doFinal(data.cipherText());
        return new String(plainText, StandardCharsets.UTF_8);
    }

    public SecretKey generateKey() throws Exception {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256, SecureRandom.getInstanceStrong());
        return keyGen.generateKey();
    }
}

public record EncryptedData(byte[] iv, byte[] cipherText) {}
```

```java
// AES-CBC는 레거시 - Padding Oracle 공격에 취약할 수 있음
// 사용해야 한다면 HMAC으로 무결성 검증 추가 필요

public class AesCbcWithHmac {

    public byte[] encrypt(byte[] plainText, SecretKey aesKey, SecretKey hmacKey)
            throws Exception {
        // CBC 암호화
        byte[] iv = new byte[16];
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        cipher.init(Cipher.ENCRYPT_MODE, aesKey, new IvParameterSpec(iv));
        byte[] cipherText = cipher.doFinal(plainText);

        // HMAC 계산 (Encrypt-then-MAC)
        Mac hmac = Mac.getInstance("HmacSHA256");
        hmac.init(hmacKey);
        hmac.update(iv);
        byte[] mac = hmac.doFinal(cipherText);

        // IV + CipherText + MAC
        ByteBuffer buffer = ByteBuffer.allocate(16 + cipherText.length + 32);
        buffer.put(iv);
        buffer.put(cipherText);
        buffer.put(mac);
        return buffer.array();
    }
}
```

### 비대칭 암호화 (Asymmetric)

```
다른 키로 암호화/복호화 (공개키/개인키)

  Public Key                    Private Key
      │                              │
      ▼                              ▼
┌─────┐    ┌─────────┐    ┌─────┐    ┌─────────┐    ┌─────┐
│Plain├───►│Encrypt  ├───►│Cipher├───►│Decrypt  ├───►│Plain│
│text │    │(PubKey) │    │text │    │(PrivKey)│    │text │
└─────┘    └─────────┘    └─────┘    └─────────┘    └─────┘

알고리즘: RSA, ECC, Ed25519
용도: 키 교환, 디지털 서명, 인증서
장점: 키 공유 불필요
단점: 느림, 큰 데이터에 부적합
```

#### RSA 암호화

```java
public class RsaEncryption {

    private final KeyPair keyPair;

    public RsaEncryption() throws Exception {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048, SecureRandom.getInstanceStrong());
        this.keyPair = keyGen.generateKeyPair();
    }

    // 공개키로 암호화 (누구나 가능)
    public byte[] encrypt(String plainText) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, keyPair.getPublic());
        return cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));
    }

    // 개인키로 복호화 (소유자만 가능)
    public String decrypt(byte[] cipherText) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.DECRYPT_MODE, keyPair.getPrivate());
        byte[] plainText = cipher.doFinal(cipherText);
        return new String(plainText, StandardCharsets.UTF_8);
    }

    // 개인키로 서명 (소유자만 가능)
    public byte[] sign(byte[] data) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initSign(keyPair.getPrivate());
        signature.update(data);
        return signature.sign();
    }

    // 공개키로 검증 (누구나 가능)
    public boolean verify(byte[] data, byte[] signatureBytes) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initVerify(keyPair.getPublic());
        signature.update(data);
        return signature.verify(signatureBytes);
    }
}
```

### 하이브리드 암호화

```java
// 실제로는 대칭 + 비대칭을 조합하여 사용
// RSA로 AES 키를 암호화하고, AES로 데이터를 암호화

public class HybridEncryption {

    public HybridEncryptedData encrypt(String plainText, PublicKey recipientPublicKey)
            throws Exception {
        // 1. 랜덤 AES 키 생성
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256, SecureRandom.getInstanceStrong());
        SecretKey aesKey = keyGen.generateKey();

        // 2. RSA로 AES 키 암호화
        Cipher rsaCipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        rsaCipher.init(Cipher.ENCRYPT_MODE, recipientPublicKey);
        byte[] encryptedKey = rsaCipher.doFinal(aesKey.getEncoded());

        // 3. AES-GCM으로 데이터 암호화
        byte[] iv = new byte[12];
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher aesCipher = Cipher.getInstance("AES/GCM/NoPadding");
        aesCipher.init(Cipher.ENCRYPT_MODE, aesKey, new GCMParameterSpec(128, iv));
        byte[] encryptedData = aesCipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

        return new HybridEncryptedData(encryptedKey, iv, encryptedData);
    }

    public String decrypt(HybridEncryptedData data, PrivateKey recipientPrivateKey)
            throws Exception {
        // 1. RSA로 AES 키 복호화
        Cipher rsaCipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        rsaCipher.init(Cipher.DECRYPT_MODE, recipientPrivateKey);
        byte[] aesKeyBytes = rsaCipher.doFinal(data.encryptedKey());
        SecretKey aesKey = new SecretKeySpec(aesKeyBytes, "AES");

        // 2. AES-GCM으로 데이터 복호화
        Cipher aesCipher = Cipher.getInstance("AES/GCM/NoPadding");
        aesCipher.init(Cipher.DECRYPT_MODE, aesKey, new GCMParameterSpec(128, data.iv()));
        byte[] plainText = aesCipher.doFinal(data.encryptedData());

        return new String(plainText, StandardCharsets.UTF_8);
    }
}

public record HybridEncryptedData(byte[] encryptedKey, byte[] iv, byte[] encryptedData) {}
```

---

## 솔팅과 페퍼링

### Salt (솔트)

```
문제: 레인보우 테이블 공격
- 사전 계산된 해시 테이블로 비밀번호 역추적
- 같은 비밀번호 → 같은 해시

해결: 솔트 추가
- 각 비밀번호에 고유한 랜덤 값 추가
- 같은 비밀번호 + 다른 솔트 → 다른 해시

솔트 없음:
password123 → 해시A
password123 → 해시A (모든 사용자 동일)

솔트 있음:
password123 + salt1 → 해시X
password123 + salt2 → 해시Y (같은 비밀번호라도 다름)
```

```java
// 솔트의 특성:
// 1. 사용자마다 고유 (unique per user)
// 2. 충분히 길어야 함 (16+ bytes 권장)
// 3. 암호학적으로 안전한 난수 생성
// 4. 해시와 함께 저장 (비밀이 아님)

public class SaltedHash {

    public SaltedHashResult hash(String password) throws Exception {
        // 16 바이트 솔트 생성
        byte[] salt = new byte[16];
        SecureRandom.getInstanceStrong().nextBytes(salt);

        // PBKDF2로 해싱
        int iterations = 310000;  // OWASP 권장
        int keyLength = 256;

        PBEKeySpec spec = new PBEKeySpec(
            password.toCharArray(),
            salt,
            iterations,
            keyLength
        );

        SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
        byte[] hash = factory.generateSecret(spec).getEncoded();

        return new SaltedHashResult(salt, hash, iterations);
    }

    public boolean verify(String password, SaltedHashResult storedResult) throws Exception {
        PBEKeySpec spec = new PBEKeySpec(
            password.toCharArray(),
            storedResult.salt(),
            storedResult.iterations(),
            256
        );

        SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
        byte[] hash = factory.generateSecret(spec).getEncoded();

        // 시간 일정 비교 (Timing Attack 방지)
        return MessageDigest.isEqual(hash, storedResult.hash());
    }
}

public record SaltedHashResult(byte[] salt, byte[] hash, int iterations) {}
```

### Pepper (페퍼)

```
솔트와의 차이:
- Salt: 사용자마다 다름, 해시와 함께 저장
- Pepper: 모든 사용자 공통, 별도 비밀 저장소에 보관

목적:
- 데이터베이스 탈취 시에도 해시 크래킹 어렵게 함
- 애플리케이션 레벨의 추가 보안 계층

저장 위치:
- 환경 변수
- 비밀 관리 시스템 (AWS Secrets Manager, HashiCorp Vault)
- HSM (Hardware Security Module)
```

```java
@Service
public class PepperedPasswordService {

    @Value("${security.password.pepper}")
    private String pepper;

    private final Argon2PasswordEncoder encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();

    public String hashPassword(String password) {
        // 페퍼를 비밀번호 앞에 추가
        String pepperedPassword = pepper + password;
        return encoder.encode(pepperedPassword);
    }

    public boolean verifyPassword(String rawPassword, String encodedPassword) {
        String pepperedPassword = pepper + rawPassword;
        return encoder.matches(pepperedPassword, encodedPassword);
    }
}

// application.yml (또는 환경 변수)
security:
  password:
    pepper: ${PASSWORD_PEPPER}  # 최소 32자 랜덤 문자열
```

```java
// HMAC 기반 페퍼링 (더 안전)
@Service
public class HmacPepperedPasswordService {

    private final SecretKey pepperKey;
    private final Argon2PasswordEncoder encoder;

    public HmacPepperedPasswordService(@Value("${security.password.pepper}") String pepper)
            throws Exception {
        byte[] keyBytes = Base64.getDecoder().decode(pepper);
        this.pepperKey = new SecretKeySpec(keyBytes, "HmacSHA256");
        this.encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
    }

    public String hashPassword(String password) throws Exception {
        // HMAC으로 페퍼 적용
        Mac hmac = Mac.getInstance("HmacSHA256");
        hmac.init(pepperKey);
        byte[] pepperedBytes = hmac.doFinal(password.getBytes(StandardCharsets.UTF_8));
        String pepperedPassword = Base64.getEncoder().encodeToString(pepperedBytes);

        return encoder.encode(pepperedPassword);
    }

    public boolean verifyPassword(String rawPassword, String encodedPassword) throws Exception {
        Mac hmac = Mac.getInstance("HmacSHA256");
        hmac.init(pepperKey);
        byte[] pepperedBytes = hmac.doFinal(rawPassword.getBytes(StandardCharsets.UTF_8));
        String pepperedPassword = Base64.getEncoder().encodeToString(pepperedBytes);

        return encoder.matches(pepperedPassword, encodedPassword);
    }
}
```

---

## Key Derivation Function

### KDF란?

Key Derivation Function은 비밀번호나 마스터 키에서 암호화 키를 안전하게 유도하는 함수입니다.

```
용도:
1. 비밀번호로부터 암호화 키 생성
2. 마스터 키에서 여러 서브 키 유도
3. 비밀번호 해싱 (bcrypt, Argon2도 KDF의 일종)

특성:
- 느린 연산 (brute-force 저항)
- 출력 길이 조절 가능
- 솔트 지원
```

### PBKDF2 (Password-Based Key Derivation Function 2)

```java
// PBKDF2 - RFC 8018
// 장점: 표준화됨, 널리 지원됨, FIPS 인증
// 단점: 메모리 하드하지 않음 (GPU 공격에 취약)

public class Pbkdf2KeyDerivation {

    public SecretKey deriveKey(String password, byte[] salt, int iterations)
            throws Exception {
        // OWASP 권장: PBKDF2-HMAC-SHA256, 310,000+ iterations
        PBEKeySpec spec = new PBEKeySpec(
            password.toCharArray(),
            salt,
            iterations,  // 310000 권장
            256          // 키 길이 (bits)
        );

        SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
        byte[] keyBytes = factory.generateSecret(spec).getEncoded();

        return new SecretKeySpec(keyBytes, "AES");
    }

    // 비밀번호로 파일 암호화 예제
    public EncryptedFile encryptWithPassword(byte[] data, String password) throws Exception {
        // 솔트 생성
        byte[] salt = new byte[16];
        SecureRandom.getInstanceStrong().nextBytes(salt);

        // 비밀번호에서 키 유도
        SecretKey key = deriveKey(password, salt, 310000);

        // AES-GCM 암호화
        byte[] iv = new byte[12];
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
        byte[] cipherText = cipher.doFinal(data);

        return new EncryptedFile(salt, iv, cipherText, 310000);
    }
}

public record EncryptedFile(byte[] salt, byte[] iv, byte[] cipherText, int iterations) {}
```

### HKDF (HMAC-based Key Derivation Function)

```java
// HKDF - RFC 5869
// 마스터 키에서 여러 서브 키를 유도할 때 사용
// 비밀번호에는 부적합 (빠른 연산)

public class HkdfKeyDerivation {

    // Extract: 입력 키 자료에서 고정 길이 PRK(Pseudo-Random Key) 추출
    public byte[] extract(byte[] salt, byte[] inputKeyMaterial) throws Exception {
        Mac hmac = Mac.getInstance("HmacSHA256");

        if (salt == null || salt.length == 0) {
            salt = new byte[32];  // zero-filled
        }

        hmac.init(new SecretKeySpec(salt, "HmacSHA256"));
        return hmac.doFinal(inputKeyMaterial);
    }

    // Expand: PRK에서 원하는 길이의 키 유도
    public byte[] expand(byte[] prk, byte[] info, int outputLength) throws Exception {
        Mac hmac = Mac.getInstance("HmacSHA256");
        hmac.init(new SecretKeySpec(prk, "HmacSHA256"));

        int hashLen = 32;  // SHA-256
        int n = (int) Math.ceil((double) outputLength / hashLen);

        byte[] result = new byte[outputLength];
        byte[] t = new byte[0];

        for (int i = 1; i <= n; i++) {
            hmac.reset();
            hmac.update(t);
            if (info != null) {
                hmac.update(info);
            }
            hmac.update((byte) i);
            t = hmac.doFinal();

            int copyLen = Math.min(hashLen, outputLength - (i - 1) * hashLen);
            System.arraycopy(t, 0, result, (i - 1) * hashLen, copyLen);
        }

        return result;
    }

    // 마스터 키에서 서브 키들 유도
    public DerivedKeys deriveKeys(byte[] masterKey) throws Exception {
        byte[] prk = extract(null, masterKey);

        // 각 용도별로 다른 키 유도
        byte[] encryptionKey = expand(prk, "encryption".getBytes(), 32);
        byte[] macKey = expand(prk, "mac".getBytes(), 32);
        byte[] ivSeed = expand(prk, "iv".getBytes(), 16);

        return new DerivedKeys(encryptionKey, macKey, ivSeed);
    }
}

public record DerivedKeys(byte[] encryptionKey, byte[] macKey, byte[] ivSeed) {}
```

---

## 실무 구현

### Spring Security 비밀번호 인코더 설정

```java
@Configuration
public class PasswordEncoderConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        // Argon2 권장
        return Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();
    }

    // 또는 DelegatingPasswordEncoder로 여러 알고리즘 지원
    @Bean
    public PasswordEncoder delegatingPasswordEncoder() {
        String defaultEncoder = "argon2";
        Map<String, PasswordEncoder> encoders = Map.of(
            "argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8(),
            "bcrypt", new BCryptPasswordEncoder(12),
            "scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8()
        );

        DelegatingPasswordEncoder encoder = new DelegatingPasswordEncoder(defaultEncoder, encoders);
        encoder.setDefaultPasswordEncoderForMatches(new BCryptPasswordEncoder(12));
        return encoder;
    }
}
```

### 데이터 암호화 서비스

```java
@Service
@Slf4j
public class EncryptionService {

    private final SecretKey encryptionKey;

    public EncryptionService(@Value("${encryption.key}") String base64Key) {
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        this.encryptionKey = new SecretKeySpec(keyBytes, "AES");
    }

    public String encrypt(String plainText) {
        try {
            byte[] iv = new byte[12];
            SecureRandom.getInstanceStrong().nextBytes(iv);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE, encryptionKey, new GCMParameterSpec(128, iv));
            byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

            // IV + CipherText를 Base64로 인코딩
            ByteBuffer buffer = ByteBuffer.allocate(12 + cipherText.length);
            buffer.put(iv);
            buffer.put(cipherText);

            return Base64.getEncoder().encodeToString(buffer.array());
        } catch (Exception e) {
            log.error("Encryption failed", e);
            throw new EncryptionException("암호화 실패", e);
        }
    }

    public String decrypt(String encryptedText) {
        try {
            byte[] combined = Base64.getDecoder().decode(encryptedText);

            ByteBuffer buffer = ByteBuffer.wrap(combined);
            byte[] iv = new byte[12];
            buffer.get(iv);
            byte[] cipherText = new byte[buffer.remaining()];
            buffer.get(cipherText);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE, encryptionKey, new GCMParameterSpec(128, iv));
            byte[] plainText = cipher.doFinal(cipherText);

            return new String(plainText, StandardCharsets.UTF_8);
        } catch (Exception e) {
            log.error("Decryption failed", e);
            throw new EncryptionException("복호화 실패", e);
        }
    }
}
```

### JPA AttributeConverter를 사용한 필드 암호화

```java
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    @Autowired
    private EncryptionService encryptionService;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        if (attribute == null) {
            return null;
        }
        return encryptionService.encrypt(attribute);
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }
        return encryptionService.decrypt(dbData);
    }
}

@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    @Convert(converter = EncryptedStringConverter.class)
    private String email;  // 암호화되어 저장됨

    @Convert(converter = EncryptedStringConverter.class)
    private String phoneNumber;  // 암호화되어 저장됨

    // ...
}
```

---

## 참고 자료

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [NIST SP 800-63B - Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 8018 - PKCS #5: Password-Based Cryptography Specification](https://datatracker.ietf.org/doc/html/rfc8018)
- [RFC 5869 - HKDF (HMAC-based Key Derivation Function)](https://datatracker.ietf.org/doc/html/rfc5869)
- [Argon2 - PHC Winner](https://www.password-hashing.net/)
