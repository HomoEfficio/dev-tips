# StackTrace - fillInStackTrace() 이슈

## 문제

GlobalExceptionHandler 를 만들어서 예외 처리 채널을 일원화해서 사용하고 있다.

스택트레이스를 보면 아래와 같이 예외 발생 위치는 안 나오고 그냥 GlobalExceptionHandler 를 가리키는, 문제 해결에 도움이 안 되는 내용만 출력되는 경우가 종종 있었다.

```
2021-08-30 10:42:43.486 ERROR 60053 --- [nio-8080-exec-1] k.c.a.y.e.a.c.l.e.GlobalExceptionHandler : 

java.lang.IllegalArgumentException: 그냥 만든 예외
	at xxx.yyy.zzz.api.common.lib.error.GlobalExceptionHandler.handleException(GlobalExceptionHandler.kt:44) ~[main/:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:564) ~[na:na]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver.doResolveHandlerMethodException(ExceptionHandlerExceptionResolver.java:428) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.handler.AbstractHandlerMethodExceptionResolver.doResolveException(AbstractHandlerMethodExceptionResolver.java:75) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.handler.AbstractHandlerExceptionResolver.resolveException(AbstractHandlerExceptionResolver.java:141) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.handler.HandlerExceptionResolverComposite.resolveException(HandlerExceptionResolverComposite.java:80) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.processHandlerException(DispatcherServlet.java:1324) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.processDispatchResult(DispatcherServlet.java:1135) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1081) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:909) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:681) ~[tomcat-embed-core-9.0.52.jar:4.0.FR]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:764) ~[tomcat-embed-core-9.0.52.jar:4.0.FR]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:227) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) ~[tomcat-embed-websocket-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:327) ~[spring-security-web-5.5.2.jar:5.5.2]
  ...
```

## 원인

아래와 같이 `e.fillInStackTrace()`가 원인이었다.

```
log.error("", e.fillInStackTrace())
```


## 해결

아래와 같이 수정하니 제대로 된 의미있는 스택트레이스가 출력된다.

```
log.error(e.stackTraceToString())
```

아래와 같이 실제 예외가 발생한 위치를 포함하는 제대로 된 스택트레이스가 출력된다.

```
2021-08-30 10:47:39.158 ERROR 62222 --- [nio-8080-exec-1] k.c.a.y.e.a.c.l.e.GlobalExceptionHandler : java.lang.IllegalArgumentException: 그냥 만든 예외
	at xxx.yyy.zzz.api.service.command.UserCommandService.createUser(UserCommandService.kt:20)    // 여기!!
	at xxx.yyy.zzz.api.service.command.UserCommandService$$FastClassBySpringCGLIB$$eb5846a.invoke(<generated>)    // 여기!!
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:779)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750)
	at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:123)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:388)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:692)
	at xxx.yyy.zzz.api.service.command.UserCommandService$$EnhancerBySpringCGLIB$$3e1cf503.createUser(<generated>)    // 여기!!
	at xxx.yyy.zzz.api.controller.SignController.createUser(SignController.kt:28)    // 여기!!
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:564)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:808)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1064)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:909)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:681)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:764)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:227)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
  ...
```
