# Java ClassLoader 훑어보기

아주 예전에 SCJP 시험볼 때나 살펴본 이후로 자바의 클래스로더를 직접 다뤄야 할 일은 솔직히 없었다. 그래서 거의 잊고 살아왔는데 요즘 Quartz를 다루면서 Quartz에 없는 기능인 외부 Job 클래스 로딩 기능을 만들면서 정말로 오랜만에 들여다보게 됐다.

클래스로더는 Java9에 모듈 시스템이 도입되면서 적지 않은 변경이 있었다. 자세한 내용은 https://docs.oracle.com/javase/9/migrate/toc.htm#JSMIG-GUID-D867DCCC-CEB5-4AFA-9D11-9C62B7A3FAB1 를 참고하고, 먼저 Java8 까지 적용됐던 내용을 기준으로 되짚어보자.


## 3가지 기본 클래스로더

![Imgur](https://i.imgur.com/cs5Qyoe.png)

### Bootstrap ClassLoader

- 부트스트랩 클래스로더는 3가지 기본 클래스로더 중 최상위 클래스로더로서, 쉽게 말하면 `jre/lib/rt.jar`에 담긴 JDK 클래스 파일을 로딩한다. 
- Native C로 구현돼 있어서, `String.class.getClassLoader()`는 그냥 `null`을 반환한다. Primordial ClassLoader 라고 불리기도 한다.

### Extension ClassLoader

- 익스텐션 클래스로더는 `jre/lib/ext` 폴더나 `java.ext.dirs` 환경 변수로 지정된 폴더에 있는 클래스 파일을 로딩한다. 
- Java로 구현되어 있으며 `sun.misc.Launcher` 클래스 안에 static 클래스로 구현되어 있으며, `URLClassLoader`를 상속하고 있다.

### Application ClassLoader

- 애플리케이션 클래스로더는 `-classpath(또는 -cp)`나 JAR 파일 안에 있는 Manifest 파일의 `Class-Path` 속성값으로 지정된 폴더에 있는 클래스를 로딩한다.
- 익스텐션 클래스로더와 마찬가지로 Java로 구현되어 있으며, `sun.misc.Launcher` 클래스 안에 static 클래스로 구현되어 있으며, `URLClassLoader`를 상속하고 있다.
- 개발자가 애플리케이션 구동을 위해 직접 작성한 대부분의 클래스는 이 애플리케이션 클래스로더에 의해 로딩된다.


## 3가지 원칙

자바 클래스로더는 3이라는 숫자와 친해 보인다. 기본 클래스로더가 3가지이고, 작동 원칙도 3가지다.

### Delegation Principle

위임 원칙은 클래스 로딩이 필요할 때 **3가지 기본 클래스로더의 윗 방향으로 클래스 로딩을 위임하는 것**을 말한다. `main()` 메서드가 포함된 `ClassLoaderRunner` 클래스에서 개발자가 직접 작성한 `Internal` 클래스를 로딩하는 과정을 그림으로 표현하면 다음과 같다.

![Imgur](https://i.imgur.com/kijdBjb.png)

1. `ClassLoaderRunner`는 자기 자신을 로딩한 애플리케이션 클래스로더에게 `Internal` 클래스 로딩을 요청한다.

1. 클래스 로딩 요청을 받은 애플리케이션 클래스로더는 `Internal`을 스스로 직접 로딩하지 않고 상위 클래스로더인 익스텐션 클래스로더에게 위임한다.

1. 클래스 로딩 요청을 받은 익스텐션 클래스로더도 `Internal`을 스스로 직접 로딩하지 않고 상위 클래스로더인 부트스트랩 클래스로더에게 위임한다.

1. 부트스트랩 클래스로더는 `rt.jar`에서 `Internal`을 찾아서

    4.1 있으면 로딩 후 반환하고

1. 없으면 익스텐션 클래스로더가 `jre/lib/ext` 폴더나 `java.ext.dirs` 환경 변수로 지정된 폴더에서 `Internal`을 찾아서
  
    5.1 있으면 로딩 후 반환하고

1. 없으면 애플리케이션 클래스로더가 클래스패스에서 `Internal`을 찾아서
  
    6.1 있으면 로딩 후 반환하고

1. 없으면 `ClassNotFoundException`이 발생한다.

이런 식으로 동작하는 이유는 두 번째 원칙인 Visibility Principle과 관련이 있다.

### Visibility Principle

가시범위 원칙은 **하위 클래스로더는 상위 클래스로더가 로딩한 클래스를 볼 수 있지만, 상위 클래스로더는 하위 클래스로더가 로딩한 클래스를 볼 수 없다**는 원칙이다.

클래스로더를 3가지로 나눈 이유가 있을 것이다. 그런데 만약에 개발자가 만든 클래스를 로딩하는 애플리케이션 클래스로더가 부트스트랩 클래스로더에 의해 로딩된 `String.class`를 볼 수 없다면 애플리케이션은 `String.class`를 사용할 수 없을 것이다. 따라서 하위에서는 상위를 볼 수 있지만 상위에서는 하위를 볼 수 없어야 애플리케이션이 제대로 동작할 수 있다.

### Uniqueness Principle

유일성 원칙은 **하위 클래스로더는 상위 클래스로더가 로딩한 클래스를 다시 로딩하지 않게 해서 로딩된 클래스의 유일성을 보장**하는 것이다. 유일성을 식별하는 기준은 클래스의 `binary name`이며 자세한 내용은 https://docs.oracle.com/javase/specs/jls/se8/html/jls-13.html#jls-13.1 를 참고한다.

---

Todo

## URLClassLoader

- External 클래스가 out 폴더와 외부 폴더 동시에 있으면 URLClassLoader로 로딩하더라도 애플리케이션 클래스에 의해 로딩된다.
- External 클래스가 out 폴더에 없고 외부 폴더에만 있으면 URLClassLoader에 의해 로딩된다.
- External 클래스의 Field로 Internal을 추가해서 컴파일 후 외부 폴더에만 있는 상태에서 URLClassLoader로 External을 로딩하면 Internal도 애플리케이션 클래스로더에 의해 로딩된다. 즉, 애플리케이션 내에 있는 클래스와 애플리케이션 밖에 있는 클래스를 조합해서 사용할 수 있다.

---

## Java9에서는 ZipInfo.class 가 컴파일도 안됨

