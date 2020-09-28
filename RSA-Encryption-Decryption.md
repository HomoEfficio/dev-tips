# RSA encryption/decryption

보안을 위해 서버-클라이언트 사이에 username, password 를 비대칭키(ex: RSA)로 암호화/복호화해서 사용하기도 한다. 이렇게 비대칭키를 사용해서 로그인 정보를 암/복호화하면 SSL을 사용하지 않더라도 로그인 정보에 대한 보안성을 SSL과 비슷한 수준으로 유지할 수 있다.

대략 다음과 같은 흐름이다.

![Imgur](https://i.imgur.com/ECNakG7.png)

이 과정을 코드와 함께 알아보자.

# 서버 - 공개키/비밀키 생성 및 공개키 정보(modulus, exponent) 등 반환

```java
// GET 을 쓴 이유는 https://homoefficio.github.io/2019/12/25/GET이냐-POST냐-그것이-문제로다/ 를 참고한다
@GetMapping(value = "/pub-key") 
public ResponseEntity<PubKeyOut> getPublicKey(HttpServletRequest request) {

    try {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(1024);

        // 공개키/비밀키 생성
        KeyPair keyPair = generator.genKeyPair();
        PublicKey publicKey = keyPair.getPublic();
        PrivateKey privateKey = keyPair.getPrivate();

        // 비밀키 찾기 ID 생성 (sessionId를 사용)
        HttpSession session = request.getSession();
        String sessionId = session.getId();
        
        // 비밀키는 인메모리그리드(HazelCast)에 저장
        IMap<Object, Object> privateKeyMap = hazelcastInstance.getMap("PrivateKeys");
        privateKeyMap.put(sessionId, privateKey, 100L, TimeUnit.SECONDS);

        // 공개키 정보 추출
        RSAPublicKeySpec publicSpec = KeyFactory.getInstance("RSA")
            .getKeySpec(publicKey, RSAPublicKeySpec.class);
        String publicModulus = publicSpec.getModulus().toString(16);
        String publicExponent = publicSpec.getPublicExponent().toString(16);

        // 공개키 정보, 비밀키찾기ID 반환
        return ResponseEntity.ok(
            PubKeyOut.builder()
                .modulus(publicModulus)
                .exponent(publicExponent)
                .sessionId(sessionId)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}

@Data
@Builder
class PubKeyOut {
    private final String modulus;
    private final String exponent;
    private final String sessionId;
}
```

# 클라이언트 - 공개키로 로그인 정보 암호화 및 로그인 요청

```javascript
const rsa = new RSAKey();
// 서버로부터 받은 공개키 정보를 이용해서 공개키 설정
rsa.setPublic(publicModulusFromServer, publicExponentFromServer);

// 공개키로 로그인 정보 암호화
const encryptedUsername = this.rsa.encrypt(username);  // 16진수 고정 길이(256) 문자열
const encryptedPassword = this.rsa.encrypt(password);  // 16진수 고정 길이(256) 문자열
const sessionId = sessionIdFromServer;

// 암호화한 정보로 로그인 요청
...
````

# 서버 - 비밀키로 로그인 정보 복호화

```java
public Credential getDecryptedCredential(HttpServletRequest request) {
    // 비밀키 조회
    String plainSessionId = request.getParameter("sessionId");
    PrivateKey privateKey = getPrivateKey(plainSessionId);

    // 암호화 된 로그인 정보 복호화
    String encryptedUsername = request.getParameter("username");
    String encryptedPassword = request.getParameter("password");
    String plainUserName = getDecryptedValue(privateKey, encryptedUsername);
    String plainPassword = getDecryptedValue(privateKey, encryptedPassword);

    return Credential.builder().username(plainUsername).pasword(plainPassword).build();
}

@Data
@Builder
class Credential {
    private final String username;
    private final String password;
}

private PrivateKey getPrivateKey(String sessionId) {
    IMap<String, PrivateKey> privateKeyMap = hazelcastInstance.getMap("PrivateKeys");
    return privateKeyMap.getOrDefault(sessionId, null);
}

private String getDecryptedValue(PrivateKey privateKey, String encryptedValue) {
    try {
        Cipher cipher = Cipher.getInstance("RSA");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return new String(cipher.doFinal(hexToByteArray(encryptedValue)), "utf-8");
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}

// see https://www.tutorialspoint.com/convert-hex-string-to-byte-array-in-java
private byte[] hexToByteArray(String hex) {
    byte[] bytes = new byte[hex.length() / 2];
    for (int i = 0; i < bytes.length; i++) {
        bytes[i] = (byte) Integer.parseInt(hex.substring(2 * i, 2 * i + 2), 16);
    }
    return bytes;
}
```

# 서버에서 공개키 암호화

위 프로세스에서는 공개키 암호화를 클라이언트에서만 수행하지만 테스트 등을 위해 공개키 암호화를 서버에서 수행할 수도 있다. 결국 JavaScript 에서 RSAKey, publicModulus, publicExponent 를 사용해서 암호화하는 로직을 Java 로 구현해야 한다. 따라서 publicModulus, publicExponent 에서 공개키를 추출하고 암호화하는 과정이 필요하다.

주의해야 할 점이 하나 있는데, BigInteger 생성 시 16진수 문자열임을 반드시 명시해줘야 한다.

```java
public String getEncryptedValue(String hexPubModulus, String hexPubExponent, String plainText) {
    // 16 을 표시해주는 게 중요!! 
    // 아래와 같이 하지 않고 hexPubModulus.getBytes() 등을 사용하면 암호과 결과 길이가 256이 아니라 512로 나오며,
    // 길이가 512면 나중에 복호화 할 때 오류 발생
    BigInteger modulus = new BigInteger(hexPubModulus, 16);
    BigInteger exponent = new BigInteger(hexPubExponent, 16);
    
    try {
        PublicKey pubKey = KeyFactory.getInstance("RSA")
                .generatePublic(new RSAPublicKeySpec(modulus, exponent));
        Cipher cipher = Cipher.getInstance("RSA");
        cipher.init(Cipher.ENCRYPT_MODE, pubKey);

        return bytesToHex(cipher.doFinal(plainText.getBytes("UTF-8")));
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}

// see https://stackoverflow.com/a/9855338
private String byteArrayToHex(byte[] bytes) {
    char[] HEX_ARRAY = "0123456789abcdef".toCharArray();
    char[] hexChars = new char[bytes.length * 2];
    for (int j = 0; j < bytes.length; j++) {
        int v = bytes[j] & 0xFF;
        hexChars[j * 2] = HEX_ARRAY[v >>> 4];
        hexChars[j * 2 + 1] = HEX_ARRAY[v & 0x0F];
    }
    return new String(hexChars);
}
```
