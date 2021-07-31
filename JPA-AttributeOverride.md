# JPA AttributeOverride

아래와 같이 NotAllowedReason 클래스를 rejectedReason, forbiddenReason 으로 임베드하려고 한다.

![Imgur](https://i.imgur.com/n5N2Lj6.png)

NotAllowedReason 은 다음과 같다.

![Imgur](https://i.imgur.com/dYH38y0.png)

이대로 실행하면 아래와 같은 에러가 발생한다.

```
Repeated column in mapping for entity: xxx.xxx.xxx.WorldVersion column: rejected_message_key
```

하나의 WorldVersion 클래스 안에서 NotAllowedReason 클래스가 rejectedReason, forbiddenReason 두 번 매핑되므로,  
하나의 world_version 테이블에 NotAllowedReason 에 들어있는 rejected_message_key 컬럼이 두 번 매핑되면서 발생하는 현상이다.

이럴 때는 `@AttributeOverride`를 사용해서 테이블에 사용될 이름을 원하는 대로 지정해줄 수 있다.

![Imgur](https://i.imgur.com/mkHCCTI.png)

애플리케이션을 실행하면 다음과 같이 지정한 이름으로 컬럼이 추가된다.

![Imgur](https://i.imgur.com/LEmhNUE.png)

