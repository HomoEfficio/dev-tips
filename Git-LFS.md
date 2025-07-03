# Git LFS

파일 업로드 테스트에 사용되는 mp4, gif 등의 파일을 포함해서 git push 를 하면 다음과 같은 에러가 발생하며 실패한다. 파일 크기가 5M 이하로 작은데도 에러가 발생한다.

```
Enumerating objects: 81, done.
Counting objects: 100% (81/81), done.
Delta compression using up to 10 threads
Compressing objects: 100% (53/53), done.
error: RPC failed; HTTP 400 curl 22 The requested URL returned error: 400
send-pack: unexpected disconnect while reading sideband packet
Writing objects: 100% (57/57), 8.21 MiB | 18.51 MiB/s, done.
Total 57 (delta 27), reused 0 (delta 0), pack-reused 0
fatal: the remote end hung up unexpectedly
Everything up-to-date
```

메시지에는 LFS(Large File System) 관련 언급이 전혀 없지만 LFS 처리를 해주면 해결된다.

## 해결 흐름

- `git push` 를 실행하는 로컬에 git LFS 설치
- `git lfs` 활성화
- `git lfs track` 으로 업로드 할 파일 트래킹 지정
  - tracking 내용이 .gitattributes 파일에 추가된다
  - 업로드 할 파일이 `git lfs track` 으로 지정한 경로에 존재하지 않더라도 트래킹 지정은 가능하다
- 파일을 포함하여 커밋 후 push

[GitHub 문서](https://docs.github.com/ko/repositories/working-with-files/managing-large-files/about-large-files-on-github)에 관련 내용이 있지만 파편화 돼 있어서 종합적으로 이해하기 어렵다.

git LFS에 대해 세부적으로는 모르지만 위 흐름을 볼 때 mp4,  
gif 같은 파일은 로컬에서 기본적으로 tracking 하지 않기 때문에 push 할 때 에러가 발생하는 것으로 추정된다.

그래서 결국 로컬에서 해당 파일들을 tracking 하게 해줘야 push 할 때 에러가 발생하지 않는다.

git LFS의 설치부터 push 성공까지 구체적인 내용은 https://git-lfs.com/ 여기에 잘 나와있다.

위 절차를 수행하고 push 하면 다음 메시가 출력되며 성공한다.

```
Locking support detected on remote "origin". Consider enabling it with:
  $ git config lfs.https://YOUR-GIT-REPO/info/lfs.locksverify true
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
remote: 
remote: Create a pull request for 'YOUR-BRANCH' on GitHub by visiting:
remote:      https://YOUR-GIT-REPO/pull/new/YOUR-BRANCH
remote: 
To https://YOUR-GIT-REPO
 * [new branch]            YOUR-BRANCH -> YOUR-BRANCH

```

맨 위에 제안하고 있는 `git config lfs.https://YOUR-GIT-REPO/info/lfs.locksverify true`를 로컬에서 실행해주면 다음부터는 Lock support 관련 메시지가 표시되지 않는다.

## 기타

반드시 발생하는 오류인지까지는 확인하지 못했지만,  
A 브랜치에서 위와 같이 해결해서 A 브랜치 push 는 성공했는데,  
A 브랜치를 B 브랜치에 반영하고 B 브랜치를 push 하면 실패할 수도 있다.

이 때는 B 브랜치에서 `git lfs` 활성화, `git lfs track`으로 트래킹 지정을 먼저 해준 후에 A 브랜치를 병합하고 B 브랜치를 push 하면 성공한다.
