# 커스텀 애노테이션을 활용한 Bean Validation

필드 하나만 검증하는 건 간단한데, 여러 필드 사이에도 제약 조건이 있을 때는 어떻게 검증하는 것이 좋을까? 예를 들어, 일정 기간을 대상으로 검색하는 시나리오가 있다고 치자.

검색어와 조건을 받아들이는 DTO과 다음과 같다면 '`endDate`는 `startDate`로부터 90일 이내'라는 하는 제약 조건을 어떻게 구현할까?

```java
package homo.efficio.springboot.scratchpad.validation;

import lombok.Getter;
import lombok.Setter;
import org.springframework.format.annotation.DateTimeFormat;

import javax.validation.constraints.NotNull;
import java.time.LocalDate;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-04-04
 */
@Getter
@Setter
public class SearchDto {

    @NotNull(message = "keyword가 명시되어야 합니다.")
    private String keyword;

    @NotNull(message = "기간 시작 일자가 명시되어야 합니다.")
    @DateTimeFormat(pattern = "yyyyMMdd")
    private LocalDate startDate;

    @NotNull(message = "기간 종료 일자가 명시되어야 합니다.")
    @DateTimeFormat(pattern = "yyyyMMdd")
    private LocalDate endDate;

    public SearchDto() {
    }

    public SearchDto(String keyword, LocalDate startDate, LocalDate endDate) {
        this.keyword = keyword;
        this.startDate = startDate;
        this.endDate = endDate;
    }

    @Override
    public String toString() {
        return "SearchDto{" +
                "keyword='" + keyword + '\'' +
                ", startDate='" + startDate + '\'' +
                ", endDate='" + endDate + '\'' +
                '}';
    }
}
```

여기에 검색 대상 기간을 90일로 한정하고 싶다. 어떻게 하는 것이 좋을까? 

여러 방법이 있겠지만 Java의 Bean Validation 프레임워크만으로 해결해보자. Java EE 8부터는 기본으로 포함되어 있고, 없다면 MvnRepository에서 검색해서 build.gradle이나 pom.xml에 다음과 같은 의존 관계를 추가해야 한다.

```
compile group: 'javax.validation', name: 'validation-api', version: '2.0.1.Final'
```

준비는 끝났고 실제로 구현해보자. 과정은 대략 다음과 같다.

1. validation 메서드를 가진 껍데기 인터페이스 `ValidPeriod` 생성
1. `SearchDTO`를 `ValidPeriod` 인터페이스를 구현하도록 변경하고, `ValidPeriod` 인터페이스 메서드를 override하는 구현 메서드 A에 실제 validation 로직 구현
1. `ConstraintValidator`를 구현하는 커스텀 `PeriodValidator` 클래스 생성하고 `isValid()` 메서드에서 A를 호출하도록 구현
1. `PeriodValidator`를 제약 조건으로 사용하는 커스텀 애노테이션 인터페이스인 `LimitSearchPeriod` 생성
1. `SearchDTO`에 `@LimitSearchPeriod` 애노테이션 추가

구체적 코드로 알아보자.

## 1. validation 메서드를 가진 껍데기 인터페이스 `ValidPeriod` 생성

```java
package homo.efficio.springboot.scratchpad.validation;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-04-04
 */
public interface ValidPeriod {

    boolean isValidPeriod();
}
```

## 2. `SearchDTO`를 `ValidPeriod` 인터페이스를 구현하도록 변경하고, `ValidPeriod` 인터페이스 메서드를 override하는 구현 메서드 A에 실제 validation 로직 구현

기간 유효성 체크 로직을 추가한다. 이 때 `startDate`와 `endDate`에 `null` 체크 로직을 추가해주는 것이 중요하다. 

스프링은 클라이언트에서 넘어온 파라미터 값을 DTO 객체에 바인딩할 때 `20181322` 와 같이 유효하지 않은 값이 넘어오면, 예외를 바로 던지지 않고 일단 DTO의 해당 속성에 `null`을 할당하고, 발생한 예외를 모두 모아서 담아두고 나중에 한 번에 처리를 한다. 

그렇지 않고 오류마다 하나하나 처리하면 사용자가 파라미터1, 파라미터2, 파라미터3에 모두 오류가 있는 값을 입력했을 때, 먼저 파라미터1 오류만 보여주고, 사용자가 파라미터1 값을 바로잡으면 파라미터2 오류를 보여주고, 사용자가 파라미터2 값을 바로잡으면 파라미터3 오류를 보여주는데, 이렇게 하면 여러 번 수정 작업을 거치므로 비효율적이다. 이런 방식 보다는 오류를 모두 담아두고 한 번에 모든 오류를 보여주면 사용자가 한 번에 모든 오류를 수정해서 보낼 수 있으므로 더 효율적이다. 이 과정은 `AbstractPropertyAccessor`의 `setPropertyValues()` 메서드에서 확인할 수 있다.

암튼 그래서 `startDate`나 `endDate`에 바인딩 될 값에 오류가 있으면, Custom Validator가 호출될 때 두 속성에 `null`이 들어와 있으므로, `null` 처리를 해주지 않으면 의도하지 않은 `NullPointerException`이 발생한다.


