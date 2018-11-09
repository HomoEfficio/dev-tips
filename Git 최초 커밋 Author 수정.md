# Git 최초 커밋 Author 수정

보통 CLI 도구로 프로젝트를 생성하면서 git repository도 함께 생성될 때는 그냥 디폴트 git 사용자를 Author로 해서 최초 commit이 생성된다.

![Imgur](https://i.imgur.com/xPZCuJc.png)

최초 커밋에 남겨진 Author를 다른 Author로 바꾸려면 어떻게 해야할까?

1. 다음 명령 실행

    >git rebase -i --root

1. pick을 edit로 수정

    ![Imgur](https://i.imgur.com/xvSERvk.png)

1. 저장하면 다음과 같은 안내가 나온다.

    ![Imgur](https://i.imgur.com/rvC740A.png)

1. 다음 명령으로 Author를 변경

    >git commit --amend --author="hanmomhanda <hanmomhanda@gmail.com>"

1. 다음과 같이 Author 변경 작업이 진행되는 것을 알 수 있다.

    ![Imgur](https://i.imgur.com/Ft2L0Oo.png)
    
1. 저장하면 다음과 같은 안내가 나온다.

    ![Imgur](https://i.imgur.com/fiLfzwY.png)
    
1. git log로 확인하면 Author가 변경된 것을 확인할 수 있다.

    ![Imgur](https://i.imgur.com/pLPoeeN.png)


