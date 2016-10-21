# DTO와 Bean Validation

## DTO(Data Transfer Object)란?
DTO는 클라이언트 요청에 포함된 데이터를 담아 서버 측에 전달하고, 또 서버 측의 결과 데이터를 담아 클라이언트에 전달하는 역할을 한다. 이름 그대로의 설명만으로는 뭔가 허전하고 존재 의의를 알기가 좀 어려운데, 아래의 그림을 보면 좀 더 와닿는 점이 있다.

![티스토리](http://cfile25.uf.tistory.com/image/210EBB3C523001D029D7DD)

출처: http://netframework.tistory.com/entry/16-Model-%EA%B8%B0%EC%88%A0-%EC%A0%95%EB%A6%AC-%EB%B0%8F-%EB%B9%84%EA%B5%90

>DTO는 Domain Object에서 
>- View 관련 로직을 제거해서 Domain Object가 도메인 로직에 대한 책임만 가지도록 한다. 

## Validation

클라이언트가 보낸 정보는 서버 단으로 넘어오면서 DTO에 담겨지게 된다. 이 단계에서 DTO에 반드시 넘겨져야 할 필수 정보가 누락되어 넘어온다면 오류가 발생하는데, 이 오류를 클라이언트의 잘못으로 처리할 것이냐 서버의 잘못으로 할 것이냐의 갈림길에 놓이게 된다.

기능적으로야 갈림길에 놓이지만, 논리적으로는 클라이언트의 잘못이라고 보는 맞다. 넘겨 줘야할 필수 정보를 넘겨주지 않은 것은 클라이언트의 잘못이기 때문이다. 실제로 QA에서도 이런 경우 `4XX` 에러를 반환하면 정상 처리로 판별하고, `5XX` 에러를 반환하면 부적절한 처리로 판별하는 것이 일반적이다. 이처럼 클라이언트가 넘겨준 정보의 유효성을 판별하는 절차가 Bean Validation이다. 

Validation을 통과하지 못하면 클라이언트 쪽의 오류인 `4XX`에러로 처리해야하는데, 여러가지 Validation 조건 중에서 필수 정보의 누락 여부는 `@NotNull`을 이용해서 감지할 수 있다.

아래와 같이 필수 정보에 `@NotNull` 애노테이션을 지정하면, 클라이언트가 넘긴 정보를 컨트롤러 단에서 DTO에 바인딩할 때 Validation과정에서 `4XX`로 처리할 수 있는 교두보를 확보하게 된다.

```java
...
import javax.validation.constraints.NotNull;
...
public class ProductDto {

	@NotNull
	private String productCode;
    
	...
}
```
필수 정보인데도 `@NotNull`을 명시하지 않으면 어떻게 될까? 아마도 Validation을 통과하게 되고, 잠재적인 오류는 서비스든 DAO 또는 Repository든 어디론가 계속 전파되어 되어 결국 서버 측의 잘못인 `5XX` 오류로 비극적인 결말을 맞게 된다.

`@NotNull` 지정으로 교두보를 확보 했으니 꼼꼼한 마무리 작업이 남았다. 편의상 SpringMVC 기준의 코드를 예로 하면, 컨트롤러의 해당 메서드의 파라미터인 DTO에 아래와 같이 `@Valid`를 지정해주면, 필수 정보의 누락 등 Validation을 통과하지 못할 때 `BindException`을 던지고, 결과적으로 `4XX` 오류로 처리되게 할 수 있다.

```java
...
@Controller
public class ProductController {
	...
	@RequestMapping(value = "/whatever", method = RequestMethod.POST)
	public void saveProduct(@Valid ProductDto productDto,
    						BindingResult bindingResult) throws BindException {
        if (bindingResult.hasErrors()) {
            throw new BindException(bindingResult);
        }
        ...
	  }
    ...
}
```

물론 `@NotNull` 외에도 다양한 Validation 조건이 존재한다.(http://beanvalidation.org/ 참고)

## 정리

>1. DTO는 Domain Object를 순수하게 도메인 로직만을 담당하는 객체로 살아갈 수 있게 해준다.
>
>1. 클라이언트가 보낸 정보에 잘못이 있을 경우, Validation에 실패해서 `4XX` 오류로 처리되도록 `javax.validation` 패키지를 이용해서 DTO와 컨트롤러에 적절한 애노테이션을 지정해야 한다.

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>의 저작물인 이 저작물은(는)

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
