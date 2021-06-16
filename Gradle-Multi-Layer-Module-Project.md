# Gradle Multi Layer Module Project 구성

그레이들은 멀티 모듈 프로젝트를 구성할 수 있고 관련 자료도 많다. 그래서 아래와 같이 구성하는 자료는 쉽게 찾을 수 있다.

```
root-project
  ㄴsubmodule-1/
  ㄴsubmodule-2/
  ㄴsubmodule-3/
  ㄴgradle/
  ㄴgradlew
  ㄴgradlew.bat
  ㄴsettings.gradle.kts
```

그런데 아래와 같이 구성하는 것도 가능할까?

```
root-project
  ㄴsubsystem-1
    ㄴsubsystem-1-1
      ㄴsubmodule-1/
      ㄴsubmodule-2/
      ㄴsubmodule-3/
    ㄴsubmodule-1/
    ㄴsubmodule-2/
    ㄴsubmodule-3/
  ㄴsubsystem-2
    ㄴsubmodule-1/
    ㄴsubmodule-2/
    ㄴsubmodule-3/
  ㄴsubsystem-3
    ㄴsubmodule-1/
    ㄴsubmodule-2/
    ㄴsubmodule-3/
  ㄴgradle/
  ㄴgradlew
  ㄴgradlew.bat
  ㄴsettings.gradle.kts
```

가능하다.

사실 그레이들에는 멀티 레이어 개념은 없는 것 같다. 어느 서브시스템에 있는 submodule-1,2,3 이든 gradle 입장에서는 그저 모듈일 뿐이다.

그리고 위와 같은 멀티 레이어 구조(처럼 보이는 구조)에서 subsystem-1, subsystem-1-1, subsystem-2, subsystem-3 은 그저 단순한 디렉터리일 뿐이다.

그레이들 스크립트에서는 디렉터리 구분을 나타내는 `/`를 사용할 수 없고 대신에 `:`를 사용해야 하므로,  
`rootProject / susbystem-1 / subsystem-1-1 / submodule-1` 은 그레이들 스크립트 안에서 `:subsystem-1:subsystem-1-1:submodule-1` 라고 지칭할 수 있는 모듈일 뿐이고,  
`rootProject / susbystem-2 / submodule-1` 은 그레이들 스크립트 안에서 `:subsystem-2:submodule-1` 라고 지칭할 수 있는 모듈일 뿐이다.


그래서 모듈을 만들 때 아래와 같이 물리적인 위치만 적절하게 지정해주고,

![Imgur](https://i.imgur.com/IQm4OVc.png)

아래와 같이 settings.gradle.kts 에서 모듈 이름을 올바르게 지정해주면 멀티 레이어(인 것 같은) 모듈 프로젝트를 구성할 수 있다.

```kotlin
// settings.gradle.kts

rootProject.name = "root-project"

include("subsystem-1:submodule-1-1:submodule-1", "subsystem1:submodule-1-1:submodule-2", "subsystem1:submodule-1-1:submodule-3")
include("subsystem-2:submodule-1", "subsystem-2:submodule-2", "subsystem-2:submodule-3")
include("subsystem-3:submodule-1", "subsystem-3:submodule-2", "subsystem-3:submodule-3")
```
