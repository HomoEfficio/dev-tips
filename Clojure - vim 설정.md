# Clojure - vim 설정

vim에서 Clojure 문법을 지원해주는 플러그인이 있는데, 설치 방법을 짧게 정리해본다.

## pathogen.vim

https://github.com/tpope/vim-pathogen

아래 명령을 실행해서 `~/.vim`에 설치

```
mkdir -p ~/.vim/autoload ~/.vim/bundle && \
curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim
```

`~/.vimrc`에 아래 내용 저장

```
execute pathogen#infect()
syntax on
filetype plugin indent on
```

이제부터는 vim 플러그인을 `~/.vim/bundle` 아래에 놓기만 하면 런타임 `runtimepath`에 자동으로 추가되어 사용 가능해진다.


## vim-fireplace

https://github.com/tpope/vim-fireplace

아래 명령을 실행해서 `~/.vim/bundle`에 `vim-fireplace`를 저장하면 끝.

```
cd ~/.vim/bundle
git clone git://github.com/tpope/vim-fireplace.git
```

## 플러그인 설정 후

원래는 흑백으로 밋밋하게 나오는데, 

![Imgur](http://i.imgur.com/JtmOPUT.png)

위 플러그인을 설치한 후에는 다음과 같이 나오고, 자동 들여쓰기도 지원이 된다.

![Imgur](http://i.imgur.com/HazmLMj.png)




