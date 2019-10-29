# private 메서드와 테스트

테스트 코드를 작성하다보면 테스트 하고 싶은 로직이 직접 테스트가 불가능한 private 메서드 안에 담겨 있어서 답답할 때가 있다.

이럴 때 어떻게 해야할까? PowerMock 같은 도구를 써서 기어코 private 메서드를 직접 테스트 해야할까?

먼저 private 메서드가 뭔지부터 다시 한 번 생각해보자.


## private 메서드란?

보통 '외부에는 공개되지 않아 외부에서 직접 호출할 수 없고 메서드가 속한 객체 내부에서만 호출 가능한 메서드'를 private 메서드라고 한다.

외부에 공개하지 않는 이유는 공개되면 결합도를 높이고 유지보수성을 떨어뜨리며 시간이 갈 수록 시스템의 전체적인 유연성이 크게 저하되기 때문이다.


## private 메서드란 언제 만들어지는가?

신체 검사를 담당하는 클래스가 있다고 하자. 가장 원초적인 코드는 대략 다음과 같을 것이다.

```java
public class BodyChekup {

    public Result getResult(Member member) {
        // 키 측정
        // 몸무게 측정
        // 혈압 측정
        ...
        // 종합해서 반환
    }
}
```

하나의 책임을 완수하는 코드를 만들었다. TDD 였다면 `getResult()`가 완성되기 전에 테스트 코드가 만들어져 있을테고, TDD가 아니라면 지금 테스트 코드를 만드는 게 좋다. 그래서 `getResult()`에 대한 테스트 코드를 만들고 테스트를 통과했다고 하자.

다시 보니 `getResult()` 혼자 너무 많은 일을 한다. 책임을 살짝 나눠보자.

```java
public class BodyChekup {

    public Result getResult(Member member) {
        float height = getHeight(member);  // 키 측정
        // 몸무게 측정
        // 혈압 측정
        ...
        // 종합해서 반환
    }

    private float getHeight(Member member) {
        return 키_측정값;// 키 측정값 반환
    }
}
```

코드가 바뀌었으니 테스트 코드를 돌려 본다. 테스트 코드를 돌리면 `getResult()`가 호출되고 내부적으로 `getHeight()`도 호출된다. 테스트가 성공하면 `getHeight()` 구현이 책임을 완수하는 데 적합하다는 뜻이다.

바로 이 시점이 `getHeight()`라는 private 메서드가 만들어진 시점이다. 지금 일어난 일이 뭐냐면 **public 메서드인 `getResult()` 안에 있던 키 측정이라는 작은 책임을 private 메서드인 `getHeight()` 안으로 재배치**한 것이다. 

그리고 현재 상태에서 **`getHeight()`에 대해서는 별도의 테스트 코드를 작성할 필요가 없다. private 메서드인 `getHeight()`를 호출하는 public 메서드인 `getResult()`에 의해 간접적으로 테스트 되기 때문이다.**

작은 책임의 재배치 뿐 아니라 중복된 코드를 하나로 모을 때도 private 메서드를 만들게 되는데, 이 역시도 중복된 코드가 결국은 하나의 작은 책임이라는 점을 생각해보면 위에서 설명한 것처럼 재배치라고 볼 수 있다.


## private 메서드의 변경

이 당시 길이를 재는 도구가 30cm 자 밖에 없었다고 하자. 그럼 `getHeight()`는 정확하게는 다음과 같을 것이다.

```java
    private float getHeight(Member member) {
        return 30cm_자로_잰_키_측정값;// 키 측정값 반환
    }
```

그런데 이제 1m 짜리 줄자가 있다고 하자. 그럼 `getHeight()`는 다음과 같이 바꿀 수 있다.

```java
    private float getHeight(Member member) {
        return 1m_자로_잰_키_측정값;// 키 측정값 반환
    }
```

코드가 바뀌었으니 이미 작성되어 있던 테스트 코드를 돌려본다. 테스트 코드를 돌리면 `getResult()`가 호출되고 내부적으로 `getHeight()`도 호출된다. 테스트가 성공하면 `getHeight()` 구현이 책임을 완수하는 데 적합하다는 뜻이다.

즉, **`getHeight()`이 바뀌었더라도, `getHeight()`에 대한 직접적인 테스트 코드를 추가하지 않아도 `getHeight()`의 구현이 책임을 완수하는 데 적합하다는 것을 확신할 수 있다.**

결국 **private 메서드란 객체의 책임을 완수하는 데 필요한 구체적인 로직의 일부를 담당하며 객체 내에 적절히 재배치된 메서드**를 말한다. 그리고 **객체가 책임을 완수하는지 테스트 하는 과정에서 private 메서드 구현의 적절 여부가 간접적으로 테스트 된다.**


## 진짜와 가짜

그런데 외부에 공개되지 않도록 `private`이 붙어있다고 해서 다 같은 private 메서드가 아니다.

어떤 놈은 지금까지 설명한 것처럼 잘 설계된 로직의 일부를 담당하며 적절히 재배치 된 진짜 private 메서드고,  
어떤 놈은 잘못 설계되어 잘못 배치된 가짜 private 메서드다.

**가짜 private 메서드는 실은 다른 클래스의 public 메서드인데 자기 스스로도 자기 정체를 모른다.**  
그래서 개발자가 찾아서 본래의 모습인 다른 클래스의 public 메서드로 돌려놔야 한다.


## 가짜 찾아내기

그렇다면 가짜를 어떻게 가려낼 수 있을까?

앞에서 설명한 것처럼 private 일 때는 간접적으로 테스트 될 수 밖에 없지만, public 이 되면 직접 테스트가 가능하다.  
**public으로 만들어서라도 직접 테스트 하고 싶은 마음이 들 정도로 중요한 `책임`을 가지고 있다면 그놈은 가짜 private 일 가능성이 높다.**

여기서 강조하고 싶은 것은 직접 테스트 하고 싶을 정도로,
- `구현`이 복잡한 놈이 가짜 private이 아니라,  
- `책임`이 무거운 놈이 가짜 private  

라는 점이다.


## 그래서 어쩌라고?

>private 메서드를 테스트하고 싶은 마음이 들면,  
>- PowerMock 같은 도구로 직접 테스트를 할 게 아니라,  
>- 구현이 복잡한 건지, 책임이 무거운 건지 생각해보고,  
>- 구현이 복잡한 거라면 public 메서드를 통한 기존의 간접 테스트로 만족하고,  
>- 책임이 무거운 거라면 별도의 클래스로 분리하고 public 메서드로 만든 후에 public 메서드로서 직접 테스트 해야 한다.


## 더 읽어보기

- https://enterprisecraftsmanship.com/posts/unit-testing-private-methods/  
- 이규원 님 블로그: https://justhackem.wordpress.com/2017/09/29/should-private-methods-be-tested/

