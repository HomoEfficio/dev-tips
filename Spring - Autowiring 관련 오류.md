# Spring - Autowiring 관련 오류

잘 되던 애플리케이션인데 갑자기 아래와 같은 에러를 뱉으면서 Bean이 없어서 Autowiring이 안 되는 경우가 있다.

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Field xxxRepository in yyyService required a bean of type '어쩌구.XXXRepository' that could not be found.


Action:

Consider defining a bean of type '어쩌구.XXXRepository' in your configuration.
```

이 때는 고민하지 말고 다음과 같이 Component Scan 위치를 명시해주자.

```java
@SpringBootApplication(scanBasePackages = {"어쩌구가속한적당한패키지"})
public class XXXApplication {
```

저렇게 패키지 위치를 지정해주지 않아도 Autowiring이 잘 되는 클래스도 있어서 당황스럽지만, 이런 거에 시간 빼앗기지 말고 그냥 명시해주자.
