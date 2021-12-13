# Spring Boot 2.6 Upgrade

## 검토 사항

- 스프링 부트 공식 문서: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes
- 스프링 클라우드 버전: https://spring.io/projects/spring-cloud

## 업그레이드 절차

- build.gradle.kts 파일에서 다음과 같이 버전 변경

```
// ...
plugins {    
    id("org.springframework.boot") version "2.6.1"
    // ...
}
// ...
extra["springCloudVersion"] = "2021.0.0"
// ...
```

## Troubleshooting

### Jupiter 관련 문제

#### 테스트 실행 시 다음과 같은 오류 발생

```
org.gradle.api.internal.tasks.testing.TestSuiteExecutionException: Could not complete execution for Gradle Test Executor 4.
	at org.gradle.api.internal.tasks.testing.SuiteTestClassProcessor.stop(SuiteTestClassProcessor.java:63)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
	at org.gradle.internal.dispatch.ReflectionDispatch.dispatch(ReflectionDispatch.java:36)
	at org.gradle.internal.dispatch.ReflectionDispatch.dispatch(ReflectionDispatch.java:24)
	at org.gradle.internal.dispatch.ContextClassLoaderDispatch.dispatch(ContextClassLoaderDispatch.java:33)
	at org.gradle.internal.dispatch.ProxyDispatchAdapter$DispatchingInvocationHandler.invoke(ProxyDispatchAdapter.java:94)
	at com.sun.proxy.$Proxy2.stop(Unknown Source)
	at org.gradle.api.internal.tasks.testing.worker.TestWorker.stop(TestWorker.java:135)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
	at org.gradle.internal.dispatch.ReflectionDispatch.dispatch(ReflectionDispatch.java:36)
	at org.gradle.internal.dispatch.ReflectionDispatch.dispatch(ReflectionDispatch.java:24)
	at org.gradle.internal.remote.internal.hub.MessageHubBackedObjectConnection$DispatchWrapper.dispatch(MessageHubBackedObjectConnection.java:182)
	at org.gradle.internal.remote.internal.hub.MessageHubBackedObjectConnection$DispatchWrapper.dispatch(MessageHubBackedObjectConnection.java:164)
	at org.gradle.internal.remote.internal.hub.MessageHub$Handler.run(MessageHub.java:414)
	at org.gradle.internal.concurrent.ExecutorPolicy$CatchAndRecordFailures.onExecute(ExecutorPolicy.java:64)
	at org.gradle.internal.concurrent.ManagedExecutorImpl$1.run(ManagedExecutorImpl.java:48)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at org.gradle.internal.concurrent.ThreadFactoryImpl$ManagedThreadRunnable.run(ThreadFactoryImpl.java:56)
	at java.base/java.lang.Thread.run(Thread.java:834)
Caused by: org.junit.platform.commons.JUnitException: TestEngine with ID 'junit-jupiter' failed to discover tests
	at org.junit.platform.launcher.core.EngineDiscoveryOrchestrator.discoverEngineRoot(EngineDiscoveryOrchestrator.java:111)
	at org.junit.platform.launcher.core.EngineDiscoveryOrchestrator.discover(EngineDiscoveryOrchestrator.java:85)
	at org.junit.platform.launcher.core.DefaultLauncher.discover(DefaultLauncher.java:92)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:75)
	at org.gradle.api.internal.tasks.testing.junitplatform.JUnitPlatformTestClassProcessor$CollectAllTestClassesExecutor.processAllTestClasses(JUnitPlatformTestClassProcessor.java:99)
	at org.gradle.api.internal.tasks.testing.junitplatform.JUnitPlatformTestClassProcessor$CollectAllTestClassesExecutor.access$000(JUnitPlatformTestClassProcessor.java:79)
	at org.gradle.api.internal.tasks.testing.junitplatform.JUnitPlatformTestClassProcessor.stop(JUnitPlatformTestClassProcessor.java:75)
	at org.gradle.api.internal.tasks.testing.SuiteTestClassProcessor.stop(SuiteTestClassProcessor.java:61)
	... 25 more
Caused by: java.lang.NoClassDefFoundError: org/junit/jupiter/api/ClassOrderer
	at org.junit.jupiter.engine.config.DefaultJupiterConfiguration.<clinit>(DefaultJupiterConfiguration.java:50)
	at org.junit.jupiter.engine.JupiterTestEngine.discover(JupiterTestEngine.java:66)
	at org.junit.platform.launcher.core.EngineDiscoveryOrchestrator.discoverEngineRoot(EngineDiscoveryOrchestrator.java:103)
	... 32 more
Caused by: java.lang.ClassNotFoundException: org.junit.jupiter.api.ClassOrderer
	at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:581)
	at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178)
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:522)
	... 35 more
```

#### 해결

buildSrc 안에 있는 `*.gradle.kts` 파일 중에서 아래와 같이 jupiter 버전을 고정해둔 것을 5.8.1 로 변경 후 테스트 실행하면 성공

```
//...
    // Use JUnit Jupiter API for testing.
    // testImplementation("org.junit.jupiter:junit-jupiter-api:5.7.1")
    testImplementation("org.junit.jupiter:junit-jupiter-api:5.8.1")
//...
```


