# zsh config

## oh my zsh

https://github.com/ohmyzsh/ohmyzsh#basic-installation

## solarized

https://ethanschoonover.com/solarized/

다운 받아서 압축 풀고,

터미널 > 환결설정 > 프로파일 > 가져오기 > solarized > osx-terminal.app-colors-solarized 아래에 있는 Solairzed Dark 선택

'선택 부분(드래그로 선택)' 색상 살짝 밝게 변경

## sorin 테마 적용

>vi ~/.zshrc

```
ZSH_THEME="sorin"
```

### PROMPT 변경

>vi ~/.oh-my-zsh/themes/sorin.zsh-theme

```
PROMPT='%{$fg[cyan]%}%c$(git_prompt_info) 🍺🦑🍺🍕🍺 %(!.%{$fg_bold[red]%}#.%{$fg_bold[green]%}❯)%{$reset_co
    lor%} '
```


## plugins 적용

>vi ~/.zshrc

```
plugins=(git gradle jenv node npm rust)
```



## Default Terminal

기본 터미널에 설정하는 방법은 이게 제일 나은 듯

http://ericbanisadr.com/tutorials/solarizing-the-macos-terminal.html

### Powerline Fonts

https://github.com/powerline/fonts

### Solarized dark + agnoster

근데 Solarized dark + agnoster 해도 iterm이 아닌 기본 터미널에서는 이쁘게 안 나온다.. 정도가 아니라 식별이 안 돼서 못 쓸 정도 ㅠㅜ

![Imgur](https://i.imgur.com/WpwxNSA.png)

## Default Terminal 2

이것도 괜찮은 듯 : https://www.freecodecamp.org/news/how-to-configure-your-macos-terminal-with-zsh-like-a-pro-c0ab3f3c1156/

이거도 괜찮 : https://wayhome25.github.io/etc/2017/03/12/zsh-alias/

## Dracula

https://draculatheme.com/zsh/

oh-my-zsh 설치 위치는 ~/.oh-my-zsh

ln -s /Users/1003604/gitRepo/util/dracula/zsh/dracula.zsh-theme  ~/.oh-my-zsh/themes/dracula.zsh-theme

![Imgur](https://i.imgur.com/RdZz4NL.png)

걍 아쉬운대로 Dracula 쓰는 걸로 ㅋㅋ

## sorin

oh-my-zsh 설치하면 함께 설치되는 것 중 sorin 테마가 괜찮은 듯

~/.oh-my-zsh/themes/sorin.zsh-theme 파일에 다음과 같이 맥주 이모지를 넣었다.

![Imgur](https://i.imgur.com/57J6l3M.png)

