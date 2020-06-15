# 새 MacBook 설정

## 마우스 스크롤

환경설정 > 마우스

사진 

## 키보드

### 한/영키 설정

사진

### CTRL-Command 전환

사진

## 터미널

https://github.com/HomoEfficio/dev-tips/blob/master/zsh-config.md

## git

git --version

명령 행 개발도구 설치? 예

alias: https://github.com/HomoEfficio/dev-tips/blob/master/Git-alias.md

## brew

https://brew.sh/index_ko

## java

>brew cask install adoptopenjdk/openjdk/adoptopenjdk8
>brew cask install adoptopenjdk/openjdk/adoptopenjdk14

## jenv

### 설치

>brew install jenv

>$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc  
>$ echo 'eval "$(jenv init -)"' >> ~/.zshrc
>source ~/.zshrc

혹시 source ~/.zshrc 실행 후 다음과 같은 오류 나면 해당 디렉토리로 가서 안내받은 명령(`compaudit | xargs chmod g-w,o-w`) 실행

```
[oh-my-zsh] Insecure completion-dependent directories detected:
drwxrwxr-x  3 1003604  admin   96  6 15 17:59 /usr/local/share/zsh
drwxrwxr-x  4 1003604  admin  128  6 15 18:01 /usr/local/share/zsh/site-functions

[oh-my-zsh] For safety, we will not load completions from these directories until
[oh-my-zsh] you fix their permissions and ownership and restart zsh.
[oh-my-zsh] See the above list for directories with group or other writability.

[oh-my-zsh] To fix your permissions you can do so by disabling
[oh-my-zsh] the write permission of "group" and "others" and making sure that the
[oh-my-zsh] owner of these directories is either root or your current user.
[oh-my-zsh] The following command may help:
[oh-my-zsh]     compaudit | xargs chmod g-w,o-w

[oh-my-zsh] If the above didn't help or you want to skip the verification of
[oh-my-zsh] insecure directories you can set the variable ZSH_DISABLE_COMPFIX to
[oh-my-zsh] "true" before oh-my-zsh is sourced in your zshrc file.
```

### 버전 관리 대상 추가=

>jenv add /Library/Java/JavaVirtualMachines/adoptopenjdk-8.jdk/Contents/Home/
>jenv add /Library/Java/JavaVirtualMachines/adoptopenjdk-14.jdk/Contents/Home/

### 로컬 설정

jdk 이름 확인: jenv versions

해당 디렉터리에서 `jenv local {{{사용할jdk이름}}}

## intellij

사이트에서 다운로드로 설치

### 설정 import

github > etc > settings 파일 다운로드 및 import

jdk 경로도 같이 import 되는데 다를 수 있으므로 불필요한 건 import 안 하는 것도 방법

