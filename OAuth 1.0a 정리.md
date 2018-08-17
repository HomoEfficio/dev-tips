# OAuth  

## 배경  

- 많은 서비스가 API를 통해 서로 연결  

  > 내가 A라는 앱에 쓴 글이 내 트위터 타임라인에도 표시되면 좋겠다.  

- A 앱은 내 트위터 타임라인에 글을 쓸 권한이 없다.  

- 물론 A 앱이 내 트위터 계정 정보를 알고 있다면 A 앱이 내 트위터 타임라인에 글을 쓸 수 있겠지만,  

- 필요한 것은 글을 쓸 수 있는 권한 뿐인데 계정 정보를 모두 A 앱에게 알려줄 필요는 없다.  

- 내가 A 앱에게 글을 쓸 권한을 줬다는 사실을 트위터에게 알려주고,  

- 그 사실을 트위터가 확인할 수 있다면,  

- 내 계정 정보를 A 앱에게 알려주지 않고도 A 앱이 트위터에 글을 쓸 수 있게 된다.  

- `내가 A 앱에 쓴 글을 내 트위터 타임라인에도 표시`하려면 결국 다음의 질문에 대한 답이 필요하다.  

  > 내가 A 앱에게 권한을 줬다는 사실을 트위터에게 어떻게 알려줄 수 있을까?  


## OAuth 1.0a  

어떤 행위(내가 트위터에게 권한을 준 행위)가 이루어졌음을 프로그래밍을 통해 증명하는 여러 방식 중에 대표적으로 서명(Signature)라는 것이 있다.

아래 그림은 서명 방식 중에서 HMAC(Hashed Message Authentication Code)를 보여주고 있다. 

- 송신자와 수신자가 비밀키를 공유하고, 
- 송신자는 평문(빨간 문서 아이콘)과 비밀키를 해시한 값(MAC)과 평문을 함께 수신자에게 보내면, 
- 수신자는 송신자와 공유하고 있는 비밀키와 평문을 해시한 값(Hash Output)을 계산하고, 
- 계산한 값이 송신자가 보낸 MAC 값과 같은지 비교해서 평문이 송신자로부터 전송되었음을 확인한다.
  
![](https://www.thinqloud.com/wp-content/uploads/2017/07/blog_banner_2-1.jpg)
(출처: https://www.thinqloud.com/hmac-authentication-in-salesforce/)  

OAuth 1.0a는 권한을 줬다는 사실을 위와 같은 서명 방식을 이용해서 증명한다. 이를 좀더 구체적으로 알아보자.  

### 등장 인물  

- User: 트위터 계정을 가지고 있는 트위터 사용자. 앱 A에 대한 사용권한도 가지고 있다.  
- Consumer: 트위터 API를 이용해서 트위터에 글을 남기려는 앱 A.  
- Provider: API로 서비스를 제공하는 트위터.  
  
### 사전 조건  
  
- Consumer는 Provider의 API를 이용할 수 있도록 등록되어 있어야 한다.
- User는 Consumer와 Provider 모두를 사용할 수 있는 권한을 가지고 있다.
  
### Provider로부터 확인 받아야 하는 사항  

- Consumer는 User로부터 권한 부여 요청을 받았다는 사실을 Provider로부터 확인 받아야 함 - (1)  
- User는 Provider의 사용자임을 Provider로부터 확인 받아야 함 - (2)  
- User는 Consumer에게 권한을 부여했음을 Provider로부터 확인 받아야 함 - (3)  

이 3가지 확인을 받기위한 절차를 개략적으로 생각해보자  
  
### 절차 개요  

1. User가 Consumer에 글을 쓰고 'Provider에도 남기기' 버튼을 누른다.  

1. Consumer는 자신의 등록 정보의 Signature를 만들고 Provider에게 보내서 권한 부여 요청을 받았음을 Provider에게 알리고, Provider는 권한 부여 요청을 확인했다는 증표를 저장하고 증표를 Consumer에게 회신한다. (1)  

1. Consumer는 권한 부여 요청 확인 증표와 함께 User의 요청을 Provider의 인가(권한 부여) 화면으로 리다이렉트한다.  

1. User가 Provider에 로그인 한 상태가 아니라면 로그인 한다. (2)  

1. 인가 화면에는 'Consumer에게 권한 부여' 버튼이 표시된다.  

1. User가 'Consumer에게 권한 부여' 버튼을 클릭하면, Provider는 권한 부여를 확인하고, 확인 증표를 저장 및 User에게 반환하고 Consumer가 제공하는 callback 화면으로 리다이렉트한다. (3)  

1. 리다이렉트를 통해 권한 부여 확인 증표를 받은 Consumer는 앱 등록 확인 증표 및 권한 부여 확인 증표의 Signature를 만들고 Provider에게 보낸다.  

1. Provider는 Consumer가 보낸 Signature를 확인하고 User만 접근할 수 있었던 보호 자원에 대한 접근 증표를 Consumer에게 발급한다.  

1. 이후 Consumer는 접근 증표를 Provider에게 보여주면서 User를 대신해서 보호 자원에 접근한다.  

6번까지 진행되면 확인해야 할 3가지 사항은 모두 확인했으므로 바로 보호 자원에 대한 접근 증표를 발급할 수 있지만, 6번에서 발급하면 증표가 User에게 반환되고 User의 Local Storage나 Session에 남을 수 있으므로 유출 가능성이 발생한다. 따라서 6번에서는 발급하지 않고 8번에서 Consumer에게 발급한다.  

위 과정에서 '권한 부여 요청 확인 증표'를 `RequestToken`, '권한 부여 확인 증표'를 `AuthorizationCode`, '보호 자원 접근 증표'를 `AccessToken`이라고 부른다.  

### Sequence Diagram  

위 절차 개요를 좀더 상세하게 시퀀스 다이어그램으로 표현해보면 다음과 같다.  

![Imgur](https://i.imgur.com/jhqnHFp.png)  
  
(http://commandlinefanatic.com/cgi-bin/showarticle.cgi?article=art014 내용 참고)

초록색 화살표는 브라우저와 웹 서버의 통신을 나타내며, 파란색 화살표는 HTTP API 호출을 나타낸다.

이제 시퀀스 다이어그램을 토대로 실제 구현해보자.

## OAuth 1.0a 구현

