# SpringMVC에서 Collection의 Validation

## 문제

아래와 같이 작성된 Controller 가 있다.
RequestBody로 `MemberDto`의 배열을 전달받아 처리하는 역할을 한다.

```java
@RestController
public class TestController {

    @RequestMapping(value = "/test/members", method = RequestMethod.POST)
    public ResponseEntity<List<MemberDto>> saveMembers(
            @RequestBody @Valid List<MemberDto> memberDtos,
            BindingResult bindingResult
    ) throws BindException {
        if (bindingResult.hasErrors()) {
            throw new BindException(bindingResult);
        }
        
        // save 했다고 치자..
        
        return ResponseEntity.ok(memberDtos);
    }
}
```

`MemberDto`는 다음과 같다.
```java
@Getter
@Setter
@ToString
public class MemberDto {

    @Id
    @NotNull
    private Long id;

    @NotNull
    private String nickname;
}
```

이제 아래와 같이 `nickname`에서 마지막 e를 치다가 바로 위의 3까지 함께 눌러버린 오타가 포함된 json을 body로 하는 HttpReqeust를 날리면 `BindException` 예외로 인한 에러가 발생할까 안할까? 

```javascript
[
  {
    "id": 1,
    "nickname": "아재"
  }, {
    "id": 2,
    "nickname3": "개저씨"    // nickname 이 아니라 nickname3
  }
]
```

## 답

에러는 발생하지 않고, 너무나 순진한 척 아래와 같은 수줍은 결과를 내놓는다.

```javascript
[
  {
    "id": 1,
    "nickname": "아재"
  },
  {
    "id": 2
    // nickname 개저씨 어디 갔니~
  }
]
```
`List<MemberDto>`에 `@Valid`를 붙여줬고, 
`MemberDto`를 보면 `nickname`은 `@NotNull`이므로, 
request body에 `nickname`가 없으면 에러가 나야한다.

따라서 위와 같이 실수로 `nickname` 대신에 `nickname3`이 들어가있다면, request에는 `nickname`이 없으므로 `@NotNull`을 위배하기 때문에 에러가 나야되는거 아닌가?

그런데 왜 에러가 안나지?

## 왜?

