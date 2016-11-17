# Spring-Vaadin Custom Servlet

[Vaadin](https://vaadin.com/home)은 Java로 WebUI를 만들 수 있게 해주는 오픈소스 프레임워크다.

Java 만으로 WebUI를 구현할 수 있으므로, 별도의 프론트엔드 담당 조직이 없어도 프론트엔드 쪽을 개발할 수 있는 좋은 대안이 될 수 있다. 특히 Java 개발 시 가장 많이 사용되는 프레임워크인 Spring과도 통합할 수 있게 되어 있다는 점도 좋다.

하지만 Vaadin에서 미리 만들어둔 컴포넌트를 통해 WebUI를 구현하므로, 아무래도 일반적인 프론트엔드 기술로 구현할 때처럼 자유롭게 원하는 대로 만들 수 없다는 한계도 있다. 그래도 어드민 용 웹 페이지를 만들 때는 꽤 좋은 선택이 될 수 있다고 생각한다.

개발할 애플리케이션이 오로지 Vaadin 애플리케이션 역할만 하면 문제가 없지만, 그 안에 구현된 로직을 재활용해서 API 서버 역할을 겸하게 한다든가 하는 방식으로 확장하다보면 UI 측면 외의 제약 사항에 맞닥뜨리게 되는데, **원래 클라이언트가 보낸 Request에 포함되어 있던 정보가 Vaadin에서 제공하는 서블릿을 통과하면 모두 사라진다**는 것이다.

이럴 때는  Vaadin에서 기본으로 제공하는 서블릿 대신 커스텀 서블릿을 만들어 사용해야 한다.

## Custom Vaadin Servlet

Spring과 통합된 Spring-Vaadin으로 개발할 때 Custom Vaadin Servlet을 다음과 같이 만들면, 원래 클라이언트가 보낸 Request에 있던 정보를 `VaadinSession`이나 Vaadin의 View나 Spring의 컨트롤러에 전달해 줄 수 있게 된다.

```java
@Component("vaadinServlet")
public class CustomSpringVaadinServlet extends SpringVaadinServlet {

    // @Value 나 @Autowired 등 Spring의 애노테이션을 사용할 수 있다.
    @Value("${vaadin.key.userId}")
    private String USER_KEY;
    
    @Override
    protected VaadinServletRequest createVaadinRequest(HttpServletRequest request) {

        if (getServiceUrlPath() != null) {        
            // 원래 request에 있던 필요한 정보를 담는다.
            setOriginalRequestInfo(request);

            VaadinServletService vaadinServletService = getService();
            
            return new SpringVaadinServletRequest(request, vaadinServletService, true);

        } else {
            return new VaadinServletRequest(request, getService());
        }
    }

    private void setOriginalRequestInfo(HttpServletRequest request) {
        
        // 원래 request에 있던 필요한 정보를 HttpSession에 담으면
        // 나중에 VaadinSession이나 Vaadin이 아닌 일반 Spring Controller에서
        // HttpSession을 통해 필요한 정보를 꺼내서 쓸 수 있다.
        HttpSession session = request.getSession();

        String userId = request.getParameter(USER_KEY);
        if (!StringUtils.isEmpty(userId)) {
            session.setAttribute(POC_USER_KEY, userId);
        }

        ...
    }
}```
