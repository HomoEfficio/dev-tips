# Git .gitignore가 안 먹을 때

git을 쓰다보면 간혹 아래와 같이 

![](http://i.imgur.com/BEenCcM.png)

`.gitignore` 파일에 지정해둔 폴더 내의 변경도 무시되지 않고 아래와 같이 관리에 포함될 때가 있다.

![](http://i.imgur.com/2Epc8IH.png)

# 해결

그럴 때는 다음과 같이 staging area 안에 있는 파일을 비워주고 다시 올바른 내용으로 채워주면 된다.

1. 일단 작업 중인 내용은 commit을 해두고,

    ![](http://i.imgur.com/BkuU5TX.png)

2. `git rm -r --cached .` 명령으로 staging area 를 비운다.

    ![](http://i.imgur.com/ju9CNlr.png)

3. `git st`로 확인해보면 뭔가 죄다 지워진 것 같지만 걱정할 필요 없다.

    ![](http://i.imgur.com/kUgL439.png)

4. 더 아래를 보면 `.gitignore`에 지정된 내용을 제외하고, 버전 관리에 포함되어야 할 내용만 올바르게 'Untracked files'에 포함되어 있다.

    ![](http://i.imgur.com/HKaC9PX.png)

5. 다시 `git add .`와 `git commit -m '커밋메시지'`로 Untracked files 를 추가해주면 된다.

    ![](http://i.imgur.com/wkHHpIR.png)
