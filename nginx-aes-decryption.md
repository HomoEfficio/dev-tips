# nginx - decypting AES encrypted request body and proxy_pass

외부서버 - 내부망(ReverseProxy - targetServer) 통신을 하는데,
외부서버 -> ReverseProxy 요 구간에 전송되는 정보에 AES 암호화를 적용해야 하고,
ReverseProxy에서는 복호화 한 다음 targetServer에 요청을 전송해야 한다.

결국 ReverseProxy를 어떻게 구성할 것인가의 문제다.

nginx에서도 Lua 모듈을 사용해서 AES 암복호화를 할 수 있으므로 이를 적용해보기로 한다.

nginx에 Lua 모듈을 적용하는 방법도 여러가지겠지만 기본 nginx 대신에 nginx를 wrapping하고 Lua 등 추가 모듈을 사용할 수 있는 OpenResty를 설치하는 것이 가장 간단하다.

## OpenResty 설치

https://openresty.org/en/linux-packages.html 여기에 리눅스 배포판 별 바이너리 및 설치 방법이 잘 나와 있다.

여기에서는 CentOS 기준으로 진행하며, `/usr/local/openresty`에 설치된다.


## Lua 모듈 사용을 위한 nginx.conf 설정

`lua_package_path`를 지정해준다.

```
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  escape=none '$remote_addr - $remote_user [$time_local] "$request"'
                      ' $status $body_bytes_sent "$http_referer"'
                      ' "$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    lua_package_path "/usr/local/openresty/lualib/resty/?.lua;;";  # 여기!!

    include my-server.conf;

    include my-upstream.conf;
}
```

## AES 복호화 Lua 스크립트

AES 256, CBC, iv를 사용해서 암호화한 후 Base64 인코딩 된 요청 본문을 읽고 복호화해서 다시 요청 본문에 저장하는 Lua 스크립트는 다음과 같다.

```
-- 파일 경로: /usr/local/openresty/nginx/conf/request-body-decrypter.lua

local aes = require "resty.aes"
local str = require "resty.string"
local cjson = require "cjson.safe"
local key = ngx.var.my_key

ngx.req.read_body()
local base64_encoded_rb = ngx.req.get_body_data()
-- ngx.log(ngx.DEBUG, "base64_encoded_rb : ", base64_encoded_rb)
local encrypted_rb = ngx.decode_base64(base64_encoded_rb)
-- ngx.log(ngx.DEBUG, "encrypted_rb : ", encrypted_rb)

local aes_256_cbc = aes:new(key, nil, aes.cipher(256, "cbc"), {iv = string.rep(string.char(0), 16)}, nil, nil, true)
local decrypted_rb = aes_256_cbc:decrypt(encrypted_rb)
-- ngx.log(ngx.DEBUG, "decrypted_rb: ", decrypted_rb)

ngx.req.set_body_data(decrypted_rb)
-- ngx.log(ngx.INFO, "requestbody: ", ngx.req.get_body_data())
ngx.req.set_method(ngx.HTTP_POST)

ngx.req.set_header("Content-Length", #decrypted_rb)
ngx.req.set_header("Content-Type", "application/json")
```

- iv 값은 암호호 한 쪽에서 정한 값을 사용하면 되며, 별도로 지정지 않았다면 위와 같이 `string.rep(string.char(0), 16)`로 지정하면 된다.

## nginx.conf

복호화 Lua 스크립트를 읽어서 사용하려면 다음과 같이 `rewrite_by_lua_file`을 지정한다.

```
# my-server.conf

server {

    location /lua-decrypt {
        access_log  logs/lua-decrypt.access.log  main;
        set $my_key "abcdefg";
        rewrite_by_lua_file conf/request-body-decrypter.lua;  # 여기!!
        include proxy.conf;
        rewrite  ^/lua-decrypt/(.*)$ /$1 break;
        proxy_pass  https://any-upstream;
    }
}
```

주의할 점은 proxy_pass를 실행할 때는 `content_by_lua_file`을 적용하면 Lua 스크립트가 실행되지 않으므로 반드시 `rewrite_by_lua_file`를 사용해야 한다는 점이다. 왜 그런지는 아래 그림을 참고하자.

![](https://cloud.githubusercontent.com/assets/2137369/15272097/77d1c09e-1a37-11e6-97ef-d9767035fc3e.png)



