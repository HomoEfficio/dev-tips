# Git 과거의 특정 커밋 수정하기

과거의 특정 커밋에 포함된 내용을 수정해야할 때가 있다.

`git rebase`를 사용하면 가능하기는 한데, 수정 후 remote에 올릴 때 결국 `git push --force`(또는 조금이라도 안전하게 하려면 `git push --force-with-lease`)를 써서 기존의 내용을 덮어써야 하므로, 기존의 내용을 공유하고 있던 공동 작업자가 있는 환경에서는 뒷처리가 복잡하다.

따라서 가급적 과거의 이력을 바꾸기보다 그냥 현재 상태에 수정 사항을 적용하는 것이 바람직하지만, 그래도 꼭 해야겠다면 뭐.. 해야지.

## 큰 흐름

작업의 큰 흐름은 다음과 같다.

- 수정하려는 커밋의 바로 이전 커밋을 base로 다시(re) 설정, 즉 rebase 한다.
- 내용을 수정하고 `git add`, `commit --amend`로 커밋도 수정한다.
- git rebase --continue로 마무리.
- rebase 완료 후에는 수정한 커밋 이후의 커밋들도 새로운 커밋번호가 할당되어, 수정 커밋 및 그 이후의 커밋들은 사실 상 새로운 커밋이 된다.

이 정도를 알아두고 실제 화면을 보며 이해해보자.

## git log

수정할 커밋을 확인하고, 바꾸려는 커밋의 바로 이전 커밋을 `git rebase --interactive`의 target으로 지정한다.

![Imgur](http://i.imgur.com/i5vxEeR.png)


## git rebase --interactive

`git rebase --interactive`를 실행하면 다음과 같은 화면이 표시된다.

![Imgur](http://i.imgur.com/gM3SKOb.png)

아래와 같이 수정할 커밋에 `pick`라고 표시된 것을 `edit`로 수정한다.

![Imgur](http://i.imgur.com/keN0obw.png)

저장하면 다음과 같이 수정 후 `commit --amend`, `rebase --continue`를 실행하라는 간단한 안내가 표시된다.

![Imgur](http://i.imgur.com/1skcCKh.png)


## 원하는 커밋의 내용 수정

다음과 같이 수정하기를 원했던 파일을 열어서 수정한다.

![Imgur](http://i.imgur.com/5pdes49.png)

## git add . & git commit --amend

수정 후 `git status`로 확인하면 다음과 같이 표시된다. 수정한 파일을 add 하고,

![Imgur](http://i.imgur.com/PgCqApP.png)

`git commit --amend`로 수정 내용을 커밋한다.

![Imgur](http://i.imgur.com/wlJ1vF5.png)

커밋하면 다음과 같이 커밋 내용에 대한 화면이 표시된다.

![Imgur](http://i.imgur.com/SeRegph.png)

## git rebase --continue

`:q!`를 입력해서 빠져 나온 후, `git rebase --continue`를 실행한다.

![Imgur](http://i.imgur.com/YjvA2DA.png)

다음과 같이 성공 메시지가 표시된다.

![Imgur](http://i.imgur.com/f6M9egA.png)

## 히스토리 확인

히스토리를 확인해보면, 다음과 같이 내용은 동일하지만 rebase의 target으로 지정했던 '수정한 커밋 바로 이전 커밋'에서 분기되어 새로운 커밋들이 master 브랜치에 생겨난 것을 확인할 수 있다.

![Imgur](http://i.imgur.com/8tuDiq6.png)

git log로 확인해보면 수정한 커밋 바로 이전 커밋의 번호는 그대로지만, 수정한 커밋부터 그 이후의 커밋은 모두 커밋 번호가 달라져있음을 확인할 수 있다. 하지만 커밋 메시지나 커밋 시간은 바꾸지 않았으므로 예전과 동일하다.

![Imgur](http://i.imgur.com/339qZF6.png)

git status로 확인해봐도 수정한 커밋을 포함하여 그 이후 5개의 커밋이 origin/master와 달라져있다는, 위의 히스토리 및 로그와 동일한 내용을 확인할 수 있다.

![Imgur](http://i.imgur.com/qckdlTs.png)

## git push

이 상태에서 `git push`를 실행하면 거절된다.

![Imgur](http://i.imgur.com/nXKC1wl.png)

그렇다고 `git fetch` 후 `git rebase` 또는 `git merge`를 하면, 원래의 목적인 '특정 커밋만 수정하기'는 수포로 돌아간다.

방법은 `git push --force`로 특정 커밋만 수정한 내 로컬 버전을 원격에 강제로 덮어쓰는 방법 밖에는 없다. 조금이라도 더 안전하게 작업하려면, 덮어쓰기 전에 로컬의 `remotes/브랜치A`가 참조하고 있는 것과 현재 원격의 `브랜치A`가 참조하고 있는 내용이 동일할 경우에만, 즉, 다른 누군가가 원격의 `브랜치A`에 push를 하지 않은 상태에서만 `git push --force`를 실행하는 `git push --force-with-lease`를 실행할 수도 있다. 

하지만 새로운 내용으로 기존 내용을 덮어쓴다는 것 자체는 동일하며, 이 경우 처음에 얘기한 것처럼 공동 작업자에게는 불필요한 일거리를 넘겨주게 된다.

## 정리

>- 특정 커밋만을 수정해야 한다면, `git rebase --interactive`로 하면된다.
>
>- 다만 공동작업자가 있다면 미리 해당 내용을 공유하고 뒤처리를 최소화 할 수 있는  계획을 수립한 후에 진행한다.









