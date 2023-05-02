# findXXX() vs getXXX()

find 와 get 을 구분하는 기준이 뭘까?

https://softwareengineering.stackexchange.com/questions/182113/how-and-why-to-decide-between-naming-methods-with-get-and-find-prefixes 여기에 비교적 여러 의견들이 있다.

대체로 다음과 같다.

>find
>1. 호출 결과가 null 일 수도 있을 때
>2. DB 조회처럼 적지 않은 시간이 소요될 때
>
>get
>1. 호출 결과 항상 정상적인 값이 있어야하고, 없으면 예외로 처리해야할 때
>2. 인메모리 조회처럼 굉장히 짧은 시간이 소요될 때

의견 중에 'DB 조회라서 find 를 사용했는데, 이후 DB 대신 해시테이블을 사용하게 됐다면 api도 get 으로 바꿔야되니, 구현 방식에 종속되는 2번 기준은 적당하지 않다'는 얘기도 있는데 타당한 지적이라고 본다. 그래서 2번은 적절한 기준이 아닌 것 같다.

그런데 nullable 필드인 zzz 의 값을 반환하는 api 는 그럼 getZzz() 로 하면 안 되고, findZzz() 로 해야되나? 하고 생각해보면 그건 기존 관례와 너무 크게 출돌되는 것 같다.

조금 덧붙이면 다음과 같이 짧게 정리할 수 있겠다.

>- find
>    - 호출 결과가 null 일 수도 있을 때
>- get
>    - getter
>    - 호출 결과가 반드시 존재하며, 없다면 예외로 처리해야할 때

대표적으로 JpaRepository 에 있는 findById 는 null 이나 Optional 을 반환한다. 따라서 find 가 적절하다.

이 정의에 따르면 findById 를 호출하는 곳에서 늘 아래와 같이 예외 처리 코드를 추가해야된다.

```kotlin
val dbXxx = xxxRepository.findById(id) ?: throw YyyException()
```

귀찮기도 하고 중복이기도 해서 `?: throw YyyException()` 이 부분을 공통화한 커스텀 메서드를 사용하기도 했는데,  
이제부턴 이런 커스텀 메서드는 위 기준에 맞게 get 을 사용해야겠다.
