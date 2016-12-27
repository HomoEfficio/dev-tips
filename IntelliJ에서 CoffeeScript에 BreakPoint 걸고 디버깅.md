# IntelliJ에서 CoffeeScript에 BreakPoint 걸고 디버깅

결론부터 말하면,

>CoffeeScript 파일에 직접 BreakPoint를 걸고 디버깅하는 것은 불가능하다.

그럼 CoffeeScript로 개발하면 BreakPoint 없이 오로지 `console.out`로 값을 찍어가면서 개발할 수 밖에 없는건가..
천만 다행스럽게도 다른 방법이 있다.

>CoffeeScript를 transpile 해서 나온 JavaScript 파일의 동일한 위치에 BreakPoint를 걸면 Break가 걸린다.

이 방법을 구체적으로 알아보자.

## 전체 흐름

위에 쓴 방법을 조금 구체적으로 정리해보면,

>- 원하는 것: `A.coffee` 파일의 행번호 B에 BreakPoint를 걸고 싶다.
>
>- 해결 방법
>    1. `A.coffee` 파일을 transpile해서 `A.js`, `A.js.map` 파일 생성
>    1. `A.js` 에서 `A.coffee`의 B 위치에 해당하는 행에 BreakPoint를 건다.
>    1. Debug 모드로 애플리케이션을 실행한다.

`.coffee` -> `.js`, `.js.map`으로 transpile 하기 위해 필요한 사전 절차가 있으니, `File Watcher` 플러그인의 설치와 설정이다.

## File Watcher

IntelliJ에는 CoffeeScript 파일을 Transpile 해주는 `File Watcher`라는 플러그인이 있다. 이름은 뭔가 지켜보고 있다가 자동으로 뭔가를 해주는 분위기를 풍기지만, 기본적으로는 자동으로 동작하지는 않고 `Run File Watcher`라는 Action을 수동으로 실행 시켜줘야 한다.

File Watcher의 설치와 설정은 다음과 같다.

1. File Watcher 플러그인 설치

    ![Imgur](http://i.imgur.com/DHN03p3.png)
    
1. CoffeeScript 템플릿 생성

    ![Imgur](http://i.imgur.com/PzMaVei.png)
    
1. CoffeeScript 템플릿 설정은 기본 그대로 둔다.

    ![Imgur](http://i.imgur.com/wEWYHDH.png)
    
## Transpile(File Watcher 실행)
    
1. Transpile 할 CoffeeScript 파일 선택 및 `Run File Watchers` 액션 실행

    ![Imgur](http://i.imgur.com/J9ZGyc0.png)
    
1. Transpile 실행 결과로 `.js` 및  `.js.map` 파일 자동 생성

    ![Imgur](http://i.imgur.com/IE6HJBi.png)
    
## BreakPoint 지정 및 BreakPoint 동작 확인
        
1. `.js`에서 원하는 위치에 BreakPoint 지정 

    ![Imgur](http://i.imgur.com/c8tHtxO.png)
    
1. Debug 모드로 애플리케이션 실행 후, BreakPoint 지정한 부분이 호출되면 실행이 멈추고 Debugger가 활성화 된다.

    ![Imgur](http://i.imgur.com/ILWfo9q.png)

## 실무적으로 아쉬운 점

### 자동 생성 파일의 배포 제외 불가

- Transpile 해서 자동으로 생성된 `.js` 파일이나 `.js.map` 파일은 배포할 때는 포함되지 않게 하려면 다음과 같은 방법이 가능할 것 같다.
	- 특정 디렉터리 A에 모아놓고, A 디렉터리를 `.gitignore` 파일에 추가해서 Repository에 올라가지 않게 하는 방법
	- 자동 생성되는 파일의 확장자를 `.js.gen`, `.js.map.gen` 등으로 생성되게 하고 `.gitignore` 파일에 `.js.gen`, `.js.map.gen`를 추가해서 Repository에 올라가지 않게 하는 방법
- 하지만, 그게 현재 버전(2016.3)에서는 현실적으로 불가능한 것 같다. File Watcher의 옵션으로 이것저것 시도해봤으나, 모두 실패..

### CoffeeScript를 JS로 전환 후 배포

- 그럼 아예 `.coffee`로 작성해서 개발하고, 배포 시에는 일괄적으로 Transpile 한 파일만 포함되게 하는건 어떨까?
	- 가능해 보인다. 다만, 원래 `.coffee`에 작성되었던 **`//` 주석은 Transpile 하고 나면 모두 사라진다.**
	- 다만, 아래와 같이 [CoffeeScript 전용 주석 문법을 활용하면 Transpile 후에도 보존된다](http://stackoverflow.com/questions/9724206/maintaining-comments-in-js-files-after-compilation-from-coffee)고 한다.
	
		```coffeescript
		###*
       # This will be preserved in a block comment in the javascript
       ###
       ```
       와 같이 작성되면 아래와 같이 Transpile 후에도 보존된다.       
       
       ```
       /**
         * This will be preserved in a block comment in the javascript
         */
       ```

개인적으로는 지금 시점(2016.12)에서는 CoffeeScript를 쓰기보다 ES6로 작성하는 것이 훨씬 낫다고 본다.

----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.
