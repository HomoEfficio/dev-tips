# 서버 실제 IP 구하는 방법

Spring Boot 의 Embedded Servlet Container를 사용한다면 `@Value("${server.address}")`로 서버 IP를 쉽게 가져올 수 있다.

그런데 그렇지 않다면 다음과 같이 InetAddress를 사용해서 서버 IP를 가져올 수도 있는데, 이렇게 하면 127.0.0.1 을 반환하기도 한다.

```java
private String getServerIp() {
    try {
        return InetAddress.getLocalHost().getHostAddress();
    } catch (UnknownHostException e) {
        // 예외 나면 그냥 공백 반환
    }
    return "";
}
```

이럴 때는 아래와 같이 NetworkInterface를 사용하면 된다.

```java
private String getServerIp() {
    try {
        Enumeration<NetworkInterface> enis = NetworkInterface.getNetworkInterfaces();
        List<NetworkInterface> nis = Collections.list(enis);
        for (NetworkInterface ni : nis) {
            if (ni.isUp() && !ni.isLoopback() && ni.getHardwareAddress() != null) {
                List<InterfaceAddress> addresses = ni.getInterfaceAddresses();
                for (InterfaceAddress address : addresses) {
                    if (!StringUtils.isEmpty(address.toString())) {
                        return address.getAddress().getHostAddress();
                    }
                }
            }
        }
    } catch (SocketException e) {
        // 예외 나면 그냥 공백 반환
    }
    return "";
}
```

코드에는 간단하게 그냥 첫 번째 나오는 값을 반환하고 있지만, 서버 장비에 NIC 카드가 여러 개일 수 있고, IP도 여러 개일 수 있으므로 적절한 조건으로 필터링해서 원하는 값을 가져와야 한다.

