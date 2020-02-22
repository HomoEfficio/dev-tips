# IntelliJ Gradle Entity 모듈 분리

프로젝트 상황에 따라 엔티티를 공유하고 그 외 api 서버는 개별 멀티 모듈로 구성할 때 Entity를 분리하는 방법

예를 들어 user-api 서버에 해당하는 user-api 모듈과 admin-api 서버에 해당하는 admin-api 모듈이 있을 때 어떻게 처리해야 할까?

다음과 같은 순서로 진행하면 된다.

1. entity 모듈 생성
2. 모듈 간 컴파일 의존관계 구성
3. 실제 파일 이동

## 1. entity 모듈 생성

entity 모듈을 생성하고 루트 프로젝트의 settings.gradle 에 다음과 같이 추가

```groovy
rootProject.name = 'xxx-api'

include 'xxx-entity'
include 'xxx-admin-api'
include 'xxx-user-api'
```

## 2. 모듈 간 컴파일 의존관계 구성

### build.gradle

admin-api, user-api 의 build.gradle 에 다음과 같이 entity 모듈에 대한 컴파일 의존관계 추가

```groovy
...

dependencies {
    compile project(':xxx-entity')

...
```

### Project Structure

IntelliJ 의 Project Structure에서도 의존관계를 구성해준다.

![Imgur](https://i.imgur.com/f8vp4BS.png)

![Imgur](https://i.imgur.com/Y8YSZxK.png)

![Imgur](https://i.imgur.com/uo8TudK.png)

![Imgur](https://i.imgur.com/moIi8uX.png)


## 3. 실제 파일 이동


