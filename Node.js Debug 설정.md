# Node.js Debug 설정

Node.js와 CoffeeScript를 일부 사용해서 개발된 API 서버를, 
인텔리제이에서 BreakPoint 잡아가며 Debug 하려니 

1. 어찌어찌 Debug 모드로 실행해도 BreakPoint가 안 먹더라는..
1. 또는 아예 Debug 모드 자체가 활성화 되지 않더라는..

이래저래 해보다가 원인과 답을 알아냈다.

- CLI 상에서 grunt-cli로 애플리케이션을 실행한다고 하더라도, 애플리케이션 상에서 BP를 잡아가며 디버그 하려면, grunt 용 Run Config가 아니라 Node.js 용 Run Config를 생성해야 한다.
- 실행할 메인 JavaScript 파일에 *.coffee 파일(A라고 하자)을 지정하면 Node.js Run/Debug Configuration을 생성해도 Debug 버튼이 활성화 되지 않는다.
- 따라서 그 A 파일을 동일한 내용의 *.js파일(B라고 하자)로 작성해서 메인 JavaScript로 지정해주면 Debug 버튼이 활성화 되며, 옵션을 아래와 같이 적절하게 지정해주면 BP가 제대로 동작한다!!

- 설정 캡쳐

	![](http://i.imgur.com/zGFmAsr.png)


### 뭔가 부족하다..

1. 자바처럼 Debug 모드로 실행한 후에는 원할 때 원하는 곳 어디든 BP를 걸 수 있는 게 아니라, Debug 모드로 띄우기 전에 미리 BP를 설정해놔야 한다.

    - 마찬가지로, Debug 모드로 떠 있는 상태에서 이미 지정되어 있던 BP를 해제한다고 해도, 실제로는 그 지점에 BP가 남아서 실행이 계속 정지된다.
    
    - BP를 재설정하려면 애플리케이션을 다시 띄워야 한다.

1. `.coffee` 파일에 직접 BP를 걸어서 정지 시킬 수 없다.

    - `.coffee`는 결국 `.js`로 트랜스파일되어 실행되므로, `.coffee`에 지정한 BP가 원하는 대로 안 먹는게 당연하다는 생각도 든다.
    
    - BP가 안 되는데 어떻게 개발을 해온걸까.. (혹시 `console.log`로 도배? ... 상상하지 말자..)


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
