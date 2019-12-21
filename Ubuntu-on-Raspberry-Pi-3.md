# Ubuntu on Raspberry Pi 3

## 준비물

- Raspberry Pi 3
- MicroSD 카드
- USB Keyboard
- Monitor + HDMI 케이블
- Ubuntu 이미지를 다운로드 하고 MicroSD에 Ubuntu 이미지를 Flash 할 인터넷 연결 컴퓨터

## Ubuntu 이미지 파일 다운로드

- https://ubuntu.com/download/raspberry-pi
  - `64-bit for Raspberry Pi 3 and 4` 다운로드

다운로드 하는 동안 아래의 'Image Flash 프로그램 다운로드 및 설치', 'MicroSD 메모리 카드 준비' 수행

## Image Flash 프로그램 다운로드 및 설치

- https://sourceforge.net/projects/win32diskimager/files/latest/download

## MicroSD 메모리 카드 준비

윈도우 10 기준

- MicroSD 메모리 카드를 컴퓨터에 연결
  - 포맷 등의 팝업이 뜨면 모두 무시
  - ![Imgur](https://i.imgur.com/93adzTR.png)
  - ![Imgur](https://i.imgur.com/UM67S8u.png)

- 기존에 사용하던 카드라 파티션이 나뉘어 있는 경우 파티션 삭제
  - 윈도우 버튼 우클릭 > 디스크 관리자 실행
  - ![Imgur](https://i.imgur.com/cGeAsxt.png)
  - MicroSD 메모리에 있던 볼륨(파티션) 모두 삭제
  - ![Imgur](https://i.imgur.com/hr9w14r.png)
  - 아래와 같이 모두 삭제 되면 준비 완료
  - ![Imgur](https://i.imgur.com/rPKVxcJ.png)

## MicroSD 메모리 카드에 Ubuntu 이미지 파일 Flash

- Ubuntu 이미지 파일 압축 해제
  - 필요 시 반디집 설치 후 해제
- Image Flash 프로그램 실행
  - Ubuntu 이미지 파일 위치 지정 및 Flash 할 대상 디바이스(MicroSD 카드) 지정 후 Write
  - ![Imgur](https://i.imgur.com/hFTpk31.png)
  - ![Imgur](https://i.imgur.com/p2inqJO.png)
- 사양에 따라 다르겠지만 약 2~3분 후 다음과 같이 Flash 완료
  - ![Imgur](https://i.imgur.com/00yqpDc.png)
- 포맷 팝업창이 다시 뜨면 무시
  - ![Imgur](https://i.imgur.com/uV31oML.png)
- 탐색기에서 꺼내기 후 MicroSD 메모리 카드를 빼낸다.

## Raspberry Pi 부팅

- Ubuntu 이미지가 Flash 된 MicroSD 카드를 Raspberry PI 에 연결
- 모니터와 연결된 HDMI 케이블과 키보드를 Raspberry PI 에 연결
- Raspberry PI 전원 연결
- Ubuntu 로 부팅되며 몇 분간 자동 설정 후 로그인 프롬프트 나옴

## 최초 로그인 및 비번 변경

- 초기 아이디/비번은 ubuntu/ubuntu 이며 로그인 후 비번 변경 필요
