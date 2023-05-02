# NoClassDefFoundError

IDE 에서 실행하면 정상적으로 실행 되는데, `java -jar` 로 실행하면 다음과 같은 오류가 나올 때가 있다.

```
🍺🦑🍺🍕🍺 ❯ java -jar target/maven-fat-jar-test-1.0-SNAPSHOT.jar                                                                        ✹
Exception in thread "main" java.lang.NoClassDefFoundError: org/slf4j/LoggerFactory
        at io.homo_efficio.App.<clinit>(App.java:9)
Caused by: java.lang.ClassNotFoundException: org.slf4j.LoggerFactory
        at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:602)
        at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178)
        at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:522)
        ... 1 more

```

처음에는 Slf4j 문제인가 싶어 이것저것 검색하면서 slf4j-simple 을 maven dependency에 추가했으나 마찬가지.

**원인은 Slf4j 가 아니라 실행 시에 사용한 was.jar 파일이 fat-jar 가 아니기 때문**이었다.

아래와 같은 메이븐 플러그인 설정은 실행 가능한 jar 파일을 만들어주기는 하지만, 의존 관계를 포함한 fat-jar 를 만들어주지는 않는다.

```
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.0.2</version>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Built-By>Homo Efficio</Built-By>
                        </manifestEntries>
                        <manifest>
                            <mainClass>io.homo_efficio.App</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

참고로 위와 같이 플러그인을 통해 main 클래스를 지정해주지 않아도 메이븐은 jar 파일을 만들어주지만, 그렇게 만들어진 jar 파일을 실행하면 다음과 같이 Manifest 속성이 없다고 나온다. 이 때는 위의 maven-jar-plugin 으로 main 클래스 위치를 지정해줘야 한다.

```
🍺🦑🍺🍕🍺 ❯ java -jar target/maven-fat-jar-test-1.0-SNAPSHOT.jar                                                                        ✹
target/maven-fat-jar-test-1.0-SNAPSHOT.jar에 기본 Manifest 속성이 없습니다.

```

정리하면, **'IDE 에서 잘 되는데 jar 파일로는 NoClassDefFoundError 가 발생해요~'라는 상황이라면 fat-jar 를 떠올리자.**

메이븐에서 fat-jar 를 만드는 방법은 https://github.com/HomoEfficio/dev-tips/blob/master/Maven-fat-jar.md 에 있다.

참고로 logback 과 slf4j 를 함께 쓸 때 logback-classic 만 maven dependency 로 명시하면 되고, 내부적으로 slf4j-api 에 대한 dependency 가 포함돼 있으므로 따로 slf4j-api 를 추가할 필요 없다. 게다가 logback-classic 은 slf4j 인터페이스를 직접 구현해서 만든 native 구현체가 포함돼 있으므로 slf4j-simple 도 추가할 필요 없다.



