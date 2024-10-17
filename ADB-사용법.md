# ADB 사용

- Android Debug Bridge 는 PC 에서 안드로이드 폰을 디버깅 할 수 있게 해주는 도구

## 큰 흐름

- PC에 ADB 설치
- USB로 PC-폰 유선 연결
- adb로 PC-폰 무선 연결
- PC-폰 유선 연결 제거
- adb로 폰 제어
- adb로 PC-폰 무선 연결 종료
- 폰 무선 연결 포트 해제
- adb 서버 종료


## ADB 설치

- https://developer.android.com/tools/releases/platform-tools 에서 운영체제 별로 다운로드 및 압축 해제
- 사용 편의를 위해 소프트링크 설정
  `sudo ln -s 다운로드위치/platform-tools/adb /usr/local/bin/adb`
- 확인
  ```
  adb version

  Android Debug Bridge version 1.0.41
  Version 35.0.2-12147458
  Installed as /usr/local/bin/adb
  Running on Darwin 23.6.0 (arm64)
  ```

## USB로 PC-폰 유선 연결

- PC 에서 다음 명령으로 연결된 디바이스 없음 확인 및 데몬 실행
  ```
  adb devices
  
  * daemon not running; starting now at tcp:5037
  * daemon started successfully
  List of devices attached
  ```
- USB로 PC-폰 유선 연결하면 폰에 뜨는 '휴대전화 데이터에 접근 허용' 팝업에서 '허용' 선택
- PC 에서 다음 명령으로 유선 연결 확인
  ```
  adb devices

  List of devices attached
  R3CX60E4LYA	device
  ```

## adb로 PC-폰 무선 연결

- 폰의 TCP/IP 5555 포트 활성화
  ```
  adb tcpip 5555
  
  restarting in TCP mode port: 5555
  ```
  
- 폰 설정에서 무선 연결 IP 확인: 192.168.45.224 라고 가정
- 폰에 무선 연결
  ```
  adb connect 192.168.45.224:5555
  
  connected to 192.168.45.224:5555
  ```

- 연결된 디바이스 확인

  ```
  adb devices
  
  List of devices attached
  R3CX60E4LYA	device
  192.168.45.224:5555	device
  ```

- 유선 연결 해제
  - 유선 연결을 해제하지 않으면 adb 명령 실행 시 마다 `adb -s 'R3CX60E4LYA' ...` 이나 `adb -s '192.168.45.224:5555` 이렇게 연결 대상을 지정해줘야 해서 불편
  - 그냥 USB 선을 뽑아도 되고 아래 명령 실행 후 선 뽑으면 됨
    ```
    adb disconnect R3CX60E4LYA
    
    disconnected R3CX60E4LYA
    ```

## adb로 폰 제어

- 여러 adb 명령으로 무선 연결을 통해 폰 제어
  - 아래는 adb 명령으로 DeepLink를 실행하는 예
  ```
  adb shell am start -a android.intent.action.VIEW -d "SAMPLE-SCHEME://HOME/..."

  Starting: Intent { act=android.intent.action.VIEW dat=SAMPLE-SCHEME://HOME/... }
  ```

## adb로 PC-폰 무선 연결 종료

- 무선 연결 종료
  ```
  adb disconnect 192.168.45.224:5555
  
  disconnected 192.168.45.224:5555
  ```
- 연결 디바이스 목록 조회 결과 없음
  ```
  adb devices 
  
  List of devices attached
  ```

## 폰 무선 연결 포트 해제

- `adb disconnect` 명령만으로는 폰의 5555 포트가 해제되지 않으며, 5555에서 계속 listening 하고 있으므로 명시적으로 해제 필요
- USB 모드로 연결하면 폰의 5555 포트가 해제됨
- 따라서 USB 선으로 PC-폰 다시 연결한 후 다음 명령으로 USB 모드 활성화하면 폰의 5555 포트 해제됨
  ```
  adb usb
  
  restarting in USB mode
  ```
- USB 선을 뽑은 후 다음 명령으로 USB 연결 해제 확인
  ```
  adb devices
  
  List of devices attached
  
  ```

## adb 서버 종료

- 종료

  ```
  adb kill-server
  
  
  ```
