# 윈도우에서 Gradle 사용 시 MS949 인코딩 에러 처리

윈도우에서 Gradle 사용할 때 아래와 같이 인코딩 관련 에러가 날 수 있다.

![](http://i.imgur.com/hIqspLO.png)

IDE에서 인코딩을 UTF-8로 다 맞춰줘도 여전히 발생하는데 환경 변수를 지정해주면 해결할 수 있다.

## JAVA_OPTS 추가

- 다음과 같이 `JAVA_OPTS`에 `-Dfile.encoding=UTF-8`을 설정해주고,

    ![](http://i.imgur.com/cIsdxAM.png)

- IDE를 재시작하고 gradle 명령을 다시 실행하면 인코딩 관련 에러가 사라진다.

    ![](http://i.imgur.com/8E56wvH.png)

- JAVA 전체에 영향을 미치게 하지 않고 GRADLE에만 영향을 주고 싶다면 `JAVA_OPTS` 대신에 `GRADLE_OPTS` 환경 변수에 `-Dfile.encoding=UTF-8`를 설 설정하면 된다.
