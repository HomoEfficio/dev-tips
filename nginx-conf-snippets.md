# nginx conf snippets

## log

```
error_log  /var/log/nginx/error.log debug;

http {

    log_format  main  escape=none '$remote_addr - $remote_user [$time_local] "$request"'
                      ' $status $body_bytes_sent "$http_referer"'
                      ' "$http_user_agent" "$http_x_forwarded_for" "$content_type" "$request_body"';
```

- error_log 의 로그 포맷으로 debug 를 지정하면 아래와 같이 rewritten 정보 등 추가 정보 확인 가능
  ```
  2023/04/12 16:38:25 [notice] 115609#115609: *1862 "^XXXXX$" matches "YYYYY", client: AAA, server: BBB, request: "POST CCC HTTP/1.1", host: DDD
  2023/04/12 16:38:25 [notice] 115609#115609: *1862 rewritten data: "ZZZZZ", args: "", client: AAA, server: BBB, request: "POST CCC HTTP/1.1", host: DDD
  ```
- `main` 커스텀 로그 포맷 정의
  - `access_log  /var/log/nginx/access.log  main;` 이런 식으로 사용
- `escape=none`: `/x22` 같은 인코딩 없이 깔끔하게 데이터 표시
