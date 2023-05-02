# 자바 Manifest 파일

- Jar 파일의 메타 정보를 담는 파일
- 전반적인 내용은 https://docs.oracle.com/javase/tutorial/deployment/jar/manifestindex.html 참고
- Class-Path 설정 관련은 http://todayguesswhat.blogspot.com/2011/03/jar-manifestmf-class-path-referencing.html 참고

## SpringBoot의 Manifest 파일

- SpringBoot 애플리케이션에서 build.gradle을 다음과 같이 작성하면
    ```groovy
    bootJar {
        manifest {
            attributes (
                    'Spring-Boot-Version': '2.1.6.RELEASE',
                    'Class-Path': '/Users/tester/tmp/quartz/remote-job-repo/quartz-remote-jobs.jar BOOT-INF/lib/quartz-2.3.1.jar'
            )
        }
    }
    ```
- META-INF/MANIFEST.MF는 다음과 같이 생성된다.
    ```
    Manifest-Version: 1.0 
    Spring-Boot-Version: 2.1.6.RELEASE
    Class-Path: /Users/tester/tmp/quartz/remote-job-repo/quartz-remote-jo
     bs.jar BOOT-INF/lib/quartz-2.3.1.jar
    Main-Class: org.springframework.boot.loader.JarLauncher
    Start-Class: io.homo_efficio.quartz.scheduler.QuartzSchedulerApplicati
     on
    Spring-Boot-Classes: BOOT-INF/classes/
    Spring-Boot-Lib: BOOT-INF/lib/
    ```
    - 'Main-Class'는 `JarLauncher`나 `PropertiesLauncer` 등 애플리케이션을 띄우는 진입점을 지정
    - 'Start-Class'는 스프링부트 애플리케이션의 루트 애플리케이션 클래스를 지정
