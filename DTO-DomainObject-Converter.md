# DTO, Domain Object, Converter

https://github.com/HomoEfficio/dev-tips/blob/master/DTO와%20Bean%20Validation.md 에 이어지는 글이라고 봐도 좋을 것 같다.


## DTO의 필요성

DTO(Data Transfer Object)가 왜 필요한지 또는 어떤 역할을 하는지는 앞의 링크 글에 잘 나와 있는데 요약하면 다음과 같다.

>DTO = Domain Information + View Information

DTO가 View Information을 포함하고 View와 맞닿아 의사소통하는 덕분에, Domain 객체는 View에 구애받지 않고, 순수하게 도메인 로직만을 담당하는 객체로 살아갈 수 있다.

하지만 View Information이 별로 필요하지 않다면, 굳이 DTO을 일부러 따로 만들 필요 없이 Domain 객체만 사용해도 괜찮다. 어차피 View는 DTO인지 Domain 객체인지 알지도 못하고 알 필요도 없다. 그저 View를 그리는데 필요한 정보만 모두 담겨있다면 무엇이든 상관없다.

Domain 객체만으로 모두 커버할 수 있다면 가장 단순하고 우아한 해법일 것이다. 하지만 View는 기대만큼 단순하지 않기 마련이다. 그래서 View Information이 필요하고 결국 DTO가 필요한 경우가 많다.


## DTO - Domain 객체 변환 위치

DTO가 있어야 하는 상황이라면, 프론트엔드의 View에는 결국 DTO가 전달되고, 백엔드의 데이터 저장소에는 결국 Domain 객체가 전달된다. 따라서 중간의 어딘가에서는 DTO와 Domain 객체가 서로 변환되는 지점이 있어야 한다. 그게 어디일까? 


## 컨트롤러 vs 서비스

처음에는 컨트롤러냐 서비스냐가 고민이었다. 컨트롤러 메서드의 인자로 DTO가 사용되는 점을 감안하면 컨트롤러에 변환 로직을 두는 게 나을 것 같다. 하지만, DTO를 Domain 객체로 만들 때는 Repository를 통한 조회가 필요할 때가 종종 있고, 이 때는 컨트롤러에서 처리할 수가 없다. 그래서 서비스에서 DTO - Domain 객체 변환을 담당하게 했다.

좀 지나서는 별도로 Converter로 빼는 게 좋을 것 같아서 별도의 Converter 계층을 두고 변환 로직을 모두 Converter에 담고, 각 Converter를 서비스에 주입해서 서비스가 DTO - Domain 객체간 변환을 Converter에게 위임해서 처리하게 했다.

더 지나고 보니 방향마다 다르게 보는 것이 낫다는 생각이 들었다.


## Domain 객체 -> DTO -> View 방향

Domain 객체 -> DTO -> View 방향의 흐름에서는 필요한 모든 Domain 객체를 서비스에서 Repository를 통해 얻어올 수 있고, DTO에 전달할 데이터를 Domain 객체가 모두 가지고 있으므로, Domain 객체를 파라미터로 받는 Converter의 메서드에서 View 관련 데이터(ex SelectBox에 들어갈 특정 데이터 목록 등)를 추가하고 DTO로 변환 하는데 아무 어려움이 없다. 

따라서 Converter에서는 서비스로부터 호출될 때 Domain 객체만 인자로 넘겨받는다면 독립적으로 변환 처리가 가능하다.


## View -> DTO -> Domain 객체 방향

하지만 View -> DTO -> Domain 객체 방향의 흐름에서는 View에서 전달받은 정보만으로 Domain 객체를 구성할 수가 없다. 쉽게 말해 View에서는 서버에 ID만 전달하기도 하는데, ID만으로는 Domain 객체를 구성할 수 없으니 ID 외의 정보를 Repository를 통해 조회한 후에나 Domain 객체를 구성할 수 있다. 

따라서 Converter에서 독립적으로 처리가 불가능하고 Repository 계층에 의존하게 된다. Converter가 Repository에 직접 접근하게 하면서까지 DTO -> Domain 객체 변환을 Converter에서 처리해야하는 걸까? 아니라고 생각한다. 그러느니 DTO -> Domain 객체 변환 책임을 서비스에게 넘기는 것이 낫다고 본다.

## 정리

정리하면 다음과 같다.

>Domain 객체 -> DTO 의 변환은 Converter에서 담당하고, Converter를 서비스에 주입해서 처리
>
>DTO -> Domain 객체의 변환은 서비스의 private 메서드에서 처리 
