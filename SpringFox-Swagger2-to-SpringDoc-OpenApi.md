# SpringFox-Swagger2 => SpringDoc-OpenApi

- Replace in Files 에서 `Cc`, `.*` 선택
- 각 단계 종료 후 `Optimize Imports` 수행


## Authorization 헤더 입력란 제거

- springdoc-openapi 에서는 OpenApiConfig 에서 Authorization 정보를 입력하도록 설정하므로, 개별 메서드에 있는 Authorization 헤더 입력 애너테이션은 제거한다.

```


<공백>
```


## `@Api` -> `@Tag`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import io.swagger.annotations.Api;

import io.swagger.v3.oas.annotations.tags.Tag;
```

```
@Api\(tags = \{"(.*?)"\}, produces = "(.*?)"

@Tag(name = "$1", description = "$2"
```

```
@Api\(tags = \{"(.*?)"\}

@Tag(name = "$1"
```

```
@Api\(tags

@Tag(name
```


## `@ApiModel` -> `@Schema`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화


```
import io.swagger.annotations.ApiModel;

import io.swagger.v3.oas.annotations.media.Schema;
```

```
@ApiModel\(value

@Schema(description
```


## `@ApiOperation` -> `@Operation`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import io.swagger.annotations.ApiOperation;

import io.swagger.v3.oas.annotations.Operation;
```

```
@ApiOperation\(\s*value = "(.*?)",\s*notes

@Operation\(summary = "$1",
      description
```

```
@ApiOperation\(value

@Operation(summary
```

```
notes

description
```


## 잘못 사용된 `@ApiModelProperty` -> `@ApiParam`

- `@ModelAttribute` + `*Request` 인 조건에 맞으면 필드에 `@ApiModelProperty`가 아니라 `@ApiParam`을 사용해야 함
  - interfaces 디렉터리에서 Find in Files > `@ModelAttribute \w+Request`로 검색해서 나오는 결과 미리보기에서 `*Request` 클래스 CTRL+B 클릭해서 `@ApiModelProperty`로 작성된 클래스에 대해서는 클래스 파일별로 다음 작업을 수행해서 `@ApiModelProperty` -> `@ApiParam` 으로 변경

```
import io.swagger.annotations.ApiModelProperty;

import io.swagger.annotations.ApiParam;
```

```
@ApiModelProperty

@ApiParam
```

  - ShopContentsRequest
  - ContentsRequestByCategory

- `*Param` 이고 파라미터로 사용되는 클래스
  - interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화 > File mask를 `*Param.java` 로 하고 아래 내용 수행
  - template, background, gesture 쪽 Param 클래스들

```
import io.swagger.annotations.ApiModelProperty;

import io.swagger.annotations.ApiParam;
```

```
@ApiModelProperty

@ApiParam
```



## 잘못 사용된 `@ApiParam` -> `@ApiModelProperty`

- 파라미터에만 사용돼야 하지만 요청본문에 잘못 사용된 사례 수동 정정
  - interfaces 디렉터리에서 Find in Files > File mask 를 `*Request.java`로 지정하고 `@ApiParam`으로 검색해서 나오는 클래스 CTRL+B 클릭해서 사용처 확인해서 `@RequestBody` 붙어 있으면 `@ApiParam`이 잘못 사용된 사례임
- 잘못 사용된 클래스 파일에서 다음 작업 수행
  ```
  import io.swagger.annotations.ApiParam;

  import io.swagger.annotations.ApiModelProperty;
  ```

  ```
  @ApiParam

  @ApiModelProperty
  ```

  - BoothsIdsRequest
  - FaceCodeIdsRequest



## `@RequestBody` 앞에 잘못 붙어 있는 `@ApiParam` 제거

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화 > File mask `*`로 하고 아래 수행
```
@ApiParam\(value = "(.*?)", type = "string"\)
(.*)@RequestBody

@io.swagger.v3.oas.annotations.parameters.RequestBody(description = "$1"\)
$2@RequestBody
```



## `@ApiParam` -> `@Parameter`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import io.swagger.annotations.ApiParam;

import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Schema;
```

```

\s*allowableValues = "android, ios"

 allowableValues = "android, ios"
```

```
^(?!.*@Request)(?!.*example = )(.*?)defaultValue = "(.*?)"

$1example = "$2"
```

```
(.*)@ApiParam\((.*)(, example = ".*"), (defaultValue = ".*")(, .*)

$1@ApiParam($2$3$5
```

```
(.*)@ApiParam\((.*)(, example = ".*"), (defaultValue = ".*")\)

$1@ApiParam($2$3)
```


```
@ApiParam\((.*), type = "string"(.*)

@ApiParam($1$2
```

```
@ApiParam\((.*), (allowableValues = ".*")(, .*")

@ApiParam($1$3, $2
```


```
@ApiParam\(([^)]*?)value\s*=\s*"([^"]*)"([^)]*)\)

@Parameter($1description = "$2"$3)
```

```
@Parameter\((.*), (allowableValues = ".*")

@Parameter($1, schema = @Schema($2)
```

- interfaces 디렉터리에서 Find in Files > `allowableValues`로 검색해서 `@Schema` 밖에 있는 `allowableValues` 를 `@Schema` 안으로 수동 조정
  - `schema = @Schema(allowableValues = "...")`로 변경
  - `@ApiModelProperty` 안에 있는 allowableValues 는 그대로 둔다
  - ContentsRequestByCategory

- interfaces 에서 Find in Files 로 `@ApiParam` 검색해서 안 나오면 ok


## `@ApiIgnore` -> `@Parameter(hidden = true)`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import springfox.documentation.annotations.ApiIgnore;

import io.swagger.v3.oas.annotations.Parameter;
```

```
@ApiIgnore

@Parameter(hidden = true)
```


## `@ApiImplicitParam` -> `@Parameter`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import io.swagger.annotations.ApiImplicitParam;

import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.enums.ParameterIn;
```

```
@ApiImplicitParam\(name = "(.*?)", value = (.*?)

@Parameter(name = "$1", description = $2
```

```
paramType = "header", dataTypeClass = String.class

in = ParameterIn.HEADER
```

## `@ApiImplicitParams` -> `@Parameters`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import io.swagger.annotations.ApiImplicitParams;

import io.swagger.v3.oas.annotations.Parameters;
```

```
@ApiImplicitParams

@Parameters
```


## `@ApiModelProperty` -> `@Schema`

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화


```
import io.swagger.annotations.ApiModelProperty;

import io.swagger.v3.oas.annotations.media.Schema;
```

```
@ApiModelProperty\(value = "(.*?)", required = true

@Schema(description = "$1", requiredMode = Schema.RequiredMode.REQUIRED
```

```
@ApiModelProperty\(required = true, value = "(.*?)"\)

@Schema(description = "$1", requiredMode = Schema.RequiredMode.REQUIRED)
```

```
@ApiModelProperty\(example = "(.*?)"\)

@Schema(example = "$1")
```

```
@ApiModelProperty\(value = "(.*?)", example = "(.*?)"\)

@Schema(description = "$1", example = "$2")
```

```
@ApiModelProperty\(value = "(.*?)"\)

@Schema(description = "$1")
```

```
@ApiModelProperty\(value = "

@Schema(description = "
```

```
@ApiModelProperty\("

@Schema(description = "
```


## application, domain 계층에 있는 `@ApiModelProperty` -> `@Schema`

- application, domain 디렉터리 모두 선택 후 Replace in Files

```
import io.swagger.annotations.ApiModelProperty;

import io.swagger.v3.oas.annotations.media.Schema;
```

```
@ApiModelProperty\(value = "(.*?)", example = "(.*?)"\)

@Schema(description = "$1", example = "$2")
```

```
@ApiModelProperty\(value

@Schema(description
```


## enum 에 붙은 `allowableValues` 삭제

- interfaces 디렉터리에서 Find in Files > `allowableValues` 로 검색
- enum 타입에는 붙은 `allowableValues` 및 `schema = @Schema` 내부에 있는 `allowables` 아예 삭제(항목이 UI 상에 중복 표시될 수 있음)
  - GiftableContentPageRequest > sortType
  - PagingItemsOfCategoriesRequest > cashType
  - CountLogRequest > type


## String 에 붙은 `allowableValues` 배열로 변경

- interfaces 디렉터리에서 Find in Files > `allowableValues` 로 검색
- String 타입에는 다음과 같이 수정
  - `allowableValues = "ZEM, COIN, NA"` -> `allowableValues = {"ZEM, COIN, NA"}`


## `UserAgent`에 `@Parameter(hidden = true)` 추가

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
^([ \t]*)UserAgent\s+userAgent([,\)])

$1@Parameter(hidden = true) UserAgent userAgent$2
```

- maven 에서 compile 해서 `Parameter` import 안 돼 있으면 import 추가
  - NoticeController


## `*Param` 클래스 선언에 `@ParameterObject` 추가

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
(public class [A-Za-z0-9]+Param.*)

@ParameterObject
$1
```

- 변경된 파일들 대상으로

```
import lombok.Data;

import lombok.Data;
import org.springdoc.api.annotations.ParameterObject;
```


## `@ModelAttribute`에 `@ParameterObject` 추가

- interfaces 디렉터리에서 Replace in Files, 대소문자 구별, 정규표현식 활성화

```
import org.springframework.web.bind.annotation.ModelAttribute;

import org.springframework.web.bind.annotation.ModelAttribute;
import org.springdoc.api.annotations.ParameterObject;
```

```
@ModelAttribute

@ModelAttribute @ParameterObject
```


## `*Request` 중에서 `@RequestBody`나 `@ParameterObject` 가 안 붙어 있는 것에 `@ParameterObject` 추가

- interfaces 디렉터리에서 Find in Files > File mask: `*Controller.java` 지정하고 `\b(?<![.@])(?![a-z])(?!HttpServletRequest\b)(\w+Request)\b`로 검색 후
- 검색 결과에서 `@RequestBody`나 `@ParameterObject` 가 안 붙어 있는 곳에 수동으로  `@ParameterObject` 추가
  - BoothController 의 boothsPostFormType()의 BoothsRequest

## springdoc-openapi 설정

- 커밋 내용 확인
  - pom.xml 에 springdoc-openapi 추가
  - springdoc-openapi 컴파일 에러 해결을 위해 apache commons-lang3 3.17 적용
  - springdoc swagger-ui 정렬 등 yml 설정
  - 파라미터를 기존 swagger2와 동일하게 header, path, query 순 및 그룹 내에서 파라미터 이름순으로 표시
  - 스키마 표시 순서를 기존 swagger2와 동일하게 알파벳순으로 표시
  - OpenApiConfig 적용
  - pom.xml 에서 springfox-swagger2 제거 및 guava-32.1.3-jre 추가
    ```
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>32.1.3-jre</version>
    </dependency>
    ```

---

# 결과 확인

## `@Parameter`

- `@Parameter`로 지정된 것은 데이터 타입에 맞는 기능이 렌더링돼야 한다.
- `schema = @Schema`가 지정된 것은 제대로 렌더링 되지 않을 수 있으므로 가급적 사용하지 말아야 한다.

### boolean

- dropdown list 로 렌러링 돼야 한다.
- `schema = @Schema`를 사용하지 않은 케이스
  - TemplateV2Controller > GET /templates (PointCursorPageQueryParam) > isNonPinnedListIncludedInPreviousPage
- `schema = @Schema`를 사용한 케이스
  - Contents V2 > GET /contents/shop/official/items (GiftableContentRequest) > giftable

### string enum

- 데이터 타입은 String 이지만 `allowableValues`를 지정하면 dropdown list 로 렌더링 돼야 한다.
- Contents V2 > /contents/shop/new (ContentsRequestByCategory) > rootCategory

### enum

- dropdown list 로 렌러링 돼야 한다.
- `schema = @Schema`를 사용하지 않은 케이스
  - TemplateV2Controller > GET /templates (PointCursorPageQueryParam) > sortType
- `schema = @Schema`를 사용한 케이스
  - Category Contents > /contents/categories/items (PagingItemsOfCategoriesRequest) > cashType, direction

### `List<String>`

- `Add string item` 버튼이 렌더링 돼야 한다.
- Category Contents > /contents/categories/items (PagingItemsOfCategoriesRequest) > categories

## `@Schema`

- Request Body 나 Response Body 에서 'Schema' 클릭 시 description 이 제대로 나와야 한다.

### enum 

- Contents V2 > POST /contents/creator (ContentsRequest) > RequestBody > Schema 에서 sort, direction 의 선택 가능한 값이 제대로 나와야 한다.

### `List<String>`

- Contents V2 > POST /contents/creator (ContentsRequest) > RequestBody > Schema 에서 hasItems 의 description 이 제대로 나와야 한다.

### `List<CashType>`

- Contents V2 > POST /contents/creator (ContentsRequest) > RequestBody > Schema 에서 creditType 이 배열로 표시돼야 한다.

### `Map<>`

- Content Log Collector > /v1/log/count (CountLogRequest) > RequestBody > Schema 에서 data



---
# OpenAPI/Swagger 관련 향후 작성 시 준수 사항

## 용어 정리

- 파라미터: 개별 변수를 통해서든 객체를 통해서든 쿼리 스트링으로 전달받은 파라미터.
- 모델: 파라미터 외에 요청 본문, 응답 본문 및 그 안에 중첩되어 사용되는 객체

## 계층

- OpenAPI/Swagger 애너테이션은 컨트롤러 계층에만 붙이며, 애플리케이션이나 도메인 계층에는 붙이지 않는다.

## 파라미터로 사용되는 객체

- 클래스의
  - 선언부에 `@ParameterObject`를 붙인다
  - 필드에 `@Parameter`를 붙인다

## 모델로 사용되는 객체

- 요청 본문 클래스의 설명이 필요하다면 클래스 선언부에 `@io.swagger.v3.oas.annotations.parameters.RequestBody(description = "설명")`를 붙인다
- 필드에 `@Schema`를 붙인다
  - `@Schema`가 아닌 `@ApiParam` 이나 `@Parameter`를 붙이면 Swagger UI 상에서 example 값이 json으로 렌더링되지 않는다

## enum

- 파라미터로 사용되는 객체의 enum 필드에는 `@Parameter`를 붙이고, 모델로 사용되는 객체의 enum 필드에는 `@Schema`를 붙이는 것은 동일


## `List<String>` 파라미터

- `type = "string"`를 지정하지 않는다

## 기본값

- Swaggger UI에 기본값을 지정하려면 example을 사용한다
- example 항목이 사실상 기본값으로 동작하고, example 이 없으면 `schema = @Schema`의 defaultValue가 기본값으로 동작한다.
- 다음과 같이 지정돼 있으면, dropdown list에 fashion 이 기본적으로 선택된다.
  ```
  @Parameter(description = "1차 카테고리",
      example = "fashion",
      schema = @Schema(defaultValue = "room", allowableValues = {"fashion", "room", "creators", "face", "space"})
  )
  private String rootCategory;
  ```
- 다음과 같이 지정돼 있으면, ko가 기본값으로 미리 입력된다
  ```
  @Parameter(description = "language",
      required = true,
      example = "ko",
      schema = @Schema(defaultValue = "jp")
  )
  private String language = "en";
  ```
- `@Schema` 안의 defaultValue 는 example 이 없을 때만 효력을 발휘한다.


# 기타 권고

- 파라미터 객체는 `*Param`
- 요청 본문 객체는 `*Request`
- 파라미터 필드에 String 대신에 enum 사용
