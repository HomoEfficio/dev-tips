# Gradle 4 Spring Boot 2 Multi Module 프로젝트에 QueryDSL 적용

- Java 8
- Gradle 4.10.3
- Spring Boot 2.2.4
- QueryDSL 4.2.2

## Multi Module 프로젝트 구성

- entity 서브 모듈에 엔티티 클래스 및 Repository 클래스 포함
- admin-api, user-api 는 entity 서브 모듈에 의존
- entity 서브 모듈에 QueryDSL 적용

```
root
  ㄴadmin-api
  ㄴentity
  ㄴuser-api
```

# 결론부터: entity 모듈의 build.gradle

- JPA와 QueryDSL에 필요한 의존관계만 포함

```groovy
plugins {
    id 'org.springframework.boot' version '2.2.4.RELEASE'
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
    id 'com.ewerk.gradle.plugins.querydsl' version '1.0.10'  // (1) QueryDSL 플러그인 설치
    id 'java'
}

group = 'io.homo_efficio'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.querydsl:querydsl-jpa'  // (2) QueryDSL 설치
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
}

test {
    useJUnitPlatform()
}

bootJar {
    enabled = false
}

jar {
    enabled = true
}

// (3) QueryDSL 설정
def querydslDir = 'build/generated/querydsl'

querydsl {
    library = 'com.querydsl:querydsl-apt'  // 이거 해주면 4.1.4 사라짐
    jpa = true
    querydslSourcesDir = querydslDir
}

sourceSets {
    main.java.srcDirs += [querydslDir]
}

configurations {
    querydsl.extendsFrom compileClasspath  // 오타 주의: complieClasspath 이렇게 오타치면 망함
}

compileQuerydsl {
    options.annotationProcessorPath = configurations.querydsl
}

```

# 과정별 보기

## (1) QueryDSL 플러그인 설치

설치 후 Gradle 패널에 다음과 같은 기능이 추가된다.

![Imgur](https://i.imgur.com/6i8Pvl5.png)

## (2) QueryDSL 설치

설치 후 다음과 같은 라이브러리가 추가된다.

![Imgur](https://i.imgur.com/5XD9B27.png)

4.1.4와 4.2.2가 함께 추가되어 있는데, 아래 설정의 `library = 'com.querydsl:querydsl-apt'` 가 적용되면 4.1.4는 사라지고 4.2.2만 남는다.

## (3) QueryDSL 설정 및 확인

다음과 같이 QueryDSL 관련 디렉터리 등 설정 후 `compileQuerldsl`을 실행해서 Q 클래스들이 생성되면 OK

![Imgur](https://i.imgur.com/Zp22rxP.png)


# 번외편

`compileClasspath`를 오타치면 다음과 같은 참상이 벌어진다.

![Imgur](https://i.imgur.com/Ber2Px1.png)

