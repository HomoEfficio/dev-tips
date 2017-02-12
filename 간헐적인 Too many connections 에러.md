# 간헐적인 Too many connections 에러

예전에 개발했던 SpringMVC 애플리케이션이 DB 접근 시 아무 문제가 없었는데, 다른 곳에 넘겨주고 나니 DB 연결이 되다 안 되다 하면서 안 될 때는 대략 다음과 같은 에러가 난다고 한다.

```java
2017-02-10 11:13:35.116 ERROR 42888 --- [nio-8080-exec-2] o.a.tomcat.jdbc.pool.ConnectionPool      : Unable to create initial connections of pool.

com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Data source rejected establishment of connection,  message from server: "Too many connections"
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[na:1.8.0_92]
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[na:1.8.0_92]
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[na:1.8.0_92]
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423) ~[na:1.8.0_92]
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:425) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.Util.getInstance(Util.java:408) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:918) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:897) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:886) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.MysqlIO.doHandshake(MysqlIO.java:1040) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.ConnectionImpl.coreConnect(ConnectionImpl.java:2253) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:2284) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2083) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:806) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:47) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[na:1.8.0_92]
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[na:1.8.0_92]
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[na:1.8.0_92]
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423) ~[na:1.8.0_92]
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:425) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:410) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:328) ~[mysql-connector-java-5.1.40.jar:5.1.40]
	at org.apache.tomcat.jdbc.pool.PooledConnection.connectUsingDriver(PooledConnection.java:310) ~[tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.PooledConnection.connect(PooledConnection.java:203) ~[tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.ConnectionPool.createConnection(ConnectionPool.java:718) [tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection(ConnectionPool.java:650) [tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.ConnectionPool.init(ConnectionPool.java:468) [tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.ConnectionPool.<init>(ConnectionPool.java:143) [tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.DataSourceProxy.pCreatePool(DataSourceProxy.java:118) [tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.DataSourceProxy.createPool(DataSourceProxy.java:107) [tomcat-jdbc-8.5.6.jar:na]
	at org.apache.tomcat.jdbc.pool.DataSourceProxy.getConnection(DataSourceProxy.java:131) [tomcat-jdbc-8.5.6.jar:na]
	at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:111) [spring-jdbc-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:77) [spring-jdbc-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.mybatis.spring.transaction.SpringManagedTransaction.openConnection(SpringManagedTransaction.java:82) [mybatis-spring-1.3.1.jar:1.3.1]
	at org.mybatis.spring.transaction.SpringManagedTransaction.getConnection(SpringManagedTransaction.java:68) [mybatis-spring-1.3.1.jar:1.3.1]
	at org.apache.ibatis.executor.BaseExecutor.getConnection(BaseExecutor.java:336) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.executor.ReuseExecutor.prepareStatement(ReuseExecutor.java:88) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.executor.ReuseExecutor.doQuery(ReuseExecutor.java:59) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:324) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:156) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:109) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:83) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:148) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:141) [mybatis-3.4.2.jar:3.4.2]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:77) [mybatis-3.4.2.jar:3.4.2]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_92]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_92]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_92]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_92]
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:433) [mybatis-spring-1.3.1.jar:1.3.1]
	at com.sun.proxy.$Proxy78.selectOne(Unknown Source) [na:na]
	at org.mybatis.spring.SqlSessionTemplate.selectOne(SqlSessionTemplate.java:166) [mybatis-spring-1.3.1.jar:1.3.1]
	at kr.co.apexsoft.gradnet.framework.persistence.dao.CommonDAOMyBatisImpl.queryForObject(CommonDAOMyBatisImpl.java:71) [main/:na]
	at kr.co.apexsoft.gradnet.user.service.UserAccountServiceImpl.retrieveUser(UserAccountServiceImpl.java:75) [main/:na]
	at kr.co.apexsoft.gradnet.user.service.UserAccountServiceImpl$$FastClassBySpringCGLIB$$4bc9769.invoke(<generated>) [main/:na]
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204) [spring-core-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:720) [spring-aop-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157) [spring-aop-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:99) [spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:282) [spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96) [spring-tx-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) [spring-aop-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92) [spring-aop-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) [spring-aop-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:655) [spring-aop-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at kr.co.apexsoft.gradnet.user.service.UserAccountServiceImpl$$EnhancerBySpringCGLIB$$2d7cffa2.retrieveUser(<generated>) [main/:na]
	at kr.co.apexsoft.gradnet.framework.security.UserDetailsServiceImpl.loadUserByUsername(UserDetailsServiceImpl.java:23) [main/:na]
	at org.springframework.security.authentication.dao.DaoAuthenticationProvider.retrieveUser(DaoAuthenticationProvider.java:114) [spring-security-core-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.authentication.dao.AbstractUserDetailsAuthenticationProvider.authenticate(AbstractUserDetailsAuthenticationProvider.java:144) [spring-security-core-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.authentication.ProviderManager.authenticate(ProviderManager.java:174) [spring-security-core-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter.attemptAuthentication(UsernamePasswordAuthenticationFilter.java:94) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at kr.co.apexsoft.gradnet.framework.security.CustomLoginFilter.attemptAuthentication(CustomLoginFilter.java:34) [main/:na]
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:212) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:331) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:121) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:331) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.header.HeaderWriterFilter.doFilterInternal(HeaderWriterFilter.java:66) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:331) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:56) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:331) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:105) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:331) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:214) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:177) [spring-security-web-4.1.3.RELEASE.jar:4.1.3.RELEASE]
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:346) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:262) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:89) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:77) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.springframework.boot.actuate.autoconfigure.MetricsFilter.doFilterInternal(MetricsFilter.java:107) [spring-boot-actuator-1.4.2.RELEASE.jar:1.4.2.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:197) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-4.3.4.RELEASE.jar:4.3.4.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:198) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:108) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:472) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:349) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:784) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:802) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1410) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_92]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_92]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-8.5.6.jar:8.5.6]
	at java.lang.Thread.run(Thread.java:745) [na:1.8.0_92]
```

