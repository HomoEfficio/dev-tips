# catch (Exception e) vs catch (Throwable t)

## 둘의 차이

>https://docs.oracle.com/javase/specs/jls/se15/html/jls-11.html#jls-11.1.1
>
>Exception is the superclass of all the exceptions from which ordinary programs may wish to recover.  
>Error is the superclass of all the exceptions from which ordinary programs are not ordinarily expected to recover.
>
>`Exception`은 일반적인 프로그램에서 복구되기를 바라는(복구할 수 있는) 예외 상황을 의미하며,  
>`Error`는 일반적인 프로그램에서 복구될 것이라고 바라지 않는(복구할 수 없는) 예외 상황을 의미한다.

그리고 `Throwable`은 `Exception`과 `Error`의 부모 클래스다.

따라서 `catch (Exception e)`는 복구 가능한 예외를 잡아서 복구하겠다는 표현이다. 합리적이다.  
그래서 학습용 예제에도 자주 나오고, 눈에 익다보니 무심결에 쓰다 보면 `catch (Exception e)`를 쓰게 된다.

그런데 `catch (Throwable t)`은 복구 불가능한 `Error`도 포함해서 잡겠다는 표현이다. 복구도 불가능한데 잡아서 뭐하지? 합리적이지 않은데?  
그래서 잘 보이지 않는다. `Throwable`이 뭔지 알려주는 예제에서나 볼 수 있을 뿐이고, 그래서 잘 사용되지 않는다.

## catch (Throwable t)를 써야 할 때

`Error`는 checked exception이 아니므로, catch 하지 않아도 된다. 발생하면 복구할 수 없으므로 그대로 프로그램이 오류로 종료된다.

그런데 왜 잡아야 할까?

간단하다, 단순히 프로그램 종료로 끝나면 되는 게 아니라,  
**프로그램 상태를 '오류'로 어딘가에 남겨야 하는 상황이라면 잡아야 한다.**  
잡아서 상태를 '오류'로 바꾸고 남겨야 한다.

그럼 끝?

아니다. 잊지 말고 다시 throw 해줘야 된다. 안 그러면 프로그램이 종료되지 않고 의도와 다르게 정상 실행 경로를 따르게 된다.

```java
try {
  ...
} catch (Throwable t) {
  // '오류'로 상태 변경 후 어딘가에 남기고

  // unchecked exception으로 감싸서 다시 던진다.
  throw new RuntimeException(t);
}
```

## 범위를 좁히자

`catch (Throwable t)`를 써야할 자리에 `catch (Exception e)`를 쓰지 말자는 거지(물론 그 반대로도 쓰지 말자), 둘을 애용하자는 건 아니다.  
범위가 너무 넓기 때문이다.

특히 `catch (Throwable t)`는 자바 언어로 표현할 수 있는 모든 예외 상황을 포괄하므로 예외 처리를 이것 하나만으로 처리하는 건 대부분 적합하지 않다.

필요한 범위만큼만 잡아서 사용하는 것이 좋다.

