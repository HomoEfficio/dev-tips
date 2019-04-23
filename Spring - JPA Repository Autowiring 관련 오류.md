# Spring - Autowiring 관련 오류

아래와 같이 Bean이 없다면서 Autowiring이 안 되는 경우가 있다.

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Field xxxRepository in yyyService required a bean of type '어쩌구.XXXRepository' that could not be found.


Action:

Consider defining a bean of type '어쩌구.XXXRepository' in your configuration.
```

## ComponentScan or scanBasePackages

일반적으로는 `@ComponentScan(basePackages = {})` 또는 `@SpringBootApplication(scanBasePackages = {})`로 해당 Bean이 존재하는 패키지를 지정해주면 된다.

```java
@SpringBootApplication(scanBasePackages = {"어쩌구가속한적당한패키지"})
public class XXXApplication {
```

## JPA Repository인 경우

JPA Repository일 때는 `@EnableJpaRepositories(basePackages = {})`를 확인한다. `XXXRepository` 클래스가 속한 패키지가 지정되어 있지 않은 상태일 것이다. 적절히 패키지를 지정해주면 된다.
