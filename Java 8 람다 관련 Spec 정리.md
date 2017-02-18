공사중입니다 6^^조금 보완해서 더 좋게 정리해보겠습니다~


# Java 8 람다 관련 Spec 정리

## method signature

https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.4.2

메서드 시그너처는 다음의 2가지로 형성된다.

- 메서드의 이름
- 파라미터의 타입


따라서 다음은 메서드 시그너처와 아무 상관이 없다.

- modifier
- 반환 타입
- throws 절

아래 메서드의 method signature는 `calculateAnswer(double, int, double, double)`이다.
    
```java        
public double calculateAnswer(double wingSpan, int numberOfEngines, double length, double grossTons) {
    //do the calculation here
}
```

## method descriptor

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3

메서드 디스크립터는 다음의 2가지로 형성된다.

- parameter descriptor
- return descriptor

따라서 다음은 메서드 디스크립터와 아무 상관이 없다.

- modifier
- 메서드 이름
- throws 절
- instance 메서드냐 class 메서드냐

아래 메서드의 method descriptor는 `(IDLjava/lang/Thread;)Ljava/lang/Object;`이다. 정식은 아니지만 알아볼 수 있게 풀어쓰면 `(integer, double, Thread) Object` 정도로 할 수 있겠다.

```java
Object m(int i, double d, Thread t) {...}
```

## type of a method

https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.2

메서드의 타입은 다음의 4가지로 형성된다.

- 타입 파라미터
- 인자 타입
- 리턴 타입
- throws 절

## function type of a functional interface

https://docs.oracle.com/javase/specs/jls/se8/html/jls-9.html#jls-9.9

함수형 인터페이스의 추상 메서드를 override 하는데 사용되는 메서드의 타입을 의미하며, 구성 요소는 `type of a method`과 같다.

## type of a lambda expression

https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3

람다식의 타입을 의미하며, 구성 요소는 `type of a method`과 같다.

## type of a method reference expression

https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.2

메서드 레퍼런스의 타입을 의미하며, 구성 요소는 `type of a method`과 같다.

## ground target type derived from functional interface

람다 또는 메서드 레퍼런스와 함수형 인터페이스가 합동적(congruent)인지, 즉 람다나 메서드 레퍼런스가 어떤 함수형 인터페이스의 자리에 들어가서 사용될 수 있는지 판별하는데 기준이 되는 타입으로, 함수형 인터페이스에서 유도되며, 구성 요소는 `type of a method`과 같다.

groud target type을 유도하는 방식이 스펙에 구구절절 나와있다.

## implicit lambda expression

파라미터의 타입이 명시되지 않은 람다식

## explicit lambda expression

파라미터의 타입이 명시된 람다식

## 람다와 메서드 레퍼런스의 차이

둘 모두 타입이 맞으면 함수형 인터페이스가 사용될 자리에 들어갈 수 있다는 점은 같다.

하지만 메서드 레퍼런스는 타입 파라미터가 있는 제네릭 메서드도 사용될 수 있는데, 람다는 타입 파라미터를 선언할 수 있는 문법이 지원되지 않으므로 제네릭 메서드를 사용할 수 없다.

```java
interface ListFactory {
    <T> List<T> make();
}

ListFactory lf  = ArrayList::new;

// 메서드 레퍼런스는 이게 가능하다.
List<String> ls = lf.make();
List<Number> ln = lf.make();
``` 

## void 호환성

https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3 의 마지막

반환 타입이 void인 abstract 메서드 하나를 가지는 functional interface에는,
반환 타입이 void가 아닌 함수 바디를 가지는 람다가 할당될 수 있다.

```java
// list.add()는 boolean을 반환하지만,
// 반환 타입이 void인 accept(T t) 메서드를 가진 Consumer<T>에 할당 가능
Consumer<String> b = s -> list.add(s);   
```

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
