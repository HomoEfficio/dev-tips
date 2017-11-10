# git revert multiple-commits

보통 히스토리를 유지한 채로 이전 상태로 돌아가려면 `git revert 이전상태커밋`을 실행하면 된다.

그런데 이전 상태가 바로 직전이 아니라 한~참 이전이라면 어떻게 해야할까?

## git revert a..b

전지전능한 git에는 역시 답이 있다. 아래와 같이 `git revert` 명령에 단일 커밋이 아니라 `..`를 이용해서 일정 범위의 커밋을 인자로 넘기면 된다.

>git revert a..b

간단하다! 

그런데 유의해야 할 것이 있다. 바로 `a`와 `b`를 어떻게 지정할 것이냐다. 아래와 같이 히스토리가 구성되어 있고, 

```
9a77e4a getUidGroup는 Widget에서 어쩌구저쩌구
a169beb [REC-1096] 쿠키 싱크는 타 도메인 어쩌구저쩌구
3294d57 [REC-1096] 외부 광고 플랫폼 어쩌구저쩌구
825fe0a [REC-1096] 쿠키 싱크 관련 쿠키 수명을 어쩌구저쩌구
fdd3a41 [REC-1096] 쿠키 싱크 리다이렉션 어쩌구저쩌구
5b7aef3 쿠키 싱크 시 고객 사이트 어쩌구저쩌구
06528fd 쿠키 생성 도메인을 어쩌구저쩌구
3988976 [REC-1096] 외부 UID 가져와서 어쩌구저쩌구
8c0ecc4 [REC-1096] xhr.withCredentials 어쩌구저쩌구
ec4100b getUidGroup을 plugin.coffee에서 어쩌구저쩌구
c2b4e04 [REC-1096] 쿠키 싱크 관련 주석 어쩌구저쩌구
658c512 [REC-1096] 쿠키 싱크 시 트래픽 어쩌구저쩌구
0fb308a [REC-1085] Social buzz 어쩌구저쩌구 <== 이 상태로 돌리고 싶다!!
```

이중에서 `9a77e4a getUidGroup는 Widget에서 어쩌구저쩌구`에서 `658c512 [REC-1096] 쿠키 싱크 시 트래픽 어쩌구저쩌구`까지를 반영하지 않은 상태로, 즉, `0fb308a [REC-1085] Social buzz 어쩌구저쩌구`까지만 반영이 된 상태로 revert  하고 싶다면 아래와 같이 범위를 지정해야 한다.

>git revert 0fb308a..9a77e4a
>
>즉 git revert `되돌아 갈 과거 커밋`..`되돌리기 시작할 최근 커밋` 요렇게 보면 된다.

그럼 아래와 같이 revert commit이 주구장창 생긴다.

```
🍺  git revert 0fb308a..9a77e4a
[master 51ddc8c] Revert "getUidGroup는 Widget에서 어쩌구저쩌구"
 1 file changed, 7 insertions(+), 5 deletions(-)
[master 60e2ada] Revert "[REC-1096] 쿠키 싱크는 타 도메인 어쩌구저쩌구"
 2 files changed, 160 insertions(+)
 create mode 100644 public/test/dmp-uid-sync-creating-cookie.test.html
 create mode 100644 public/test/dmp-uid-sync-existing-cookie.test.html
[master f5871f8] Revert "[REC-1096] 외부 광고 플랫폼 어쩌구저쩌구"
 1 file changed, 90 insertions(+), 16 deletions(-)
[master 506d983] Revert "[REC-1096] 쿠키 싱크 관련 쿠키 수명을 어쩌구저쩌구"
 2 files changed, 30 insertions(+), 23 deletions(-)
[master 05a078d] Revert "[REC-1096] 쿠키 싱크 리다이렉션 어쩌구저쩌구"
 1 file changed, 18 insertions(+), 29 deletions(-)
[master ec6e477] Revert "쿠키 싱크 시 고객 사이트 어쩌구저쩌구"
 1 file changed, 1 deletion(-)
[master e527f26] Revert "쿠키 생성 도메인을 어쩌구저쩌구"
 1 file changed, 5 insertions(+)
[master 0e540d2] Revert "[REC-1096] 외부 UID 가져와서 어쩌구저쩌구"
 1 file changed, 52 insertions(+), 142 deletions(-)
[master c73189e] Revert "[REC-1096] xhr.withCredentials 어쩌구저쩌구"
 1 file changed, 12 deletions(-)
[master 7a539ba] Revert "getUidGroup을 plugin.coffee에서 어쩌구저쩌구"
 2 files changed, 5 insertions(+), 5 deletions(-)
[master a5278ad] Revert "[REC-1096] 쿠키 싱크 관련 주석 어쩌구저쩌구"
 1 file changed, 19 insertions(+), 21 deletions(-)
[master 05c0f30] Revert "[REC-1096] 쿠키 싱크 시 트래픽 어쩌구저쩌구"
```

