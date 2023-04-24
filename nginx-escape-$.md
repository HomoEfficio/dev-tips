# nginx escaping $

nginx 설정 중 `$`가 포함된 문자열을 변수로 지정하면 의도하지 않은 에러가 발생한다.

```
set $my_key "3t6w9z$C&E)H";
```

위 `$my_key`를 읽어서 사용하려고 하면 다음과 같은 에러가 발생한다.

```
nginx: [emerg] unknown "c" variable
```

이유는 위 문자열에 `$C`가 포함돼 있고 여기서 `$`가 escape 처리되지 않았으므로 `$C`라는 변수를 찾고, 그런 변수가 없어서 에러가 발생한다.

그럼 `$`를 `\$`와 같이 escape 처리해주면 간단한 일인데 아직까지 nginx에 `$`를 escape 할 수 있는 공식적인 방법이 없는 것 같다.

그래서 검색해보니 https://openresty.org/download/agentzh-nginx-tutorials-en.html#nginx-variables-escaping-dollar 여기에 답이 있었다.

간단하게 요약하면 `geo` directive 안에서는 variable interpolation이 적용되지 않는다는 성질을 이용한 편법이다.

`geo` directive는 `http` directive 안에서 정의할 수 있으므로 다음과 같이 정의하면 `$`를 escape 한 것과 같은 효과를 얻을 수 있다.

```
http {
    
    geo $dollar {
        default "$";
    }

    set $my_key "3t6w9z${dollar}C&E)H";  // ${dollar}가 $로 대체되면서 escape 된 것처럼 일반 문자열로 취급된다.
}
```
