# Mac Java Version Change

M1 Monterey 기준으로 현재 Java Version 확인: `/usr/libexec/java_home -V` 에서 맨 위에 나오는 버전

```
🦑🍺 ❯ /usr/libexec/java_home -V
Matching Java Virtual Machines (3):
    18.0.1.1 (arm64) "Oracle Corporation" - "OpenJDK 18.0.1.1" /Users/user/Library/Java/JavaVirtualMachines/dep-openjdk-18.0.1.1/Contents/Home
    14.0.2 (x86_64) "AdoptOpenJDK" - "AdoptOpenJDK 14" /Library/Java/JavaVirtualMachines/adoptopenjdk-14.jdk/Contents/Home
    11.0.11 (x86_64) "AdoptOpenJDK" - "AdoptOpenJDK 11" /Library/Java/JavaVirtualMachines/adoptopenjdk-11.jdk/Contents/Home
/Users/user/Library/Java/JavaVirtualMachines/dep-openjdk-18.0.1.1/Contents/Home
```

이 상태에서 기본 버전을 14.0.2 로 변경하고 싶으면 18.0.1.1 로 등록된 `/Users/user/Library/Java/JavaVirtualMachines/dep-openjdk-18.0.1.1/Contents/Home`를 삭제하면 된다.

삭제 후 위 명령을 다시 실행하면 아래와 같이 14.0.2 로 변경된 것을 확인할 수 있다.

```
🦑🍺 ❯ /usr/libexec/java_home -V
Matching Java Virtual Machines (2):
    14.0.2 (x86_64) "AdoptOpenJDK" - "AdoptOpenJDK 14" /Library/Java/JavaVirtualMachines/adoptopenjdk-14.jdk/Contents/Home
    11.0.11 (x86_64) "AdoptOpenJDK" - "AdoptOpenJDK 11" /Library/Java/JavaVirtualMachines/adoptopenjdk-11.jdk/Contents/Home
/Library/Java/JavaVirtualMachines/adoptopenjdk-14.jdk/Contents/Home
```