눈여겨 볼 것은 revert 커밋은 원래 커밋과 역순으로 진행된다는 점이다. 다시 말하면 최신 커밋부터 시작해서 과거 커밋으로 진행되는데, revert가 과거로 되돌리는 작업이므로 사실 당연하기도 하다.

그리고 위에 `[master #######]`로 표시된 것중 `#######`는 revert 커밋의 커밋 ID는 또 아니다. 아래와 같이 `git log`로 확인되는 커밋 ID와 `#######`는 다르다. 이 부분은 일단 그냥 넘어가고 아래와 같이 revert 결과를 확인해보자.

### revert 결과

```
🍺 git log --pretty=oneline
b6c428f Revert "[REC-1096] 쿠키 싱크 시 트래픽 어쩌구저쩌구"
fc185c3 Revert "[REC-1096] 쿠키 싱크 관련 주석 어쩌구저쩌구"
7c31406 Revert "getUidGroup을 plugin.coffee에서 어쩌구저쩌구"
c8124b8 Revert "[REC-1096] xhr.withCredentials 어쩌구저쩌구"
654ee33 Revert "[REC-1096] 외부 UID 가져와서 어쩌구저쩌구"
6b770d2 Revert "쿠키 생성 도메인을 어쩌구저쩌구"
f4ec6fc Revert "쿠키 싱크 시 고객 사이트 어쩌구저쩌구"
1a34511 Revert "[REC-1096] 쿠키 싱크 리다이렉션 어쩌구저쩌구"
8631002 Revert "[REC-1096] 쿠키 싱크 관련 쿠키 수명을 어쩌구저쩌구"
acb30ad Revert "[REC-1096] 외부 광고 플랫폼 어쩌구저쩌구"
50eaad6 Revert "[REC-1096] 쿠키 싱크는 타 도메인 어쩌구저쩌구"
c2362c0 Revert "getUidGroup는 Widget에서 어쩌구저쩌구"
```

자 이렇게 일단 긴급하게 과거 상태로는 돌려놨고, 개발은 계속 이어가야 하므로 revert 된 여러 커밋을 다시 revert 해야 할 수도 있다. 이건 어떻게 해야할까?

## revert 한 걸 다시 revert 하기

마찬가지로 범위를 지정해서 `git revert`를 실행하면 된다. 범위를 어떻게 지정할지가 문제다.

일단 안전을 위해 `git co -b revert`로 새로운 `revert` 브랜치를 하나 따서 `revert` 브랜치에서 시도해보자.

`git co -b revert` 아래와 같이 revert 커밋 모두를 범위로 해서 `git revert`를 실행한다.

```
🍺  git revert c2362c0..b6c428f
[revert b76a261] Revert "Revert "[REC-1096] 쿠키 싱크 시 어쩌구저쩌구""
 1 file changed, 12 insertions(+), 12 deletions(-)
[revert 4826f4b] Revert "Revert "[REC-1096] 쿠키 싱크 관련 주석 어쩌구저쩌구""
 1 file changed, 21 insertions(+), 19 deletions(-)
[revert 976031d] Revert "Revert "getUidGroup을 plugin.coffee에서 어쩌구저쩌구""
 2 files changed, 5 insertions(+), 5 deletions(-)
[revert 0bbafdb] Revert "Revert "[REC-1096] xhr.withCredentials 어쩌구저쩌구""
 1 file changed, 12 insertions(+)
[revert 58cbe24] Revert "Revert "[REC-1096] 외부 UID 가져와서 어쩌구저쩌구""
 1 file changed, 142 insertions(+), 52 deletions(-)
[revert 5955478] Revert "Revert "쿠키 생성 도메인을 어쩌구저쩌구""
 1 file changed, 5 deletions(-)
[revert 98ca146] Revert "Revert "쿠키 싱크 시 고객 사이트 어쩌구저쩌구""
 1 file changed, 1 insertion(+)
[revert 7c2ad96] Revert "Revert "[REC-1096] 쿠키 싱크 리다이렉션 어쩌구저쩌구""
 1 file changed, 29 insertions(+), 18 deletions(-)
[revert 923c33d] Revert "Revert "[REC-1096] 쿠키 싱크 관련 쿠키 수명을 어쩌구저쩌구""
 2 files changed, 23 insertions(+), 30 deletions(-)
[revert 7d75d42] Revert "Revert "[REC-1096] 외부 광고 플랫폼 어쩌구저쩌구""
 1 file changed, 16 insertions(+), 90 deletions(-)
[revert 7e38b24] Revert "Revert "[REC-1096] 쿠키 싱크는 타 도메인 어쩌구저쩌구""
 2 files changed, 160 deletions(-)
 delete mode 100644 public/test/dmp-uid-sync-creating-cookie.test.html
 delete mode 100644 public/test/dmp-uid-sync-existing-cookie.test.html
```

