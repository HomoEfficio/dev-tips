# Mac Java jenv 환경구성

## 원하는 자바 버전 설치

https://mkyong.com/java/how-to-install-java-on-mac-osx/

## jenv 설치

>brew install jenv

### 셸 별 설정

#### Bash

>$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.bash_profile
>
>$ echo 'eval "$(jenv init -)"' >> ~/.bash_profile

#### Zsh

>$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
>
>$ echo 'eval "$(jenv init -)"' >> ~/.zshrc

## jenv 로 폴더별 사용 자바 버전 지정

https://jojoldu.tistory.com/329

