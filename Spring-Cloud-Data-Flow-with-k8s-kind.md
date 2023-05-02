# Spring Cloud Data Flow with k8s - kind

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

# from https://kind.sigs.k8s.io/docs/user/quick-start/#multinode-clusters
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  # data flow server
  - hostPort: 8081  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31081  # k8s service에서 nodePort로 지정한 포트 번호
  # skipper server
  - hostPort: 8082  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31082  # k8s service에서 nodePort로 지정한 포트 번호
  # grafana
  - hostPort: 3000  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31030  # k8s service에서 nodePort로 지정한 포트 번호
  # prometheus web
  - hostPort: 9001  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31091  # k8s service에서 nodePort로 지정한 포트 번호
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

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
🍺🦑🍺🍕🍺 ❯ 
```

클러스터 생성 후 `~/.kube/config` 파일 내용이 다음과 같이 채워진다.

```
apiVersion: v1                                                                                                                                                                    
clusters:
- cluster:
    certificate-authority-data: XXX...생략...
    server: https://127.0.0.1:56486
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


### 클러스터 종료 (나중에 필요 시 실행)

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

>kubectl cluster-info --context kind-my-k8s-cluster

```
🍺🦑🍺🍕🍺 ❯ kubectl cluster-info --context kind-my-k8s-cluster
Kubernetes master is running at https://127.0.0.1:56486
KubeDNS is running at https://127.0.0.1:56486/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
🍺🦑🍺🍕🍺 ❯ 
```

브라우저에서 `https://127.0.0.1:56486`에 접속하거나 `curl -k https://127.0.0.1:56486`를 실행하면 다음과 같이 403 인증 에러 발생. 일단 넘어감

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

크게 다음 4개 카테고리로 구성

- Message Broker
- Database
- Monitoring
- SCDF



# Meesage Broker

