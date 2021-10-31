# throw Exception 주의

잘 작성된 코드라면 `throw Exception(...)` 같은 코드를 볼 일이 없겠지만, 늘 잘 작성된 코드만 바라보며 사는 행운은 흔치 않다.

이 코드는 바람직하지 않을 뿐만 아니라 스프링 환경에서 사용될 때는 꽤 심각한 내재적 오류를 가지고 있다.

>**Checked Exception이 던져지면 롤백이 되지 않는다.**
>
>참고: https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction-declarative-rolling-back

특히 Checked Exception이 던져지더라도 메서드에 `throws`를 명시하지 않아도 컴파일 에러가 나지 않는 코틀린에서는 더 편하게 오용될 수 있으니 주의가 필요하다.

물론 기본 설정에서만 롤백이 되지 않는 것이고 `rollbackFor`를 지정하면 Check Exception이 던져질 때도 롤백이 된다.

## 왜?

근데 왜 Checked Exception 일 때는 롤백되지 않게 했을까?

찾아보니 다음과 같은 내용이 있었다.

>Although EJB container default behavior automatically rolls back the transaction on a system exception (usually a runtime exception), EJB CMT does not roll back the transaction automatically on an application exception (that is, a checked exception other than java.rmi.RemoteException). While the Spring default behavior for declarative transaction management follows EJB convention (roll back is automatic only on unchecked exceptions), it is often useful to customize this behavior.
>
>참고: https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction-declarative

요는 EJB의 Container Managed Transactions 의 롤백 정책을 스프링도 그대로 준용한 것으로 보인다. 그럼 EJB의 Container Managed Transactions 은 왜 그런 롤백 정책을 선택했는가를 알아봐야겠지만, 알아보는 노력대비 큰 실익은 없을 듯해서 이쯤에서 줄인다.
