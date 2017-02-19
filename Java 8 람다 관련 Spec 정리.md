# Java 8 람다 관련 Spec 정리

자바는 아직 함수가 독립적으로 어떤 값에 할당되거나, 어떤 함수의 인자로 사용되거나, 반환값으로 사용될 수 없다. 

대신에 Java8에서부터는 추상 메서드를 한 개만 가지고 있는 함수형 인터페이스라는 것을 언어의 기능으로 추가해서 할당, 인자 또는 반환에 사용하고, 함수형 인터페이스의 자리에 람다식이나 메서드 레퍼런스를 사용할 수 있게 해서, 간접적이지만 실질적으로 람다식이나 메서드 레퍼런스를 할당, 인자, 반환에 사용될 수 있게 했다.

어떤 A가 사용되는 곳에 A와는 다른 어떤 B가 사용되려면, B는 A에 대해 어떤 형태로든 호환성이 있어야 한다. 같은 맥락으로 람다나 메서드 레퍼런스가 함수형 인터페이스가 들어갈 자리에 들어가서 대신 사용되려면, 일종의 타입 비교 같은 호환성 체크가 필요하다. 

그렇다면 무엇을 기준으로 타입을 정해서 비교할 수 있을까?

자바 가상머신 스펙(JVMS)에는 `메서드 시그너처`라는 용어가 있고, 자바 언어 스펙(JLS)에는 `메서드 시그너처(signature)`, `메서드 디스크립터(descriptor)`, `메서드 타입`, `함수 타입`이라는, 뭔가 타입을 정할 목적으로 사용되는 것 같은 용어가 여러가지 등장하는데, 일단 결론적으로 **호환성 검사를 하는데 필요한 기본 정보는 메서드 타입과 함수 타입이다**.

