# 사용자 입력 값은 Path Variable로 받지 말자

Path Variable로 받은 값에 `/`가 있으면 의도하지 않은 URL로 인식되어 별도의 처리가 필요하다.

스프링 환경에서 처리할 수 있는 방법은 [여기](https://stackoverflow.com/questions/13482020/encoded-slash-2f-with-spring-requestmapping-path-param-gives-http-400)에 여러가지가 나와있다.

1. 하위 URL을 `/**`로 뭉뜽그린 후에 컨트롤러 메서드에서 파싱해서 처리
1. `WebMvcConfigurerAdapter`(이건 deprecated됨)나 `WebMvcConfigurationSupport`와 `UrlPathHelper`를 통해 자동 decode 방지
1. `RequestMappingHandlerMapping`의 `urlPathHelper`를 override해서 자동 decode 방지

스프링 시큐리티를 사용하면 `org.springframework.security.web.firewall.DefaultHttpFirewall` 클래스에서 다음과 같이 판별하고,

```java
    private boolean containsInvalidUrlEncodedSlash(String uri) {
        if (!this.allowUrlEncodedSlash && uri != null) {
            return uri.contains("%2f") || uri.contains("%2F");
        } else {
            return false;
        }
    }
```

다음과 같이 예외 처리한다.

```java
    public FirewalledRequest getFirewalledRequest(HttpServletRequest request) throws RequestRejectedException {
        FirewalledRequest fwr = new RequestWrapper(request);
        if (this.isNormalized(fwr.getServletPath()) && this.isNormalized(fwr.getPathInfo())) {
            String requestURI = fwr.getRequestURI();
            if (this.containsInvalidUrlEncodedSlash(requestURI)) {
                throw new RequestRejectedException("The requestURI cannot contain encoded slash. Got " + requestURI);
            } else {
                return fwr;
            }
        } else {
            throw new RequestRejectedException("Un-normalized paths are not supported: " + fwr.getServletPath() + (fwr.getPathInfo() != null ? fwr.getPathInfo() : ""));
        }
    }
```

결국 스프링 시큐리티에서 보안을 이유로 막고 있는 것을 굳이 우회해가면서까지 Path Variable로 써야만 하는지 한 번 더 고민해보고 Path Variable 대신 Request Parameter로 보내는 편이 나을 것이다.

