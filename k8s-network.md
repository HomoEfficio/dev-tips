# k8s 네트워크

## k8s 클러스터 생성

- k8s 클러스터가 생성되면 노드 별 Bridge가 만들어지고, Bridge-Node 매핑 테이블이 만들어진다.
- 클러스터 DNS도 만들어진다.  


## Pod

- Pod를 하나 만들면 명시적으로 지정한 한 개 이상의 애플리케이션 컨테이너가 실행되고, 묵시적으로 여러 가지 컨테이너도 자동으로 실행된다.  
- Pause 컨테이너도 묵시적으로 자동 실행되며 veth를 생성하고 Pod 내 여러 컨테이너가 veth를 공동 사용할 수 있게 해준다.
- veth는 해당 노드의 Bridge와 연결된다.
- Pod내의 컨테이너가 하나의 veth를 공동 사용하므로 Pod 내 모든 컨테이너의 IP 주소는 veth의 IP 주소이며 각 컨테이너는 Port로 구별된다.
- Pod의 IP 주소는, 즉 veth 주소는 `kubectl get pods -o wide`나 `kubectl describe pod {{POD_NAME}}`으로 확인할 수 있다.

    ```
    $ kubectl get pods -o wide
    NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
    webapp1-6b54fb89d9-jxztl   1/1     Running   0          23s   172.18.0.4   minikube   <none>           <none>
    ```

