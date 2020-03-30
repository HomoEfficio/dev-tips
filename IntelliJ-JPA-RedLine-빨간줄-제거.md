# IntelliJ JPA RedLine 빨간줄 제거

JPA `@Column` 으로 컬럼명을 지정하면 아래와 같이 빨간줄이 표시될 수 있다.

원론적인 해결 방법은 Datasource 설정을 해주는 건데,  
그게 귀찮다면 아래와 같이 Inspection 항목 설정에서 `JPA > Unresolved database references in annotations` 체크를 해제하고 몇 초 기다리면 빨간줄이 사라진다.

![Imgur](https://i.imgur.com/3tM4Qc7.png)