```java
package homo.efficio.springboot.scratchpad.validation;

import lombok.Getter;
import lombok.Setter;
import org.springframework.format.annotation.DateTimeFormat;

import javax.validation.constraints.NotNull;
import java.time.LocalDate;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-04-04
 */
@Getter
@Setter
public class SearchDto implements ValidPeriod {

    @NotNull(message = "keyword가 명시되어야 합니다.")
    private String keyword;

    @NotNull(message = "기간 시작 일자가 명시되어야 합니다.")
    @DateTimeFormat(pattern = "yyyyMMdd")
    private LocalDate startDate;

    @NotNull(message = "기간 종료 일자가 명시되어야 합니다.")
    @DateTimeFormat(pattern = "yyyyMMdd")
    private LocalDate endDate;

    public SearchDto() {
    }

    public SearchDto(String keyword, LocalDate startDate, LocalDate endDate) {
        this.keyword = keyword;
        this.startDate = startDate;
        this.endDate = endDate;
    }

    // 여기!!!
    @Override
    public boolean isValidPeriod() {
        // 개별 속성에 유효하지 않은 값이 들어오면 null 이 할당된다.
        if (startDate == null || endDate == null) return false;

        return endDate.isBefore(startDate.plusMonths(3))
                && (endDate.isEqual(startDate) || endDate.isAfter(startDate));
    }

    @Override
    public String toString() {
        return "SearchDto{" +
                "keyword='" + keyword + '\'' +
                ", startDate='" + startDate + '\'' +
                ", endDate='" + endDate + '\'' +
                '}';
    }
}
```

## 3. `ConstraintValidator`를 구현하는 커스텀 `PeriodValidator` 클래스 생성하고 `isValid()` 메서드에서 A를 호출하도록 구현

```java
package homo.efficio.springboot.scratchpad.validation;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-04-04
 */
public class PeriodValidator implements ConstraintValidator<LimitSearchPeriod, ValidPeriod> {

    @Override
    public void initialize(LimitSearchPeriod constraintAnnotation) {

    }

    @Override
    public boolean isValid(ValidPeriod validPeriod, ConstraintValidatorContext context) {
        return validPeriod.isValidPeriod();  // 여기!!!
    }
}
```

## 4. `PeriodValidator`를 제약 조건으로 사용하는 커스텀 애노테이션 인터페이스인 `LimitSearchPeriod` 생성

```java
package homo.efficio.springboot.scratchpad.validation;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-04-04
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {PeriodValidator.class})
public @interface LimitSearchPeriod {

    String message() default "기간이 유효하지 않습니다.";  // 애노테이션 지정 시 validatino rule에 맞는 메시지 지정 가능!!!

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

## 5. `SearchDTO`에 `@LimitSearchPeriod` 애노테이션 추가

```java
@Getter
@Setter
@LimitSearchPeriod(message = "조회 기간은 90일 이내여야 합니다.")  // validation rule에 맞는 메시지 지정!!!
public class SearchDto implements ValidPeriod {

    // 앞과 동일
}
```

## 스프링 MVC Controller 사례

```java
@RequestMapping("/period")
public ResponseEntity<String> searchWithinPeriod(@Valid SearchDto dto, BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        // 바인딩 오류 처리
    }
    // 어쩌구 그 잘난 비즈니스 로직을 담고 있는 서비스 호출
}
```

이제 다음과 같이 90일이 넘는 기간으로 요청을 날리면

```
http://localhost:8080/bean-validation/period?keyword=omw&startDate=20171201&endDate=20180403
```

다음과 같이 `BindException`이 발생한다.

```
2018-04-04 15:28:51.382  WARN 75195 --- [nio-8080-exec-1] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved exception caused by Handler execution: org.springframework.validation.BindException: org.springframework.validation.BeanPropertyBindingResult: 1 errors
Error in object 'searchDto': codes [LimitSearchPeriod.searchDto,LimitSearchPeriod]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [searchDto.,]; arguments []; default message []]; default message [조회 기간은 90일 이내여야 합니다.]
```

## BindException 처리

앞의 컨트롤러에서 다음과 같이 BindException 처리하는 부분이 있는데,

```java
@RequestMapping("/period")
public ResponseEntity<String> searchWithinPeriod(@Valid SearchDto dto, BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        // 바인딩 오류 처리
    }
  // 이하 생략
```

다음과 같은 Helper를 만들어서,

```java
public class BindingResultHelper {

    public static void throwCustomInvalidParameterException(BindingResult bindingResult) {

        // 개별 필드 오류 처리(ex: startDate에 20181322와 같은 값이 들어왔을 때)
        List<FieldError> fieldErrors = bindingResult.getFieldErrors();
        if (!fieldErrors.isEmpty()) {
            String message = fieldErrors.stream()
                    .map(e -> String.format("Property value [%s] of '%s' is invalid.", e.getRejectedValue(), e.getField()))
                    .collect(Collectors.joining(" && "));
            throw new CustomInvalidParameterException(message);
        }

        // 개별 필드 외의 오류 처리(ex: startDate와 endDate의 차이가 90일을 넘어갈 때)
        List<ObjectError> objectErrors = bindingResult.getAllErrors();
        if (!objectErrors.isEmpty()) {
            String message = objectErrors.stream()
                    .map(e -> String.format("Error in object '%s': %s", e.getObjectName(), e.getDefaultMessage()))
                    .collect(Collectors.joining(" && "));
            throw new CustomInvalidParameterException(message);
        }

        throw new CustomInvalidParameterException("입력값을 확인해 주십시오.");
    }
}
```

아래와 같이 쓰는 것도 괜찮다.

```java
@RequestMapping("/period")
public ResponseEntity<String> searchWithinPeriod(@Valid SearchDto dto, BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        BindingResultHelper.throwCustomInvalidParameterException(bindingResult);
    }
  // 이하 생략
```

