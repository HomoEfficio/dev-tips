# Vultr 가상 클라우드 설정

## OS

Ubuntu 18.04 로 설치

Vultr Console로 연결해서 터미널 화면을 보면 apt upgrade에서 진행이 안 되는 걸로 나올 수 있는데, 엔터를 치면 로그인 가능한 화면으로 전환된다.

![Imgur](https://i.imgur.com/Bk0uqND.png)


## 터미널 접속 후 설정

Vultr Console은 사용성이 떨어지므로 편의상 Git bash를 통해 접근해서 실행한다. 물론 다른 SSH 클라이언트를 써도 무방하다.

root 계정으로 접근하며 비밀번호는 Vultr 웹 사이트 대시보드에서 볼 수 있다.

### update & upgrade

root로 로그인 후 먼저 아래 명령을 차례로 실행한다.

```
apt get update
apt get upgrade
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

자원을 가장 적게 먹는 것으로 알려진 Xfce 데스탑을 설치한다.

```
su - hanmomhanda
sudo apt install xubuntu-desktop
```

설치 중 어느 데스크탑 매니저를 사용하겠냐고 물으면 lightdm 을 선택한다.


### 재부팅

![Imgur](https://i.imgur.com/ocPJC8c.png)

위와 같이 Xfce 설치가 완료되면 아래와 같이 Vultr 대시보드 화면에서 Restart 한다.

![Imgur](https://i.imgur.com/hzBPI5w.png)


## Xfce로 로그인

Restart 후 아래와 같이 Vultr 대시보드 화면에서 View Console 을 클릭하면

![Imgur](https://i.imgur.com/tEn1CAz.png)

Xfce 데스크탑으로 로그인할 수 있다.

![Imgur](https://i.imgur.com/R9Zd01Z.png)


## 웹 브라우저 실행

lightdm 기반의 Xfce 데스크탑에는 태스크바 조차 없다. 바탕화면에서 우클릭하면 메뉴를 확인할 수 있다.

Application에서 Web Browser를 선택하고 기본으로 설치되어 있는 Debian Sensible Browser를 선택하면 다음과 같이 에러가 발생한다.

![Imgur](https://i.imgur.com/fzAzYtw.png)






