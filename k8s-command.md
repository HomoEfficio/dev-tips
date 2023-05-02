# k8s 명령 모음

## 일반 파드

### 파드 환경 변수 표시

>kubectl exec 파드이름 -- printenv



## 다중 컨테이너 파드

### 원하는 컨테이너 셸에 진입

>kbc exec -it 파드이름 -c 컨테이너이름 /bin/bash


### 원하는 컨테이너 로그 표시

>kbc logs 파드이름 컨테이너이름


### 원하는 컨테이너 환경 변수 표시

>kbc exec 파드이름 -c 컨테이너이름 -- printenv



## 시크릿

### 시크릿 값 표시

>kubectl get secret 시크릿이름 -o jsonpath='{.data}'

### 문자열 base64 decoding

>echo 'MWYyZDFlMmU2N2Rm' | base64 --decode