에러가 나지 않는 이유는 `@Valid`는 JavaBeans에 적용되는데, List는 JavaBeans가 아니기 때문이다(참고: http://stackoverflow.com/a/35643761).

실제로 `@RequestBody @Valid List<MemberDto>`가 아니라 `@RequestBody @Valid MemberDto`로 테스트 해보면 예상대로 에러가 발생한다.

우리는 파라미터가 List같은 Collection일 때도 에러가 발생하기를 원하는데 어떻게 해야될까?

인터넷에서 찾아보면 아래와 같이 `List<MemberDto>`를 감싸는 JavaBeans 클래스를 만들어서 해결하는 방법이 나오는데, 간단하긴 하지만 왠지 좀 억지스러운 느낌이 없지 않다.

```java
public class MyList {

    @Valid
    List<MemberDto> memberDtos;
    
    ...
}
```

Client가 보낸 데이터는 Custom Validator를 만들어서 검증할 수도 있다. 그런데 보통 Custom Validator를 만들면 DTO에 지정된 `@NotNull` 등을 인식하는 것이 아니라, Custom Validator 내에 자체 작성한 로직으로 Null check를 하고, 에러 메시지도 직접 지정해줘야 하므로 일이 좀 커진다.

## 문제 정의

정리하면 아래와 같은 요구사항을 쉽게 만족시키는 방법을 알고 싶다.

>Collection 형태의 HttpRequest body 데이터를 `@Valid`로 검증하기 위해,

>Custom Validator를 쓰면서도,

>DTO에 지정된 `@NotNull` 등을 그대로 사용하고,

>Validation이 실패하면 실패 이유를 로그에 남기고 싶다.

어떻게 해야할까?

## 이렇게 해보자

Custom Validator를 만들기는 하되 DTO에 지정된 Validation 제약조건을 그대로 준용하려면, 아래와 같이 `Validation.buildDefaultValidatorFactory().getValidator()`와 `SpringValidatorAdapter`를 이용하면 된다.

```java
package homo.efficio.web.validator;

import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;
import org.springframework.validation.beanvalidation.SpringValidatorAdapter;

import javax.validation.Validation;
import java.util.Collection;

@Component
public class CustomCollectionValidator implements Validator {

    private SpringValidatorAdapter validator;

    public CustomCollectionValidator() {
        this.validator = new SpringValidatorAdapter(
                Validation.buildDefaultValidatorFactory().getValidator()
        );
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return Collection.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {

        Collection collection = (Collection) target;

        for (Object object : collection) {
            validator.validate(object, errors);
        }
    }
}

```
Controller에서 이 Validator를 이용해서 검증하는 부분만 추가하면 자연스럽게 해결할 수 있다.

```java
@Autowired
CustomCollectionValidator customCollectionValidator;

@RequestMapping(value = "/test/members", method = RequestMethod.POST)
    public ResponseEntity<List<MemberDto>> saveMembers(
            @RequestBody @Valid List<MemberDto> memberDtos,
            BindingResult bindingResult
    ) throws BindException {
    
        // 이것만 추가 - List를 직접 validate
        customCollectionValidator.validate(memberDtos, bindingResult);

        if (bindingResult.hasErrors()) {
            throw new BindException(bindingResult);
        }
        
        // save 했다고 치자..
        
        return ResponseEntity.ok(memberDtos);
    }
```

다시 HttpRequest를 날려보면, 반갑게도 `bindingResult`에 에러가 담겨지고(에러가 이렇게 반갑다니..), `BindException` 예외가 발생하고 아래와 같이 자세한 오류 내역을 response를 통해 볼 수 있다.

```json
{
  "timestamp": 1470313788279,
  "status": 400,
  "error": "Bad Request",
  "exception": "org.springframework.validation.BindException",
  "errors": [
    {
      "codes": [
        "NotNull.memberDtoList.nickname",
        "NotNull.nickname",
        "NotNull"
      ],
      "arguments": [
        {
          "codes": [
            "memberDtoList.nickname",
            "nickname"
          ],
          "arguments": null,
          "defaultMessage": "nickname",
          "code": "nickname"
        }
      ],
      "defaultMessage": "반드시 값이 있어야 합니다.",
      "objectName": "memberDtoList",
      "field": "nickname",
      "rejectedValue": null,
      "bindingFailure": false,
      "code": "NotNull"
    }
  ],
  "message": "Validation failed for object='memberDtoList'. Error count: 1",
  "path": "/test/members"
}
```
API Client에게 위와 같이 구구절절 알려줄 필요는 없으니 아래와 같이 ExceptionHandler를 만들면, 

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

import javax.servlet.http.HttpServletResponse;
import java.util.Locale;

@ControllerAdvice("homo.efficio.web")
@Order(Ordered.HIGHEST_PRECEDENCE)
public class GlobalExceptionHandler {

    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(BindException.class)
    @ResponseBody
    public ResponseEntity bindException(BindException exception, Locale locale,
                                        HttpServletResponse response) {
        response.setContentType(MediaType.APPLICATION_JSON_UTF8_VALUE);
        logger.error(exception.getFieldErrors().toString());
        return new ResponseEntity(HttpStatus.BAD_REQUEST);
    }
}
```
Client에게는 `400` Bad Request 코드만 반환하고, Error log에 아래와 같이 간단하게 남길 수 있다.

![Imgur](http://i.imgur.com/OTPCxD6.png)

----
## Kotlin 버전

```kotlin
@RestController
@RequestMapping(path = ["/api/zzz"])
class ZzzController(
    private val collectionValidator: CollectionValidator,
) {

    @PutMapping("/{zzzOid}/testers", produces = [MediaType.APPLICATION_JSON_VALUE])
    fun saveTesters(
        @PathVariable("zzzOid") zzzOid: Long,
        @Valid @RequestBody testers: List<TesterIn>,
        bindingResult: BindingResult,
    ): ResponseEntity<List<TesterOut>> {
    
        collectionValidator.validate(testers, bindingResult)

        if (bindingResult.hasErrors()) {
            throw MethodArgumentNotValidException(
                MethodParameter(
                    object{}.javaClass.enclosingMethod,  // saveTesters 메서드
                    1  // validation 대상인 파라미터 testers는 2번째 파라미터이므로 인덱스로는 1
                ),
                bindingResult
            )
        }

        ...
    }
```

---
## `List<@Size(max = 10) String>` + `-Xemit-jvm-type-annotations`

컴파일 옵션 `-Xemit-jvm-type-annotations`을 사용하면 Collection의 bracket 안에 validation annotation을 사용할 수 있다.

build.gradle.kts 파일에 다음과 같이 컴파일 옵션을 추가하면,

```kotlin
// build.gradle.kts
tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs = listOf(
            "-Xjsr305=strict",
            "-Xemit-jvm-type-annotations",  // 여기!!
        )
        jvmTarget = "11"
    }
}
```

아래와 같이 `@Size(max = 10)` 애너테이션을 `List<>` 안에 넣어서, List 안에 들어있는 각 String의 길이가 10자를 초과하면 validation exception이 발생하게 할 수 있다.

Custom Annotation도 동일한 방식으로 `List<>` 안에 지정하면 Custom Validator가 실행된다.

```kotlin
class Xxx(
    val testers: List<@Size(max = 10) String>,  // 여기!!
    val other1: OtherType1,
    val other2: OtherType2,
)

// conroller class
@PostMapping("/xxx")
fun save(@Valid @RequestBody xxx: Xxx): ResponseEntity<Xxx> {
    // ...
}
```

---
## 기타

CustomValidator 에 사용되는 payload 사용법은 https://docs.jboss.org/hibernate/validator/5.0/reference/en-US/html/validator-customconstraints.html 에서 payload 를 검색하면 나온다.

하지만 애너테이션에 애트리뷰트를 추가해서 필요한 값을 주입받을 수 있으므로, payload를 사용해야만 할 일은 아마도 없을 듯.

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
