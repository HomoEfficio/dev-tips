# Maven fat-jar

메이븐에서 실행 가능한 jar(fat-jar) 만드는 2가지 방법

## maven-assembly-plugin

pom.xml 의 `build` 부분에 다음과 같이 maven-assembly-plugin 을 추가한다.

```xml
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>  <!-- 이걸 지정해야 사용하는 lib을 포함하는 fat-jar가 만들어진다. -->
                    </descriptorRefs>
                    <archive>
                        <manifestEntries>
                            <Built-By>Homo Efficio</Built-By>
                        </manifestEntries>
                        <manifest>
                            <mainClass>io.homo_efficio.App</mainClass>  <!-- main 클래스 지정 -->
                        </manifest>
                    </archive>
                    <finalName>maven-fat-jar</finalName>  <!-- 최종으로 만들어지는 fat-jar 파일 이름 지정 -->
                    <appendAssemblyId>false</appendAssemblyId>  <!-- false로 하지 않으면 finalName 에서 지정한 이름 뒤에 -jar-with-dependencies 라는 postfix 가 붙은 fat-jar 파일이 생성된다. -->
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id> <!-- this is used for inheritance merges -->
                        <phase>package</phase> <!-- bind to the packaging phase -->
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

`mvn clean package`를 실행하면 다음과 같은 로그가 찍힌다. 최종 파일 이름 관련 경고가 나오지만 일단 현 상태로 실행하는데 문제 없다.

```
[INFO] --- maven-assembly-plugin:3.3.0:single (make-assembly) @ maven-fat-jar-test ---
[INFO] Building jar: /Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar
[WARNING] Configuration option 'appendAssemblyId' is set to false.
Instead of attaching the assembly file: /Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar, it will become the file for main project artifact.
NOTE: If multiple descriptors or descriptor-formats are provided for this project, the value of this file will be non-deterministic!
[WARNING] Replacing pre-existing project main-artifact file: /Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar-test-1.0-SNAPSHOT.jar
with assembly file: /Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar
```

생성되는 파일은 다음과 같다.

```
🍺🦑🍺🍕🍺 ❯ ll target      
total 1560
drwxr-xr-x  2 1003604  629569319    64B Jul 20 14:54 archive-tmp
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:54 classes
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:54 generated-sources
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:54 maven-archiver
-rw-r--r--  1 1003604  629569319   2.8K Jul 20 14:54 maven-fat-jar-test-1.0-SNAPSHOT.jar
-rw-r--r--  1 1003604  629569319   774K Jul 20 14:54 maven-fat-jar.jar
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:54 maven-status

```

실행 결과 정상 동작한다.

```
🍺🦑🍺🍕🍺 ❯ java -jar target/maven-fat-jar.jar 
15:05:21.907 [main] INFO io.homo_efficio.App - maven으로 만든 fat-jar에서 logback + slf4j 정상 동작함
```


## maven-shade-plugin

pom.xml 의 `build` 부분에 다음과 같이 maven-shade-plugin 을 추가한다.

maven-assebmly-plugin 는 build 바로 하위의 finalName 을 무시하고 최종 파일 이름 지정을 별도로 해야되지만, maven-shade-plugin 은 최종 파일 이름을 별도로 지정하지 않고 build 바로 하위의 finalName 에서 정한 최종 파일 이름을 그대로 따른다.

```xml
    <build>
        <finalName>maven-fat-jar</finalName>  <!-- 최종으로 만들어지는 fat-jar 파일 이름 지정 -->
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.4</version>
                <configuration>
                    <transformers>
                        <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                            <mainClass>io.homo_efficio.App</mainClass>  <!-- main 클래스 지정 -->
                        </transformer>
                    </transformers>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

`maven clean package`를 실행하면 대략 다음과 같은 로그가 찍힌다. 경고 로그가 있지만 실행에 문제 없다.

```
[INFO] --- maven-shade-plugin:3.2.4:shade (default) @ maven-fat-jar-test ---
[INFO] Including ch.qos.logback:logback-classic:jar:1.2.3 in the shaded jar.
[INFO] Including ch.qos.logback:logback-core:jar:1.2.3 in the shaded jar.
[INFO] Including org.slf4j:slf4j-api:jar:1.7.25 in the shaded jar.
[WARNING] logback-classic-1.2.3.jar, logback-core-1.2.3.jar, maven-fat-jar.jar, slf4j-api-1.7.25.jar define 1 overlapping resource: 
[WARNING]   - META-INF/MANIFEST.MF
[WARNING] maven-shade-plugin has detected that some class files are
[WARNING] present in two or more JARs. When this happens, only one
[WARNING] single version of the class is copied to the uber jar.
[WARNING] Usually this is not harmful and you can skip these warnings,
[WARNING] otherwise try to manually exclude artifacts based on
[WARNING] mvn dependency:tree -Ddetail=true and the above output.
[WARNING] See http://maven.apache.org/plugins/maven-shade-plugin/
[INFO] Replacing original artifact with shaded artifact.
[INFO] Replacing /Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar with /Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar-test-1.0-SNAPSHOT-shaded.jar
[INFO] Dependency-reduced POM written at: /Users/1003604/gitRepo/study/maven-fat-jar-test/dependency-reduced-pom.xml
```

로그를 보면 의존 관계가 최적화된 dependency-reduced-pom.xml 파일도 생성해주는 것을 알 수 있다.

생성되는 파일은 다음과 같다.

```
🍺🦑🍺🍕🍺 ❯ ll target 
total 1584
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:59 classes
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:59 generated-sources
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:59 maven-archiver
-rw-r--r--  1 1003604  629569319   785K Jul 20 14:59 maven-fat-jar.jar
drwxr-xr-x  3 1003604  629569319    96B Jul 20 14:59 maven-status
-rw-r--r--  1 1003604  629569319   3.1K Jul 20 14:59 original-maven-fat-jar.jar

```

실행 결과 정상 동작한다.

```
🍺🦑🍺🍕🍺 ❯ java -jar target/maven-fat-jar.jar
15:03:56.935 [main] INFO io.homo_efficio.App - maven으로 만든 fat-jar에서 logback + slf4j 정상 동작함
```

전체 코드는 https://github.com/HomoEfficio/maven-fat-jar-test 에 있다.
