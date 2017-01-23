# 우분투 bash 프롬프트 커스터마이징

참으로 잉여스러운 일이랄 수 있는데,프롬프트에 아무런 색깔이 없으믄 무쟈게 심심하다.

아래의 명령을 `.bashrc`에 추가하면 컬러로 전체 경로를 표시하는 두 줄로 구성된 프롬프트가 뜬다.

>export PS1="\e[33m[\e[36m\u\e[37m@\e[35m\h\e[33m] \e[34m\w\e[m\n\$ "

대충 감으로 해석해보면

`\e`: 아마도 element로 추정되는데, 걍 구분자로 보면 될 듯
`[31m`: ANSI Color 값 - 나중에 추가
`\u`: 로그인 사용자 이름

와 같은 형식이 반복된다.


----
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="크리에이티브 커먼즈 라이선스" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

<a href='https://www.facebook.com/hanmomhanda' target='_blank'>HomoEfficio</a>가 작성한 이 저작물은

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">크리에이티브 커먼즈 저작자표시-비영리-동일조건변경허락 4.0 국제 라이선스</a>에 따라 이용할 수 있습니다.