역시나 문제는 connection-pool의 개수였다. 개발용으로 로컬에서만 쓸 때마저 connection-pool의 초기 connection 개수가 50으로 설정되어 있었다.

```
application.yml

spring.datasource:
    type: org.apache.tomcat.jdbc.pool.DataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://any-server:3306/any-app_test?useUnicode=true&amp;characterEncoding=UTF-8&amp;characterSetResults=UTF-8&amp;zeroDateTimeBehavior=convertToNull
    username: any-username
    password: any-password
    tomcat.default-auto-commit: false
    tomcat.initial-size: 50
    tomcat.validation-query: SELECT 1
```

그리고 MySQL 5.6은 my.cnf에 설정된 내용을 일부러 바꾸지 않으면 [151개로 설정](https://dev.mysql.com/doc/refman/5.6/en/too-many-connections.html) 된다.

그래서 초기 connection 개수 50개인 커넥션 풀을 사용하는 데이터소스를 사용하는 애플리케이션을 개발팀원 여러명이 함께 실행시키면 연결이 될 때도 있고, Too many connections 에러가 날 때도 있었던 것이다.

해결은 간단하다. application.yml에 프로파일을 적절히 적용해주고 애플리케이션 기동 시 적절한 프로파일을 지정해주면 된다.

```
application.yml

spring.datasource:
    type: org.apache.tomcat.jdbc.pool.DataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://any-server:3306/any-app_test?useUnicode=true&amp;characterEncoding=UTF-8&amp;characterSetResults=UTF-8&amp;zeroDateTimeBehavior=convertToNull
    username: any-username
    password: any-password
    tomcat.default-auto-commit: false
    tomcat.initial-size: 50
    tomcat.validation-query: SELECT 1

---
spring.profiles: dev

spring.datasource:
    type: org.apache.tomcat.jdbc.pool.DataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://any-server:3306/any-app_test?useUnicode=true&amp;characterEncoding=UTF-8&amp;characterSetResults=UTF-8&amp;zeroDateTimeBehavior=convertToNull
    username: any-username
    password: any-password
    tomcat.default-auto-commit: false
    tomcat.initial-size: 3  # 적절한 수치로 조정
    tomcat.validation-query: SELECT 1
```
