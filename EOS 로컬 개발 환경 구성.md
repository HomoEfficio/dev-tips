# EOS 로컬 개발 환경 구성

공식 문서인 https://github.com/EOSIO/eos/wiki/Local-Environment 를 기준으로 약간의 커스터마이징을 가미해서 EOS 로컬 개발 환경을 직접 구성해보자.

## 사전 조건

### OS

공식 문서에 아래와 같이 안내 되어 있다. 이 문서는 Linux Mint 18.3에서 빌드하고 작성했다.

1. Amazon 2017.09 and higher.
1. Centos 7.
1. Fedora 25 and higher (Fedora 27 recommended).
1. Mint 18.
1. Ubuntu 16.04 (Ubuntu 16.10 recommended).
1. MacOS Darwin 10.12 and higher (MacOS 10.13.x recommended).

### 용량

- RAM: 8G 이상
- Disk: 20G 이상


## 소스 가져오기

문서에는 EOSIO의 리포지토리를 직접 clone하도록 안내하고 있지만, Pull Request도 보내면서 개발하려면 각자 fork 뜬 리포지토리를 clone하는 것이 좋다.

각자 fork 뜬 리포지토리의 URL을 `https://github.com/YOUR_USERNAME/eos`라고 하면, 다음과 같이 clone 하면 된다.

>$ git clone https://github.com/YOUR_USERNAME/eos --recursive

### 원본 리포지토리와 Sync를 위한 upstream 추가

https://help.github.com/articles/configuring-a-remote-for-a-fork/ 를 참고해서 다음을 실행한다.

1. 현재 remote 확인

    >$ git remote -v

    ```
    origin  https://github.com/YOUR_USERNAME/eos.git (fetch)
    origin  https://github.com/YOUR_USERNAME/eos.git (push)
    ```

1. 원본 리포지토리를 가리키는 upstream 추가

    >$ git remote add upstream https://github.com/EOSIO/eos.git

1. upstream 확인

    >$ git remote -v

    ```
    origin    https://github.com/YOUR_USERNAME/eos.git (fetch)
    origin    https://github.com/YOUR_USERNAME/eos.git (push)
    upstream  https://github.com/EOSIO/eos.git (fetch)
    upstream  https://github.com/EOSIO/eos.git (push)
    ```

### 원본 리포지토리와 Sync

그냥 일반적인 remote 리포지토리와 Sync하는 것과 다르지 않다.

https://help.github.com/articles/syncing-a-fork/ 를 참고하되, `git merge` 뿐아니라 `git rebase`도 물론 쓸 수 있다.


## 빌드

EOSIO 에서 제공하는 빌드 스크립트를 실행하면 된다.

>$ cd eos
>
>$ ./eosio_build.sh

### 자잘한 버그 수정

그런데 우분투 용 빌드 스크립트에는 의존 관계 구성 관련 사소한 버그가 있어서, 빌드 스크립트를 살짝 고치고 Pull Request를 날려두었다.

https://github.com/EOSIO/eos/pull/2979/files

이 글을 보는 시점에 Pull Request가 반영되어있다면 아래 내용은 무시하고 다음 단원으로 건너뛰면 된다.

원본 그대로 실행하면 먼저 다음과 같이 라이브러리를 설치해야 한다는 안내가 나온다. 

![Imgur](https://i.imgur.com/FdCnNY9.png)

메시지가 살짝 깨져 나오지만 일단 넘어가고, 1 을 입력하고 엔터를 치면 설치가 안 되고 다음과 같은 에러가 난다.

![Imgur](https://i.imgur.com/9EyuWR0.png)

위의 Pull Request 링크를 참고해서 `scripts/eosio_build_ubuntu.sh`파일을 수정하고 다시 `./eosio_build.sh`를 실행해서, 라이브러리 설치 문의 시 1을 입력하고 엔터를 치면 다음과 같이 라이브러리 설치가 정상적으로 진행된다.

![Imgur](https://i.imgur.com/uI88CPW.png)

## 기다림

라이브러리 설치 후 Boost, mongoDB, LLVM 등도 빌드하는데 넉넉 잡아 120분 정도 소요되었다. 느긋한 기다림이 필요하다.

몇 가지 화면 캡처를 떠놨으니 미리 구경해보자.

#### Boost

별 문제없이 설치된다.

![Imgur](https://i.imgur.com/QS1BX2K.png)

#### mongoDB

별 문제없이 설치된다.

![Imgur](https://i.imgur.com/TgsjIC5.png)

![Imgur](https://i.imgur.com/KhhsDyP.png)

![Imgur](https://i.imgur.com/AvnGEG0.png)

#### secp256k1-zkp

mongoDB 설치까지 완료된 후 지루할만 하니 에러가 나준다.

![Imgur](https://i.imgur.com/HqkYJ7q.png)

`aclocal`이 없다는 얘긴데 `automake`를 설치하면 해결된다.

>$ sudo apt install automake

![Imgur](https://i.imgur.com/Dy1kTfc.png)

`automake` 설치 완료 되면 다시 `./eosio_build.sh`를 실행한다.

![Imgur](https://i.imgur.com/tkyTrmz.png)

아까 받아둔 파일이 있어서 문제라고 하니 개뿐하게 지워주고, 다시 `./eosio_build.sh`를 실행한다.

![Imgur](https://i.imgur.com/QBGDIzg.png)

#### LLVM

이제 secp256k1-zkp도 깔끔하게 설치되고 다음 단계인 LLVM으로 넘어간다. LLVM은 별 문제없이 설치된다.

![Imgur](https://i.imgur.com/KOsUI6Q.png)

![Imgur](https://i.imgur.com/zICvgid.png)

#### EOSIO

이제 드디어 EOSIO 설치로 넘어간다. EOSIO는 별 문제없이 설치된다.

![Imgur](https://i.imgur.com/wcPRLJl.png)

![Imgur](https://i.imgur.com/G4bcK1s.png)

드디어 빌드가 완성 되었다. 총 90분 정도 걸렸다고 나오는데, 중간에 오류 나기 전에 설치되는 것들까지 감안하면 CPU i5-2500 3.3GHz, RAM 8G 정도로 넉넉잡아 2시간은 걸린 것 같다.

## 테스트

위 빌드 결과에 안내해준 대로 테스트를 수행해보자.

![Imgur](https://i.imgur.com/EVWbJK4.png)

![Imgur](https://i.imgur.com/SQh7tA0.png)

테스트도 시간은 10여분 정도 걸렸지만 문제없이 모두 통과한다.

