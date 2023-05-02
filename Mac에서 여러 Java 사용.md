# Mac에서 여러 Java 사용

## jenv 설치 및 설정

https://junho85.pe.kr/736

위와 같이 해도 `mvn --version` 또는 메이븐 Wrapper 사용 시 프로젝트 루트에서 `./mvnw --version`을 해보면,  
jenv로 설정한 java 버전이 아니라 최종 설치된 자바 버전이 나올 수 있다.

아래와 같이 `jenv doctor`로 확인해보면 `JAVA_HOME` 이 제대로 설정돼 있지 않을 것이다.

```text
🍺  jenv doctor
[OK]	JAVA_HOME is not set
[OK]	Java binaries in path are jenv shims
[OK]	Jenv is correctly loaded
```

## jenv export plugin 설치

이 때는 다음과 같이 `export` 플러그인을 설치해주고,

>jenv enable-plugin export

새 터미널에서 `jenv doctor`를 실행하면 다음과 같이 `JAVA_HOME`이 jenv로 설정한 버전의 홈으로 설정된다.

```text
🍺  jenv doctor
[OK]	JAVA_HOME variable probably set by jenv PROMPT
[OK]	Java binaries in path are jenv shims
[OK]	Jenv is correctly loaded
```

`echo $JAVA_HOME`으로 확인해보면 다음과 같이 jenv로 지정한 버전이 표시된다.

```text
🍺  echo $JAVA_HOME
/Users/1003604/.jenv/versions/1.8
```

`mvn --version`으로 확인하면 다음과 같이 jenv로 설정한 버전으로 나온다.

```text
🍺  ./mvnw --version
Apache Maven 3.5.0 (ff8f5e7444045639af65f6095c62210b5713f426; 2017-04-04T04:39:06+09:00)
Maven home: /Users/1003604/.m2/wrapper/dists/apache-maven-3.5.0-bin/6ps54u5pnnbbpr6ds9rppcc7iv/apache-maven-3.5.0
Java version: 1.8.0_131, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/jre
Default locale: ko_KR, platform encoding: UTF-8
OS name: "mac os x", version: "10.14.5", arch: "x86_64", family: "mac"
```