순서를 보면 이번에도 역순으로 진행되며, revert를 두 번 하면 역순의 역순이 되어 결국   원래의 커밋 순서대로 돌아오게 된다. 순서가 헷갈린다면 순서는 그냥 신경쓰지 않아도 좋다.

### revert + revert 결과

```
🍺 git log --pretty=oneline
7e38b24 Revert "Revert "[REC-1096] 쿠키 싱크는 타 도메인 어쩌구저쩌구""
7d75d42 Revert "Revert "[REC-1096] 외부 광고 플랫폼 어쩌구저쩌구""
923c33d Revert "Revert "[REC-1096] 쿠키 싱크 관련 쿠키 수명을 어쩌구저쩌구""
7c2ad96 Revert "Revert "[REC-1096] 쿠키 싱크 리다이렉션 어쩌구저쩌구""
98ca146 Revert "Revert "쿠키 싱크 시 고객 사이트 어쩌구저쩌구""
5955478 Revert "Revert "쿠키 생성 도메인을 어쩌구저쩌구""
58cbe24 Revert "Revert "[REC-1096] 외부 UID 가져와서 어쩌구저쩌구""
0bbafdb Revert "Revert "[REC-1096] xhr.withCredentials 어쩌구저쩌구""
976031d Revert "Revert "getUidGroup을 plugin.coffee에서 어쩌구저쩌구""
4826f4b Revert "Revert "[REC-1096] 쿠키 싱크 관련 주석 어쩌구저쩌구""
b76a261 Revert "Revert "[REC-1096] 쿠키 싱크 시 트래픽 어쩌구저쩌구""
```

얼추 잘 된 것 같지만 자세히 검사해보면, 내가 원하는 상태로 되돌아오지 않고 1% 부족하다. `c2362c0 Revert "getUidGroup는 Widget에서 어쩌구저쩌구"` 이 커밋은 되돌아와 있지 않다.

### revert + revert + (revert revert되지않은커밋)

그래서 명시적으로 `git revert c2362c0`을 해줘야 아래와 같이 원하는 상태로 만들어진다.

```
🍺 git log --pretty=oneline
b353b9a Revert "Revert "getUidGroup는 Widget에서 어쩌구저쩌구""
7e38b24 Revert "Revert "[REC-1096] 쿠키 싱크는 타 도메인 어쩌구저쩌구""
7d75d42 Revert "Revert "[REC-1096] 외부 광고 플랫폼 어쩌구저쩌구""
923c33d Revert "Revert "[REC-1096] 쿠키 싱크 관련 쿠키 수명을 어쩌구저쩌구""
7c2ad96 Revert "Revert "[REC-1096] 쿠키 싱크 리다이렉션 어쩌구저쩌구""
98ca146 Revert "Revert "쿠키 싱크 시 고객 사이트 어쩌구저쩌구""
5955478 Revert "Revert "쿠키 생성 도메인을 어쩌구저쩌구""
58cbe24 Revert "Revert "[REC-1096] 외부 UID 가져와서 어쩌구저쩌구""
0bbafdb Revert "Revert "[REC-1096] xhr.withCredentials 어쩌구저쩌구""
976031d Revert "Revert "getUidGroup을 plugin.coffee에서 어쩌구저쩌구""
4826f4b Revert "Revert "[REC-1096] 쿠키 싱크 관련 주석 어쩌구저쩌구""
b76a261 Revert "Revert "[REC-1096] 쿠키 싱크 시 트래픽 어쩌구저쩌구""
```

## 정리

1. 여러 커밋을 revert 하려면 아래와 같이 `..`를 이용해서 범위를 지정하면 된다.

    >git revert `되돌아 갈 과거 커밋`..`되돌리기 시작할 최근 커밋`
    
1. revert 한 여러 커밋을 다시 revert 하려면 아래와 같이 2단계를 거쳐야 한다.

    1. revert된 커밋 범위로 revert 다시 수행
    
        >git revert `revert된 가장 과거 커밋`..`revert된 최신 커밋`
        
    1. revert된 과거 커밋을 대상으로 다시 한 번 revert 수행

        >git revert `revert된 가장 과거 커밋`