RabbitMQ와 Kafka 중 [선택](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/#choose-a-message-broker), 여기에서는 RabbitMQ 선택

## RabbitMQ

### 배포

`src/kubernetes/rabbitmq/rabbit-svc.yaml` 파일을 열어보면 rabbitmq 서비스의 타입은 지정돼있지 않다. 따라서 기본값인 Cluster IP가 사용됨.

>kubectl create -f src/kubernetes/rabbitmq

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/rabbitmq
deployment.apps/rabbitmq created
service/rabbitmq created
🍺🦑🍺🍕🍺 ❯ 
```

### 배포 확인

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=rabbitmq

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=rabbitmq
NAME                            READY   STATUS    RESTARTS   AGE
pod/rabbitmq-78b6c44c49-s6ls8   1/1     Running   0          28s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/rabbitmq   ClusterIP   10.96.154.123   <none>        5672/TCP   28s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rabbitmq   1/1     1            1           28s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/rabbitmq-78b6c44c49   1         1         1       28s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### 배포 회수 (나중에 필요 시 실행)

>kubectl delete all -l app=rabbitmq



# Database

MySQL, Postgres, H2 중 선택. 여기에서는 MySQL 선택.

## MySQL

### 배포

`src/kubernetes/mysql/mysql-svc.yaml` 파일을 열어보면 mysql 서비스의 타입은 지정돼있지 않다. 따라서 기본값인 Cluster IP가 사용됨.

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

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret,persistentvolumeclaim -l app=mysql

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret,persistentvolumeclaim -l app=mysql
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-58f79dbc8c-sxfl6   1/1     Running   0          41s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/mysql   ClusterIP   10.96.184.124   <none>        3306/TCP   41s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql   1/1     1            1           41s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-58f79dbc8c   1         1         1       41s

NAME           TYPE     DATA   AGE
secret/mysql   Opaque   2      41s

NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql   Bound    pvc-93581cce-a4d3-487d-8d92-e7da164d66b8   8Gi        RWO            standard       41s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### 배포 회수 (나중에 필요 시 싫행)

>kubectl delete all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret,persistentvolumeclaim -l app=mysql



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
      targetPort: 8080  # prometheus-proxy-deployment.yaml 의 containerPort 와 같아야 함
    - name: rsocket  # rsocket으로 scdf 서버 metric 수집
      port: 7001
      targetPort: 7001  # prometheus-proxy-deployment.yaml 의 containerPort 와 같아야 함
  # type: LoadBalancer  # 로컬 클러스터에서는 외부 클라우드 제공자의 로드밸런서가 없으므로 제거
```

외부 클라우드 제공자의 로드밸런서를 사용하고 있지 않으므로 `LoadBalancer`를 제거해서 기본 type인 Cluster IP 가 사용되게 한다.

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

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=prometheus-proxy

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=prometheus-proxy
NAME                                    READY   STATUS    RESTARTS   AGE
pod/prometheus-proxy-5b958c7fd4-sxqwb   1/1     Running   0          21s

NAME                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
service/prometheus-proxy   ClusterIP   10.96.78.84   <none>        8080/TCP,7001/TCP   22s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-proxy   1/1     1            1           22s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-proxy-5b958c7fd4   1         1         1       22s

NAME                                                            ROLE                        AGE
clusterrolebinding.rbac.authorization.k8s.io/prometheus-proxy   ClusterRole/cluster-admin   22s

NAME                              SECRETS   AGE
serviceaccount/prometheus-proxy   1         22s
🍺🦑🍺🍕🍺 ❯ 
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

서비스의 네트워크는 다음과 같이 확인 가능하다.

>kubectl describe services/prometheus-proxy

```
🍺🦑🍺🍕🍺 ❯ kubectl describe services/prometheus-proxy
Name:              prometheus-proxy
Namespace:         default
Labels:            app=prometheus-proxy
Annotations:       <none>
Selector:          app=prometheus-proxy
Type:              ClusterIP
IP:                10.96.78.84
Port:              scrape  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.244.1.4:8080
Port:              rsocket  7001/TCP
TargetPort:        7001/TCP
Endpoints:         10.244.1.4:7001
Session Affinity:  None
Events:            <none>
🍺🦑🍺🍕🍺 ❯  
```

### prometheus-proxy 배포 회수 (나중에 필요 시 실행)

>kubectl delete all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=prometheus-proxy

### prometheus 배포

클러스터 외부에서 NodePort로 접근할 수 있도록 prometheus-service.yaml 파일을 다음과 같이 NodePort 타입으로 변경

```yml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  labels:
    app: prometheus
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '9090'
spec:
  selector:
    app: prometheus
  type: NodePort  # 클러스터 외부에서 NodePort로 접근 가능
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 31091  # kind cluster config 파일의 extraPortMappings 의 containerPort 가 이 nodePort 값을 가리켜야 한다.
```

>kubectl create -f src/kubernetes/prometheus/

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/prometheus/
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
configmap/prometheus created
deployment.apps/prometheus created
service/prometheus created
serviceaccount/prometheus created
```

### prometheus 배포 확인

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=prometheus

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=prometheus
NAME                              READY   STATUS    RESTARTS   AGE
pod/prometheus-7fbc58dcf5-mmcwl   1/1     Running   0          4m41s

NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/prometheus   NodePort   10.96.254.177   <none>        9090:31091/TCP   4m42s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus   1/1     1            1           4m42s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-7fbc58dcf5   1         1         1       4m42s

NAME                   DATA   AGE
configmap/prometheus   1      4m42s

NAME                                               CREATED AT
clusterrole.rbac.authorization.k8s.io/prometheus   2021-01-27T08:17:38Z

NAME                                                      ROLE                     AGE
clusterrolebinding.rbac.authorization.k8s.io/prometheus   ClusterRole/prometheus   4m42s

NAME                        SECRETS   AGE
serviceaccount/prometheus   1         4m42s
🍺🦑🍺🍕🍺 ❯ 
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### prometheus 배포 회수 (나중에 필요 시 실행)

>kubectl delete all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=prometheus

### 웹 콘솔 확인

kind 클러스터 config 파일에 prometheus 용으로 지정한 포트로 접속 `http://localhost:9001`

![Imgur](https://i.imgur.com/KEHlxSv.png)


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
#  type: LoadBalancer
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 31030  # kind cluster config 파일의 containerPor 값이 이 nodePort 값을 가리켜야 함
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

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=grafana

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=grafana
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-7dc7d95456-zs4hb   1/1     Running   0          37s

NAME              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/grafana   NodePort   10.96.41.163   <none>        3000:31030/TCP   37s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           37s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-7dc7d95456   1         1         1       37s

NAME                DATA   AGE
configmap/grafana   1      37s

NAME             TYPE     DATA   AGE
secret/grafana   Opaque   2      37s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### 배포 회수 (나중에 필요 시 실행)

>kubectl delete all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=grafana

### 웹 콘솔 확인

kind 클러스터 config 파일에 grafana 용으로 지정한 포트로 접속 `http://localhost:3000`

![Imgur](https://i.imgur.com/pbLx3RS.png)

로그인 정보는 grafana-secret.yaml 파일에 지정돼 있다. base64 로 인코딩 된 값을 지정히야 하며, 디코딩 한 값을 로그인에 사용하면 된다.


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
🍺🦑🍺🍕🍺 ❯ kubectl describe service/grafana
Name:                     grafana
Namespace:                default
Labels:                   app=grafana
Annotations:              <none>
Selector:                 app=grafana
Type:                     NodePort
IP:                       10.96.41.163
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  31030/TCP
Endpoints:                10.244.2.4:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
🍺🦑🍺🍕🍺 ❯  
```

grafana 서비스의 Endpoints에 표시된 IP, PORT 값을 Data Flow 서버 서비스 설정 파일인 `src/kubernetes/server/server-config.yaml`의 dashboard url에 다음과 같이 지정한다. 이렇게 해야 나중에 Data Flow 대시보드 상에서 버튼 클릭으로 grafana 대시보드를 띄울 수 있다.

kind 클러스터를 사용할 때는 dashboard url에 grafana 용 nodePort 정보를 입력해야 된다.

```
...
    spring:
      cloud:
        dataflow:
          metrics.dashboard:
            # url: 'https://grafana:3000'
            # url: 'https://10.244.2.4:3000'
            url: http://localhost:3000  # kind의 config 파일의 extraPortMapping 참고
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

### Service Account

>kubectl create -f src/kubernetes/server/service-account.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/service-account.yaml
serviceaccount/scdf-sa created
🍺🦑🍺🍕🍺 ❯  
```

### Role, RoleBinding, Service Account 확인

>kubectl get roles,rolebindings,sa

```
🍺🦑🍺🍕🍺 ❯ kubectl get roles,rolebindings,sa
NAME                                       CREATED AT
role.rbac.authorization.k8s.io/scdf-role   2021-01-27T08:32:35Z

NAME                                            ROLE             AGE
rolebinding.rbac.authorization.k8s.io/scdf-rb   Role/scdf-role   21s

NAME                              SECRETS   AGE
serviceaccount/default            1         30m
serviceaccount/prometheus         1         15m
serviceaccount/prometheus-proxy   1         18m
serviceaccount/scdf-sa            1         9s
🍺🦑🍺🍕🍺 ❯  
```

### Role, Binding, Service Account 배포 회수 (나중에 필요 시 실행)

>kubectl delete role scdf-role
>
>kubectl delete rolebinding scdf-rb
>
>kubectl delete serviceaccount scdf-sa


## Skipper 서버

2021-01-27 현재 SCDF 버전은 2.7.0 이지만 아래와 같이 Skipper 서버 최신 이미지는 2.6.0 이며, 

```
🍺🦑🍺🍕🍺 ❯ docker images springcloud/spring-cloud-skipper-server
REPOSITORY                                TAG       IMAGE ID       CREATED        SIZE
springcloud/spring-cloud-skipper-server   2.6.0     d7eba9aa59f8   2 months ago   389MB
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

src/kubernetes/skipper/skipper-config-rabbit.yaml 파일의 prometheus 항목 설정값을 확인한다.

```
...
data:
  application.yaml: |-
    management:
      metrics:
        export:
          prometheus:
            enabled: true
            rsocket:
              enabled: true
              host: prometheus-proxy
              port: 7001  # prometheus-proxy service 의 rsocket 포트 값을 가리켜야 한다.
...
```


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
  selector:
    app: skipper
  # If you are running k8s on a local dev box, using minikube, or Kubernetes on docker desktop you can use type NodePort instead
  # type: LoadBalancer
  type: NodePort
  ports:
  - port: 80
    targetPort: 7577
    nodePort: 31082
```

>kubectl create -f src/kubernetes/skipper/skipper-svc.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/skipper/skipper-svc.yaml
service/skipper created
🍺🦑🍺🍕🍺 ❯ 
```

### Skipper 서버 배포 확인

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=skipper

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=skipper
NAME                           READY   STATUS    RESTARTS   AGE
pod/skipper-54d985cd5b-dpnpm   1/1     Running   0          2m3s

NAME              TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/skipper   NodePort   10.96.49.40   <none>        80:31082/TCP   89s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/skipper   1/1     1            1           2m3s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/skipper-54d985cd5b   1         1         1       2m3s

NAME                DATA   AGE
configmap/skipper   1      2m14s
🍺🦑🍺🍕🍺 ❯   
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### Skipper 서버 배포 회수 (나중에 필요 시 실행)

>kubectl delete all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=skipper

### 웹 콘솔 확인

kind 클러스터 config 파일에 skipper 서버 용으로 지정한 포트로 접속 `http://localhost:8082`

```json
{
  "_links" : {
    "deployers" : {
      "href" : "http://localhost:8082/api/deployers{?page,size,sort}",
      "templated" : true
    },
    "repositories" : {
      "href" : "http://localhost:8082/api/repositories{?page,size,sort}",
      "templated" : true
    },
    "packageMetadata" : {
      "href" : "http://localhost:8082/api/packageMetadata{?page,size,sort,projection}",
      "templated" : true
    },
    "releases" : {
      "href" : "http://localhost:8082/api/releases{?page,size,sort}",
      "templated" : true
    },
    "about" : {
      "href" : "http://localhost:8082/api/about"
    },
    "release" : {
      "href" : "http://localhost:8082/api/release"
    },
    "package" : {
      "href" : "http://localhost:8082/api/package"
    },
    "profile" : {
      "href" : "http://localhost:8082/api/profile"
    }
  }
}
```


## Data Flow 서버

구성 파일은 `src/kubernetes/server/server-deployment.yaml`에 있으며, 기본 버전도 2.7.0 으로 돼있어 그대로 사용

### Deployment

Skipper 서버가 먼저 구동되고 있어야 하며, Skipper 서버의 IP, PORT 를 확인한다.

```
🍺🦑🍺🍕🍺 ❯ kubectl describe service/skipper
Name:                     skipper
Namespace:                default
Labels:                   app=skipper
                          spring-deployment-id=scdf
Annotations:              <none>
Selector:                 app=skipper
Type:                     NodePort
IP:                       10.96.47.21
Port:                     <unset>  80/TCP
TargetPort:               7577/TCP
NodePort:                 <unset>  31082/TCP
Endpoints:                10.244.1.6:7577
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
🍺🦑🍺🍕🍺 ❯ 
```

Skipper 서비스의 IP, Port 정보를 `src/kubernetes/server/server-deployment.yaml` 파일에 아래와 같이 지정해줘야 함

```
...
          # Provide the Skipper service location
          # kubectl describe service/skipper 실행 후 표시 내용 중 IP, PORT 값을 아래 value 에 설정
        - name: SPRING_CLOUD_SKIPPER_CLIENT_SERVER_URI
          # value: 'http://${SKIPPER_SERVICE_HOST}:${SKIPPER_SERVICE_PORT}/api'
          value: 'http://10.96.49.40:80/api' # IP:Port 또는 Endpoints에 있는 ip:port TODO
...
```

>kubectl create -f src/kubernetes/server/server-deployment.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-deployment.yaml
deployment.apps/scdf-server created
🍺🦑🍺🍕🍺 ❯ 
```

RabbitMQ 연동을 위한 ConfigMap 은 `src/kubernetes/skipper/skipper-config-rabbit.yaml`에 있다고 하는데, 그걸 복사해서 적절히 바꿔서 `src/kubernetes/server/server-config-rabbit.yaml`로 만들어 쓰라는 건지 그냥 있다는 건지 알 수 없음. 일단 만들지 않고 진행 TODO

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
  selector:
    app: scdf-server
  # If you are running k8s on a local dev box or using minikube, you can use type NodePort instead
  # type: LoadBalancer
  type: NodePort
  ports:
    - name: scdf-server
      port: 80
      targetPort: 80
      nodePort: 31081
```

>kubectl create -f src/kubernetes/server/server-svc.yaml

```
🍺🦑🍺🍕🍺 ❯ kubectl create -f src/kubernetes/server/server-svc.yaml
service/scdf-server created
🍺🦑🍺🍕🍺 ❯ 
```


### Data Flow 서버 배포 확인

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=scdf-server

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=scdf-server
NAME                               READY   STATUS    RESTARTS   AGE
pod/scdf-server-6b98fd5d54-jsckz   1/1     Running   1          4m8s

NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/scdf-server   NodePort   10.96.129.175   <none>        80:31081/TCP   3m21s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scdf-server   1/1     1            1           4m9s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/scdf-server-6b98fd5d54   1         1         1       4m9s

NAME                    DATA   AGE
configmap/scdf-server   1      3m56s
🍺🦑🍺🍕🍺 ❯  
```

pod와 deployment 의 READY 항목이 `0/1`로 표시되면 대략 1 ~ 2분 후 다시 실행해서 `1/1`로 표시되는 것을 확인해야 한다.

### Data Flow 서버 배포 회수 (나중에 필요 시 실행)

>kubectl delete all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret -l app=scdf-server

### SCDF IP 확인 및 웹 접근

아래 내용은 레퍼런스 문서에 나와 있으나, kind 클러스터 기반으로 구성한 상황에서는 EXTERNAL-IP 값이 없으므로 적용 불가

>kubectl get svc scdf-server

```
🍺🦑🍺🍕🍺 ❯ kubectl get svc scdf-server
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
scdf-server   NodePort   10.96.142.141   <none>        80:31081/TCP   61m
🍺🦑🍺🍕🍺 ❯ 
```

`EXTERNAL_IP` 값으로 나온 IP로 SCDF Shell 에서 접속 가능

`curl CLUSTER-IP:PORT의왼쪽값 == curl NODE-IP:PORT의오른쪽값` 으로 접속 가능

minikube나 kind로 구성한 경우 외부 로드 밸런서를 구성하지 않았으므로 `EXTERNAL_IP` 값이 `<pending>`으로 표시됨

minikube를 사용했다면 `minikube service --url scdf-server` 실행 결과 나오는 IP, PORT를 사용할 수 있다고 한다.

kind를 사용했다면 다음 명령으로 URL 확인 가능

>curl CLUSTER-IP:PORT의왼쪽값 == curl NODE-IP:PORT의오른쪽값

### 웹 콘솔 확인

kind 클러스터 config 파일에 data flow server 용으로 지정한 포트로 접속 `http://localhost:8081`

```json
{
  "_links": {
    "dashboard": {
      "href": "http://localhost:8081/dashboard"
    },
    "audit-records": {
      "href": "http://localhost:8081/audit-records"
    },
    "streams/definitions": {
      "href": "http://localhost:8081/streams/definitions"
    },
    "streams/definitions/definition": {
      "href": "http://localhost:8081/streams/definitions/{name}",
      "templated": true
    },
    "streams/validation": {
      "href": "http://localhost:8081/streams/validation/{name}",
      "templated": true
    },
    "runtime/streams": {
      "href": "http://localhost:8081/runtime/streams{?names}",
      "templated": true
    },
    "runtime/streams/{streamNames}": {
      "href": "http://localhost:8081/runtime/streams/{streamNames}",
      "templated": true
    },
    "runtime/apps": {
      "href": "http://localhost:8081/runtime/apps"
    },
    "runtime/apps/{appId}": {
      "href": "http://localhost:8081/runtime/apps/{appId}",
      "templated": true
    },
    "runtime/apps/{appId}/instances": {
      "href": "http://localhost:8081/runtime/apps/{appId}/instances",
      "templated": true
    },
    "runtime/apps/{appId}/instances/{instanceId}": {
      "href": "http://localhost:8081/runtime/apps/{appId}/instances/{instanceId}",
      "templated": true
    },
    "streams/deployments": {
      "href": "http://localhost:8081/streams/deployments"
    },
    "streams/deployments/{name}{?reuse-deployment-properties}": {
      "href": "http://localhost:8081/streams/deployments/{name}?reuse-deployment-properties=false",
      "templated": true
    },
    "streams/deployments/{name}": {
      "href": "http://localhost:8081/streams/deployments/{name}",
      "templated": true
    },
    "streams/deployments/history/{name}": {
      "href": "http://localhost:8081/streams/deployments/history/{name}",
      "templated": true
    },
    "streams/deployments/manifest/{name}/{version}": {
      "href": "http://localhost:8081/streams/deployments/manifest/{name}/{version}",
      "templated": true
    },
    "streams/deployments/platform/list": {
      "href": "http://localhost:8081/streams/deployments/platform/list"
    },
    "streams/deployments/rollback/{name}/{version}": {
      "href": "http://localhost:8081/streams/deployments/rollback/{name}/{version}",
      "templated": true
    },
    "streams/deployments/update/{name}": {
      "href": "http://localhost:8081/streams/deployments/update/{name}",
      "templated": true
    },
    "streams/deployments/deployment": {
      "href": "http://localhost:8081/streams/deployments/{name}",
      "templated": true
    },
    "streams/deployments/scale/{streamName}/{appName}/instances/{count}": {
      "href": "http://localhost:8081/streams/deployments/scale/{streamName}/{appName}/instances/{count}",
      "templated": true
    },
    "streams/logs": {
      "href": "http://localhost:8081/streams/logs"
    },
    "streams/logs/{streamName}": {
      "href": "http://localhost:8081/streams/logs/{streamName}",
      "templated": true
    },
    "streams/logs/{streamName}/{appName}": {
      "href": "http://localhost:8081/streams/logs/{streamName}/{appName}",
      "templated": true
    },
    "tasks/platforms": {
      "href": "http://localhost:8081/tasks/platforms"
    },
    "tasks/definitions": {
      "href": "http://localhost:8081/tasks/definitions"
    },
    "tasks/definitions/definition": {
      "href": "http://localhost:8081/tasks/definitions/{name}",
      "templated": true
    },
    "tasks/executions": {
      "href": "http://localhost:8081/tasks/executions"
    },
    "tasks/executions/name": {
      "href": "http://localhost:8081/tasks/executions{?name}",
      "templated": true
    },
    "tasks/executions/current": {
      "href": "http://localhost:8081/tasks/executions/current"
    },
    "tasks/executions/execution": {
      "href": "http://localhost:8081/tasks/executions/{id}",
      "templated": true
    },
    "tasks/validation": {
      "href": "http://localhost:8081/tasks/validation/{name}",
      "templated": true
    },
    "tasks/logs": {
      "href": "http://localhost:8081/tasks/logs/{taskExternalExecutionId}{?platformName}",
      "templated": true
    },
    "tasks/schedules": {
      "href": "http://localhost:8081/tasks/schedules"
    },
    "tasks/schedules/instances": {
      "href": "http://localhost:8081/tasks/schedules/instances/{taskDefinitionName}",
      "templated": true
    },
    "jobs/executions": {
      "href": "http://localhost:8081/jobs/executions"
    },
    "jobs/executions/name": {
      "href": "http://localhost:8081/jobs/executions{?name}",
      "templated": true
    },
    "jobs/executions/status": {
      "href": "http://localhost:8081/jobs/executions{?status}",
      "templated": true
    },
    "jobs/executions/execution": {
      "href": "http://localhost:8081/jobs/executions/{id}",
      "templated": true
    },
    "jobs/executions/execution/steps": {
      "href": "http://localhost:8081/jobs/executions/{jobExecutionId}/steps",
      "templated": true
    },
    "jobs/executions/execution/steps/step": {
      "href": "http://localhost:8081/jobs/executions/{jobExecutionId}/steps/{stepId}",
      "templated": true
    },
    "jobs/executions/execution/steps/step/progress": {
      "href": "http://localhost:8081/jobs/executions/{jobExecutionId}/steps/{stepId}/progress",
      "templated": true
    },
    "jobs/instances/name": {
      "href": "http://localhost:8081/jobs/instances{?name}",
      "templated": true
    },
    "jobs/instances/instance": {
      "href": "http://localhost:8081/jobs/instances/{id}",
      "templated": true
    },
    "tools/parseTaskTextToGraph": {
      "href": "http://localhost:8081/tools"
    },
    "tools/convertTaskGraphToText": {
      "href": "http://localhost:8081/tools"
    },
    "jobs/thinexecutions": {
      "href": "http://localhost:8081/jobs/thinexecutions"
    },
    "jobs/thinexecutions/name": {
      "href": "http://localhost:8081/jobs/thinexecutions{?name}",
      "templated": true
    },
    "jobs/thinexecutions/jobInstanceId": {
      "href": "http://localhost:8081/jobs/thinexecutions{?jobInstanceId}",
      "templated": true
    },
    "jobs/thinexecutions/taskExecutionId": {
      "href": "http://localhost:8081/jobs/thinexecutions{?taskExecutionId}",
      "templated": true
    },
    "apps": {
      "href": "http://localhost:8081/apps"
    },
    "about": {
      "href": "http://localhost:8081/about"
    },
    "completions/stream": {
      "href": "http://localhost:8081/completions/stream{?start,detailLevel}",
      "templated": true
    },
    "completions/task": {
      "href": "http://localhost:8081/completions/task{?start,detailLevel}",
      "templated": true
    }
  },
  "api.revision": 14
}
```

http://localhost:8081/dashboard 로 Data Flow 서버 대시보드에 접속할 수 있다.

![Imgur](https://i.imgur.com/LMOnBfw.png)


# 전체 설치 내용 및 상태 최종 확인

>kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret,persistentvolumeclaim

```
🍺🦑🍺🍕🍺 ❯ kubectl get all,cm,role,rolebinding,clusterrole,clusterrolebinding,sa,secret,persistentvolumeclaim
NAME                                    READY   STATUS    RESTARTS   AGE
pod/grafana-7dc7d95456-zs4hb            1/1     Running   0          31m
pod/mysql-58f79dbc8c-sxfl6              1/1     Running   0          50m
pod/prometheus-7fbc58dcf5-mmcwl         1/1     Running   0          40m
pod/prometheus-proxy-5b958c7fd4-sxqwb   1/1     Running   0          42m
pod/rabbitmq-78b6c44c49-s6ls8           1/1     Running   0          51m
pod/scdf-server-6b98fd5d54-jsckz        1/1     Running   1          6m47s
pod/skipper-54d985cd5b-dpnpm            1/1     Running   0          23m

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/grafana            NodePort    10.96.41.163    <none>        3000:31030/TCP      31m
service/kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP             55m
service/mysql              ClusterIP   10.96.184.124   <none>        3306/TCP            50m
service/prometheus         NodePort    10.96.254.177   <none>        9090:31091/TCP      40m
service/prometheus-proxy   ClusterIP   10.96.78.84     <none>        8080/TCP,7001/TCP   42m
service/rabbitmq           ClusterIP   10.96.154.123   <none>        5672/TCP            51m
service/scdf-server        NodePort    10.96.129.175   <none>        80:31081/TCP        6m
service/skipper            NodePort    10.96.49.40     <none>        80:31082/TCP        22m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana            1/1     1            1           31m
deployment.apps/mysql              1/1     1            1           50m
deployment.apps/prometheus         1/1     1            1           40m
deployment.apps/prometheus-proxy   1/1     1            1           42m
deployment.apps/rabbitmq           1/1     1            1           51m
deployment.apps/scdf-server        1/1     1            1           6m47s
deployment.apps/skipper            1/1     1            1           23m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-7dc7d95456            1         1         1       31m
replicaset.apps/mysql-58f79dbc8c              1         1         1       50m
replicaset.apps/prometheus-7fbc58dcf5         1         1         1       40m
replicaset.apps/prometheus-proxy-5b958c7fd4   1         1         1       42m
replicaset.apps/rabbitmq-78b6c44c49           1         1         1       51m
replicaset.apps/scdf-server-6b98fd5d54        1         1         1       6m47s
replicaset.apps/skipper-54d985cd5b            1         1         1       23m

NAME                    DATA   AGE
configmap/grafana       1      31m
configmap/prometheus    1      40m
configmap/scdf-server   1      6m34s
configmap/skipper       1      23m

NAME                                       CREATED AT
role.rbac.authorization.k8s.io/scdf-role   2021-01-27T08:32:35Z

NAME                                            ROLE             AGE
rolebinding.rbac.authorization.k8s.io/scdf-rb   Role/scdf-role   25m

NAME                                                                                                         CREATED AT
clusterrole.rbac.authorization.k8s.io/admin                                                                  2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/cluster-admin                                                          2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/edit                                                                   2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/kindnet                                                                2021-01-27T08:01:56Z
clusterrole.rbac.authorization.k8s.io/kubeadm:get-nodes                                                      2021-01-27T08:01:54Z
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role                                            2021-01-27T08:01:57Z
clusterrole.rbac.authorization.k8s.io/prometheus                                                             2021-01-27T08:17:38Z
clusterrole.rbac.authorization.k8s.io/system:aggregate-to-admin                                              2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:aggregate-to-edit                                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:aggregate-to-view                                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:auth-delegator                                                  2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:basic-user                                                      2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:certificates.k8s.io:certificatesigningrequests:nodeclient       2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:certificates.k8s.io:kube-apiserver-client-approver              2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:certificates.k8s.io:kube-apiserver-client-kubelet-approver      2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:certificates.k8s.io:kubelet-serving-approver                    2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:certificates.k8s.io:legacy-unknown-approver                     2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:attachdetach-controller                              2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:certificate-controller                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:clusterrole-aggregation-controller                   2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:cronjob-controller                                   2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:daemon-set-controller                                2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:deployment-controller                                2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:disruption-controller                                2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:endpoint-controller                                  2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:endpointslice-controller                             2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:endpointslicemirroring-controller                    2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:expand-controller                                    2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:generic-garbage-collector                            2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:horizontal-pod-autoscaler                            2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:job-controller                                       2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:namespace-controller                                 2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:node-controller                                      2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:persistent-volume-binder                             2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:pod-garbage-collector                                2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:pv-protection-controller                             2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:pvc-protection-controller                            2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:replicaset-controller                                2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:replication-controller                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:resourcequota-controller                             2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:route-controller                                     2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:service-account-controller                           2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:service-controller                                   2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:statefulset-controller                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:controller:ttl-controller                                       2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:coredns                                                         2021-01-27T08:01:54Z
clusterrole.rbac.authorization.k8s.io/system:discovery                                                       2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:heapster                                                        2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:kube-aggregator                                                 2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:kube-controller-manager                                         2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:kube-dns                                                        2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:kube-scheduler                                                  2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:kubelet-api-admin                                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:node                                                            2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:node-bootstrapper                                               2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:node-problem-detector                                           2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:node-proxier                                                    2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:persistent-volume-provisioner                                   2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:public-info-viewer                                              2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/system:volume-scheduler                                                2021-01-27T08:01:52Z
clusterrole.rbac.authorization.k8s.io/view                                                                   2021-01-27T08:01:52Z

NAME                                                                                                ROLE                                                                               AGE
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin                                          ClusterRole/cluster-admin                                                          55m
clusterrolebinding.rbac.authorization.k8s.io/kindnet                                                ClusterRole/kindnet                                                                55m
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:get-nodes                                      ClusterRole/kubeadm:get-nodes                                                      55m
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:kubelet-bootstrap                              ClusterRole/system:node-bootstrapper                                               55m
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:node-autoapprove-bootstrap                     ClusterRole/system:certificates.k8s.io:certificatesigningrequests:nodeclient       55m
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:node-autoapprove-certificate-rotation          ClusterRole/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   55m
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:node-proxier                                   ClusterRole/system:node-proxier                                                    55m
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind                            ClusterRole/local-path-provisioner-role                                            55m
clusterrolebinding.rbac.authorization.k8s.io/prometheus                                             ClusterRole/prometheus                                                             40m
clusterrolebinding.rbac.authorization.k8s.io/prometheus-proxy                                       ClusterRole/cluster-admin                                                          42m
clusterrolebinding.rbac.authorization.k8s.io/system:basic-user                                      ClusterRole/system:basic-user                                                      55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:attachdetach-controller              ClusterRole/system:controller:attachdetach-controller                              55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:certificate-controller               ClusterRole/system:controller:certificate-controller                               55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:clusterrole-aggregation-controller   ClusterRole/system:controller:clusterrole-aggregation-controller                   55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:cronjob-controller                   ClusterRole/system:controller:cronjob-controller                                   55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:daemon-set-controller                ClusterRole/system:controller:daemon-set-controller                                55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:deployment-controller                ClusterRole/system:controller:deployment-controller                                55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:disruption-controller                ClusterRole/system:controller:disruption-controller                                55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:endpoint-controller                  ClusterRole/system:controller:endpoint-controller                                  55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:endpointslice-controller             ClusterRole/system:controller:endpointslice-controller                             55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:endpointslicemirroring-controller    ClusterRole/system:controller:endpointslicemirroring-controller                    55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:expand-controller                    ClusterRole/system:controller:expand-controller                                    55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:generic-garbage-collector            ClusterRole/system:controller:generic-garbage-collector                            55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:horizontal-pod-autoscaler            ClusterRole/system:controller:horizontal-pod-autoscaler                            55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:job-controller                       ClusterRole/system:controller:job-controller                                       55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:namespace-controller                 ClusterRole/system:controller:namespace-controller                                 55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:node-controller                      ClusterRole/system:controller:node-controller                                      55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:persistent-volume-binder             ClusterRole/system:controller:persistent-volume-binder                             55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:pod-garbage-collector                ClusterRole/system:controller:pod-garbage-collector                                55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:pv-protection-controller             ClusterRole/system:controller:pv-protection-controller                             55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:pvc-protection-controller            ClusterRole/system:controller:pvc-protection-controller                            55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:replicaset-controller                ClusterRole/system:controller:replicaset-controller                                55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:replication-controller               ClusterRole/system:controller:replication-controller                               55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:resourcequota-controller             ClusterRole/system:controller:resourcequota-controller                             55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:route-controller                     ClusterRole/system:controller:route-controller                                     55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:service-account-controller           ClusterRole/system:controller:service-account-controller                           55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:service-controller                   ClusterRole/system:controller:service-controller                                   55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:statefulset-controller               ClusterRole/system:controller:statefulset-controller                               55m
clusterrolebinding.rbac.authorization.k8s.io/system:controller:ttl-controller                       ClusterRole/system:controller:ttl-controller                                       55m
clusterrolebinding.rbac.authorization.k8s.io/system:coredns                                         ClusterRole/system:coredns                                                         55m
clusterrolebinding.rbac.authorization.k8s.io/system:discovery                                       ClusterRole/system:discovery                                                       55m
clusterrolebinding.rbac.authorization.k8s.io/system:kube-controller-manager                         ClusterRole/system:kube-controller-manager                                         55m
clusterrolebinding.rbac.authorization.k8s.io/system:kube-dns                                        ClusterRole/system:kube-dns                                                        55m
clusterrolebinding.rbac.authorization.k8s.io/system:kube-scheduler                                  ClusterRole/system:kube-scheduler                                                  55m
clusterrolebinding.rbac.authorization.k8s.io/system:node                                            ClusterRole/system:node                                                            55m
clusterrolebinding.rbac.authorization.k8s.io/system:node-proxier                                    ClusterRole/system:node-proxier                                                    55m
clusterrolebinding.rbac.authorization.k8s.io/system:public-info-viewer                              ClusterRole/system:public-info-viewer                                              55m
clusterrolebinding.rbac.authorization.k8s.io/system:volume-scheduler                                ClusterRole/system:volume-scheduler                                                55m

NAME                              SECRETS   AGE
serviceaccount/default            1         55m
serviceaccount/prometheus         1         40m
serviceaccount/prometheus-proxy   1         42m
serviceaccount/scdf-sa            1         24m

NAME                                  TYPE                                  DATA   AGE
secret/default-token-ctn92            kubernetes.io/service-account-token   3      55m
secret/grafana                        Opaque                                2      31m
secret/mysql                          Opaque                                2      50m
secret/prometheus-proxy-token-s68tw   kubernetes.io/service-account-token   3      42m
secret/prometheus-token-tn6nf         kubernetes.io/service-account-token   3      40m
secret/scdf-sa-token-fgccl            kubernetes.io/service-account-token   3      24m

NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql   Bound    pvc-93581cce-a4d3-487d-8d92-e7da164d66b8   8Gi        RWO            standard       50m
🍺🦑🍺🍕🍺 ❯ 
```


# dmp-batch-app을 SCDF에 등록 및 실행

SCDF가 도커 레지스트리에 있는 dmp-batch-app의 이미지를 pull 해서 등록

## 도커 로컬 레지스트리 구성

- 로컬 테스트에만 사용하며 나중에 실제 docker repo 로 구성 필요

>docker pull registry

>docker container run -d -p 5555:5000 --name local_registry registry

## dmp-batch-app 도커 이미지 빌드

- Dockerfile 이 있는 폴더에서

>docker build --tag=localhost:5555/dmp-batch-app:latest --rm=true .

## dmp-batch-app 도커 이미지를 로컬 레지스트리에 등록

>docker push localhost:5555/dmp-batch-app

## SCDF에 dmp-batch-app 등록

![Imgur](https://i.imgur.com/JKvo0Go.png)

![Imgur](https://i.imgur.com/THLktWW.png)

![Imgur](https://i.imgur.com/eUpfEfV.png)

## SCDF에 task 등록

![Imgur](https://i.imgur.com/VPQXvP1.png)

![Imgur](https://i.imgur.com/XEJTcbh.png)

![Imgur](https://i.imgur.com/gmePHzb.png)

![Imgur](https://i.imgur.com/EWYNYkf.png)

![Imgur](https://i.imgur.com/ygWlhUm.png)

![Imgur](https://i.imgur.com/1TIZdqg.png)


---

# Batch Processing

## 간단한 Pre-built 앱 등록

https://dataflow.spring.io/docs/batch-developer-guides/getting-started/task/#creating-the-task 따라 진행한다. 단, 사전에 아래와 같이 application을 먼저 추가해야 한다.

### Application 추가

https://cloud.spring.io/spring-cloud-task-app-starters/#quick-start 에 timestamp 를 포함하는 application 정보가 있다.

이 중 Docker 타입의 https://dataflow.spring.io/task-docker-latest 를 Data Flow 대시보드 > Add applications > Import application coordinates from an HTTP URI location > URI 에 입력하고 Import Application(s)를 클릭한다.

![Imgur](https://i.imgur.com/QEhd8y6.png)

![Imgur](https://i.imgur.com/vMif569.png)

### Task 생성

![Imgur](https://i.imgur.com/d1t6D5T.png)

![Imgur](https://i.imgur.com/BzzGfhO.png)

![Imgur](https://i.imgur.com/8smt6of.png)

### Task 실행
