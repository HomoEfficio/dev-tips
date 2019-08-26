# Java 동적 클래스로딩과 스프링 @Transactional

스프링 애플리케이션 startup time 말고 runtime 중간에 동적으로 로딩한 외부 클래스 A에,  
`@Transactional` 붙은 메서드 B가 포함돼있다면,  
B가 호출될 때 트랜잭션 관리 기능이 자동으로 적용되게 하자. 가 목표였고, 결론은 불가..

까먹기 전에 살짝 정리해보면,  
A를 런타임에 로딩해서 사용하려면 URLClassLoader를 상속한 커스텀 클래스로더를 만들어서 동적으로 로딩하면 된다.  
하지만 스프링의 프록시 자동 생성 로직을 태워서 A에 대한 프록시를 생성할 때는 기본적으로는 정해진 클래스로더(톰캣 환경이면 WebAppClassLoader, 스프링부트 내장 WAS면 LaunchedURLClassLoader, devtools를 쓰는 로컬 개발환경이면 RestartClassLoader)를 통해 클래스를 로딩하게 돼 있다.

톰캣 환경이면 context.xml 파일에 <Loader> 로 커스텀 클래스로더를 지정할 수 있고, 스프링부트는 PropertiesLauncher+loader.path로 클래스패스를 지정해줄 수 있고, 가장 범용적으로는 manifest 파일에 Class-Path로 클래스패스를 지정해 줄 수 있다.

그런데 A에 대한 클래스패스를 추가하는 걸로 끝나는 게 아니라 A가 의존하는 다른 클래스가 의존하는 다른 클래스가 의존하는 다른 클래스가 의존하는.. 클래스패스도 모두 직접 추가해줘야 한다는 문제가 있다.

이 문제를 해결하지 못하면 현실에서는 사실상 쓸 수 없는 방법이지만, 억지로라도 의존하는 클래스를 가능한 최소화해서 클래스패스를 지정해주고 실행해보니 드디어 고귀한 프록시가 생성된다는 TRACE 로그가 보였다. 오오 드디어~~

그러나 바로 다음에 아래 그림과 같이 'o.s.aop.framework.CglibAopProxy : Unable to apply any optimizations to advised method: @Transactional붙어있는 메서드'라는 로그가 나오며, 결국에는 `@Transactional` 이 있더라도 Tx 관리 기능이 동작하지 않더라능..

![Imgur](https://i.imgur.com/Vm9NDZf.png)

게다가 이렇게 스프링이 제공해주는 프록시 생성 로직을 통해 프록시로 등록되면, 해당 프록시가 캐시 돼서 A의 내용을 바꿔서 A를 포함하는 jar를 새로 배포해도 바뀐 내용 기준으로 새 프록시가 생성되지 않으며, 결국 바뀐 내용이 반영되지 않는다. 이러면 런타임에 동적 로딩할 이유가 사라져버린다능.. ㅠㅜ

`@Transactional`은 포기하고 걍 PlatformTransactionManager를 직접 써주거나 아니면 프록시 비스무리한 효과를 낼 수 있는 다른 방법을 고민해봐야 할 듯