- Bridge-Node 매핑 테이블에 의해 다른 노드에 있는 Pod 안에 있는 컨테이너와도 통신할 수 있다.
- Pod 네트워크 그림  
    ![Imgur](https://i.imgur.com/qEVjpjR.png)  
    그림 출처: https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727

### containerPort

- Deployment에서 개별 컨테이너를 지칭하는 `spec.template.spec.containers` 아래에서 `ports.containerPort`로 지정하는 값이 개별 컨테이너의 Port 값으로 사용된다.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: webapp1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: webapp1
      template:
        metadata:
          labels:
            app: webapp1
        spec:
          containers:
          - name: webapp1
            image: katacoda/docker-http-server:latest
            ports:
            - containerPort: 80  # 컨테이너에 접근할 수 있도록 공개되는 컨테이너 포트
    ```

- 클러스터 내에서 {{POD_IP}}:{{containerPort}} 로 특정 Pod 내 특정 컨테이너에 접근할 수 있다.


## Service

- Bridge-Node 매핑 테이블에 의해 다른 노드에 있는 Pod 안에 있는 컨테이너와도 통신할 수 있지만, Pod는 생멸을 반복하며 그에 따라 Pod의 IP도 계속 변경될 수 있다.
- Service는 Pod의 IP 대신 이름으로 통신할 수 있게 해주므로 IP 변경에도 문제 없이 통신할 수 있게 해준다.
- Service를 생성할 때 지정한 `spec.selector` 값으로 destination Pod를 지정할 수 있다.
- Service에는 ClusterIP 가 부여되는데 노드의 eth나 Bridge, Pod의 IP와는 다른 대역에 있어 직접 통신이 안 된다.(왜 이렇게 했을까?)
  - 클러스터에서는 노드와 Pod 모두 생멸을 반복하므로 Service의 IP를 이들과 같은 대역에 두면 라우팅이 매우 복잡해 질 수 있기 때문에 아예 다른 대역으로 지정

    ```
    $ kubectl describe svc webapp1
    Name:                     webapp1-svc
    Namespace:                default
    Labels:                   app=webapp1
    Annotations:              <none>
    Selector:                 app=webapp1
    Type:                     NodePort
    IP:                       10.102.122.15  # 노드의 eth나 Pod의 veth와 다른 대역
    Port:                     <unset>  80/TCP  # 서비스의 port 값
    TargetPort:               80/TCP  # 서비스의 targetPort 값
    NodePort:                 <unset>  30080/TCP  # 서비스의 nodePort 값
    Endpoints:                172.18.0.4:80  # {{POD IP}}:{{targetPort == Pod의 containerPort}}
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>
    ```

- Service가 생성되면 클러스터 DNS에 Service 이름과 Service의 ClusterIP가 저장되고, ClusterIP와 destination Pod IP 정보가 노드 별 리눅스 netfilter에 등록된다.
- kube-proxy가 마스터 서버인 control-plain의 API를 통해 컨테이너 health 감지하고 netfilter 규칙을 계속 수정하면서 최신화한다.
- 클러스내 클라이언트는 Service 이름으로 Service를 호출할 수 있다. 클러스터 DNS에 의해 Service의 IP를 얻고, 노드의 netfilter에 의해 Service의 destination IP를 얻어서 최종 destination 에 접근하게 된다.
- 서비스 정보 및 netfilter 규칙 그림  
    ![Imgur](https://i.imgur.com/y0wSnr2.png)  
    그림 출처: https://medium.com/finda-tech/kubernetes-네트워크-정리-fccd4fd0ae6

### port, targetPort

- Service의 포트 정보는 `spec.ports`의 port, targetPort 로 지정할 수 있다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: webapp1-svc
      labels:
        app: webapp1
    spec:
      type: ClusterIP  # type 값이 생략되면 기본값인 ClusterIP가 적용됨
      ports:
      - name: http  # 클라이언트에서 숫자 port 값 대신 {{POD_IP}}:{{name}} 형식으로 호출 가능
        port: 80  # 서비스에 접근할 수 있도록 공개되는 서비스 포트
        targetPort: 80  # 생략하면 port 값과 같은 값으로 지정되며, 서비스의 destination인 컨테이너의 containerPort 값과 같아야 한다.
      selector:
        app: webapp1
    ```

### NodePort

- 클러스터 외부에서 클러스터 내부에 있는 Service를 호출할 수 있게 해주는 기능
- 서비스의 type을 NodePort로 지정하고 nodePort 값을 지정하면 클러스터 외부에서 {{노드의 IP}}:{{nodePort}}로 노드 내에 있는 서비스에 접근할 수 있다.

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: webapp1-svc
      labels:
        app: webapp1
    spec:
      type: NodePort  # 클러스터 외부로부터의 접근을 NodePort 방식으로 허용
      ports:
      - port: 80
        targetPort: 80
        nodePort: 30080
      selector:
        app: webapp1

    # 클러스터 외부에서 {{노드의 IP}}:{{nodePort}}로 접근하면 서비스의 port로 연결되고,  
    # 서비스는 접근 요청을 selector.app 에서 지정한 컨테이너의 targetPort로 포워딩 한다.
    # {{노드의 IP}}:{{nodePort}} --> {{서비스의 IP}}:{{port}} --> {{destination 컨테이너의 IP}}:{{targetPort}}
    ```

### kind 클러스터 + NodePort

- kind(Kubernetes IN Docker, https://kind.sigs.k8s.io/) 를 사용해서 로컬에 멀티노드 클러스터를 구성할 수 있다.
- kind 클러스터에 사용되는 노드는 로컬 머신 입장에서 볼 때 하나의 도커 컨테이너이며, 로컬 머신에서 kind 클러스터 내 노드에 접근하려면 다음과 같이 extraPortMapping을 설정해줘야 한다.

```yaml
# from https://kind.sigs.k8s.io/docs/user/quick-start/#multinode-clusters
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  # data flow server
  - hostPort: 8081  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31081  # 노드 컨테이너에서 오픈한 포트, k8s service에서 nodePort로 지정한 포트 값을 사용해야 해당 서비스로 연결
  # skipper server
  - hostPort: 8082  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31082  # 노드 컨테이너에서 오픈한 포트, k8s service에서 nodePort로 지정한 포트 값을 사용해야 해당 서비스로 연결
  # grafana
  - hostPort: 3000  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31030  # 노드 컨테이너에서 오픈한 포트, k8s service에서 nodePort로 지정한 포트 값을 사용해야 해당 서비스로 연결
  # prometheus web
  - hostPort: 9001  # localhost:PORT 에서 PORT 값으로 사용할 번호
    containerPort: 31091  # 노드 컨테이너에서 오픈한 포트, k8s service에서 nodePort로 지정한 포트 값을 사용해야 해당 서비스로 연결
- role: worker
- role: worker

# {{로컬 머신 IP}}:{{extraPortMapping의 hostPort}} --> {{노드의 IP}}:{{nodePort}} --> {{서비스의 IP}}:{{port}} --> {{destination 컨테이너의 IP}}:{{targetPort}}
```



# 참고 자료

- https://medium.com/finda-tech/kubernetes-네트워크-정리-fccd4fd0ae6
- https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727
  - 번역: https://coffeewhale.com/k8s/network/2019/04/19/k8s-network-01/
- https://medium.com/google-cloud/understanding-kubernetes-networking-services-f0cb48e4cc82
  - 번역: https://coffeewhale.com/k8s/network/2019/05/11/k8s-network-02/
- https://medium.com/google-cloud/understanding-kubernetes-networking-ingress-1bc341c84078
  - 번역: https://coffeewhale.com/k8s/network/2019/05/30/k8s-network-03/

