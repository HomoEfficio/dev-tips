# Docker Image Platform

도커 이미지를 만들고 이 이미지로 k8s 상에서 pod를 생성하며 로그를 보니 다음과 같이 표시된다.

```
Failed to write to log, write logpipe: bad file descriptor
exec /usr/bin/bash: exec format error
Failed to write to log, write logpipe: bad file descriptor
Failed to write to log, write logpipe: bad file descriptor
```

별 정보가 없어 막막한데 검색해보니 이미지 생성 시 사용한 플랫폼과 실제 이미지를 통해 컨테이너를 생성할 때 해당 장비의 플랫폼이 다르면 위와 같은 에러가 난다고 한다.
내 경우도 맥북(aarch64)에서 이미지를 만들고 amd64 장비에서 pod를 생성해서 발생하는 에러였다.

그래서 도커 이미지 생성 시 다음과 같이 플랫폼을 지정해주고 문제를 해결할 수 있었다.

```
docker buildx build --platform linux/amd64 -t xxx:yyy .
```
