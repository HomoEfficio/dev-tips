# IDE 에서는 되는데 jar 에서는 안 돼요 - Java Resource

>한 줄 요약: 웬만하면 `getResource()` 쓰지 말고 `getResourceAsStream()` 씁시다.

## 기본 폴더 구조

자바에서는 메이븐이 널리 사용되면서 아래와 같은 폴더 구조가 표준처럼 사용되고 있다.

![Imgur](https://i.imgur.com/zpDIgeL.png)

src/main/java 폴더 하위에 있는 java 파일은 빌드 후 target/classes 하위에 위치하게 되고,  
src/main/resources/static 폴더는 빌드 후 target/static 폴더 바로 아래에 위치하게 된다.

자바 파일이든 그 외 파일이든 결국 빌드 후에는 target 디렉터리가 루트 디렉터리가 된다.


## main 클래스

```java
@Slf4j
public class App {

    public static void main(String[] args) throws IOException {
        ResourceLoader resourceLoader =
                new ResourceLoader("/static", Path.of("/static"));
        resourceLoader.loadResourceAsFile("/folder1/sample1");
    }
}
```


## getResource()

`getResource()`를 사용해서 파일로 읽어들인 후 출력하는 프로그램은 다음과 같다.

```java
@Slf4j
@RequiredArgsConstructor
public class ResourceLoader {

    private final String root;
    private final Path rootPath;


    public void loadResourceAsFile(String resourceLocation) throws IOException {
        log.info("*** getResource() + File 방식");
        log.info("content root: {}", rootPath);
        log.info("resourceLocation: {}", resourceLocation);

        URL resourceURL = this.getClass().getResource(root + resourceLocation);
        log.info("resourceURL: {}", resourceURL);

        String fileLocation = resourceURL.getFile();
        log.info("fileLocation from URL: {}", fileLocation);

        File file = new File(fileLocation);
        FileReader fileReader = new FileReader(file);
        char[] chars = new char[(int) file.length()];
        fileReader.read(chars);

        log.info("resource contents: {}", new String(chars));
    }
}
```

### IDE 에서 동작 확인

IDE 에서 실행하면 다음과 같이 정상적으로 출력된다.

```
00:56:15.152 [main] INFO io.homo_efficio.ResourceLoader - *** getResource() + File 방식
00:56:15.155 [main] INFO io.homo_efficio.ResourceLoader - content root: /static
00:56:15.156 [main] INFO io.homo_efficio.ResourceLoader - resourceLocation: /folder1/sample1
00:56:15.158 [main] INFO io.homo_efficio.ResourceLoader - resourceURL: file:/Users/1003604/gitRepo/study/maven-fat-jar-test/target/classes/static/folder1/sample1
00:56:15.158 [main] INFO io.homo_efficio.ResourceLoader - fileLocation from URL: /Users/1003604/gitRepo/study/maven-fat-jar-test/target/classes/static/folder1/sample1
00:56:15.159 [main] INFO io.homo_efficio.ResourceLoader - resource contents: Sample File 1
```

resourceURL 값이 `file:` 로 시작한다는 것을 기억해두자.

읽을 파일 경로는 `/Users/1003604/gitRepo/study/maven-fat-jar-test/target/classes/static/folder1/sample1`로 표시되는데 이는 실제 존재하는 경로와 일치한다.


### fat-jar 에서 동작 확인

하지만 다음과 같이 `java -jar` 명령으로 fat-jar 파일을 실행하면 다음과 같이 오류가 발생한다. maven에서 fat-jar 만드는 방법은 https://github.com/HomoEfficio/dev-tips/blob/master/Maven-fat-jar.md 를 참고한다.

```
maven-fat-jar-test git:master 🍺🦑🍺🍕🍺 ❯ java -jar target/maven-fat-jar.jar
00:58:11.479 [main] INFO io.homo_efficio.ResourceLoader - *** getResource() + File 방식
00:58:11.481 [main] INFO io.homo_efficio.ResourceLoader - content root: /static
00:58:11.482 [main] INFO io.homo_efficio.ResourceLoader - resourceLocation: /folder1/sample1
00:58:11.484 [main] INFO io.homo_efficio.ResourceLoader - resourceURL: jar:file:/Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar!/static/folder1/sample1
00:58:11.484 [main] INFO io.homo_efficio.ResourceLoader - fileLocation from URL: file:/Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar!/static/folder1/sample1
Exception in thread "main" java.io.FileNotFoundException: file:/Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar!/static/folder1/sample1 (No such file or directory)
        at java.base/java.io.FileInputStream.open0(Native Method)
        at java.base/java.io.FileInputStream.open(FileInputStream.java:212)
        at java.base/java.io.FileInputStream.<init>(FileInputStream.java:154)
        at java.base/java.io.FileReader.<init>(FileReader.java:75)
        at io.homo_efficio.ResourceLoader.loadResourceAsFile(ResourceLoader.java:38)
        at io.homo_efficio.App.main(App.java:19)
```

에러 메시지를 보면 파일 경로(정확히는 URL)가 `file:/Users/1003604/gitRepo/study/maven-fat-jar-test/target/maven-fat-jar.jar!/static/folder1/sample1` 라고 표시된다. fat-jar 파일이 중간에 `mavan-fat-jar.jar!` 로 표시돼 있는데 이렇게 `!`가 포함된 경로는 실제로 존재하지 않기 때문에 위와 같은 에러가 발생하게 된다.

즉 IDE에서 실행할 때는 실제 파일시스템 기준 경로를 따르므로 에러가 발생하지 않지만, jar 파일을 읽을 때는 jar 파일이 `!`와 함께 표시되기 때문에 실제 파일시스템 경로에 맞지 않아 에러가 발생한다.

rsourceURL 값이 IDE 에서 실행할 때는 `file:` 로 시작했는데, 이번에는 `jar:file:`로 시작한다. 이것도 기억해두자.

어쨌든 자바 프로그램은 실제로는 대부분 jar 로 만들어져서 실행될텐데, jar 에서 제대로 실행이 안 된다면 이를 어쩐다?


## getResourceAsStream()

getResource() 는 기본적으로 URL 을 반환한다. URL은 위와 같이 jar 파일을 `!`와 함께 표시하기 때문에, jar 실행 시 에러가 발생한다.

하지만 getResourceAsStream()은 InputStream 을 반환한다. 그리고 Java 9 에서 추가된 `InputStream.readAllBytes()`를 사용하면 편리하게 InputStream 을 읽어서 byte[] 에 저장할 수 있다.

```java
// ResourceLoader.java

    public void loadResourceAsStream(String resourceLocation) throws IOException {
        log.info("OOO getResourceAsStream() 방식");
        log.info("content root: {}", root);
        log.info("resourceLocation: {}", resourceLocation);

        InputStream resourceAsStream = this.getClass().getResourceAsStream(root + resourceLocation);
        byte[] bytes = resourceAsStream.readAllBytes();
        log.info("resource contents: {}", new String(bytes, StandardCharsets.UTF_8));
    }
```

### IDE 에서 동작 확인

IDE 에서 실행하면 다음과 같이 정상적으로 실행된다.

```
23:55:45.443 [main] INFO io.homo_efficio.ResourceLoader - OOO getResourceAsStream() 방식
23:55:45.443 [main] INFO io.homo_efficio.ResourceLoader - content root: /static
23:55:45.443 [main] INFO io.homo_efficio.ResourceLoader - resourceLocation: /folder1/sample1
23:55:45.443 [main] INFO io.homo_efficio.ResourceLoader - resource contents: Sample File 1
```

### fat-jar 에서 동작 확인

fat-jar 실행 시에도 정상적으로 실행된다.

```
maven-fat-jar-test git:master 🍺🦑🍺🍕🍺 ❯ java -jar target/maven-fat-jar.jar
00:01:31.774 [main] INFO io.homo_efficio.ResourceLoader - OOO getResourceAsStream() 방식
00:01:31.775 [main] INFO io.homo_efficio.ResourceLoader - content root: /static
00:01:31.776 [main] INFO io.homo_efficio.ResourceLoader - resourceLocation: /folder1/sample1
00:01:31.778 [main] INFO io.homo_efficio.ResourceLoader - resource contents: Sample File 1
```

따라서 `getResource()` 보다는 `getResourceAsStream()`을 사용하자. 끗?

이렇게 끝내면 너무 싱거우니 `getResourceAsStream()`로 하면 왜 jar 에서도 정상 실행되는지 살짝만 들여다보자.


## 속사정

`getResourceAsStream()` 호출을 따라가보면 Java 14 기준 `BuiltinClassLoader` 클래스에서 아래와 같은 코드를 만나게 되는데,

![Imgur](https://i.imgur.com/m87zAsl.png)

`openStream()` 을 따라가면 왜 되는지 알 수 있다. `openConnection()`은 URLConnection 을 반환하는데, 이 URLConnection 에는 여러가지 SubClass가 있어서 다형적으로 동작한다.

![Imgur](https://i.imgur.com/oZz9Wu2.png)

앞에서 IDE 에서 실행할 때는 URL 값이 `file:` 로 시작하고, jar 로 실행할 떄는 URL 값이 `jar:file:` 로 시작하는 것을 기억해두자고 한 것을 상기해보면 답이 보일 것이다.

URL 이 `file:` 로 시작하는 IDE 에서는 FileURLConnection 이 사용되고, URL 이 `jar:file:` 로 시작하는 jar 실행에서는 JarURLConnection 이 사용된다. `getResourceAsStream()`은 다형적으로 동작하도록 구현돼 있어서 두 상황 모두에서 잘 동작할 수 있다.

그럼 `getResource()`는?

사실 문제는 `getResource()` 가 아니라 `getResource()` 이 반환하는 URL 을 어떻게 쓰느냐에 있다. 똑같이 `getResource()` 를 사용하더라도 다음과 같이 `openStream()` 을 사용하면 `getResource()` 을 사용해도 jar 에서도 잘 동작한다.

즉, URL 에서 File 을 생성하면 다형성이 적용되지 않아 IDE 에서는 되지만 jar 에서는 안 되는 상황이 연출되고,  
URL 에서 InputStream 을 뽑아서 사용하면 다형성이 적용돼서 IDE, jar 모두에서 잘 동작한다.

```java
    public void loadResourceAsFile(String resourceLocation) throws IOException {
        log.info("*** getResource() + File 방식");
        log.info("content root: {}", rootPath);
        log.info("resourceLocation: {}", resourceLocation);

        URL resourceURL = this.getClass().getResource(root + resourceLocation);
        log.info("resourceURL: {}", resourceURL);

//        String fileLocation = resourceURL.getFile();
//        log.info("fileLocation from URL: {}", fileLocation);
//
//        File file = new File(fileLocation);
//        FileReader fileReader = new FileReader(file);
//        char[] chars = new char[(int) file.length()];
//        fileReader.read(chars);
//
//        log.info("resource contents: {}", new String(chars));

        URL resource = this.getClass().getResource(root + resourceLocation);
        InputStream inputStream = resource.openStream();
        byte[] bytes = inputStream.readAllBytes();
        log.info("resource contents: {}", new String(bytes, StandardCharsets.UTF_8));
    }
```
