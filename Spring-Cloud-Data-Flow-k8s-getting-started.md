# Spring Cloud Data Flow with k8s

k8s 기반 Spring Cloud Data Flow 실습 자료인 https://dataflow.spring.io/getting-started/ 에서 k8s 플랫폼 기준 내용 요약 및 실제 실행 내용 추가

# Install k8s

k8s를 로컬에서 구성하는 데 사용하는 도구도 여러가지가 있다.


레퍼런스 문서는 minikube 기준으로 돼 있지만,  
[관련 자료](https://medium.com/dev-genius/kubernetes-for-local-development-a6ac19f1d1b2) 검토 결과,  
multiple node cluster 가능한 Kind(K8s INside Docker) 선택

## Kind

k8s 로컬 구성 지원 도구

### 설치

https://kind.sigs.k8s.io/docs/user/quick-start/

>brew install kind

```
...
==> Summary
🍺  /usr/local/Cellar/kind/0.9.0: 8 files, 9.2MB
```

클러스터가 생성 전에는 `~/.kube/config` 파일의 내용이 다음과 같이 거의 비어 있다.

```
apiVersion: v1                                                       
kind: Config
preferences: {}
```

### 클러스터 생성 및 실행

3-node(1 control + 2 worker) 클러스터 모드로 실행

https://kind.sigs.k8s.io/docs/user/quick-start/#advanced

```yml
kind-minimal-cluster-example-config.yml

# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

>kind create cluster --config kind-minimal-cluster-example-config.yml --name my-k8s-cluster

```
🍺🦑🍺🍕🍺 ❯ kind create cluster --config kind-minimal-cluster-example-config.yml --name my-k8s-cluster
Creating cluster "my-k8s-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼 
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-my-k8s-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-my-k8s-cluster

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
🍺🦑🍺🍕🍺 ❯ 
```

클러스터 생성 후 `~/.kube/config` 파일 내용이 다음과 같이 채워진다.

```
apiVersion: v1                                                                                                                                                                    
clusters:
- cluster:
    certificate-authority-data: XXX...생략...
    server: https://127.0.0.1:58878
  name: kind-my-k8s-cluster
contexts:
- context:
    cluster: kind-my-k8s-cluster
    user: kind-my-k8s-cluster
  name: kind-my-k8s-cluster
current-context: kind-my-k8s-cluster
kind: Config
preferences: {}
users:
- name: kind-my-k8s-cluster
  user:
    client-certificate-data: YYY...생략...
```

>kubectl cluster-info --context kind-my-k8s-cluster

```
Kubernetes master is running at https://127.0.0.1:58878
KubeDNS is running at https://127.0.0.1:58878/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### 클러스터 종료

>kind delete cluster --name my-k8s-cluster

```
🍺🦑🍺🍕🍺 ❯ kind delete cluster --name my-k8s-cluster
Deleting cluster "my-k8s-cluster" ...
🍺🦑🍺🍕🍺 ❯ kubectl get all
The connection to the server localhost:8080 was refused - did you specify the right host or port?
🍺🦑🍺🍕🍺 ❯ 
```

종료 후 `~/.kube/config` 파일 내용은 다시 초기화된다.

```
apiVersion: v1                                                       
kind: Config
preferences: {}
```


## kubectl 설치

kind가 kubectl을 반드시 필요로 하진 않지만 여러 예제가 kubectl을 사용하므로 설치

https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-homebrew-on-macos

>brew install kubectl

```
...
==> Summary
🍺  /usr/local/Cellar/kubernetes-cli/1.20.1: 246 files, 46.1MB
```


## 클러스터 정보 확인 

>kubectl cluster-info --context kind-kind

```
🍺🦑🍺🍕🍺 ❯ kubectl cluster-info --context kind-my-k8s-cluster
Kubernetes master is running at https://127.0.0.1:58878
KubeDNS is running at https://127.0.0.1:58878/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
🍺🦑🍺🍕🍺 ❯ 
```

`https://127.0.0.1:58878`에 접속하면 다음과 같이 403 인증 에러 발생. 일단 넘어감

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```

`kubectl cluster-info dump`를 실행하면 엄청난 양의 정보가 출력됨


# k8s 구성 파일

kubectl로 배포되는 여러 컴포넌트의 yml 파일이 git에 있어서 clone으로 가져옴

## git repo clone

>git clone https://github.com/spring-cloud/spring-cloud-dataflow  
>cd spring-cloud-dataflow  
>git checkout v2.7.0



# k8s-SCDF 구성

크게 다음 5개 카테고리로 구성

- Message Broker
- Database
- Monitoring
- SCDF



# Meesage Broker

RabbitMQ와 Kafka 중 [선택](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/#choose-a-message-broker), 여기에서는 RabbitMQ 선택

## RabbitMQ

### 배포

`src/kubernetes/rabbitmq/rabbit-svc.yaml` 파일을 열어보면 rabbitmq 서비스의 타입은 지정돼있지 않다.

>kubectl create -f src/kubernetes/rabbitmq

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/rabbitmq
deployment.apps/rabbitmq created
service/rabbitmq created
🍺🦑🍺🍕🍺 ❯ 
```

### 배포 확인

>kubectl get all -l app=rabbitmq

```
🍺🦑🍺🍕🍺 ❯ kubectl get all -l app=rabbitmq                                                                                              ✹ ✭
NAME                            READY   STATUS    RESTARTS   AGE
pod/rabbitmq-78b6c44c49-bpln4   1/1     Running   0          77s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/rabbitmq   ClusterIP   10.96.135.124   <none>        5672/TCP   77s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rabbitmq   1/1     1            1           77s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/rabbitmq-78b6c44c49   1         1         1       77s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### 배포 회수

>kubectl delete all -l app=rabbitmq



# Database

MySQL, Postgres, H2 중 선택

## MySQL

### 배포

`src/kubernetes/mysql/mysql-svc.yaml` 파일을 열어보면 rabbitmq 서비스의 타입은 지정돼있지 않다.

>kubectl create -f src/kubernetes/mysql/

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/mysql/
deployment.apps/mysql created
persistentvolumeclaim/mysql created
secret/mysql created
service/mysql created
🍺🦑🍺🍕🍺 ❯  
```

### 배포 확인

>kubectl get all -l app=mysql

```
🍺🦑🍺🍕🍺 ❯ kubectl get all -l app=mysql
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-58f79dbc8c-n7slk   1/1     Running   0          31s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/mysql   ClusterIP   10.96.183.164   <none>        3306/TCP   31s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql   1/1     1            1           31s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-58f79dbc8c   1         1         1       31s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### 배포 회수

>kubectl delete all,pvc,secrets -l app=mysql



# Monitoring

Prometheus로 지표 수집, Grafana 대시보드 사용

문서에는 WaveFront도 나오지만 오픈 소스가 아니며 선택인 듯 해서 skip

## Prometheus

모니터링 지표 수집을 위해 Prometheus 배포

### prometheus-proxy 배포

`src/kubernetes/prometheus-proxy/prometheus-proxy-service.yaml`를 열어보면 다음과 같이 타입이 `LoadBalancer`로 지정돼 있다.

```
apiVersion: v1                                      
kind: Service
metadata:
  name: prometheus-proxy
  labels:
    app: prometheus-proxy
spec:
  selector:
    app: prometheus-proxy
  ports:
    - name: scrape
      port: 8080
      targetPort: 8080
    - name: rsocket
      port: 7001
      targetPort: 7001
  # type: LoadBalancer
  type: NodePort
```

외부 클라우드 제공자의 로드밸런서를 사용하고 있지 않으므로 `LoadBalancer`를 `NodePort`로 대체한다.

>kubectl create -f src/kubernetes/prometheus-proxy/

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/prometheus-proxy
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/prometheus-proxy created
deployment.apps/prometheus-proxy created
service/prometheus-proxy created
serviceaccount/prometheus-proxy created
🍺🦑🍺🍕🍺 ❯  
```

### prometheus-proxy 배포 확인

>kubectl get all -l app=prometheus-proxy

```
🍺🦑🍺🍕🍺 ❯ kubectl get all -l app=prometheus-proxy
NAME                                    READY   STATUS    RESTARTS   AGE
pod/prometheus-proxy-5b958c7fd4-xk5lq   1/1     Running   0          24m

NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
service/prometheus-proxy   NodePort   10.96.20.147   <none>        8080:32036/TCP,7001:31986/TCP   95s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-proxy   1/1     1            1           24m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-proxy-5b958c7fd4   1         1         1       24m
🍺🦑🍺🍕🍺 ❯ 
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

서비스의 네트워크는 다음과 같이 확인 가능하다.

>kubectl describe services/prometheus-proxy

```
🍺🦑🍺🍕🍺 ❯ kubectl describe services/prometheus-proxy
Name:                     prometheus-proxy
Namespace:                default
Labels:                   app=prometheus-proxy
Annotations:              <none>
Selector:                 app=prometheus-proxy
Type:                     NodePort
IP:                       10.96.20.147
Port:                     scrape  8080/TCP
TargetPort:               8080/TCP
NodePort:                 scrape  32036/TCP
Endpoints:                10.244.1.3:8080
Port:                     rsocket  7001/TCP
TargetPort:               7001/TCP
NodePort:                 rsocket  31986/TCP
Endpoints:                10.244.1.3:7001
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
🍺🦑🍺🍕🍺 ❯  
```

### prometheus-proxy 배포 회수

>kubectl delete all,cm,svc -l app=prometheus-proxy

### prometheus 배포

>kubectl create -f src/kubernetes/prometheus/prometheus-configmap.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/prometheus/prometheus-configmap.yaml
configmap/prometheus created
🍺🦑🍺🍕🍺 ❯ 
```

>kubectl create -f src/kubernetes/prometheus/prometheus-deployment.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/prometheus/prometheus-deployment.yaml
deployment.apps/prometheus created
🍺🦑🍺🍕🍺 ❯ 
```

`src/kubernetes/prometheus/prometheus-service.yaml` 파일을 열어보면 prometheus 서비스의 타입은 지정돼있지 않다.

>kubectl create -f src/kubernetes/prometheus/prometheus-service.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/prometheus/prometheus-service.yaml
service/prometheus created
🍺🦑🍺🍕🍺 ❯ 
```

### prometheus 배포 확인

>kubectl get all -l app=prometheus

```
🍺🦑🍺🍕🍺 ❯ kubectl get all -l app=prometheus
NAME                              READY   STATUS    RESTARTS   AGE
pod/prometheus-7fbc58dcf5-24bvv   1/1     Running   0          20s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/prometheus   ClusterIP   10.96.87.216   <none>        9090/TCP   11s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus   1/1     1            1           20s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-7fbc58dcf5   1         1         1       20s
🍺🦑🍺🍕🍺 ❯ 
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### prometheus 배포 회수

>kubectl delete all,cm,svc -l app=prometheus


## Grafana

### 배포

`src/kubernetes/grafana/grafana-service.yaml`를 열어보면 다음과 같이 타입이 `LoadBalancer`로 지정돼 있는데, 외부 클라우드 제공자의 로드밸런서를 사용하고 있지 않으므로 `LoadBalancer`를 `NodePort`로 대체한다.

```
apiVersion: v1                                                              
kind: Service
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  selector:
    app: grafana
  # type: LoadBalancer
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
```

>kubectl create -f src/kubernetes/grafana/

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/grafana/
configmap/grafana created
deployment.apps/grafana created
secret/grafana created
service/grafana created
🍺🦑🍺🍕🍺 ❯ 
```

### 배포 확인

>kubectl get all -l app=grafana

```
🍺🦑🍺🍕🍺 ❯ kubectl get all -l app=grafana
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-7dc7d95456-bhj6l   1/1     Running   0          43s

NAME              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/grafana   NodePort   10.96.116.14   <none>        3000:30050/TCP   43s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           43s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-7dc7d95456   1         1         1       43s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### 배포 회수

>kubectl delete all,cm,svc,secrets -l app=grafana


# SCDF

## prometheus-proxy 관련 설정

Prometheus가 Data Flow 서버와 Skipper 서버 지표를 수집할 수 있도록 설정해줘야 한다.

Data Flow 서버 설정 파일 `src/kubernetes/server/server-config.yaml`에는 이미 Prometheus가 구성돼 있고, Skipper 서버 설정 파일에는 구성돼 있지 않다. 따라서 `src/kubernetes/server/server-config.yaml` 내용 참고해서 아래와 같이 Skipper 서버 설정을 변경하면 된다.

`src/kubernetes/skipper/skipper-config-{kafka|rabbit}.yaml` 파일에 아래 `management` 내용을 `data > application.yaml`아래에 추가한다.

```
data:
  application.yaml: |-
    ...
    management:
      metrics:
        export:
          prometheus:
            enabled: true
            rsocket:
              enabled: true
              host: prometheus-proxy
              port: 7001
...
```

## grafana 관련 설정

### grafana Endpoint 확인

>kubectl describe service/grafana

```
🍺🦑🍺🍕🍺 ❯ kubectl describe service/grafana                                                                                             ✹ ✭
Name:                     grafana
Namespace:                default
Labels:                   app=grafana
Annotations:              <none>
Selector:                 app=grafana
Type:                     NodePort
IP:                       10.96.116.14
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  30050/TCP
Endpoints:                10.244.2.7:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
🍺🦑🍺🍕🍺 ❯  
```

grafana 서비스의 Endpoints에 표시된 IP, PORT 값을 Data Flow 서버 서비스 설정 파일인 `src/kubernetes/server/server-config.yaml`의 dashboard url에 다음과 같이 지정한다. 이렇게 해야 나중에 웹 브라우저를 통해 grafana 대시보드에 접근할 수 있다.

```
...
    spring:
      cloud:
        dataflow:
          metrics.dashboard:
            # url: 'https://grafana:3000'
            url: 'https://10.244.2.7:3000'
...
```

## SCDF Role, Binding, Service Account

### Role

>kubectl create -f src/kubernetes/server/server-roles.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-roles.yaml
role.rbac.authorization.k8s.io/scdf-role created
🍺🦑🍺🍕🍺 ❯ 
```

### Role Binding

>kubectl create -f src/kubernetes/server/server-rolebinding.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-rolebinding.yaml
Warning: rbac.authorization.k8s.io/v1beta1 RoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 RoleBinding
rolebinding.rbac.authorization.k8s.io/scdf-rb created
🍺🦑🍺🍕🍺 ❯  
```

### Role Binding 확인

>kubectl get roles

```
🍺🦑🍺🍕🍺 ❯ kubectl get roles
NAME        CREATED AT
scdf-role   2020-12-27T09:29:28Z
🍺🦑🍺🍕🍺 ❯  
```

### Service Account

>kubectl create -f src/kubernetes/server/service-account.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/service-account.yaml
serviceaccount/scdf-sa created
🍺🦑🍺🍕🍺 ❯  
```

### Service Account 확인

>kubectl get sa

```
🍺🦑🍺🍕🍺 ❯ kubectl get sa
NAME               SECRETS   AGE
default            1         5h54m
prometheus         1         43m
prometheus-proxy   1         4h30m
scdf-sa            1         43s
🍺🦑🍺🍕🍺 ❯ 
```

### Role, Binding, Service Account 배포 회수 

>kubectl delete role scdf-role
>
>kubectl delete rolebinding scdf-rb
>
>kubectl delete serviceaccount scdf-sa


## Skipper 서버

2020-12-27 현재 SCDF 버전은 2.7.0 이지만 아래와 같이 Skipper 서버 최신 이미지는 2.6.0 이며, 

```
🍺🦑🍺🍕🍺 ❯ docker images springcloud/spring-cloud-skipper-server      
REPOSITORY                                TAG       IMAGE ID       CREATED       SIZE
springcloud/spring-cloud-skipper-server   2.6.0     d7eba9aa59f8   4 weeks ago   389MB
🍺🦑🍺🍕🍺 ❯ 
```

`src/kubernetes/skipper/skipper-deployment.yaml` 파일에도 2.6.0 으로 돼 있다.

```
...
spec:
  selector:
    matchLabels:
      app: skipper
  replicas: 1
  template:
    metadata:
      labels:
        app: skipper
    spec:
      containers:
      - name: skipper
        image: springcloud/spring-cloud-skipper-server:2.6.0  #
...
```

### Skipper-RabbitMQ 배포

>kubectl create -f src/kubernetes/skipper/skipper-config-rabbit.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/skipper/skipper-config-rabbit.yaml
configmap/skipper created
🍺🦑🍺🍕🍺 ❯ 
```

### Skipper 서버 배포

>kubectl create -f src/kubernetes/skipper/skipper-deployment.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/skipper/skipper-deployment.yaml
deployment.apps/skipper created
🍺🦑🍺🍕🍺 ❯ 
```

`src/kubernetes/skipper/skipper-svc.yaml`를 열어보면 다음과 같이 타입이 `LoadBalancer`로 지정돼 있는데, 외부 클라우드 제공자의 로드밸런서를 사용하고 있지 않으므로 `LoadBalancer`를 `NodePort`로 대체한다.

```
apiVersion: v1
kind: Service
metadata:
  name: skipper
  labels:
    app: skipper
    spring-deployment-id: scdf
spec:
  # If you are running k8s on a local dev box, using minikube, or Kubernetes on docker desktop you can use type NodePort instead
  # type: LoadBalancer
  type: NodePort                                                                                                 
  ports:
  - port: 80
    targetPort: 7577
  selector:
    app: skipper
```

>kubectl create -f src/kubernetes/skipper/skipper-svc.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/skipper/skipper-svc.yaml
service/skipper created
🍺🦑🍺🍕🍺 ❯ 
```

### Skipper 서버 배포 확인

>kubectl get all,cm -l app=skipper

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm -l app=skipper
NAME                           READY   STATUS    RESTARTS   AGE
pod/skipper-54d985cd5b-nd6wn   1/1     Running   0          20m

NAME              TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/skipper   NodePort   10.96.89.15   <none>        80:31353/TCP   3s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/skipper   1/1     1            1           20m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/skipper-54d985cd5b   1         1         1       20m

NAME                DATA   AGE
configmap/skipper   1      12m
🍺🦑🍺🍕🍺 ❯   
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### Skipper 서버 배포 회수

>kubectl delete all,cm -l app=skipper


## Data Flow 서버

구성 파일은 `src/kubernetes/server/server-deployment.yaml`에 있으며, 기본 버전도 2.7.0 으로 돼있어 그대로 사용

Skipper 서버가 먼저 구동되고 있어야 하며, Skipper 서버의 IP, PORT를 `src/kubernetes/server/server-deployment.yaml` 파일에 아래와 같이 지정해줘야 함

```
...
          # Provide the Skipper service location
          # kubectl describe service/skipper 실행 후 표시 내용 중 IP, PORT 값을 아래 value 에 설정
        - name: SPRING_CLOUD_SKIPPER_CLIENT_SERVER_URI
          # value: 'http://${SKIPPER_SERVICE_HOST}:${SKIPPER_SERVICE_PORT}/api'
          value: 'http://10.96.89.15:80/api'
...
```

RabbitMQ 연동을 위한 ConfigMap 은 `src/kubernetes/skipper/skipper-config-rabbit.yaml`에 있다고 하는데, 그걸 복사해서 적절히 바꿔서 `src/kubernetes/server/server-config-rabbit.yaml`로 만들어 쓰라는 건지 그냥 있다는 건지 알 수 없음 TODO

### ConfigMap 생성

>kubectl create -f src/kubernetes/server/server-config.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-config.yaml
configmap/scdf-server created
🍺🦑🍺🍕🍺 ❯ 
```

### Data Flow 서버 배포

`src/kubernetes/server/server-svc.yaml`를 열어보면 다음과 같이 타입이 `LoadBalancer`로 지정돼 있는데, 외부 클라우드 제공자의 로드밸런서를 사용하고 있지 않으므로 `LoadBalancer`를 `NodePort`로 대체한다.

```
kind: Service                                                              
apiVersion: v1
metadata:
  name: scdf-server
  labels:
    app: scdf-server
    spring-deployment-id: scdf
spec:
  # If you are running k8s on a local dev box or using minikube, you can use type NodePort instead
  # type: LoadBalancer
  type: NodePort
  ports:
    - port: 80
      name: scdf-server
  selector:
    app: scdf-server
```

>kubectl create -f src/kubernetes/server/server-svc.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-svc.yaml
service/scdf-server created
🍺🦑🍺🍕🍺 ❯ 
```

>kubectl create -f src/kubernetes/server/server-deployment.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-deployment.yaml
deployment.apps/scdf-server created
🍺🦑🍺🍕🍺 ❯ 
```

### Data Flow 서버 배포 확인

>kubectl get all,cm -l app=scdf-server

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm -l app=scdf-server
NAME                               READY   STATUS    RESTARTS   AGE
pod/scdf-server-5cdc56dd64-s494f   1/1     Running   0          91s

NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/scdf-server   NodePort   10.96.177.217   <none>        80:31467/TCP   97s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scdf-server   1/1     1            1           91s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/scdf-server-5cdc56dd64   1         1         1       91s

NAME                    DATA   AGE
configmap/scdf-server   1      108s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### Data Flow 서버 배포 회수

>kubectl delete all,cm -l app=scdf-server

### SCDF IP 확인

>kubectl get svc scdf-server

```
🍺🦑🍺🍕🍺 ❯ kubectl get svc scdf-server
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
scdf-server   NodePort   10.96.177.217   <none>        80:31467/TCP   5m26s
🍺🦑🍺🍕🍺 ❯ 
```

`EXTERNAL_IP` 값으로 나온 IP로 SCDF Shell 에서 접속 가능

minikube나 kind로 구성한 경우 외부 로드 밸런서를 구성하지 않았으므로 `EXTERNAL_IP` 값이 `<pending>`으로 표시됨

minikube를 사용했다면 `minikube service --url scdf-server` 실행 결과 나오는 IP, PORT를 사용할 수 있다고 한다.

kind를 사용했다면 다음 명령으로 URL 확인 가능

>TODO

# 전체 설치 내용 최종 확인

>kubectl get all,cm

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm
NAME                                    READY   STATUS    RESTARTS   AGE
pod/grafana-7dc7d95456-bhj6l            1/1     Running   0          90m
pod/mysql-58f79dbc8c-n7slk              1/1     Running   0          5h36m
pod/prometheus-7fbc58dcf5-24bvv         1/1     Running   0          96m
pod/prometheus-proxy-5b958c7fd4-xk5lq   1/1     Running   0          5h26m
pod/rabbitmq-78b6c44c49-bpln4           1/1     Running   0          5h39m
pod/scdf-server-5cdc56dd64-s494f        1/1     Running   0          7m35s
pod/skipper-54d985cd5b-nd6wn            1/1     Running   0          45m

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
service/grafana            NodePort    10.96.116.14    <none>        3000:30050/TCP                  90m
service/kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP                         6h50m
service/mysql              ClusterIP   10.96.183.164   <none>        3306/TCP                        5h36m
service/prometheus         ClusterIP   10.96.87.216    <none>        9090/TCP                        95m
service/prometheus-proxy   NodePort    10.96.20.147    <none>        8080:32036/TCP,7001:31986/TCP   5h3m
service/rabbitmq           ClusterIP   10.96.135.124   <none>        5672/TCP                        5h39m
service/scdf-server        NodePort    10.96.177.217   <none>        80:31467/TCP                    7m41s
service/skipper            NodePort    10.96.89.15     <none>        80:31353/TCP                    25m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana            1/1     1            1           90m
deployment.apps/mysql              1/1     1            1           5h36m
deployment.apps/prometheus         1/1     1            1           96m
deployment.apps/prometheus-proxy   1/1     1            1           5h26m
deployment.apps/rabbitmq           1/1     1            1           5h39m
deployment.apps/scdf-server        1/1     1            1           7m35s
deployment.apps/skipper            1/1     1            1           45m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-7dc7d95456            1         1         1       90m
replicaset.apps/mysql-58f79dbc8c              1         1         1       5h36m
replicaset.apps/prometheus-7fbc58dcf5         1         1         1       96m
replicaset.apps/prometheus-proxy-5b958c7fd4   1         1         1       5h26m
replicaset.apps/rabbitmq-78b6c44c49           1         1         1       5h39m
replicaset.apps/scdf-server-5cdc56dd64        1         1         1       7m35s
replicaset.apps/skipper-54d985cd5b            1         1         1       45m

NAME                    DATA   AGE
configmap/grafana       1      90m
configmap/prometheus    1      96m
configmap/scdf-server   1      7m52s
configmap/skipper       1      37m
🍺🦑🍺🍕🍺 ❯ 
```



