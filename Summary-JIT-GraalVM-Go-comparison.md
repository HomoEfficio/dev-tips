# JIT-GraalVM-Go 벤치마크 요약

원문: https://blog.devgenius.io/java-kotlin-graalvm-and-golang-7fb979ef4c03?gi=c79eb437d6de


## 구성

 \- | JVM | Go
--- | --- | ---
http | reactor-netty | fasthttp
serde | dsl-json | jsoniter
postgres-driver | vertx-pg-client | pgxpool

## 결과

 \- | JIT | GraalVM | Go
 --- | --- | --- | ---
결과물 크기 | 12M | 49M -> 11M(upx로 압축) | 14M -> 5.5M(upx로 압축)
도커 컨테이너 크기 | 306M | 30M | 17M
Cold start Mem | 34M | 75M | 2M
While testing Mem | 75M | 99M | 22M
While testing CPU | 45% 이하 | 150% - 250% | 250% - 330%
Req/s | 332 | 337 | 223

- 참고로 Rust로 구현 시 결과물 크기가 1M 였다고..

## 결론

- 늘 그렇지만 성능 벤치마크는 어떤 환경에서 어떤 라이브러리를 사용했는지에 따라 다를 수 있다.
- 이번 벤치마크에서는 JVM의 처리량이 Go보다 나았다.
- Go가 JVM보다 자원을 덜 쓰는 것은 확실하다.
- GraalVM이 결과물(artifact) 크기를 줄여주는 것은 확실하다. 다만 환경 설정 등이 훨씬 복잡하다.
- GraalVM은 AOT 때문에 컴파일이 오래 걸리므로 개발 생산성을 해칠 수 있다.
- Java 개발자 입장에서 Go의 에러 처리는 쉽지 않다.
- Go의 컴파일 속도가 가장 빠르다.
