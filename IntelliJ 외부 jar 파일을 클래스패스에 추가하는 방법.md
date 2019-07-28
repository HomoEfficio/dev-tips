# IntelliJ 외부 jar 파일을 클래스패스에 추가하는 방법

인텔리제이에서 애플리케이션을 실행할 때 maven이나 gradle 등을 통해 예쁘게 추가된 jar 말고, 예를 들어 내가 만든 외부 jar 파일을 클래스패스에 추가하려면 어떻게 해야 할까?


## 인텔리제이의 java 실행 옵션 확인

먼저 인텔리제이가 `java` 명령과 `-classpath` 옵션으로 클래스패스를 지정해준 것을 확인해보자. 다음과 같이 콘솔에 회색으로 표시되는 부분을 클릭하면,

![Imgur](https://i.imgur.com/IJmwSYX.png)

다음과 같이 축약 표시됐던 자바 실행 옵션이 모두 표시된다. 이중 클래스패스는 `-classpath` 옵션으로 지정된다.

![Imgur](https://i.imgur.com/wp3FSAA.png)


## Run Configuration에서 클래스패스 추가

내가 만든 jar를 클래스패스에 포함하기 위해 다음과 같이 'Run Configuration' 화면의 'Program arguments'에 추가하면 클래스패스에 정상적으로 추가될까?

![Imgur](https://i.imgur.com/uQCXdTp.png)

아쉽지만 안 된다. 실행해보면 다음과 같이 인텔리제이의 `-classpath` 옵션 내용에 포함되는 대신에 맨 마지막에 `-cp` 옵션으로 별도로 추가되는데 이렇게 되면 클래스패스로 인식되지 못한다.

![Imgur](https://i.imgur.com/0M9dtb2.png)

그럼 어떻게 해야될까? 


## Project Structure에서 클래스패스 추가

다음과 같이 'Project Structure' 화면에서 'main' 모듈?에 jar를 직접 추가해주면 된다.

![Imgur](https://i.imgur.com/cHDaEyD.png)

![Imgur](https://i.imgur.com/EnCc09u.png)

![Imgur](https://i.imgur.com/th6CebB.png)

![Imgur](https://i.imgur.com/ezRF90P.png)

이렇게 추가해준 후 실행해보면 다음과 같이 인텔리제이의 `-classpath` 옵션 내용에 포함되며, jar 파일 안에 있는 클래스도 문제 없이 로딩된다.

![Imgur](https://i.imgur.com/XOdatWp.png)