메서드 시그너처나 메서드 디스크립터는 나중에 [여기](https://github.com/HomoEfficio/dev-tips/blob/master/%EC%9A%95%20%EB%82%98%EC%98%A4%EB%8A%94%20%EC%9E%90%EB%B0%94%20%EC%8A%A4%ED%8E%99%20-%20%EB%A9%94%EC%84%9C%EB%93%9C%20%EC%8B%9C%EA%B7%B8%EB%84%88%EC%B2%98.md)를 참고하는 걸로 하고 지금은 일단 넘어가자. 먼저 람다와 메서드 레퍼런스의 타입 기준이 되는 `메서드 타입`부터 알아보자. 

## 메서드 타입

`메서드 타입`, 즉, 메서드의 타입은 [JLS 8.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.2)에 기술되어 있다.

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.2
>
>For a method, an ordered 4-tuple consisting of:
>
>- **type parameters**: the declarations of any type parameters of the method member.
>
>- **argument types**: a list of the types of the arguments to the method member.
>
>- **return type**: the return type of the method member.
>
>- **throws clause**: exception types declared in the throws clause of the method member.
>
>(대략) 메서드의 타입은 **타입 파라미터**, **인자 타입**, **반환 타입**, **예외 타입**으로 구성된다.

일단 쉽게 생각해서 **'타입 파라미터, 인자 타입, 반환 타입, 예외 타입, 이렇게 4가지의 정보가 메서드의 타입을 결정한다.'**라고 이해하자.

## 함수 타입

메서드의 타입을 `메서드 타입`으로 규정할 수 있다면, 함수형 인터페이스의 어떤 타입과 비교할 수 있을 것이다. 그 어떤 타입이 무엇일까? 이에 대한 답이 바로 `함수 타입(Function type)`, 구체적으로는 함수형 인터페이스의 함수 타입(Function type of a functional interface)이다.

`함수 타입`은 [JLS 9.9](https://docs.oracle.com/javase/specs/jls/se8/html/jls-9.html#jls-9.9)에 기술되어 있다.

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-9.html#jls-9.9
>
>The function type of a functional interface I is a method type (§8.2) that can be used to override (§8.4.8) the abstract method(s) of I.
>
>(대략) 함수형 인터페이스 I의 함수 타입은 I의 추상 메서드를 override 하는데 사용되는 메서드 타입을 의미한다.
>
>The function type of I consists of the following:
>
>- Type parameters, formal parameters, and return type:
>
>    (중략)
>
>- throws clause:
>
>    (중략)

뭔가 구구절절 설명이 더 있는데, 쉽게 요약하면 `함수 타입`의 구성 요소도 `메서드 타입`의 구성 요소와 같이 다음의 4가지다.

- 타입 파라미터
- 인자 타입
- 반환 타입
- 예외 타입

함수 타입의 기준까지는 비교적 간단한데, 실제 함수 타입을 구하는 과정은 여러 케이스에 대해 구체적이고 복잡하고 방대한 내용이 펼쳐진다. 여기서 더 나아가면 추론(Inference) 및 제네릭(Generic)에 대한 전반적인 지식이 필요하므로 자세한 설명은.. 여러가지 이유로.. 생략하기로..(하아.. 이렇게 맘이 편할 수가..)

함수 타입을 구하는 자세한 절차와 방법이 궁금하다면 [여기](https://docs.oracle.com/javase/specs/jls/se8/html/jls-9.html#jls-9.9)를 참고하고, 여기에서는 스펙에 나와있는 `함수 타입` 예제 하나만 구경해보고 넘어가자.

```java
interface G1 {
    <E extends Exception> Object m() throws E;
}
interface G2 {
    <F extends Exception> String m() throws Exception;
}
interface G extends G1, G2 {}
```
일 때, `G`의 함수 타입은 다음과 같다.

```java
<F extends Exception> ()->String throws F
```

이처럼 함수 타입은 실제로 스펙에도 `<F extends Exception> ()->String throws F`와 같은 형식으로 기술하고 있는데, 

`<타입 파라미터>`, `(인자 타입)`, `-> 반환 타입`, `throws 예외 타입`, 이렇게 생김새만 봐도 4가지 구성 요소로 이루어져있다는 걸 알 수 있다.


## 람다식 타입

람다식의 타입(Type of a lambda expression)은 [JLS 15.27.3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3)에 기술되어 있다.

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3
>
>A lambda expression is compatible in an assignment context, invocation context, or casting context with a target type T if T is a functional interface type (§9.8) and the expression is congruent with the function type of the ground target type derived from T.
>
>(대략) 어떤 람다식이, 함수형 인터페이스 T에서 유도된 ground target type의 함수 타입과 합동(congruent)이면, 그 람다식은 할당, 호출, 캐스팅에 대해 함수형 인터페이스 T와 호환된다.

대략 써도 알아먹기 쉽지 않은데, 일단 욕 해주고 싶은 부분은 람다식의 타입에 대한 정의나 람다식의 타입은 어떤 요소로 구성되는 설명도 없이, 합동이면 콜~ 이라고 얘기하고는 합동 조건에 대해 설명을 이어간다는 점이다.

암튼 완전히 정확하지는 않더라도 쉬운 이해를 위해 단순화하면, **람다식은 언어적으로 타입 파라미터를 사용할 수 없으므로, 결국 인자 타입과 반환 타입, 예외 타입으로 함수형 인터페이스와의 호환성을 검사**할 수 있다.

## 메서드 레퍼런스 타입

메서드 레퍼런스의 타입(Type of a method reference expression)은 [JLS 15.13.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.2)에 기술되어 있다.

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.2
>
>A method reference expression is compatible in an assignment context, invocation context, or casting context with a target type T if T is a functional interface type (§9.8) and the expression is congruent with the function type of the ground target type derived from T.
>
>(대략) 어떤 메서드 레퍼런스식이, 함수형 인터페이스 T에 유도된 ground target type의 함수 타입과 합동이면, 그 메서드 레퍼런스식은 할당, 호출, 캐스팅에 대해 함수형 인터페이스 T와 호환된다.

람다식의 타입과 거의 비슷한 설명이다. 메서드 레퍼런스의 정의나 구성 요소에 대한 내용은 역시나 없다. 메서드 레퍼런스가 결국은 메서드를 지칭하는 것이므로, 메서드 레퍼런스의 타입은 결국 `메서드 타입`과 같다고 봐도 무방할 것 같다. 따라서 **메서드 레퍼런스식은 타입 파라미터, 인자 타입, 반환 타입, 예외 타입으로 구성**된다고 할 수 있겠다.

## 호환성 비교

자 이제 람다와 메서드 레퍼런스의 `메서드 타입`과 함수형 인터페이스의 `함수 타입`이 호환성 비교의 기준이라는 건 알게 되었다. 이제 실질적인 호환성 비교, 그러니까 앞에서 합동(congruent) 여부를 검사하는 과정을 알아보자. congruent와 함께 ground target type이라는 용어도 나왔는데, 이것부터 먼저 알아보자.

### ground target type derived from functional interface

람다 또는 메서드 레퍼런스와 함수형 인터페이스가 합동적(congruent)인지, 즉 람다나 메서드 레퍼런스가 어떤 함수형 인터페이스의 자리에 들어가서 사용될 수 있는지 판별하는데 기준이 되는 인터페이스의 타입을 말한다.

좀 쉽게 얘기하면, 

- `interface I`나 `interface I<Integer>`와 같이 와일드카드 파라미터를 사용하지 않는 함수형 인터페이스의 ground target type은 `I`이고, 
- `I<? extends P>`와 같이 와일드카드를 사용하는 파라미터화 된 함수형 인터페이스의 ground target type은, 스펙에 정해진 방법으로 `?`를 예를 들어 `S`로 구체화 했을 때 `I<S>`가 된다.
- 람다나 메서드 레퍼런스의 사용을 위한 실질적인 타입 비교는, 이 `I`나 `I<S>`의 함수 타입이 사용된다.

간단한 설명을 위해 일단 이제부터는 함수형 인터페이스에서 유도된 ground target type은 그냥 함수형 인터페이스의 타입과 같다고 간주하고, 'ground target type의 함수 타입'을 그냥 '함수 타입'이라고 칭하기로 하자(이렇게해도 틀리는 것은 아니다. 두 가지 중 간단한 것 하나만 취하고 나머지는 간단한 설명을 위해 배제했을 뿐이다).

이제 람다식과 메서드 레퍼런스와 함수 타입(정확하게는 ground target type의 함수 타입이지만)의 합동(즉, 호환됨)에 대해 차례대로 살펴보자.

### 람다식 타입과 함수 타입의 합동

람다식 타입과 함수 타입의 합동은 [JLS 15.27.3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3)에 설명되어 있다. 분량 상 원문 생략하고 간단하게 합동 조건을 요약하면,

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3
>
>람다식 타입과 함수 타입이 합동이려면 다음의 모든 조건을 만족해야 한다.
>
>- 함수 타입에 타입 파라미터가 없어야 한다.
>
>- 람다의 인자의 수는 함수 타입의 인자 타입의 수와 같아야 한다.
>
>- 인자 타입이 명시된 람다식의 인자 타입은 함수 타입의 파라미터 타입과 같아야 한다.
>
>- 람다식의 인자 타입이 함수 타입의 파라미터 타입과 같다고 간주되면,
>
>    - 함수 타입의 결과값이 void 이면, 람다식의 body는 문장식이거나 void-호환 블럭이어야 한다.  
>    - 함수 타입의 결과값이 void가 아닌 R이라면,
>         - 람다 body는 할당에 대해 R과 호환되는 식이거나,
>         - 람다 body는 값 호환이고, 각 결과식이 할당에 대해 R과 호환되어야 한다.

이 또한 크게 틀리지 않는 한에서 쉽게 풀어보면

>람다식 A가 함수형 인터페이스 B 대신 사용되려면, 다음 조건을 모두 만족해야 한다.
>
>- B의 함수 타입에 타입 파라미터가 사용되고 있지 않아야 한다.
>- A의 인자 수와 B의 함수 타입의 인자 수가 같아야 한다.
>- A의 인자 타입과 B의 함수 타입의 인자 타입이 같아야 한다.
>    - A가 인자 타입이 명시되지 않은 람다라면, 인자의 타입 추론 결과가 B의 함수 타입의 인자 타입과 같아야 한다.
>- A의 반환 타입은 B의 함수 타입의 반환 타입에 할당될 수 있어야 한다.

글로만 보면 별로 와닿지 않으니 코드로 요점만 짚어보면 다음과 같다.

```java
// Integer가 Object를 상속하고 있으므로
// Object obj = new Integer(3); 은 가능하지만,
// 아래와 같이 람다식의 인자 파라미터가 명시된 경우
// 함수 타입의 인자 파라미터와 람다식의 인자 파라미터 타입은 할당 가능이 아니라 일치 해야만 한다.
Consumer<Object> consumer1 = (Integer i) -> System.out.println(i);  // 컴파일 에러(Incompatible parameter type)

// 람다식의 인자 파라미터가 명시되지 않으면 추론에 의해 아래와 같은 람다 사용이 가능하다.
Consumer<Object> consumer2 = (i) -> System.out.println(i);  // 이건 가능(cf가 Object로 추론됨)

// 람다식의 반환 타입 Integer은 Object에 할당가능하므로 아래와 같은 람다 사용이 가능하다.
Callable<Object> callable1 = () -> new Integer(1);

// 함수 타입의 반환 타입이 void인 Runnable에 statement expression이 아닌 단순한 값 3은 사용 불가
Runnable runnable1 = () -> 3;  // 컴파일 에러(Bad return type)

// Runnable의 함수 타입의 반환 타입은 void지만, statement expression이 해당하는 인스턴스 생성식은 사용 가능
Runnable runnable2 = () -> new Integer(3);

// 함수 타입에 타입 파라미터가 있는 경우 람다를 쓸 수 없다.
interface Lister {
    <T> List<T> makeList();
}
Lister lister = () -> ...어쩌라고... // 람다식은 언어 차원에서 타입 파라미터가 지원되지 않는다.

// 함수 타입에 throws가 있는 경우
interface WithThrows {
    Integer makeTrouble() throws IOException;
}
// 함수 타입에 throws IOException 이 있으므로
// 아래와 같이 body에서 IOException을 던지는 람다 사용 가능
WithThrows withThrows1 = () -> {
    if (1 == 1)
        throw new IOException();
    return new Integer(3);
};
// 함수 타입의 throws IOException을 상속한 EOFException을 던지는 람다도 사용 가능
WithThrows withThrows2 = () -> {
    if (1 == 1)
        throw new EOFException();
    return new Integer(1);
};
// 함수 타입의 throws IOException을 상속하지 않은 예외를 던지는 람다는 사용 불가
WithThrows withThrows3 = () -> {
    if (1 == 1)
        throw new InterruptedException();  // 컴파일 에러(Unhandled exception)
    return new Integer(1);
};
// body에서 Unchecked Exception을 던지는 람다는 함수 타입의 예외 타입과 관계 없이 사용 가능 
WithThrows withThrows4 = () -> {
    if (1 == 1)
        throw new RuntimeException();
    return new Integer(1);
};
// body에서 Unchecked Exception을 던지는 람다는 함수 타입에 throws 가 없더라도 사용 가능
Runnable runnable3 = () -> {
    if (1 != 1)
        throw new RuntimeException();
    System.out.println("Unchecked Exception in a lambda body is OK");
};
```

### 메서드 레퍼런스 타입과 함수 타입의 합동

메서드 레퍼런스 타입과 함수 타입의 합동은 [JLS 15.13.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.2)에 설명되어 있다. 분량 상 원문 생략하고 간단하게 합동 조건을 요약하면,

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.2
>
>메서드 레퍼런스 타입과 함수 타입이 합동이려면 다음의 모든 조건을 만족해야 한다.
>
>- 함수 타입이 컴파일 시점에 메서드 레퍼런스가 지칭하는 메서드의 선언부를 식별할 수 있어야 한다.
>
>- 다음 둘 중의 하나가 참이어야 한다.
>
>    - 함수 타입의 결과값이 void 여야 한다.
> 
>    - 함수 타입의 결과값이 R이라면, 함수 타입이 컴파일 시점에 식별한 메서드 레퍼런스의 선언부에 명시된 결과값의 타입이 할당에 대해 R과 호환되어야 한다.

역시 크게 틀리지 않는 선에서 쉽게 풀어보면

>메서드 레퍼런스 A가 함수형 인터페이스 B 대신 사용되려면, 다음 조건을 모두 만족해야 한다.
>
>- B의 함수 타입이 컴파일 시점에 메서드 레퍼런스가 가리키는 메서드의 선언부를 식별할 수 있어야 한다.
>
>- 함수 타입의 반환값이 void 이거나, 
>- 반환값이 void가 아니라면, 컴파일 시점에 메서드 레퍼런스가 가리키는 메서드 선언부에 명시된 반환 타입이 함수 타입의 반환 타입에 할당될 수 있어야 한다.

글로만 보면 별로 와닿지 않으니 역시 코드로 요점만 짚어보면 다음과 같다.

```java
// 함수 타입의 반환 타입이 void이고, 메서드 레퍼런스의 반환 타입도 void
Runnable runnable4 = System.out::println;

// 함수 타입의 반환 타입이 void이면, 반환 타입이 void가 아닌 메서드 레퍼런스도 사용 가능
Integer integer2 = new Integer(2);
Runnable runnable6 = integer2::doubleValue;

// 함수 타입에 타입 파라미터가 있는 경우에도 메서드 레퍼런스 사용 가능
interface Lister {
    <T> List<T> makeList();
}
Lister lister1 = ArrayList::new;

// 함수 타입의 반환 타입이 void가 아니고, 반환 타입이 함수 타입의 반환 타입에 할당 불가능한 레퍼런스는 사용 불가
Integer integer1 = new Integer(1);
Callable<Integer> callable2 = integer1::doubleValue;  // 컴파일 에러(Bad return type)
// 함수 타입의 반환 타입이 void가 아니고, 반환 타입이 함수 타입의 반환 타입에 할당 가능한 메서드 레퍼런스는 가능
Callable<Object> callable3 = integer1::doubleValue;
```


## 람다와 메서드 레퍼런스의 차이

둘 모두 타입이 맞으면 함수형 인터페이스가 사용될 자리에 들어갈 수 있다는 점은 같다.

하지만 메서드 레퍼런스는 타입 파라미터가 있는 제네릭 메서드도 사용될 수 있는데, 람다는 타입 파라미터를 선언할 수 있는 문법이 지원되지 않으므로 제네릭 메서드를 사용할 수 없다.

좀 추상적이니 구체적인 코드로 보면 다음과 같은 차이가 있다.

```java
interface ListFactory {
    <T> List<T> make();
}

// 람다는 이게 안된다
ListFactory lf1 = () -> new ArrayList();  // target method is generic
        
// 메서드 레퍼런스는 이게 가능하다.
ListFactory lf2  = ArrayList::new;


List<String> ls = lf2.make();
List<Number> ln = lf2.make();
``` 

## 람다의 void 호환성

람다의 void 호환성은 [JLS 15.27.3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3)의 마지막 부분에 나와 있다.

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27.3
>
>람다의 body가 statement expression이면, 그 람다의 반환 타입이 (무엇이든 상관없이) void인 함수 타입과 호환 된다.

statement expression의 예는 다음과 같다.

>https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-StatementExpression
>
>- 할당
>
>- 전위덧셈(++a)
>
>- 전위뺄셈(--a)
>
>- 후위덧셈(a++)
>
>- 후위뺄셈(a--)
>
>- 메서드 호출
>
>- 클래스 인스턴스 생성식(new 등..)

코드로 살펴보자면, 다음과 같은 일이 가능하다는 소리다.

```java
// list.add()는 boolean을 반환하지만 메서드 호출이므로,
// 반환 타입이 void인 accept(T t) 메서드를 가진 Consumer<T>에 할당 가능
Consumer<String> b = s -> list.add(s);   
```

# 정리

>- **메서드의 타입**은 **타입 파라미터, 인자 타입, 반환 타입, 예외 타입**으로 구성된다.
>
>- **함수 타입**은 함수형 인터페이스 I의 함수 타입은 I의 추상 메서드를 override 하는데 사용되는 메서드 타입을 의미
>
>- 따라서 **람다나 메서드 레퍼런스의 메서드 타입과 함수형 인터페이스의 함수 타입의 호환성**을 검사하면, 어떤 람다나 메서드 레퍼런스가 어떤 함수형 인터페이스 대신 사용될 수 있는지 판단할 수 있다.
>
>- 람다식은 메서드 타입의 4가지 구성 요소 중 문법적으로 타입 파라미터를 지정할 수 없게 되어 있으므로, 인자 타입과 반환 타입, 예외 타입이 함수 타입의 인자 타입과 반환 타입, 예외 타입과 호환, 즉, 개수가 같고 타입이 호환되어야 한다.
>
>    - 타입 파라미터가 있는 함수 타입은 람다식과 호환되지 않는다.
>    
>    - 인자 타입이 명시된 람다는 인자 타입과 함수 타입의 인자 타입이 **동일**해야 한다.
>    
>    - 람다식의 반환 타입과 예외 타입은 함수 타입의 반환 타입과 예외 타입에 각각 할당 가능하면 동일하지 않아도 호환된다.
>    
>    - 람다식의 body가 할당, 전위덧셈, 전위뺄셈, 전위덧셈, 전위뺄셈, 메서드호출, 클래스인스턴스생성, 이렇게 7개 중의 하나에 해당한다면 void 호환성이 적용된다.
>
>- 메서드 레퍼런스는 메서드 타입의 4가지 구성 요소 모두가 함수 타입의 구성 요소 4가지 모두와 호환되어야 한다.
>
>    - 함수 타입이 컴파일 시점에 메서드 레퍼런스가 가리키는 메서드의 선언부를 식별할 수 있어야 한다.
>
>    - 함수 타입의 반환값이 void 이거나,
>     
>    - 반환값이 void가 아니라면, 컴파일 시점에 메서드 레퍼런스가 가리키는 메서드 선언부에 명시된 반환 타입이 함수 타입의 반환 타입에 할당될 수 있어야 한다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
