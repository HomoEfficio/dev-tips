# Jenkins-CI-JaCoCo

Jenkins 에 JaCoCo 플러그인을 설치하면 빌드 시 테스트를 돌리고 코드 커버리지 결과 리포트를 볼 수 있다.

Jenkins 플러그인은 단지 결과 리포트를 보여주는 역할을 하며, 결과 리포트를 생성하기 위해서는 먼저 빌드 과정에서 JaCoCo 가 커버리지 검사를 실행하도록 태스크를 설정해줘야 한다.

## Gradle 에 JaCoCo 설정

개략적인 샘플은 다음과 같고 자세한 내용은 https://docs.gradle.org/current/userguide/jacoco_plugin.html 참고

```kotlin
// build.gradle.kts

//...
plugins {
    //...
    id("jacoco")
    //...
}

//...

tasks.withType<Test> {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)  // 추가!!
}

// 아래 내용도 추가!!

// jacoco 버전 명시
jacoco {
    toolVersion = "0.8.7"
}

// jacoco 리포트 설정
tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        html.isEnabled = true
        xml.isEnabled = false
        csv.isEnabled = false
    }
    // 리포트에서 제외할 디렉터리 및 파일 설정
    classDirectories.setFrom(sourceSets.main.get().output.asFileTree.matching {
        exclude(listOf(
            "**/generated",
            "**/resources",
            "**/jib",
            "**/config",
        ))
    })
    finalizedBy(tasks.jacocoTestCoverageVerification)
}

// 코드 커버리지 강제: 기준치 이하면 빌드 실패 처리
tasks.jacocoTestCoverageVerification {
    violationRules {
        // https://docs.gradle.org/current/javadoc/org/gradle/testing/jacoco/tasks/rules/JacocoViolationRule.html
        rule {
            // 코드 커버리지 강제 처리에서 제외할 디렉터리 및 파일 설정
            excludes {
                [
                    "**/generated",
                    "**/resources",
                    "**/jib",
                    "**/config",
                ]
            }  
            limit {
                minimum = "0.10".toBigDecimal()
            }
        }
        rule {
            // https://www.eclemma.org/jacoco/trunk/doc/api/org/jacoco/core/analysis/ICoverageNode.ElementType.html
            element = "CLASS"
            enabled = true
            // https://docs.gradle.org/current/javadoc/org/gradle/testing/jacoco/tasks/rules/JacocoLimit.html
            limit {
                // https://www.eclemma.org/jacoco/trunk/doc/api/org/jacoco/core/analysis/ICoverageNode.CounterEntity.html
                counter = "LINE"
                // https://www.eclemma.org/jacoco/trunk/doc/api/org/jacoco/core/analysis/ICounter.CounterValue.html
                value = "COVEREDRATIO"
                minimum = "0.00".toBigDecimal()
            }
            limit {
                counter = "METHOD"
                value = "COVEREDRATIO"
                minimum = "0.00".toBigDecimal()
            }
        }
    }
}

//...
```

### 실행 확인

- gradle > Tasks > verification > test 실행
- 리포트 저장 위치 기본값: build/reports/jacoco


## JUnit 플러그인 설치

Jenkins > Plugin Manager 에서 JaCoCo 플러그인 설치

## Job 설정

### 테스트 실행 지정

- Build 항목에 테스트 실행 태스크를 추가, 아래는 gradle 사례

  - ![Imgur](https://i.imgur.com/Y0CJDxk.png)

### 빌드 후 조치 추가

- 테스트 결과 xml 파일을 읽어서 보여줄 수 있도록 개별 Job > 구성 > 빌드 후 조치 > 빌드 후 조치 추가 > 'Record JaCoCo coverage report' 항목 추가

  - ![Imgur](https://i.imgur.com/BJrB42d.png)

- 클래스 위치, 소스 파일 위치, 코드 커버리지 기준값 등 설정


## 테스트 결과 확인

- 빌드 후 Build History 에 아래와 같이 'Coverage Result' 메뉴가 표시된다.

  - ![Imgur](https://i.imgur.com/AgsMWuy.png)

- 클릭하면 아래와 같이 리포트를 볼 수 있다.

  - ![Imgur](https://i.imgur.com/3RLWwsu.png)

- 참고로 gradle 에 JaCoCo 태스크 설정을 안 해도 Jenkins 에서 리포트가 생성되긴 하는데, 테스트 코드에 의해 코드 커버리지 측정이 안 되었으므로 모두 빨간색으로만 표시된다.

  - ![Imgur](https://i.imgur.com/aAz56O2.png)

- Job 콘솔 로그를 보면 아래와 같이 JaCoCo 관련 로그가 표시된다

  - ![Imgur](https://i.imgur.com/iMOBQ9t.png)


## 후속 작업

- 결과 리포트에 보면 코드 커버리지 측정 대상이 아닌 디렉터리나 파일이 눈에 보일 것이다.
- 위 gradle 설정 샘플을 참고해서 `excludes` 항목으로 제외할 대상을 설정해준다.
- 현재 상태를 감안해서 코드 커버리지 기준 수치 등을 결정하고 설정한다.

