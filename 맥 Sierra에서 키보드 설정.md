## Control <=> Command 

다행스럽게도 그냥 OS 차원에서 지원해준다.

1. Apple 메뉴 > 시스템 환경설정을 선택하고 '키보드' 클릭

2. '키보드'를 클릭한 다음 화면 우하단의 '보조 키' 클릭

3. 변경하려는 각 보조 키의 팝업 메뉴를 클릭한 다음 아래 그림과 같이 `Control`과 `Command`를 서로 바꾼다. 이거 안 하면 특기인 CTRL+C, CTRL+V를 구사하는데 큰 차질이 생긴다.

    ![Imgur](http://i.imgur.com/4Yi7uUY.png)

## 한영 전환

일단 기본으로 `CapsLock` 키로 한영 전환이 가능하다.

    ![Imgur](http://i.imgur.com/rMzvWCS.png)

하지만, 적응이 안된다.. `한/영`키가 안된다면 `Shift+Space`로 하자.

http://macnews.tistory.com/297 이걸 보고 따라하자!
    
## Home/End 키

아.. 이게 되다니!! 이게 되다니!!

~/Library/KeyBindings/DefaultKeyBinding.dict 파일을 만들고, 다음의 내용을 입력하고 저장한다.

```
{
  "\UF729" = "moveToBeginningOfLine:"; /* Home */
  "\UF72B" = "moveToEndOfLine:"; /* End */
  "$\UF729" = "moveToBeginningOfLineAndModifySelection:"; /* Shift + Home */
  "$\UF72B" = "moveToEndOfLineAndModifySelection:"; /* Shift + End */
  "^\UF729" = "moveToBeginningOfDocument:"; /* Ctrl + Home */
  "^\UF72B" = "moveToEndOfDocument:"; /* Ctrl + End */
  "$^\UF729" = "moveToBeginningOfDocumentAndModifySelection:"; /* Shift + Ctrl + Home */
  "$^\UF72B" = "moveToEndOfDocumentAndModifySelection:"; /* Shift + Ctrl + End */
  "₩" = ("insertText:", "`"); /* Shift+Space로 한글 모드로 전환 후 `를 누르면 ₩가 입력되는 문제 해결 */
}
```

## Karabiner 삭제

결론적으로 예전 El Capitan에서 사용하던 Karabiner는 필요없으므로 삭제!

혹시나 필요하다면 Sierra에서는 https://github.com/tekezo/Karabiner-Elements 를 써야 한다능..

