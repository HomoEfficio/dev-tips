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


## 빌드

### grunt task 직접 실행

`tranquildpeak` 디렉터리에서 hexo server를 종료한 상태에서 아래 명령 차례로 수행

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

### main repo

`homo.efficio.io` 디렉터리에서 

- `git push origin source`

### theme repo (subtree)

`homo.efficio.io` 디렉터리에서 

- `git subtree push --prefix=themes/tranquilpeak/ custom-tranquilpeak dev`

# h1
## h2
### h3
#### h4
##### h5
###### h6
