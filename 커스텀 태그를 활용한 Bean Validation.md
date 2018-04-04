# 커스텀 태그를 활용한 Bean Validation

필드 하나만 검증하는 건 간단한데, 여러 필드 사이에도 제약 조건이 있을 떄는 어떻게 검증하는 것이 좋을까? 예를 들어, 일정 기간을 대상으로 검색하는 시나리오가 있다고 치자.

검색어와 조건을 받아들이는 DTO과 다음과 같다면 `endDate`와 `startDate`는 30일 이내여야 하는 제약 조건을 어떻게 구현할까?

```java
package homo.efficio.springboot.scratchpad.validation;

import lombok.Getter;
import lombok.Setter;

import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;

/**
 * @author homo.efficio@gmail.com
 * created on 2018-04-04
 */
@Getter
@Setter
@MostSearchPeriod
public class SearchDto {

    @NotNull(message = "keyword가 명시되어야 합니다.")
    private String keyword;  // 검색어

    @NotNull(message = "기간 시작 일자가 명시되어야 합니다.")
    @Pattern(regexp = "\\d{4}[0-1]\\d[0-3]\\d", message = "startDate는 yyyymmdd 형식이어야 합니다.")
    private String startDate;  // 대상 기간 시작일

    @NotNull(message = "기간 종료 일자가 명시되어야 합니다.")
    @Pattern(regexp = "\\d{4}[0-1]\\d[0-3]\\d", message = "endDate는 yyyymmdd 형식이어야 합니다.")
    private String endDate;  // 대상 기간 종료일

    public SearchDto() {
    }

    public SearchDto(String keyword, String startDate, String endDate) {
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

날짜 regexp가 좀 허술하지만 그냥 예시니까 일단 넘어가자 ㅋ

여기에 검색 대상 기간을 30일로 한정하고 싶다. 어떻게 하는 것이 좋을까? 

여러 방법이 있겠지만 Java의 Bean Validation 프레임워크만으로 해결해보자. Java EE 8부터는 기본으로 포함되어 있고, 없다면 MvnRepository에서 검색해서 build.gradle이나 pom.xml에 의존 관계를 추가해야 한다.

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

```java
package homo.efficio.springboot.scratchpad.validation;

import lombok.Getter;
import lombok.Setter;

import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;

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
    @Pattern(regexp = "\\d{4}[0-1]\\d[0-3]\\d", message = "startDate는 yyyymmdd 형식이어야 합니다.")
    private String startDate;

    @NotNull(message = "기간 종료 일자가 명시되어야 합니다.")
    @Pattern(regexp = "\\d{4}[0-1]\\d[0-3]\\d", message = "endDate는 yyyymmdd 형식이어야 합니다.")
    private String endDate;

    public SearchDto() {
    }

    public SearchDto(String keyword, String startDate, String endDate) {
        this.keyword = keyword;
        this.startDate = startDate;
        this.endDate = endDate;
    }


    // 여기!!!
    @Override
    public boolean isValidPeriod() {
        long startDate = Long.valueOf(this.startDate);
        long endDate = Long.valueOf(this.endDate);

        return (endDate - startDate) <= 30 && (endDate - startDate) >= 0;
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

    String message() default "기간이 유효하지 않습니다.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

## 5. `SearchDTO`에 `@LimitSearchPeriod` 애노테이션 추가

```java
@Getter
@Setter
@LimitSearchPeriod
public class SearchDto implements ValidPeriod {

    // 앞과 동일
}
```

## 스프링 MVC Controller에서는 이렇게

```java
@RequestMapping("/period")
public ResponseEntity<String> searchWithinPeriod(@ModelAttribute @Valid SearchDto dto) {
    // 어쩌구 그 잘난 비즈니스 로직을 담고 있는 서비스 호출
}
```
