# msa-simple-mall

## SW

- OpenJDK 14.0.1
- Gradle 6.3 (Java 14 지원)
- Spring Boot 


# 멀티 모듈 프로젝트

- 소스 관리 편의를 위해 하나의 멀티 모듈 프로젝트에 모든 마이크로서비스를 서브 모듈로 구성

https://github.com/HomoEfficio/dev-tips/blob/master/Gradle%20Spring%20Boot%20멀티%20모듈%20프로젝트%20구성.md

- 비즈니스 모듈은 biz 아래에
- msa 패턴 구현체는 formation 아래에

- biz, formation 단순 디렉토리로 만들면 하위에 서브 모듈을 만들 수 없으므로 biz, formation 도 gradle/java 모듈로 생성
- 생성 후 자동 생성된 src/ 및 build.gradle 제거
  - 생성된 디렉터리 내 모든 디렉터리/파일을 제거해도 biz, formation 폴더는 모듈 폴더 상태 유지되므로 하위에 서브 모듈 추가 가능



# 비즈니스 모듈 구축 순서

- 판매자, 상품 이벤트, 상품
- 구매자, 주문



# 판매자

## 서브 모듈 구성

- Spring Initlizr

![Imgur](https://i.imgur.com/vl0o6Qa.png)

- IntelliJ Java 버전 설정

![Imgur](https://i.imgur.com/r4JbSut.png)

![Imgur](https://i.imgur.com/egzvS4t.png)

- 프로젝트 정상 실행으로 구성 확인

![Imgur](https://i.imgur.com/0i9Coku.png)


## 도메인 작성

- domain 폴더에 엔티티, 리포지토리 추가

### 엔티티

- `@Table(indexes = ...)`
- `@EqualsAndHashCode(of = "id")`
- `@NotNull`: app 단에서 null 값을 허용하지 않고 DDL에도 `not null` 추가됨
- `@Column(unique)`
- `@Email`

### 리포지토리

- Spring Data JPA Repository 추가
- 단 건 반환 시 `Optional`로 반환


## 컨트롤러 작성

- 단순 CRUD 이므로 서비스 레이어 없이 컨트롤러에서 직접 리포지토리에 접근하고 DTO 반환

### DTO

- upstream DTO는 In, downstream DTO는 Out postfix 추가
- In 에는 
  - 파라미터 바인딩을 위해 `@NoArgsConstructor`, `@Setter` 추가
  - 테스트에서 사용할 수 있도록 `@AllArgsConstructor` 추가
  - 엔티티에 있는 validation 을 참고해서 `@Size(min, max, message)` 애너테이션 추가
- Out 에는
  - Entity 에서 Out 으로 변환하는 케이스만 있으면 되므로 모든 필드를 final로 선언하고 `@RequiredArgsConstructor` 추가 및 `public static Entity from(Entity)` 메서드 추가

### 컨트롤러

- 판매자 추가(회원 가입)는 나중에 Security 설정에서 따로 `permitAll()` 처리를 해줘야 하므로 `@PostMapping("/sellers/sign-up")` 과 같이 `sellers` 뒤에 `sign-up` 추가
- 서비스 레이어 생략했으므로 컨트롤러 클래스에 `@Transactional` 추가

### 테스트

- MockMvc 테스트는 별도로 설정해주지 않으면 Filter 가 적용되지 않음
  - 따라서 MockMvc 로 호출할 때는 `@EnableWebSecurity`로 설정을 해도 Security Filter 도 적용되지 않으므로 모든 API 가 열려있게 됨
- `@ParameterizedTest`, `@MethodSource` 활용
- Delete의 경우 `ResponseEntity<String>`을 반환하므로 `.andExpect(content().string(actual)` 로 테스트


## bootJar

>./gradlew bootJar


## Docker

### Dockerfile

```Dockerfile
FROM openjdk:14.0.1-jdk

EXPOSE 8080

ADD ./build/libs/*.jar app.jar

ENTRYPOINT ["java","-jar","/app.jar"]
```

### Docker 이미지 빌드

>docker build -t seller .

`-t`는 tag

```
homo.efficio ~/gitRepo/study/msa-simple-mall/biz/seller 
🍺  docker build -t seller .
Sending build context to Docker daemon  73.53MB
Step 1/4 : FROM openjdk:14.0.1-jdk
 ---> db8922a53903
Step 2/4 : EXPOSE 8080
 ---> Using cache
 ---> ac078db10d03
Step 3/4 : ADD ./build/libs/*.jar app.jar
 ---> Using cache
 ---> f430dbb80f07
Step 4/4 : ENTRYPOINT ["java","-jar","/app.jar"]
 ---> Using cache
 ---> bcab7a4f48e0
Successfully built bcab7a4f48e0
Successfully tagged seller:latest
```

### 이미지 빌드 확인

>docker images | grep seller

```
homo.efficio ~/gitRepo/study/msa-simple-mall/biz/seller 
🍺  docker images | grep seller
seller                        latest              bcab7a4f48e0        28 minutes ago      553MB
```

### 컨테이너 생성 및 실행

>docker run --rm -p8080:8080 -e 'SPRING_PROFILES_ACTIVE=docker' seller

- `--rm`은 컨테이너 종료 시 컨테이너 삭제
- `-e`는 환경변수

![Imgur](https://i.imgur.com/12hwPg2.png)

### 컨테이너 종료 및 제거 확인

`CTRL+C`

![Imgur](https://i.imgur.com/iZcYAaT.png)

'{ \
    "name": "seller01", \
    "email": "seller01@test.com", \
    "phone": "123-456-7890", \
    "loginId": "Seller01", \
    "password": "Password01" \
}'


## 부하 테스트 및 시각화

https://github.com/HomoEfficio/dev-tips/blob/master/LoadTest-K6-InfluxDB-Grafana.md 참고



# 상품 이벤트

- id, (판매자쪽)상품번호, 상품 이름, 상품 카테고리, 가격, 수량



