# Hexo Theme Tranquilpeak 수정

https://github.com/hanmomhanda/custom-tranquilpeak-hexo-theme/commits/master 참고


## 구글 어낼리틱스

- https://github.com/HomoEfficio/homoefficio.github.io/commit/d83edbd70035bb2c0ec3a7ed266110bea504c5a7
- https://github.com/HomoEfficio/homoefficio.github.io/commit/9b5fd07afff65f40ad2727636b8ecb5f883d9e39


## 구글 애드센스

- https://github.com/HomoEfficio/homoefficio.github.io/commit/e4c32f526a4454911f43000c68b65fb995d8819c
- https://github.com/HomoEfficio/homoefficio.github.io/commit/c35f9895889ca7d5d3f9870ab0bc4250182aec3b


## SCSS 커스터마이징

- https://github.com/HomoEfficio/homoefficio.github.io/commit/564478f8d0c6d9440fb702eabd8165acac704aa3
- https://github.com/HomoEfficio/homoefficio.github.io/commit/d5c4008225131656a7b84ea23e8e51f26705a87d
- https://github.com/HomoEfficio/homoefficio.github.io/commit/242de93d2782e145c984ac4b576cf2def5c94b5d (fontawesome, jquery.fancybox 관련 수정)

scss 관련 변수는 `tranquilpeak/source/_css/util/_variable.css`에 있음

## 빌드

### grunt task 직접 실행

`tranquildpeak` 디렉터리에서 hexo server를 종료한 상태에서 `build-theme.sh` 실행 (실제로는 아래 명령 실행)

>grunt clean:build
>
>grunt bower:dev
>
>grunt syncAssets
>
>grunt replace:cssFancybox
>
>grunt replace:cssTranquilpeak
>
>grunt concat
>
>grunt cssmin
>
>grunt uglify
>
>grunt linkAssetsProd


## 배포

위와 같이 스타일을 변경하고 grunt task 작업 후에도 `public/assets/css` 안에 있는 `style.css`, `style.min.css`, `tranquipeak.css` 파일의 날짜가 변경되지 않았다면, 먼저 `themes/tranquilpeak/source/assets/css` 안에 있는 `style.css`, `style.min.css`, `tranquipeak.css` 파일을 `public/assets/css`에 직접 덮어쓴다.

### home page repo

`homoefficio.github.io` 디렉터리에서 

- `hexo generate --deploy`

### main repo

`homoefficio.github.io` 디렉터리에서 

- `git push origin source`

### theme repo (subtree)

`homoefficio.github.io` 디렉터리에서 

- `git subtree push --prefix=themes/tranquilpeak/ custom-tranquilpeak dev`
- `git subtree push --prefix=themes/tranquilpeak/ custom-tranquilpeak master`

# h1
## h2
### h3
#### h4
##### h5
###### h6

# hexo 빌드 관련

- source 브랜치에서 `hexo generate --deploy` 를 실행하면, 
- source 브랜치 상에서 .deploy_git 폴더에 있는 내용을 모두 삭제하고, 
- 새로 generate(빌드)한 내용으로 .deploy_git 폴더의 내용을 채우고, 
- 새로 generate(빌드)한 내용이 origin/master 브랜치의 root 폴더 아래의 컨텐츠를 덮어쓴다.

# Brave Verification 파일 처리

- 포스트 작성 후 `hexo generate` 실행
- `.nojekyll` 파일과 `.well-known/brave-rewards-verification.txt` 파일을 `.delply_git` 폴더에 저장
- `hexo deploy`로 배포
