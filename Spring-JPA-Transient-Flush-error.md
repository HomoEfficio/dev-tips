# Spring JPA Transient Flush

저장 중 아래와 같은 에러가 날 때가 있다.

>save the transient instance before flushing - 참조하는객체 -> 참조되는객체

이 때는 참조하는객체 쪽에 `cascade`를 붙여주면 된다.

`cascade`는 대부분 `@OneToMany`에 붙이는 게 통상적이지만, 드물게 `@ManyToOne` 쪽에 붙여야 할 때도 있다.

1:N 양방향 관계에서 양쪽 모두에 붙여도 문제가 되지 않는다.

```java
// 1쪽 객체 안의 코드
@OneToMany(mappedBy = "containerObj", cascade = [CascadeType.ALL])
// N쪽 객체를 담는 collection객체
  
// N쪽 객체 안의 코드
@ManyToOne(cascade = [CascadeType.ALL])  // cascade 없으면 item 최초 저장 시 'jpa save the transient instance before flushing' 발생
@JoinColumn(name = "col_name_xxx")
// 1쪽 객체를 가리키는 객체
private XXX containerObj;
```
