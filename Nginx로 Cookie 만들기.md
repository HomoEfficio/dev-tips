# Nginx로 Cookie 만들기

request parameter로 넘어온 값을 쿠키에 넣어주는 서버를 만들게 되었다.

하는 일은 단순하지만 수십개의 쇼핑몰에서 날라오는 요청을 처리해야 하므로 트래픽은 상당히 많을 것이다.

잘은 몰라도 할 줄 아는게 스프링 뿐이긴해도, 겨우 Cookie 심는 일을 하기 위해  스프링을 쓰는 건 너무 오버 같고..

그냥 웹 서버만으로 할 수 있지 않을까 하는 생각이 들었는데, 역시나 다른 파트의 동료는 아파치 모듈을 만들어서 쿠키를 심고 있었다.

그래서 나는 Nginx 모듈을 만들어 보기로 했다. C는 써본 지 십 여년이 훌쩍 넘어서 가물가물하지만, 달랑 쿠키 심는 일인데 검색해서 찾아보면 참고할 만한게 있겠지.. 하고 검색해보니.. 역시나 아래의 두 가지가 비교적 쉽고 이해할 만 했다.

- https://github.com/perusio/nginx-hello-world-module
- https://github.com/usamadar/ngx_hello_world

하지만 더 검색을 하다보니 Nginx 모듈 조차도 필요없고, 그냥 nginx.conf 수정만으로도 가능하다는 걸 알게 됐다.

## 시나리오

- cookie.mydomain.com/set/cookie/?a_id=QWE&b_id=ZXC 와 같은 요청을 받으면
- 쿠키에 a_id=QWE, b_id=ZXC 라는 값을 심는다.

## 쿠키 세팅

Nginx에서는 nginx.conf에서 `add_header Set_Cookie` 명령을 쓰는 것 만으로 쿠키에 원하는 값을 넣어줄 수 있다. 대략 아래와 같은 형식이다.

>add_header Set-Cookie "a_id=QWE;Domain=.mydomain.com;Path=/;Max-Age=31536000";

쿠키 값을 더 넣으려면 아래와 같이 추가할 수 있다.

>add_header Set-Cookie "a_id=QWE;Domain=.mydomain.com;Path=/;Max-Age=31536000";
>
>add_header Set-Cookie "b_id=ZXC;Domain=.mydomain.com;Path=/;Max-Age=31536000";

## 파라미터로 받은 값 세팅

위의 예제는 하드코딩 된 값을 쓰고 있어 실제 현장에서 사용하기엔 부적합하다. 요청 파라미터로 받아 처리하고 싶은데 Nginx에서는 `@arg_파라미터이름` 같은 형식으로 받아올 수 있다.

그리고 "my $name is homo.efficio" 와 같은 식으로 문자열 interpolation이 가능하다. 따라서 아래와 같이 하면 파라미터로 받은 값을 쿠키에 저장할 수 있다.

>add_header Set-Cookie "a_id=@arg_a_id;Domain=.mydomain.com;Path=/;Max-Age=31536000";
>
>add_header Set-Cookie "b_id=@arg_b_id;Domain=.mydomain.com;Path=/;Max-Age=31536000";

다음과 같이 set 명령을 활용해서 default 값을 주는 것도 좋은 방법이다.

```
set $a_id -1;
set $b_id -1;
if ($arg_a_id) {
    set $a_id $arg_a_id;
}
if ($arg_a_id) {
    set $b_id $arg_b_id;
}
add_header Set-Cookie "a_id=$a_id;Domain=.mydomain.com;Path=/;Max-Age=31536000";
add_header Set-Cookie "b_id=$b_id;Domain=.mydomain.com;Path=/;Max-Age=31536000";
```

## URI 처리

hostname/set/cookie/ 를 처리해야 하므로 nginx.conf에 다음과 같이 location 설정한다. 204는 반환할 resource가 없음을 의미하는 코드다.

```
location /set/cookie {
    set $dmp_id -1;
    set $recopick_id -1;
    if ($arg_dmp_id) {
        set $dmp_id $arg_dmp_id;
    }
    if ($arg_recopick_id) {
        set $recopick_id $arg_recopick_id;
    }
    add_header Set-Cookie "dmp_id=$dmp_id;Domain=.recopick.com;Path=/;Max-Age=31536000";
    add_header Set-Cookie "recopick_id=$recopick_id;Domain=.recopick.com;Path=/;Max-Age=31536000";
    return 204;
}
```

## Nginx 실행

>sudo /usr/local/nginx/sbin/nginx

## 쿠키 생성 접근 url

>localhost/recopick/pixel/?dmp_id=asdf&recopick_id=qwer

## Nginx 종료

>sudo kill -QUIT $( cat /usr/local/nginx/logs/nginx.pid )


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

