# 새 MacBook 설정

## 마우스 스크롤

환경설정 > 마우스

![Imgur](https://i.imgur.com/yMBUFJC.png)

## 키보드

### 한/영키 설정

![Imgur](https://i.imgur.com/YV8wrwl.png)

Monterey 에서는 Fn 키를 눌러도 환경설정으로는 불가능 -> https://andrewpage.tistory.com/95 참고

### CTRL-Command 전환

![Imgur](https://i.imgur.com/l5nEvbz.png)

### 터치바 Function키 고정

![Imgur](https://i.imgur.com/R3ADXdX.png)

### Home, End

```
$ cd ~/Library
$ mkdir KeyBindings
$ cd KeyBindings
$ vi DefaultKeyBinding.dict
```

```json
{
 /* Remap Home / End keys to be correct */
  "\UF729" = "moveToBeginningOfLine:"; /* Home */
  "\UF72B" = "moveToEndOfLine:"; /* End */
  "$\UF729" = "moveToBeginningOfLineAndModifySelection:"; /* Shift + Home */
  "$\UF72B" = "moveToEndOfLineAndModifySelection:"; /* Shift + End */
  "^\UF729" = "moveToBeginningOfDocument:"; /* Ctrl + Home */
  "^\UF72B" = "moveToEndOfDocument:"; /* Ctrl + End */
  "$^\UF729" = "moveToBeginningOfDocumentAndModifySelection:"; /* Shift + Ctrl + Home */
  "$^\UF72B" = "moveToEndOfDocumentAndModifySelection:"; /* Shift + Ctrl + End */
  "₩" = ("insertText:", "`");
  "~₩" = ("insertText:", "₩");
}

```

![Imgur](https://i.imgur.com/p1kk5j5.png)

출처: https://bimmermac.com/6426

재부팅

### Auto Repeat - 이건 안 해도 됨 (Catalina 10.15.7)

터미널에서 `defaults write -g ApplePressAndHoldEnabled -bool false` 입력 후 애플리케이션 재실행

참고: https://osxdaily.com/2011/08/04/enable-key-repeat-mac-os-x-lion/

### F기능키

아래와 같이 해줘야 인텔리제이 Alt+F12 같은 명령이랑 충돌이 발생하지 않는다

![Imgur](https://i.imgur.com/HmeKLuH.png)


## 캡처 

맥 기본 캡처 비활성화 (Skitch 로 대체, Skitch로 안 되는 건 다른 맥 기본 캡처-Shift+Ctrl+4로 가능)

![Imgur](https://i.imgur.com/1270JW4.png)

### Skitch

아래 도형 시작점에 화살표 넣기는 해제해야 꼬리부터 그려짐

![Imgur](https://i.imgur.com/dUkfmQD.png)


## git

git --version

명령 행 개발도구 설치? 예

### alias

vi ~/.gitconfig 후 아래 내용 붙여넣기

https://github.com/HomoEfficio/dev-tips/blob/master/Git-alias.md


### global .gitignore

https://gomjellie.github.io/git/2017/06/15/global-git-ignore.html


## 터미널

https://github.com/HomoEfficio/dev-tips/blob/master/zsh-config.md

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


## 데스크탑 순서 고정 - Mission Control

![Imgur](https://i.imgur.com/4svUgZa.png)

## Sublime Text

### Home, End 키 변경

Preferences > Key Bindings 에 아래 내용 추가

```
  { "keys": ["home"], "command": "move_to", "args": {"to": "bol"} },
  { "keys": ["end"], "command": "move_to", "args": {"to": "eol"} },
  { "keys": ["shift+end"], "command": "move_to", "args": {"to": "eol", "extend": true} },
  { "keys": ["shift+home"], "command": "move_to", "args": {"to": "bol", "extend": true } }
```

## brew

https://brew.sh/index_ko

## java

>brew install adoptopenjdk/openjdk/adoptopenjdk8  
>brew install adoptopenjdk/openjdk/adoptopenjdk14

### java 기본 버전 변경

https://stackoverflow.com/a/44169445

- 기본적으로 설치된 가장 높은 버전이 기본 버전으로 설정됨
- 예를 들어 14, 15가 설치돼 있고, 기본이 15로 설정돼있을 때 14를 기본으로 하려면 다음과 같이 15의 Info.plist 파일을 다른 이름으로 바꾸면 된다.
    >sudo mv /Library/Java/JavaVirtualMachines/adoptopenjdk-15.jdk/Contents/Info.plist /Library/Java/JavaVirtualMachines/adoptopenjdk-15.jdk/Contents/Info.plist.disabled

- 이렇게 하면 시스템 기본 자바 버전 지정할 때만 무시될 뿐이고 JAVA_HOME 으로 15를 지정하면 지정한대로 잘 동작한다.


## jenv

### 설치

>brew install jenv

>$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc  
>$ echo 'eval "$(jenv init -)"' >> ~/.zshrc  
>source ~/.zshrc

### 버전 관리 대상 추가

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


## nvm

>brew install nvm

설치 후 나오는 메시지에 따라 추가 작업 필요

```
==> Caveats
Please note that upstream has asked us to make explicit managing
nvm via Homebrew is unsupported by them and you should check any
problems against the standard nvm install method prior to reporting.

You should create NVM's working directory if it doesn't exist:

  mkdir ~/.nvm

Add the following to ~/.zshrc or your desired shell
configuration file:

  export NVM_DIR="$HOME/.nvm"
  [ -s "/usr/local/opt/nvm/nvm.sh" ] && . "/usr/local/opt/nvm/nvm.sh"  # This loads nvm
  [ -s "/usr/local/opt/nvm/etc/bash_completion.d/nvm" ] && . "/usr/local/opt/nvm/etc/bash_completion.d/nvm"  # This loads nvm bash_completion

You can set $NVM_DIR to any location, but leaving it unchanged from
/usr/local/opt/nvm will destroy any nvm-installed Node installations
upon upgrade/reinstall.

Type `nvm help` for further information.

Bash completion has been installed to:
  /usr/local/etc/bash_completion.d
```
