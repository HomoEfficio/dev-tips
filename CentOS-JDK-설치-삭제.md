# CentOS 에서 JDK 설치-삭제

## 설치

- https://zetawiki.com/wiki/CentOS_JDK_설치
  - 이름이 아래와 같이 돼 있어서 openjdk 가 stable 버전이고, `-devel`은 뭔가 개발 중인 버전인가 하고 착각할 수 있는데,   

      ```
      java-버전-openjdk
      java-버전-openjdk-devel
      ```

  - `java-버전-openjdk`는 이름과 달리 실제로는 JRE이고, **JDK 설치하려면 `java-버전-openjdk-devel`을 설치해야함**

- 위와 같이 설치하면 `/etc/alternatives` 아래에 소프트 링크가 자동으로 추가됨

## alternatives 로 Java 버전 선택

- https://blog.seotory.com/post/2016/08/java-setting-at-centos

## 삭제

- https://zetawiki.com/wiki/CentOS_yum_패키지_삭제
