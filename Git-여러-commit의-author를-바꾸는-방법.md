# Git 여러 commit의 author를 바꾸는 방법

하나의 컴퓨터에서 여러 git 계정을 쓰다보면 잘못된 author 정보를 써서 수많은 커밋을 날려놓고는 나중에 아차.. 싶은 때가 종종 있다.  
이렇게 뒤늦게 알게되는 건 주로 개인 프로젝트에서 발생한다.  
몇 개 안 된다면 `git rebase -i`로 한땀한땀 고칠 수도 있지만, 아주 많다면?

꽤 위험하지만 한 방에 처리할 수 있는 방법이 있다. 

**기존 커밋의 hash가 모두 바뀌므로 개인 프로젝트가 아닌 협업하는 프로젝트에서는 협의 없이 절대로 해서는 안 되는 방법**이다.

`git-filter-repo`를 사용하면 기존의 `git filter-branch` 보다 훨씬 빠르게 처리할 수 있다.

맥에서는 `brew install git-filter-repo`로 설치하면 되고, 다른 OS에서는 https://github.com/newren/git-filter-repo/blob/main/INSTALL.md 를 참고해서 설치한다.

`git filter-branch`를 `git-filter-repo`로 대체하는 방법은 https://github.com/newren/git-filter-repo/blob/main/Documentation/converting-from-filter-branch.md#cheat-sheet-conversion-of-examples-from-the-filter-branch-manpage 여기에 잘 나와있다.

author 이름을 바꾸려면 아래와 같이 하면 된다.

```
git-filter-repo --force --name-callback 'return name.replace(b"OLD NAME", b"NEW NAME")'

Parsed 2457 commits
New history written in 0.32 seconds; now repacking/cleaning...
Repacking your repo and cleaning out old unneeded objects
HEAD is now at 1b7c31f0 XXXXXXXXXX
Enumerating objects: 59175, done.
Counting objects: 100% (59175/59175), done.
Delta compression using up to 10 threads
Compressing objects: 100% (18528/18528), done.
Writing objects: 100% (59175/59175), done.
Total 59175 (delta 17561), reused 59057 (delta 17465), pack-reused 0
Completely finished after 1.93 seconds.
```

위와 같이 하면 author name 이 `OLD NAME`인 커밋에 대해서만 `NEW NAME`으로 변경되고 commit id 도 변경되며, author name 이 `OLD NAME`이 아닌 커밋은 그대로 보존된다.

author 이메일을 바꾸려면 아래와 같이 하면 된다.

```
git-filter-repo --force --email-callback 'return email.replace(b"OLD EMAIL", b"NEW EMAIL")'

Parsed 2457 commits
New history written in 0.31 seconds; now repacking/cleaning...
Repacking your repo and cleaning out old unneeded objects
HEAD is now at 93c85ba3 XXXXXXXXXX
Enumerating objects: 59175, done.
Counting objects: 100% (59175/59175), done.
Delta compression using up to 10 threads
Compressing objects: 100% (18432/18432), done.
Writing objects: 100% (59175/59175), done.
Total 59175 (delta 17561), reused 59154 (delta 17561), pack-reused 0
Completely finished after 2.08 seconds.
```

두 작업 모두 2초 정도에 수행됐는데 `git filter-branch`를 사용했다면 수십 초에서 분 단위까지 걸렸을 것이다.

아래의 `git filter-branch`는 이제는 떠나 보내주자.

---

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

다만 GitHub 예제에서는 더 안전을 기하기 위해 bare repository를 하나 만들어서 수행하도록 가이드하고 있다.

위 파일 실행 후 다음과 같은 에러가 나면,

```
Cannot create a new backup.
A previous backup already exists in refs/original/
Force overwriting the backup with -f
```

다음과 같이 

`git filter-branch --env-filter '` 을 `git filter-branch -f --env-filter '` 로 바꿔서 실행하면 된다.
