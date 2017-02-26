# Mac + VirtualBox + CentOS

[실무로 배우는 빅데이터 기술](http://www.aladin.co.kr/shop/wproduct.aspx?ItemId=101507322)이라는 책을 따라 환경 구성을 하다가, 네트워크 부분이 책에 나온 내용대로 하면 인터넷 연결이 안 되는 등 문제가 발생해서 불필요한 설정은 제외하고 필요한 내용만 정리해본다.

버전은 다음과 같다.
- Mac OS X El Capitan 10.11.6
- VirtualBox 5.1.14 r11294
- CentOS 7.3.1611 (LiveGNome.iso)

참고로 Mac + VirtualBox + CentOS의 조합은 윈도우 + VMware + Linux의 조합에 비해 뭔가 느리고, 또 아무런 안내 사항이 없는 블랙스크린이 수 분 동안 떠 있는 경우도 많은데, 잘못된 것은 아니니 인내심을 가지고 그냥 기다리면 된다.

# OS 선택

책에는 `Other Linux 64bit`를 선택하라고 나오는데, **CentOS는 `Redhat 64bit`를 선택**하면 된다.

실제로 `Redhat 64bit`를 선택하고 설치하면 마우스 이동, 키보드 입력 등의 밀림 현상이 더 적다.

# 키보드 선택

**`한국어(Hangul)`로 표시된 것을 선택**하면 `Shift+Space`로 한/영 전환이 된다. 즉, **`101/104키보드 호환`으로 선택하지 말 것**.

# 네트워크 설정

네트워크 설정은 먼저 VirtualBox에서 가상 네트워크 A를 생성하고, CentOS 가상 머신에서 A를 이용해서 가상 머신의 로컬 네트워크 B를 설정한다.

그리고 마지막으로 호스트 OS의 hosts 파일에 게스트 OS의 IP와 호스트 이름을 설정해주면 된다.

## VirtualBox

http://solatech.tistory.com/277 여기에 링크되어 있는 pdf에 VirtualBox의 네트워크 설정에 대한 내용이 잘 정리되어 있다.

그 중 VirtualBox + CentOS에 사용되는 네트워크는 아래의 2가지이며, 여러 개의 가상 머신을 만들더라도 한 번씩만 생성해주면 된다.

### NAT network

게스트 OS 내부에서 홀로 외부의 인터넷을 사용할 수 있게 해주는 일반 NAT와 다르게, 다수의 VM에서 하나의 게이트웨이를 통해 외부 인터넷을 사용할 수 있게 해준다.

이 부분은 책과 동일하게 설정하면 된다.

- 설정 위치: VirtualBox > 파일 > 환경 설정 > 네트워크 > NAT 네트워크 > 추가
- 설정 값
    - 네트워크 이름: NatNetwork
    - 네트워크 CIDR: 10.0.2.0/24
    - 네트워크 옵션: DHCP 지원 체크

### Host-only network

게스트 OS와 호스트 OS가 통신할 수 있는 폐쇄망으로, 설정을 마치면 호스트 OS에 vboxnet0라는 가상 NIC가 하나 생성되며, 이를 통해 게스트 OS와 호스트 OS가 ssh 등의 통신을 할 수 있다.

이 부분은 책과 동일하게 설정하면 된다.

- 설정 위치: VirtualBox > 파일 > 환경 설정 > 네트워크 > 호스트 전용 네트워크 > 추가
- 설정 값
    - 어댑터 탭
        - IPv4 주소: 192.168.56.1
        - IPv4 서브넷 마스크: 255.255.255.0
    - DHCP 서버 탭
        - 서버 주소: 192.168.56.100
        - 서버 마스크: 255.255.255.0
        - 최저 주소 한계: 192.168.56.101
        - 최고 주소 한계: 192.168.56.254

## 가상 머신

가상 머신 생성 후 가상 머신을 실행하기 전에 네트워크 설정을 구성해준다. 아래 내용은 가상 머신을 생성할 때마다 반복적으로 지정해줘야 한다.

- 설정 위치: VirtualBox > 가상 머신 선택 > 설정 > 네트워크
- 설정 값
    - 어댑터1 탭
        - 네트워크 어댑터 사용하기 체크
        - 다음에 연결됨: NAT 네트워크
        - 이름: NatNetwork
        - 고급 클릭
        - 어댑터 종류: Intel PRO/1000 MT Desktop (82540EM)
        - MAC 주소: 자동 생성되어 있음
        - 케이블 연결됨 체크
    - 어댑터2 탭
        - 네트워크 어댑터 사용하기 체크
        - 다음에 연결됨: 호스트 전용 어댑터
        - 이름: vboxnet0
        - 고급 클릭
        - 어댑터 종류: Intel PRO/1000 MT Desktop (82540EM)
        - 무작위 모드: 모두 허용
        - MAC 주소: 자동 생성되어 있음
        - 케이블 연결됨 체크

## 가상 머신 부팅 후 설정

가상 머신 부팅 후에는 결론적으로 호스트 이름과 /etc/hosts 파일만 설정하면 된다.

나머지는 설정해 줄 필요 없다.

### 호스트 이름

아래의 명령으로 지정한다.

`sudo hostnamectl set-hostname <new hostname>`

### /etc/hosts

가상 머신끼리 호스트 이름으로 통신할 수 있도록 아래의 내용만 추가해준다.

```
192.168.56.101   server01.hadoop.com
192.168.56.102   server02.hadoop.com
192.168.56.103   server03.hadoop.com
```

### /etc/sysconfig/network-scripts/ifcfg-eth0

결론적으로 설정해 줄 필요 없다.

책에서는 /etc/sysconfig/network-scripts/ifcfg-eth0을 추가하고 정보를 입력하게 되어 있는데, 그 전에 `ifconfig -a`를 실행해서 다음과 같이 `enpOs3`과  `enpOs8`에 대한 정보가 표시되면 `ifcfg-eth0`를 구성해줄 필요 없이 그냥 되어있는대로 사용해도 된다.

책에 있는대로 `ifcfg-eth0`를 구성하면, 어댑터2(호스트 전용 어댑터)가 활성화 된 상태에서는 인터넷 연결이 안 될 수 있다.

### /etc/udev/rules.d/70-persistent-net.rules

결론적으로 설정해 줄 필요 없다.

책에는 있지만 막상 가상 머신의 터미널에서 보면 `/etc/udev/rules.d/70-persistent-net.rules`라는 파일은 존재하지 않고 대신에 주석 외에는 내용이 없는 `/etc/udev/rules.d/70-persistent-ipoib.rules`라는 파일만 있다.

### /etc/sysconfig/network

결론적으로 설정해 줄 필요 없다.

## 호스트 OS 설정

`/etc/hosts` 파일에 아래 내용만 추가해주면 된다.

```
192.168.56.101   server01
192.168.56.102   server02
192.168.56.103   server03
```

## 전체 설정 확인

### 가상 머신 하나씩 띄우고 각 가상 머신 상에서

- `hostname`으로 호스트 이름 설정 확인
- 브라우저로 인터넷 연결 확인

### 가상 머신 모두 띄우고 각 가상 머신 상에서

- `ssh bigdata@server01`, `ssh bigdata@server02`, `ssh bigdata@server03`으로 서로에게 연결 가능한지 확인

### 가상 머신 모두 띄우고 호스트 OS 상에서

- `ssh bigdata@server01`, `ssh bigdata@server02`, `ssh bigdata@server03`으로 각 가상 머신에 연결 가능한지 확인




