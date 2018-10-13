# Java URLClassLoader로 알아보는 클래스로딩

Bootstrap ClassLoader, Extension ClassLoader, Application ClassLoader 이 3가지 기본 클래스로더 말고도 `URLClassLoader`가 있다. 
사실 Extension ClassLoader와 Application ClassLoader는 `URLClassLoader`를 상속받아서 만들어진 클래스다.

3가지 기본 클래스로더에 대한 자세한 내용은 [여기](https://github.com/HomoEfficio/dev-tips/blob/master/Java%20ClassLoader%20훑어보기.md)를 참고한다.

간단한 `URLClassLoader` 예제로 클래스로딩 과정을 짚어보자.

# Java 8

## URLClassLoader 예제

### ClassLoaderRunner

```java
package homo.efficio.classloader;

import com.sun.nio.zipfs.ZipInfo;

import java.io.File;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class ClassLoaderRunner {

    public static void main(String[] args) {
        System.out.println("-----------------------------------");
        System.out.println("3 Default ClassLoader\n");
        // Bootstrap ClassLoader 확인
        final ClassLoader bootStrapClassLoader = String.class.getClassLoader();
        System.out.println("Bootstrap Classloader - ClassLoader of String.class: " + bootStrapClassLoader);

        // Extension ClassLoader 확인
        final ClassLoader extensionClassLoader = ZipInfo.class.getClassLoader();
        System.out.println("Extension Classloader - ClassLoader of ZipInfo.class: " + extensionClassLoader);

        // Application ClassLoader 확인
        final ClassLoader applicationClassLoader = Internal.class.getClassLoader();
        System.out.println("Application Classloader - ClassLoader of Internal.class: " + applicationClassLoader);

        System.out.println("-----------------------------------");
        System.out.println("ClassLoader Hierarchy\n");

        System.out.println("BootStrap ClassLoader           : " + bootStrapClassLoader);
        System.out.println("extensionClassLoader.getParent(): " + extensionClassLoader.getParent());

        System.out.println("Extension ClassLoader             : " + extensionClassLoader);
        System.out.println("applicationClassLoader.getParent(): " + applicationClassLoader.getParent());


        // 외부 폴더에 있는 파일 존재 확인
        final File classRepo = new File("C:/Temp/class-repo/");
//        System.out.println(classRepo.exists());
        final File abcClassFile = new File("C:/Temp/class-repo", "homo/efficio/classloader/External.class");
//        System.out.println(abcClassFile.exists());

        try {

            System.out.println("-----------------------------------");
            System.out.println("ClassLoader for External and Internal\n");
            final URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{ classRepo.toURI().toURL() });
            final Class<?> externalFromUrl = urlClassLoader.loadClass("homo.efficio.classloader.External");
            System.out.println("ClassLoader of External: " + externalFromUrl.getClassLoader());
            System.out.println("ClassLoader of Internal: " + Internal.class.getClassLoader());

            System.out.println("-----------------------------------");

        } catch (MalformedURLException e) {
            throw new RuntimeException("URL 형식이 잘못되었습니다.", e);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("클래스가 없습니다.", e);
        }


    }
}

```

간단하다. 먼저 3가지 기본 클래스로더를 출력해보고, 기본 클래스로더 사이의 위계 구조(Hierarchy)를 출력해서 확인해본다.

마지막으로 프로젝트 경로 외부에 있는 클래스를 `URLClassLoader`를 통해 읽어오는 부분이 있다.

유의해야할 점은 **URLClassLoader의 생성자 인자로 URL을 넘겨줄 떄 `/`로 끝나는 문자열을 줘야 `.class` 파일을 인식할 수 있다**는 점이다.

### Internal

```java
package homo.efficio.classloader;

import java.util.UUID;

public class Internal {

    private Long id;

    private UUID uuid;
    
    // getter/setter 생략
```

Internal 클래스는 그냥 별다른 특이점이 없는 일반적인 클래스다.


### External

```java
package homo.efficio.classloader;

public class External {

    private Long id;

    private String name;

    private Internal internal;
    
    // getter/setter 생략
```

External 클래스는 컴파일 한 후에 클래스 파일을 프로젝트(클래스패스) 외부로 옮겨서 URLClassLoader 에 의해 로딩되는 부분을 테스트하는데 사용한다.

Internal 클래스를 필드로 가지고 있는데, External 클래스가 외부에서 로딩될 때 Internal 클래스의 로딩 여부를 확인하는 데 사용한다.

## 실행 결과 - 1

클래스 파일을 전혀 손대지 않은 실행 결과는 다음과 같다.

```
-----------------------------------
3 Default ClassLoader

Bootstrap Classloader - ClassLoader of String.class: null
Extension Classloader - ClassLoader of ZipInfo.class: sun.misc.Launcher$ExtClassLoader@5a07e868
Application Classloader - ClassLoader of Internal.class: sun.misc.Launcher$AppClassLoader@18b4aac2
-----------------------------------
ClassLoader Hierarchy

BootStrap ClassLoader           : null
extensionClassLoader.getParent(): null
Extension ClassLoader             : sun.misc.Launcher$ExtClassLoader@5a07e868
applicationClassLoader.getParent(): sun.misc.Launcher$ExtClassLoader@5a07e868
-----------------------------------
ClassLoader for External and Internal

ClassLoader of External: sun.misc.Launcher$AppClassLoader@18b4aac2
ClassLoader of Internal: sun.misc.Launcher$AppClassLoader@18b4aac2
-----------------------------------

Process finished with exit code 0
```

### 알 수 있는 점

- Bootstrap ClassLoader는 `null` 로 표시된다. 실제로 Bootstrap ClassLoader는 Native C로 구현되어 있다.
- `jre/lib/ext` 폴더에 있는 jar 파일 안에 있는 `ZipInfo.class`를 통해 Extension ClassLoader를 확인할 수 있다.
  - 참고로 Java 9 에서는 모듈 시스템이 도입되면서 클래스로더에도 변화가 있었으며, `ZipInfo.class` 파일을 찾지 못해 컴파일 에러가 발생한다.
- External 클래스는 `URLClassLoader`를 통해 로딩을 시도하더라도, **클래스 파일이 프로젝트 외부가 아닌 내부에 존재하고 있으면 클래스로딩 위임에 의해 [여기](https://github.com/HomoEfficio/dev-tips/blob/master/Java%20ClassLoader%20훑어보기.md)에 나온 것처럼 Application ClassLoader에 의해 로딩된다.**

## External 클래스를 프로젝트 내부에서 외부로 옮긴 후 실행

다음 그림과 같이 External 클래스를 프로젝트 내부(즉 클래스패스)에서는 없애고 외부에만 존재하게 바꾸고 실행하면,

![Imgur](https://i.imgur.com/uGY0oib.png)

다음과 같이 External 클래스가 Application ClassLoader가 아니라 `URLClassLoader`에 의해 로딩됨을 알 수 있다.

```
-----------------------------------
ClassLoader for External and Internal

ClassLoader of External: java.net.URLClassLoader@2c7b84de
ClassLoader of Internal: sun.misc.Launcher$AppClassLoader@18b4aac2
```

### 알 수 있는 점

- 컴파일 된 클래스 파일을 클래스패스 외부에 두고 `URLClassLoader`로 로딩할 수 있다.
- 외부에 있는 클래스 파일에 포함되어 있는 Internal는 클래스로더 위임에 따라 원래대로 Application ClassLoader에서 로딩한다.
  - **즉 클래스패스 외부에 있는 클래스 파일이 클래스패스 내부에 있는 클래스를 참조하더라도 둘 모두 문제 없이 로딩해서 조합해서 사용할 수 있다.**


# Java 9

## URLClassLoader 예제

### ClassLoaderRunner9

```java
package homo.efficio.classloader;

import java.io.File;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

/**
 * @author homo.efficio@gmail.com
 * Created on 2018-10-11.
 */
public class ClassLoaderRunner9 {

    public static void main(String[] args) {
        System.out.println("-----------------------------------");
        System.out.println("3 Default ClassLoader\n");
        // Bootstrap ClassLoader 확인
        final ClassLoader bootStrapClassLoader = String.class.getClassLoader();
        System.out.println("Bootstrap Classloader - ClassLoader of String.class: " + bootStrapClassLoader);

        // Platform ClassLoader 확인
        final ClassLoader platformClassLoader = ClassLoader.getPlatformClassLoader();
        System.out.println("Platform Classloader - ClassLoader.getPlatformClassLoader(): " + platformClassLoader);

        // System ClassLoader 확인
        final ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println("System Classloader - ClassLoader.getSystemClassLoader()    : " + systemClassLoader);

        System.out.println("-----------------------------------");
        System.out.println("ClassLoader Hierarchy\n");

        System.out.println("BootStrap ClassLoader           : " + bootStrapClassLoader);
        System.out.println("platformClassLoader.getParent() : " + platformClassLoader.getParent());

        System.out.println("Platform ClassLoader             : " + platformClassLoader);
        System.out.println("systemClassLoader.getParent()    : " + systemClassLoader.getParent());


        // 외부 폴더에 있는 파일 존재 확인
        final File classRepo = new File("C:/Temp/class-repo/");
//        System.out.println(classRepo.exists());
        final File abcClassFile = new File("C:/Temp/class-repo", "homo/efficio/classloader/External.class");
//        System.out.println(abcClassFile.exists());

        try {

            System.out.println("-----------------------------------");
            System.out.println("ClassLoader for External and Internal\n");
            final URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{ classRepo.toURI().toURL() });
            final Class<?> externalFromUrl = urlClassLoader.loadClass("homo.efficio.classloader.External");
            System.out.println("ClassLoader of External: " + externalFromUrl.getClassLoader());
            System.out.println("ClassLoader of Internal: " + Internal.class.getClassLoader());

            System.out.println("-----------------------------------");

        } catch (MalformedURLException e) {
            throw new RuntimeException("URL 형식이 잘못되었습니다.", e);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("클래스가 없습니다.", e);
        }
    }
}
```

Java 8과 달라진 점은 Platform ClassLoader와 System ClassLoader를 `ZipInfo.class`나 `Internal.class`와 같은 개별 클래스의 `getClassLoader()`가 아니라 **`ClassLoader.getPlatformClassLoader()`, `ClassLoader.getSystemClassLoader()`와 같이 `ClassLoader`의 static 메서드를 통해 직접 가졍로 수 있다**는 점이다.

## 실행 결과

```
-----------------------------------
3 Default ClassLoader

Bootstrap Classloader - ClassLoader of String.class: null
Platform Classloader - ClassLoader.getPlatformClassLoader(): jdk.internal.loader.ClassLoaders$PlatformClassLoader@e73f9ac
System Classloader - ClassLoader.getSystemClassLoader()    : jdk.internal.loader.ClassLoaders$AppClassLoader@726f3b58
-----------------------------------
ClassLoader Hierarchy

BootStrap ClassLoader           : null
platformClassLoader.getParent() : null
Platform ClassLoader             : jdk.internal.loader.ClassLoaders$PlatformClassLoader@e73f9ac
systemClassLoader.getParent()    : jdk.internal.loader.ClassLoaders$PlatformClassLoader@e73f9ac
-----------------------------------
ClassLoader for External and Internal

ClassLoader of External: java.net.URLClassLoader@96532d6
ClassLoader of Internal: jdk.internal.loader.ClassLoaders$AppClassLoader@726f3b58
-----------------------------------
```

클래스로더의 패키지와 이름이 좀 달라지기는 했지만, 기본 클래스로더의 3계층 구조나 3가지 원칙 등 내용적으로는 Java 8과 같다.

