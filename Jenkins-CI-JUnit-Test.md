# Jenkins CI JUnit Test

Jenkins 에 JUnit 플러그인을 설치하면 빌드 시 테스트를 돌리고 결과 리포트도 볼 수 있다.

## JUnit 플러그인 설치

Jenkins > Plugin Manager 에서 JUnit 플러그인 설치

## Job 설정

### 테스트 실행 지정

- Build 항목에 테스트 실행 태스크를 추가, 아래는 gradle 사례

![Imgur](https://i.imgur.com/Y0CJDxk.png)

### 빌드 후 조치 추가

- 테스트 결과 xml 파일을 읽어서 보여줄 수 있도록 'Publish JUnit test result report' 항목 추가

![Imgur](https://i.imgur.com/Mji0fJf.png)

- Test report XMLs: gradle test 에 의해 생성되는 xml 파일의 위치 지정
  - 로컬 프로젝트 루트 dir에서 'gradlew 서브모듈경로:test' 실행 후 리포트가 생성되는 위치와 동일하게 지정

    ![Imgur](https://i.imgur.com/HfxV9v5.png)

## 테스트 결과 확인

- 빌드 후 Build History 에 아래와 같이 'Test Result' 메뉴가 표시된다.

![Imgur](https://i.imgur.com/0AJ6xYI.png)

- 클릭하면 아래와 같이 리포트를 볼 수 있다.

![Imgur](https://i.imgur.com/TuL6ixo.png)


## TroubleShooting

### Jenkins JUnit Test Result Error

빌드와 실제 테스트 실행도 모두 에러가 없었는데 콘솔 로그 마지막에 아래와 같이 표시되면서 Job이 실패로 종료되는 경우가 있다.

```
Recording test results
ERROR: Step ‘Publish JUnit test result report’ failed: Test reports were found but none of them are new. Did leafNodes run? 
For example, /xxx/build/test-results/test/TEST-xxx.yyy.zzz/ZzzTest.xml is 24 min old
```

이럴 때는 아래와 같이 Jenkins Job 설정 > Build 에서 `clean`을 추가해주면 해결된다.

![Imgur](https://i.imgur.com/fZz7rr9.png)



