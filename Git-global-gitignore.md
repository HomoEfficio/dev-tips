# global .gitignore 파일 설정

보통은 프로젝트 마다 `.gitignore` 파일을 만들어 사용하지만,  
사실상 모든 프로젝트에서 공통적으로 무시해야 할 파일들도 있다.

이런 걸 global 파일로 설정해두면 좋다. 사용법도 아주 간단하다.

먼저 `~/.gitignore` 파일에 다음 내용을 작성해 저장한다.

```text
.DS_Store
.java-version
```

git 전역 설정으로 위 파일을 지정한다.

>git config --global core.excludesfile ~/.gitignore

