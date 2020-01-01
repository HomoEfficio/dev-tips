# git 여러 commit의 author를 바꾸는 방법

하나의 컴퓨터에서 여러 git 계정을 쓰다보면 잘못된 author 정보를 써서 수많은 커밋을 날려놓고는 나중에 아차.. 싶은 때가 종종 있다.  
이렇게 뒤늦게 알게되는 건 주로 개인 프로젝트에서 발생한다.  
몇 개 안 된다면 `git rebase -i`로 한땀한땀 고칠 수도 있지만, 아주 많다면?

꽤 위험하지만 한 방에 처리할 수 있는 방법이 있다. 

**기존 커밋의 hash가 모두 바뀌므로 개인 프로젝트가 아닌 협업하는 프로젝트에서는 협의 없이 절대로 해서는 안 되는 방법**이다.

`git filter-branch`를 이용하는 방법인데, [GitHub에 아주 깔끔하게 예제가 정리](https://help.github.com/en/github/using-git/changing-author-info)돼있다.

다음 스크립트를 보면 금방 알 수 있을 것이다.

```bash
#!/bin/sh

git filter-branch --env-filter '

OLD_EMAIL="your-old-email@example.com"
CORRECT_NAME="Your Correct Name"
CORRECT_EMAIL="your-correct-email@example.com"

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

다만 에제에서는 더 안전을 기하기 위해 bare repository를 하나 만들어서 수행하도록 가이드하고 있다.
