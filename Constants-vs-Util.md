# Constants vs Util

오늘 일하다 의견을 나누게 된 Constants와 Util 얘기

애플리케이션이 실행되는 서버 IP를 가져와서 로그에 남길 일이 있다.

이유는 모르겠지만 `InetAddress.getLocalHost().getHostAddress()`로 찍으면 127.0.0.1 이 출력돼서 다른 방법을 찾아보니 아래와 같이 `NetworkInterface` 라는 놈을 써서 구할 수 있었다.

로직이 복잡해 보이긴 하지만 어쨌든 서버 IP라는 상수를 구하는 로직이라 아래와 같이 Constants 클래스에 private static 메서드로 넣었는데,

```java
public class Constants {
    // 다른 상수 2개 있음

    public static final String SERVER_IP = getServerIp();

    private static String getServerIp() {
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
            // 없으면 공백 반환 처리
        }
        return "";
    }
}
```

동료가 Constants 에는 메서드가 있으면 Constants 클래스 성격에 안 맞는다며 `getServerIp()`를 외부 Util 클래스로 빼자고 한다.

엉? 아니 Constants 클래스에 메서드가 있으면 안 되는 건가? 다른 메서드도 아니고 상수를 구하는 데 사용되는 private static 메서드인데?

나는 이 메서드는 분명히 SERVER_IP 라는 상수를 구하는 책임을 가지고 있고, 외부에서 호출할 필요가 없으므로 외부로 뺄 게 아니라 private static 으로 Constants 에 두는 게 적절하다고 봤다. 이게 정보와 로직을 함께 모아 넣자는 캡슐화의 원리에 부합하기 때문이다.

물론 상수가 많아지면 상황에 맞게 클래스를 분리해서 비대화를 막아야겠지만, 지금은 비대화와는 거리가 멀어 보이고, 분리한다고 해도 메서드의 존재 여부가 분리의 기준일 리는 없다고 생각한다.

하지만 동료는 Constants 라는 이름의 클래스는 통상적으로 상수만 모아 놓는 목적으로 만드는 거라, 이 클래스에 메서드를 넣으면 그 목적을 잃는다고 한다. 다른 동료도 이 메서드는 Util 클래스로 빼는 게 좋다고 한다.

아 그래? 메서드를 쓰면 안 되는 건가? 하고 다음과 같이 복잡해보이지만 메서드 호출이 아닌 단순 할당식으로 바꿔서 반응을 살펴봤다.

```java
public static final String SERVER_IP = ((Supplier<String>) () -> {
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
         // 없으면 공백 반환 처리
     }
     return "";
 }).get();
```

예상한대로 여전히 복잡한 로직이 있으니 외부 Util 로 빼는 게 좋다고 한다.

그러니까 결국 메서드라는 형식이 문제가 아니라 상수를 구하는 로직이 복잡하다면 외부로 빼야한다는 얘기였던 거다. 요는 **Constants 클래스에는 복잡한 로직을 두지 말고 오직 간단하고 단순한 상수만 넣자는 얘기다.**

정리하면 다음과 같다.

>SERVER_IP라는 똑같은 상수일지라도 그 값을 구하는 데,
>- 로직이 동원된다면 `Constants.SERVER_IP`로 하지 말고, `ServerUtil.SERVER_IP` 로 해야 되고,  
>- 로직이 동원되지 않는다면 `ServerUtil.SERVER_IP`로 하지 말고, `Constants.SERVER_IP`로 해야 된다는 얘기

Util 이나 Constants 나 엎어치나 메치나 좌측 궁뎅이나 우측 방뎅이나 그게 그거 같긴 하지만,  
설계라는 게 원래 책임, 역할, 이름 이런 거 고민하는 거니께, 탭이나 스페이스 보다는 유익한 얘기 아닐까?  
(헉 실수다 감히 신성한 탭/스페이스를 운운하다니..=3=3) 
여러분의 생각은?

참고로 실무적으로는 다른 동료의 의견에 따라 다음과 같이 깔쌈하게 마무리됐다!

```java
public static final String SERVER_IP = System.getProperty("java.rmi.server.hostname", "");
```
