# Vultr 가상 클라우드 설정

## OS

Ubuntu 16.04 로 설치

- Ubuntu 18.04는 2018-06-23 현재 Qt기반 애플리케이션이 실행이 안 되고 netsurf가 설치가 안되는 등 불편함

Vultr Console로 연결해서 터미널 화면을 보면 apt upgrade에서 진행이 안 되는 걸로 나올 수 있는데, 엔터를 치면 로그인 가능한 화면으로 전환된다.

(아래 그림은 18.04로 되어 있으나 위와 같은 이유로 16.04로 다시 설치해서 진행했음)
![Imgur](https://i.imgur.com/Bk0uqND.png)


## 터미널 접속 후 설정

Vultr Console은 사용성이 떨어지므로 편의상 Git bash를 통해 접근해서 실행한다. 물론 다른 SSH 클라이언트를 써도 무방하다.

root 계정으로 접근하며 비밀번호는 Vultr 웹 사이트 대시보드에서 볼 수 있다.

### update & upgrade

root로 로그인 후 먼저 아래 명령을 차례로 실행한다.

```
apt update
apt upgrade
```


### 계정 추가 및 sudo 권한 부여

아래와 같이 계정을 추가하고 홈디렉토리와 셸 설정을 지정한다.

```
useradd hanmomhanda -m -s /bin/bash
```

비밀번호를 지정한다.

```
passwd hanmomhanda
```

sudo 권한을 부여한다.

```
usermod -a -G sudo hanmomhanda
```


## Xfce 데스크탑 설치

참고: https://www.hiroom2.com/2018/05/06/ubuntu-1804-xfce-en/

자원을 가장 적게 먹는 것으로 알려진 Xfce 데스크탑(xubuntu-desktop)을 설치한다.

```
su - hanmomhanda
sudo apt install xubuntu-desktop
```

설치 중 어느 데스크탑 매니저를 사용하겠냐고 물으면 lightdm 을 선택한다.


### 재부팅

![Imgur](https://i.imgur.com/ocPJC8c.png)

위와 같이 Xfce 데스크탑 설치가 완료되면 아래와 같이 Vultr 대시보드 화면에서 Restart 한다.

![Imgur](https://i.imgur.com/hzBPI5w.png)


## Xfce로 로그인

Restart 후 아래와 같이 Vultr 대시보드 화면에서 View Console 을 클릭하면

![Imgur](https://i.imgur.com/tEn1CAz.png)

Xfce 데스크탑으로 로그인할 수 있다.

![Imgur](https://i.imgur.com/R9Zd01Z.png)


## 웹 브라우저 설치 및 설정

### 기본 브라우저 오류

lightdm 기반의 Xfce 데스크탑에는 태스크바 조차 없다. 바탕화면에서 우클릭하면 메뉴를 확인할 수 있다.

Application에서 Web Browser를 선택하고 기본으로 설치되어 있는 Debian Sensible Browser를 선택하면 다음과 같이 에러가 발생한다.

![Imgur](https://i.imgur.com/fzAzYtw.png)

### netsurf-gtk 설치 및 설정

가벼운 netsurf-gtk 브라우저를 설치한다.

```
sudo apt install netsurf-gtk
```

바탕화면 우클릭으로 아래와 같이 기본 브라우저를 설정한다.

![Imgur](https://i.imgur.com/cfIHAaG.png)

![Imgur](https://i.imgur.com/anN4yvF.png)

우클릭 > Applications > Web Browser 로 netsurf-gtk를 실행

Edit > Preferences 후 Content 탭에서 Enable JavaScript 활성화

![Imgur](https://i.imgur.com/hJVTPDX.png)


## Linda 지갑 설정

### 실행 파일 다운로드

wget 으로 실행 파일 다운로드

```
wget https://github.com/Lindacoin/Linda/releases/download/2.0.0.1/Unix.Linda-qt-2.0.0.1g.tar.gz
```

압축 해제

```
tar zxvf target_dir Unix.Linda-qt-2.0.0.1g.tar.gz
```

압축 해제하면 나오는 실행 파일 실행

```
./Linda-qt
```

Linda 지갑 종료

Linda 지갑 데이터는 `~/.Linda`에 저장됨


### 부트스트랩 다운로드

아쉽게도 netsurf-gtk 로는 드랍박스에 있는 부트스트랩 다운로드 사이트에서 다운로드 불가

vultr 말고 외부에서 다운로드 후 scp 로 vultr 내로 전송. 용량이 크므로 긴 시간 소요

```
scp LindaBootstrap.zip hanmomhanda@vultr-instance-ip:/home/hanmomhanda/Downloads
```

vultr 인스턴스로 연결된 터미널에서 압축을 풀고

```
mkdir linda-bootstrap
unzip -d linda-bootstrap LindaBootstrap.zip
```

Linda 지갑 데이터가 있는 폴더에 덮어 쓰기

```
mv -f linda-bootstrap/* ~/.Linda
```

### Linda Backup 전송

```
scp linda-google-20180623-after-repairwallet.dat hanmomhanda@vultr-instance-ip:/home/hanmomhanda/Downloads
```

### Linda Backup 파일 가져오기

```
mv -f linda-google-20180623-after-repairwallet.dat ~/.Linda/wallet.dat
```

### Linda 지갑 재실행

Vultr Console 내의 바탕화면 우클릭 > Create Launcher... 클릭 후 아래와 같이 Launcher 설정

![Imgur](https://i.imgur.com/udW8kSn.png)

바탕화면 Launcher 더블 클릭하면 아래와 같이 지갑 실행

![Imgur](https://i.imgur.com/mWiW0x2.png)

### Linda Master Node 설정

https://masternodeguides.com/setup-linda-masternode-linda-masternodes/
