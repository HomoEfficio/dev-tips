# IntelliJ Gradle Entity 모듈 분리

프로젝트 상황에 따라 엔티티를 공유하고 그 외 api 서버는 개별 멀티 모듈로 구성할 때 Entity를 분리하는 방법

예를 들어 user-api 서버에 해당하는 user-api 모듈과 admin-api 서버에 해당하는 admin-api 모듈이 있을 때 어떻게 처리해야 할까?

다음과 같은 순서로 진행하면 된다.

1. entity 모듈 생성
2. 모듈 간 컴파일 의존관계 구성
3. 실제 파일 이동
4. entity 모듈 라이브러리화
5. Entity 및 Repository 사용 설정

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

- xxx-api -> entity 의존관계 추가 전

    ![Imgur](https://i.imgur.com/122Yqn7.png)

- 다음과 같이 xxx-api -> entity 컴파일 의존관계 구성


    ```groovy
    ...

    dependencies {
        compile project(':xxx-entity')

    ...
    ```

- xxx-api -> entity 의존관계 추가 후

    ![Imgur](https://i.imgur.com/8RTA0LW.png)


### Project Structure

혹시 위와 같이 컴파일 의존관계를 구성해줬는데도, xxx-api 에서 entity 모듈 내의 클래스를 인식하지 못하거나, 인식 잘 되다가 가끔 IntelliJ를 껐다 켜면 인식 못 하는 경우에는 깨진 의존관계를 아래와 같이 수동으로 직접 다시 맺어주면 된다.

![Imgur](https://i.imgur.com/f8vp4BS.png)

![Imgur](https://i.imgur.com/Y8YSZxK.png)

![Imgur](https://i.imgur.com/uo8TudK.png)

![Imgur](https://i.imgur.com/moIi8uX.png)


## 3. 실제 파일 이동

1, 2를 수행한 후에 파일을 이동하면 도메인 계층에 잘못 들어가 있는 Jackson 라이브러리 관련 코드 외에는 참조 깨진다는 경고 없이, 동일 프로젝트 내에서의 파일 이동 처럼 참조가 깨지지 않고 부드럽게 이동된다.

**1, 2를 제대로 수행하지 않으면 참조가 깨진다는 경고가 나오며, 이를 무시하고 이동하면 나중에 패키지 명칭을 모두 맞게 해줘도 참조가 깨진 상태에서 고쳐지지 않는 ~~Hello~~ Hell로 이어진다.**

기타 자잘한 컴파일 에러는 직접 수정한다.

## 4. entity 모듈 라이브러리화

entity 모듈은 단독으로는 실행되지 않으므로 메인 클래스를 지우고, executagle jar를 만들지 않고 일반 라이브러리 jar를 만들도록 entity 모듈의 build.gradle 에 다음과 같이 설정해서 라이브러리화 한다.

**이 작업을 해주지 않으면, 나중에 빌드나 실행할 때 분명히 존재하는 패키지인데도 찾을 수 없다는 골치 아픈 에러를 만나게 된다.**

```groovy
bootJar {
	enabled = false
}

jar {
	enabled = true
}
```


## 5. Entity 및 Repository 사용 설정

4까지 수행하고 xxx-api 애플리케이션을 실행하면 다음과 같이 Repository를 찾을 수 없다며 정상 기동에 실패한다.

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Parameter 0 of constructor in kr.co.apexsoft.gradnet2.api.user.user.service.UserService required a bean of type 'kr.co.apexsoft.gradnet2.entity.user.repository.UserRepository' that could not be found.


Action:

Consider defining a bean of type 'kr.co.apexsoft.gradnet2.entity.user.repository.UserRepository' in your configuration.


Process finished with exit code 0
```

다음과 같이 xxx-api 애플리케이션 메인 클래스의 `@SpringBootApplication` 의 `scanBasePackages` 속성을 아래와 같이 지정해주면 된다.

```java
@SpringBootApplication
@EntityScan("kr.co.apexsoft.gradnet2.entity")
@EnableJpaRepositories("kr.co.apexsoft.gradnet2.entity")
public class Gradnet2ApiUserApplication {

    public static void main(String[] args) {
        SpringApplication.run(Gradnet2ApiUserApplication.class, args);
    }

}
```
