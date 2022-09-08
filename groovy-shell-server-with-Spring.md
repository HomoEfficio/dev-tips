# groovy-shell-server with Spring

- groovy-shell-server(https://github.com/bazhenov/groovy-shell-server): 실제 기동 중인 애플리케이션 서버에 터미널로 접속 후 특수한 셸(groovy-shell)에서 애플리케이션 로직을 실행할 수 있게 해주는 도구
- 아래와 같이 서버 터미널에서 ssh를 통해 groovy-shell에 접속한 후 스프링 애플리케이션 컨텍스트(ctx)를 통해 빈을 가져와서 실제 서버에서 로직 실행 가능
    ```
    root@app-567d946ffd-zs2zn:/# ssh 127.0.0.1 -p 54324
    The authenticity of host '[127.0.0.1]:54324 ([127.0.0.1]:54324)' can't be established.
    ECDSA key fingerprint is SHA256:RXExtz7UuOBJFK9rvooPLUiFJuZv3gupd4+c91YQSak.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '[127.0.0.1]:54324' (ECDSA) to the list of known hosts.
    Groovy Shell (3.0.8, JVM: 11.0.8)
    Type ':help' or ':h' for help.
    -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    groovy:000> jwtController = ctx.getBean('jwtController')
    ===> zzz.yyy.xxx.controller.JwtController@1e8a89e6
    groovy:000> jwtController.jwtToken('any-string')
    ===> <200 OK OK,eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiLsm5Trk5wg7ZSM656r7Y-8IOyWtOuTnOuvvCIsImF1dGgiOiJST0xFX1VTRVIsUk9MRV9SRVZJRVdFUixST0xFX1NVUEVSX0FETUlOIiwidWlkIjoiYW55LXN0cmluZyIsImV4cCI6MTY2MjcwNDUxOH0.nGlozID0-4-PCqCy_p7tprPFYohC3_AP_V346222KEIZkGgsx1aWXau8GRFvVI8iUshPg5soccz_VFFwh8E2Kg,[]>
    groovy:000> 
    ```

## 스프링 애플리케이션 설정

### build.gradle.kts

```
implementation("me.bazhenov.groovy-shell:groovy-shell-server:2.2.3")
```

### application.yml

```yml
groovy-shell-server:
  enabled: true
  port: 54324
```

### Configuration 클래스

- 대략 다음과 같이 GroovyShellServiceBean을 등록

```
@Configuration
class GroovyShellServerConfig(
    @Value("\${groovy-shell-server.port:23456}")    // application.yml 파일에 지정
    private val port: Int,
) {

    @ConditionalOnProperty(name = ["groovy-shell-server.enabled"], havingValue = "true")
    @Bean
    fun groovyShellServiceBean(applicationContext: ApplicationContext): GroovyShellServiceBean {
        val gssBean = GroovyShellServiceBean()
        gssBean.setPort(port)
        gssBean.isLaunchAtStart = true
        gssBean.setPublishContextBeans(false)
        gssBean.setDisableImportCompletions(true)

        gssBean.setBindings(
            applicationContext.beanDefinitionNames
                .filter { it != "groovyShellServiceBean" }
                .associateWith<String, Any> { applicationContext.getBean(it) }
        )

        log.info("🌈 Groovy Shell Server started on port $port. Type 'ssh 127.0.0.1 -p $port' to use. Good Luck!! 🍀")

        return gssBean
    }

    private val log = LoggerFactory.getLogger(javaClass)
}
```

### 애플리케이션 실행 시 로그

- 위 GroovyShellServiceBean에서 설정한 로그가 아래와 같이 출력됨

```
🌈 Groovy Shell Server started on port 54324. Type 'ssh 127.0.0.1 -p 54324' to use. Good Luck!! 🍀
```


## 셸 접근 및 사용

### 셸 접근

- 애플리케이션 서버의 터미널에 접속 후 `ssh 127.0.0.1 -p <PORT>`를 입력하고 yes를 클릭하면 groovy-shell 에 접속할 수 있다.

```
~ 🦑🍺 ❯ ssh 127.0.0.1 -p 54324         
The authenticity of host '[127.0.0.1]:54324 ([127.0.0.1]:54324)' can't be established.
ECDSA key fingerprint is SHA256:Sna/2gV4tZ9UPdz6J5E/48UhmfM+Z6K1fn6p/KsZJ9k.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:54324' (ECDSA) to the list of known hosts.
Groovy Shell (3.0.8, JVM: 14.0.2)
Type ':help' or ':h' for help.
------------------------------------------------------------------------------------------------------------------------------------------------------------------
groovy:000> 

```

혹시 아래와 같은 에러 발생 시 `/Users/user/.ssh/known_hosts`에서 해당 행을 삭제한다.

```
~ 🦑🍺 ❯ ssh 127.0.0.1 -p 54324
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:Sna/2gV4tZ9UPdz6J5E/48UhmfM+Z6K1fn6p/KsZJ9k.
Please contact your system administrator.
Add correct host key in /Users/user/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /Users/user/.ssh/known_hosts:3    <=== /Users/user/.ssh/known_hosts 파일의 3행을 삭제
Host key for [127.0.0.1]:54324 has changed and you have requested strict checking.
Host key verification failed.
```

### 애플리케이션 컨텍스트를 통해 빈 획득 및 빈 메서드 실행

- groovy-shell에서 `ctx.getBean(<빈 이름>)` 명령을 통해 빈을 획득하고 빈의 메서드를 실행할 수 있다.

```
groovy:000> jwtController = ctx.getBean('jwtController')
===> zzz.yyy.xxx.controller.JwtController@6dcbeb47
groovy:000> jwtController.jwtToken('any-string')
===> <200 OK OK,eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiLsm5Trk5wg7ZSM656r7Y-8IOyWtOuTnOuvvCIsImF1dGgiOiJST0xFX1VTRVIsUk9MRV9SRVZJRVdFUixST0xFX1NVUEVSX0FETUlOIiwidWlkIjoiYW55LXN0cmluZyIsImV4cCI6MTY2MjcwNTUxMX0.chufbjASOOJapLgVT6uxQGAGQekEyUinpNZEcX6L1UR0CgsLMAwW_6l9xsDADqADJKHA-Y_db3ioJu1uJ7p63Q,[]>
groovy:000> 

```



