# Gradle 의 plugin version 버전 지정 중앙화

멀티 프로젝트로 구성하면 plugins 에 버전 지정이 중복되기도 한다.  
어딘가에 변수를 선언해두고 사용하면 될 것 같은데 아쉽게도 plugins 안에서는 변수값을 사용하면 에러가 발생한다.

```kotlin
plugins {

    id("org.springframework.boot") version "$springBootVersion"
    // 'val springBootVersion: Any?' can't be called in this context by implicit receiver. Use the explicit one if necessary
```

찾아보니 https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_version_management 여기에 해법이 있다.

다음과 같이 gradle.properties 에 버전을 지정하고,

```
// gradle.properties

kotlinVersion=1.9.20
springDependencyManagementVersion=1.1.4
springBootVersion=3.2.1
```

다음과 같이 setting.gradle.kts 에 pluginmanagemnt 를 선언해서 gradle.properties에 정의된 버전를 사용할 수 있다.

```kotlin
// settings.gradle.kts

pluginManagement {
    // gradle.properties 에 정의된 properties 값을 읽어온다
    val kotlinVersion: String by settings
    val springDependencyManagementVersion: String by settings
    val springBootVersion: String by settings


    plugins {
        id("org.springframework.boot") version springBootVersion
        id("io.spring.dependency-management") version springDependencyManagementVersion

        kotlin("plugin.spring") version kotlinVersion
    }
}
```

이렇게 하고 하위 모듈에서는 plugin 의 version 을 명시하지 않으면 된다

```kotlin
// subproject 의 build.gradle.kts

plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")

    kotlin("plugin.spring")
}
```
