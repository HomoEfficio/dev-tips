# k8s Secret 사용

(k8s 문서)[https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/]에 따르면 k8s Secret에 비밀번호 등을 저장하고 이를 컨테이너의 환경변수에 담아서 사용할 수 있다.


## k8s Secret 생성

Secret 에도 몇 가지 타입이 있으며 일반적인 문자열은 다음과 같이 `generic` 타입으로 생성

>from https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#create-a-secret-directly-with-kubectl  
>
>kubectl create secret generic 시크릿이름 --from-literal='key1=value1' --from-literal='key2=value2'


## 환경 변수 로딩

yaml 문서에 다음과 같이 `env` 아래에 환경변수 이름 및 참조 지정하면, 컨테이너 실행 시 k8s Secret에서 값을 읽어서 환경변수에 값 저장

```yaml
# from https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/inject/pod-single-secret-env-variable.yaml

apiVersion: v1
kind: Pod
metadata:
  name: env-single-secret
spec:
  containers:
  - name: envars-test-container
    image: nginx
    env:
    - name: SECRET_FROM_K8S
      valueFrom:
        secretKeyRef:
          name: backend-user
          key: backend-username
```


## 환경 변수 사용

spring boot 인 경우 application.yml 에서 다음과 같이 환경변수값을 읽어서 사용 가능

```yaml
security.jwt:
  base64-encoded-secret: ${SECRET_FROM_K8S}
```
