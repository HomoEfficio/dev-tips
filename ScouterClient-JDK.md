# Mac Scouter Client JDK 지정

Java 14에서 잘 동작하던 Scouter Client가 JDK 17을 설치하고 난 후 아래와 같은 에러로 실행되지 않는다 ㅠㅜ

![scouter-client-jdk-17-error](https://user-images.githubusercontent.com/17228983/180632439-8901e86f-31b0-4f05-b5ba-26bf435334c0.png)

다음과 같이 package 내용을 열어서

![scouter-client-pkg-info](https://user-images.githubusercontent.com/17228983/180632463-05cc2f19-8c35-4d04-bcfe-171fbce2e023.png)

info.plist 파일을 열고 vm 옵션으로 사용 가능한 JDK를 지정하면 된다. 설치된 JDK 위치는 `/usr/libexec/java_home -V`로 확인할 수 있다.

![scouter-client-resolve-jdk](https://user-images.githubusercontent.com/17228983/180632540-0b2c7e17-aa10-45fd-ad2c-360c4db04195.png)

