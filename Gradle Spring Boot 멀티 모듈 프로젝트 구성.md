# Gradle Spring Boot 멀티 모듈 프로젝트 구성

- 하나의 루트 프로젝트 아래에 예를 들어 job-library, api-server 이렇게 다수의 서브프로젝트를 Gradle로 쉽게 구성하는 방법
- Gradle 5.5.1, Spring Boot 2.1.6, IntelliJ 기준

## 루트 프로젝트 구성

1. 루트 프로젝트 디렉토리를 만들고,
1. 만든 디렉토리로 이동해서 `gradle init` 실행
1. 프로젝트 타입에서 'basic' 선택
1. 언어에서 'Groovy' 선택(물론 Kotlin이 익숙하다면 Kotlin 선택 가능)

여기까지 실행 과정은 다음과 같다.

![Imgur](https://i.imgur.com/ykfzOT8.png)

생성되는 파일은 다음과 같다. 'src' 등이 생기지 않고 주로 Gradle 관련 파일만 생성된다.

![Imgur](https://i.imgur.com/dNPN9Zf.png)


## git 구성

생성된 파일 중에 .gitignore가 있기는 하지만 git 리포지토리까지 구성된 것은 아니다. 

1. `git init`을 실행해서 git 리포지토리를 구성한다.
1. git 리포지토리가 구성되면 `git commit -m 'Initial Commit'`을 실행해서 최초 커밋을 작성한다.

![Imgur](https://i.imgur.com/RSDnyrs.png)


## 라이브러리 모듈 생성

- IntelliJ 메인 메뉴의 New > Module.. 에서 Gradle > Java 를 선택하고 모듈 정보 입력 후 모듈생성

## 스프링부트 모듈 생성

스프링부트 모듈 생성 시 Spring Initializr를 사용하면 생성된 모듈이 별도의 프로젝트인 것처럼 인식된다.

!Gradle Task 창 캡처 그림

- IntelliJ 메인 메뉴의 New > Module.. 에서 Spring Initializr를 선택하고 Spring Boot Starter를 선택하고 모듈 생성
- 이때 서브모듈에는 있어서는 안 되는 settings.gradle 파일과 .gradle 디렉토리가 생성된다.
- 스프링부트 모듈의 settinge.gradle 파일을 지우고, 루트 프로젝트의 settings.gradle에 `include '스프링부트 모듈 프로젝트의 이름'` 추가
  - 루트 프로젝트 아래로 이동됨 !그림
- 빌드가 성공하면 .gradle 디렉토리를 삭제한다.

