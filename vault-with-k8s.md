# Vault with k8s

from: https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar?in=vault/kubernetes

- helm chart 로 vault를 설치
  - vault, vault-agent-injector 생성
- vault 에 setcret 값 저장, 정책 등 설정
- vault 에 접근하는 기능을 기존 deployment에 추가해주는 annotation 설정
- 기존 deployment 에 annotation 적용 후 재배포
  - secret 값이 저장될 파일 경로를 annotation 으로 저징
- app pod 내부의 해당 경로에 secret 값 저장된 파일 생성
- app 에서는 해당 파일 읽어서 사용


## 기본 환경 확인

```
Your Interactive Bash Terminal. A safe place to learn and execute commands.

$ docker version
Client:
 Version:      17.03.0-ce
 API version:  1.26
 Go version:   go1.7.5
 Git commit:   3a232c8
 Built:        Tue Feb 28 07:52:04 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.0-ce
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   3a232c8
 Built:        Tue Feb 28 07:52:04 2017
 OS/Arch:      linux/amd64
 Experimental: false
$ minikube version
minikube version: v1.12.0
commit: c83e6c47124b71190e138dbc687d2556d31488d6
$ helm version
version.BuildInfo{Version:"v3.2.1", GitCommit:"fe51cd1e31e6a202cba7dead9552a6d418ded79a", GitTreeState:"clean", GoVersion:"go1.13.10"}
$ minikube version
minikube version: v1.12.0
commit: c83e6c47124b71190e138dbc687d2556d31488d6
```

### minikube 구동

```
$ minikube start --vm-driver none --bootstrapper kubeadm
* minikube v1.12.0 on Ubuntu 16.04 (kvm/amd64)
* Using the none driver based on user configuration
* Starting control plane node minikube in cluster minikube
* Running on localhost (CPUs=2, Memory=2467MB, Disk=45126MB) ...
* OS release is Ubuntu 16.04.2 LTS
* Preparing Kubernetes v1.18.3 on Docker 17.03.0-ce ...
    > kubeadm.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
    > kubectl.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
    > kubelet.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
    > kubectl: 41.99 MiB / 41.99 MiB [---------------] 100.00% 53.99 MiB p/s 1s
    > kubeadm: 37.97 MiB / 37.97 MiB [---------------] 100.00% 44.66 MiB p/s 1s
    > kubelet: 108.04 MiB / 108.04 MiB [-------------] 100.00% 54.27 MiB p/s 2s

* Configuring local host environment ...
* Verifying Kubernetes components...
* Enabled addons: default-storageclass, storage-provisioner
* Done! kubectl is now configured to use "minikube"
$ 
$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

## helm hashicorp repo 추가 및 vault 설치, 확인

```
$ helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" has been added to your repositories
$
$
$
$ helm install vault hashicorp/vault --set "server.dev.enabled=true"
NAME: vault
LAST DEPLOYED: Fri May 21 01:57:02 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
$ helm status vault
NAME: vault
LAST DEPLOYED: Fri May 21 01:57:02 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
$ helm get manifest vault
---
# Source: vault/templates/injector-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-agent-injector
  namespace: default
  labels:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
---
# Source: vault/templates/server-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault
  namespace: default
  labels:
    helm.sh/chart: vault-0.11.0
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
---
# Source: vault/templates/injector-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vault-agent-injector-clusterrole
  labels:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
rules:
- apiGroups: ["admissionregistration.k8s.io"]
  resources: ["mutatingwebhookconfigurations"]
  verbs: 
    - "get"
    - "list"
    - "watch"
    - "patch"
---
# Source: vault/templates/injector-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-agent-injector-binding
  labels:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vault-agent-injector-clusterrole
subjects:
- kind: ServiceAccount
  name: vault-agent-injector
  namespace: default
---
# Source: vault/templates/server-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-server-binding
  labels:
    helm.sh/chart: vault-0.11.0
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault
  namespace: default
---
# Source: vault/templates/injector-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: vault-agent-injector-svc
  namespace: default
  labels:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
  
spec:
  ports:
  - name: https
    port: 443
    targetPort: 8080
  selector:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    component: webhook
---
# Source: vault/templates/server-headless-service.yaml
# Service for Vault cluster
apiVersion: v1
kind: Service
metadata:
  name: vault-internal
  namespace: default
  labels:
    helm.sh/chart: vault-0.11.0
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
  annotations:

spec:
  clusterIP: None
  publishNotReadyAddresses: true
  ports:
    - name: "http"
      port: 8200
      targetPort: 8200
    - name: https-internal
      port: 8201
      targetPort: 8201
  selector:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    component: server
---
# Source: vault/templates/server-service.yaml
# Service for Vault cluster
apiVersion: v1
kind: Service
metadata:
  name: vault
  namespace: default
  labels:
    helm.sh/chart: vault-0.11.0
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
  annotations:

spec:
  # We want the servers to become available even if they're not ready
  # since this DNS is also used for join operations.
  publishNotReadyAddresses: true
  ports:
    - name: http
      port: 8200
      targetPort: 8200
    - name: https-internal
      port: 8201
      targetPort: 8201
  selector:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    component: server
---
# Source: vault/templates/injector-deployment.yaml
# Deployment for the injector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-agent-injector
  namespace: default
  labels:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
    component: webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: vault-agent-injector
      app.kubernetes.io/instance: vault
      component: webhook
  template:
    metadata:
      labels:
        app.kubernetes.io/name: vault-agent-injector
        app.kubernetes.io/instance: vault
        component: webhook
    spec:
      
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: vault-agent-injector
                  app.kubernetes.io/instance: "vault"
                  component: webhook
              topologyKey: kubernetes.io/hostname
  
      
      
      serviceAccountName: "vault-agent-injector"
      hostNetwork: false
      securityContext:
        runAsNonRoot: true
        runAsGroup: 1000
        runAsUser: 100
      containers:
        - name: sidecar-injector
          
          image: "hashicorp/vault-k8s:0.10.0"
          imagePullPolicy: "IfNotPresent"
          securityContext:
            allowPrivilegeEscalation: false
          env:
            - name: AGENT_INJECT_LISTEN
              value: :8080
            - name: AGENT_INJECT_LOG_LEVEL
              value: info
            - name: AGENT_INJECT_VAULT_ADDR
              value: http://vault.default.svc:8200
            - name: AGENT_INJECT_VAULT_AUTH_PATH
              value: auth/kubernetes
            - name: AGENT_INJECT_VAULT_IMAGE
              value: "vault:1.7.0"
            - name: AGENT_INJECT_TLS_AUTO
              value: vault-agent-injector-cfg
            - name: AGENT_INJECT_TLS_AUTO_HOSTS
              value: vault-agent-injector-svc,vault-agent-injector-svc.default,vault-agent-injector-svc.default.svc
            - name: AGENT_INJECT_LOG_FORMAT
              value: standard
            - name: AGENT_INJECT_REVOKE_ON_SHUTDOWN
              value: "false"
            - name: AGENT_INJECT_CPU_REQUEST
              value: "250m"
            - name: AGENT_INJECT_CPU_LIMIT
              value: "500m"
            - name: AGENT_INJECT_MEM_REQUEST
              value: "64Mi"
            - name: AGENT_INJECT_MEM_LIMIT
              value: "128Mi"
            - name: AGENT_INJECT_DEFAULT_TEMPLATE
              value: "map"
            
          args:
            - agent-inject
            - 2>&1
          livenessProbe:
            httpGet:
              path: /health/ready
              port: 8080
              scheme: HTTPS
            failureThreshold: 2
            initialDelaySeconds: 5
            periodSeconds: 2
            successThreshold: 1
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
              scheme: HTTPS
            failureThreshold: 2
            initialDelaySeconds: 5
            periodSeconds: 2
            successThreshold: 1
            timeoutSeconds: 5
---
# Source: vault/templates/server-statefulset.yaml
# StatefulSet to run the actual vault server cluster.
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vault
  namespace: default
  labels:
    app.kubernetes.io/name: vault
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
spec:
  serviceName: vault-internal
  podManagementPolicy: Parallel
  replicas: 1
  updateStrategy:
    type: OnDelete
  selector:
    matchLabels:
      app.kubernetes.io/name: vault
      app.kubernetes.io/instance: vault
      component: server
  template:
    metadata:
      labels:
        helm.sh/chart: vault-0.11.0
        app.kubernetes.io/name: vault
        app.kubernetes.io/instance: vault
        component: server
    spec:
      
      
      
      terminationGracePeriodSeconds: 10
      serviceAccountName: vault
      
      securityContext:
        runAsNonRoot: true
        runAsGroup: 1000
        runAsUser: 100
        fsGroup: 1000
      volumes:
        
        - name: home
          emptyDir: {}
      containers:
        - name: vault
          
          image: vault:1.7.0
          imagePullPolicy: IfNotPresent
          command:
          - "/bin/sh"
          - "-ec"
          args: 
          - |
            /usr/local/bin/docker-entrypoint.sh vault server -dev 
  
          securityContext:
            allowPrivilegeEscalation: false
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: VAULT_K8S_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: VAULT_K8S_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: VAULT_ADDR
              value: "http://127.0.0.1:8200"
            - name: VAULT_API_ADDR
              value: "http://$(POD_IP):8200"
            - name: SKIP_CHOWN
              value: "true"
            - name: SKIP_SETCAP
              value: "true"
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: VAULT_CLUSTER_ADDR
              value: "https://$(HOSTNAME).vault-internal:8201"
            - name: HOME
              value: "/home/vault"
            
            - name: VAULT_DEV_ROOT_TOKEN_ID
              value: root
            - name: VAULT_DEV_LISTEN_ADDRESS
              value: "[::]:8200"
  
            
            
          volumeMounts:
          
  
  
            - name: home
              mountPath: /home/vault
          ports:
            - containerPort: 8200
              name: http
            - containerPort: 8201
              name: https-internal
            - containerPort: 8202
              name: http-rep
          readinessProbe:
            # Check status; unsealed vault servers return 0
            # The exit code reflects the seal status:
            #   0 - unsealed
            #   1 - error
            #   2 - sealed
            exec:
              command: ["/bin/sh", "-ec", "vault status -tls-skip-verify"]
            failureThreshold: 2
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 3
          lifecycle:
            # Vault container doesn't receive SIGTERM from Kubernetes
            # and after the grace period ends, Kube sends SIGKILL.  This
            # causes issues with graceful shutdowns such as deregistering itself
            # from Consul (zombie services).
            preStop:
              exec:
                command: [
                  "/bin/sh", "-c",
                  # Adding a sleep here to give the pod eviction a
                  # chance to propagate, so requests will not be made
                  # to this pod while it's terminating
                  "sleep 5 && kill -SIGTERM $(pidof vault)",
                ]
---
# Source: vault/templates/injector-mutating-webhook.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: vault-agent-injector-cfg
  labels:
    app.kubernetes.io/name: vault-agent-injector
    app.kubernetes.io/instance: vault
    app.kubernetes.io/managed-by: Helm
webhooks:
  - name: vault.hashicorp.com
    sideEffects: None
    admissionReviewVersions:
    - "v1beta1"
    - "v1"
    clientConfig:
      service:
        name: vault-agent-injector-svc
        namespace: default
        path: "/mutate"
      caBundle: ""
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    failurePolicy: Ignore
```

## 배포 확인 

```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          41s
vault-agent-injector-64b8c58fff-sfgt9   1/1     Running   0          41s
```

## vault-0 에 비밀 정보 등록

```
$ kubectl exec -it vault-0 -- /bin/sh
/ $ vault secrets enable -path=internal kv-v2
Success! Enabled the kv-v2 secrets engine at: internal/
/ $ vault kv put internal/database/config username="db-readonly-username" passwords="db-secret-password"
Key              Value
---              -----
created_time     2021-05-21T02:01:53.744360997Z
deletion_time    n/a
destroyed        false
version          1
/ $ vault kv get internal/database/config
====== Metadata ======
Key              Value
---              -----
created_time     2021-05-21T02:01:53.744360997Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key          Value
---          -----
passwords    db-secret-password
username     db-readonly-username
/ $ exit
```

## vault - k8s 연동 설정

- k8s의 Service Account Token 을 활용
- 이 토큰은 각 pod 가 실행될 때마다 pod에 제공된다?
- 정책을 vault 에 등록
- (아직 만들어지지 않은) k8s service account 를 참조하도록 설정

```
$ kubectl exec -it vault-0 -- /bin/sh
/ $ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
/ $
/ $
/ $
/ $ cat /var/run/secrets/kubernetes.io/serviceaccount/token 
eyJhbGciOiJSUzI1NiIsImtpZCI6InFwalVLLW84dXBSUEdhZF9oRDd6NXdTaUJSWHlPUjBZU0w3R3lhUHRnYkkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InZhdWx0LXRva2VuLWNrOTVzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYjE3MjA0ZDItOWQwOS00NDA3LWI1ODItYmE2NjhlYTAwOGI5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6dmF1bHQifQ.X_HASoxg_y2FXA8jQDV3hN4WmyqcFdveApNHEPBWTXYE5523SJx1iHu5nwskfmUdUegIUbex7eDQ2pvE3Pd5mMkSm1WrdKUNXDrw4R-2lMYnUS8rak2yoA0bD7WWTiZpFbpoQARSvDzvSoAC8c-SoaryI7zt6hY5nKzRgeEZ47X6DnrkRciW1YxZxgEU2K06Q-SXhOwGdKu2QwsVoRKnw1ZtsBF2unWyHqULhwlTr-ScdrLfZH23vSSANr5N3dzpc6_XiFHTm7eBbApAjIjzsTPqn6GoyehrFl0QqlljNK67i-Jua2QnplLwkXHGQTYC394UVw77Fgw4ybaMvgV9dg/ $ 
/ $
/ $
/ $ vault write auth/kubernetes/config \
>         token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
>         kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
>         kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
Success! Data written to: auth/kubernetes/config
/ $ 
/ $ 
/ $ vault policy write internal-app - <<EOF
> path "internal/data/database/config" {
>   capabilities = ["read"]
> }
> EOF
Success! Uploaded policy: internal-app
/ $ 
/ $ 
/ $ 
/ $ vault write auth/kubernetes/role/internal-app \
>         bound_service_account_names=internal-app \
>         bound_service_account_namespaces=default \
>         policies=internal-app \
>         ttl=24h
Success! Data written to: auth/kubernetes/role/internal-app
/ $ 
/ $ 
/ $ 
/ $ exit
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          22m
vault-agent-injector-64b8c58fff-sfgt9   1/1     Running   0          22m
```

## k8s service account 추가

```
$ kubectl get serviceaccounts
NAME                   SECRETS   AGE
default                1         27m
vault                  1         24m
vault-agent-injector   1         24m
$ 
$ 
$ 
$ cat service-account-internal-app.yml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app
$ 
$ 
$ 
$ kubectl apply --filename service-account-internal-app.yml
serviceaccount/internal-app created
$ 
$ 
$ kubectl get serviceaccounts
NAME                   SECRETS   AGE
default                1         28m
internal-app           1         11s
vault                  1         25m
vault-agent-injector   1         25m
```

## vault 를 사용할 애플리케이션 구동

```
$ cat deployment-orgchart.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orgchart
  labels:
    app: orgchart
spec:
  selector:
    matchLabels:
      app: orgchart
  replicas: 1
  template:
    metadata:
      annotations:
      labels:
        app: orgchart
    spec:
      serviceAccountName: internal-app
      containers:
        - name: orgchart
          image: jweissig/app:0.0.1
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          27m
vault-agent-injector-64b8c58fff-sfgt9   1/1     Running   0          27m
$ 
$ 
$ 
$ kubectl apply --filename deployment-orgchart.yml 
deployment.apps/orgchart created
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-7f6b86f74f-hn4ps               1/1     Running   0          80s
vault-0                                 1/1     Running   0          28m
vault-agent-injector-64b8c58fff-sfgt9   1/1     Running   0          28m
```

애플리케이션 pod에 있는 orgchart container에 아무 secrets 도 없는 것 확인

```
$ kubectl exec $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container orgchart -- ls /vault/secrets
ls: /vault/secrets: No such file or directory
command terminated with exit code 1
```


## secret를 애플리케이션 pod에 주입

애플리케이션 deployment 는 default namespace에 이름이 internal-app 인 k8s service account 를 가진 채 실행되고 있다.

Vault Agent Injector는 특정 애노테이션이 포함된 deployment 만 수정한다. 애노테이션은 나중에 추가할 수도 있다.

```
$ cat patch-inject-secrets.yml 
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "internal-app"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
$ 
$ 
$ 
$ kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets.yml)"
deployment.apps/orgchart patched
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-7654cd56f9-8k9zp               2/2     Running   0          13s
vault-0                                 1/1     Running   0          6m33s
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running   0          6m33s
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-7654cd56f9-8k9zp               2/2     Running   0          57s
vault-0                                 1/1     Running   0          7m17s
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running   0          7m17s
$ 
$ 
$ 
$ kubectl logs $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container vault-agent
==> Vault agent started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.7.0
             Version Sha: 4e222b85c40a810b74400ee3c54449479e32bb9f

2021-05-21T02:41:15.076Z [INFO]  sink.file: creating file sink
2021-05-21T02:41:15.076Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2021-05-21T02:41:15.080Z [INFO]  template.server: starting template server
2021-05-21T02:41:15.080Z [INFO]  auth.handler: starting auth handler
2021-05-21T02:41:15.080Z [INFO]  auth.handler: authenticating
2021-05-21T02:41:15.080Z [INFO]  sink.server: starting sink server
[INFO] (runner) creating new runner (dry: false, once: false)
[INFO] (runner) creating watcher
2021-05-21T02:41:15.094Z [INFO]  auth.handler: authentication successful, sending token to sinks
2021-05-21T02:41:15.094Z [INFO]  auth.handler: starting renewal process
2021-05-21T02:41:15.095Z [INFO]  sink.file: token written: path=/home/vault/.vault-token
2021-05-21T02:41:15.095Z [INFO]  template.server: template server received new token
[INFO] (runner) stopping
[INFO] (runner) creating new runner (dry: false, once: false)
[INFO] (runner) creating watcher
[INFO] (runner) starting
2021-05-21T02:41:15.099Z [INFO]  auth.handler: renewed auth token
$ 
$ 
$ 
$ kubectl exec $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container orgchart -- cat /vault/secrets/database-config.txt
data: map[password:db-secret-password username:db-readonly-username]
metadata: map[created_time:2021-05-21T02:37:09.175613971Z deletion_time: destroyed:false version:1]
```


## 주입된 secret를 사용할 수 있게끔 template 적용

```
$ cat patch-inject-secrets-as-template.yml 
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/role: "internal-app"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}$ 
$ 
$ 
$ 
$ kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets-as-template.yml)"
deployment.apps/orgchart patched
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running       0          10s
orgchart-7654cd56f9-8k9zp               0/2     Terminating   0          8m
vault-0                                 1/1     Running       0          14m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running       0          14m
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running   0          22s
vault-0                                 1/1     Running   0          14m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running   0          14m
$ 
$ 
$ 
$ kubectl exec -it $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") -c orgchart -- cat /vault/secrets/database-config.txtpostgresql://db-readonly-username:db-secret-password@postgres:5432/wizard
```


## 처음부터 애노테이션 적용해서 deploy

```
$ cat pod-payroll.yml 
apiVersion: v1
kind: Pod
metadata:
  name: payroll
  labels:
    app: payroll
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "internal-app"
    vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
    vault.hashicorp.com/agent-inject-template-database-config.txt: |
      {{- with secret "internal/data/database/config" -}}
      postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
      {{- end -}}
spec:
  serviceAccountName: internal-app
  containers:
    - name: payroll
      image: jweissig/app:0.0.1
$ 
$ 
$ 
$ kubectl apply --filename pod-payroll.yml
pod/payroll created
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running   0          2m31s
payroll                                 2/2     Running   0          4s
vault-0                                 1/1     Running   0          16m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running   0          16m
$ 
$ 
$ 
$ kubectl exec payroll --container payroll -- cat /vault/secrets/database-config.txt
postgresql://db-readonly-username:db-secret-password@postgres:5432/wizard
```


## secret는 service account 에 바운드

Vault k8s auth role 에서 정의된 service account가 아니라 다른 service account를 가지고 있는 pod는 vault에 저장된 secret에 접근 불가


```
$ cat deployment-website.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website
  labels:
    app: website
spec:
  selector:
    matchLabels:
      app: website
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "internal-app"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
      labels:
        app: website
    spec:
      # This service account does not have permission to request the secrets.
      serviceAccountName: website
      containers:
        - name: website
          image: jweissig/app:0.0.1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: website$ 
$ 
$ 
$ 
$ kubectl apply --filename deployment-website.yml
deployment.apps/website created
serviceaccount/website created
$ 
$ 
$ 
```

### Deploy 실패

```
$ kubectl get pods
NAME                                    READY   STATUS     RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running    0          4m49s
payroll                                 2/2     Running    0          2m22s
vault-0                                 1/1     Running    0          18m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running    0          18m
website-88fdd8759-sx4ph                 0/2     Init:0/1   0          2s
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS     RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running    0          5m4s
payroll                                 2/2     Running    0          2m37s
vault-0                                 1/1     Running    0          19m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running    0          19m
website-88fdd8759-sx4ph                 0/2     Init:0/1   0          17s
$ 
$ 
$ 
```

### Deploy 실패 로그 확인

```
$ kubectl logs $(kubectl get pod -l app=website -o jsonpath="{.items[0].metadata.name}") --container vault-agent-init
==> Vault agent started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.7.0
             Version Sha: 4e222b85c40a810b74400ee3c54449479e32bb9f

2021-05-21T02:53:51.482Z [INFO]  sink.file: creating file sink
2021-05-21T02:53:51.482Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2021-05-21T02:53:51.484Z [INFO]  template.server: starting template server
[INFO] (runner) creating new runner (dry: false, once: false)
[INFO] (runner) creating watcher
2021-05-21T02:53:51.485Z [INFO]  auth.handler: starting auth handler
2021-05-21T02:53:51.485Z [INFO]  auth.handler: authenticating
2021-05-21T02:53:51.488Z [INFO]  sink.server: starting sink server
$ 
$ 
$ 
```

### vault 에서 정의된 service account 로 수정 배포

```
$ cat patch-website.yml 
spec:
  template:
    spec:
      serviceAccountName: internal-app
$ 
$ 
$ 
$ 
$ kubectl patch deployment website --patch "$(cat patch-website.yml)"
deployment.apps/website patched
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running       0          5m48s
payroll                                 2/2     Running       0          3m21s
vault-0                                 1/1     Running       0          19m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running       0          19m
website-788d689b87-bqdhl                2/2     Running       0          3s
website-88fdd8759-sx4ph                 0/2     Terminating   0          61s
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running       0          5m58s
payroll                                 2/2     Running       0          3m31s
vault-0                                 1/1     Running       0          20m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running       0          20m
website-788d689b87-bqdhl                2/2     Running       0          13s
website-88fdd8759-sx4ph                 0/2     Terminating   0          71s
$ 
$ 
$ 
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-6cc49fd78f-s2jrc               2/2     Running   0          6m17s
payroll                                 2/2     Running   0          3m50s
vault-0                                 1/1     Running   0          20m
vault-agent-injector-64b8c58fff-j6qvp   1/1     Running   0          20m
website-788d689b87-bqdhl                2/2     Running   0          32s
$ 
$ 
$ 
$ kubectl exec \
>     $(kubectl get pod -l app=website -o jsonpath="{.items[0].metadata.name}") \
>     --container website -- cat /vault/secrets/database-config.txt
postgresql://db-readonly-username:db-secret-password@postgres:5432/wizard
```


## secret 은 namespace 에 바운드

secret은 namespace에 바운드 돼 있어서 다른 namespace에 정의된 secret에는 접근할 수 없다.

### 새로운 offsite namespace 생성 및 namespace 변경

```
$ kubectl create namespace offsite
namespace/offsite created
$ 
$ 
$ 
$ kubectl config set-context --current --namespace offsite
Context "minikube" modified.
$ 
$ 
$
```

### service account 생성 및 app deploy

deploy 에서 애노테이션으로 지정한 secret 에 접근 불가로 deploy 실패

``` 
$ cat service-account-internal-app.yml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app
$ 
$ 
$ 
$ kubectl apply --filename service-account-internal-app.yml
serviceaccount/internal-app created
$ 
$ 
$ 
$ cat deployment-issues.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: issues
  labels:
    app: issues
spec:
  selector:
    matchLabels:
      app: issues
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "internal-app"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
      labels:
        app: issues
    spec:
      serviceAccountName: internal-app
      containers:
        - name: issues
          image: jweissig/app:0.0.1
$ 
$ 
$ 
$ kubectl apply --filename deployment-issues.yml
deployment.apps/issues created
$ 
$ 
$ 
$ kubectl get pods
NAME                      READY   STATUS     RESTARTS   AGE
issues-79d8bf7cdf-rpkkl   0/2     Init:0/1   0          2s
$ 
$ 
$ 
```

### deploy 실패 로그 확인

```
$ kubectl logs $(kubectl get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") --container vault-agent-init
==> Vault agent started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.7.0
             Version Sha: 4e222b85c40a810b74400ee3c54449479e32bb9f

2021-05-21T02:58:01.848Z [INFO]  sink.file: creating file sink
2021-05-21T02:58:01.849Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2021-05-21T02:58:01.850Z [INFO]  template.server: starting template server
2021-05-21T02:58:01.850Z [INFO]  auth.handler: starting auth handler
2021-05-21T02:58:01.850Z [INFO]  auth.handler: authenticating
[INFO] (runner) creating new runner (dry: false, once: false)
[INFO] (runner) creating watcher
2021-05-21T02:58:01.851Z [INFO]  sink.server: starting sink server
$ 
$ 
$ 
```

### default namespace 로 돌아가서 offsite namespace 안에서 사용할 role 새로 생성

```
$ kubectl exec --namespace default -it vault-0 -- /bin/sh
/ $ 
/ $ 
/ $ 
/ $ vault write auth/kubernetes/role/offsite-app \
>     bound_service_account_names=internal-app \
>     bound_service_account_namespaces=offsite \
>     policies=internal-app \
>     ttl=24h
Success! Data written to: auth/kubernetes/role/offsite-app
/ $ 
/ $ 
/ $ 
/ $ exit
$ 
$ 
$ 
```

### deployment 스펙이 offsite-app role 을 사용하도록 수정

```
$ cat patch-issues.yml 
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/role: "offsite-app"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}
$ 
$ 
$ 
```

### 수정 배포 성공 및 secret 확인

```
$ kubectl patch deployment issues --patch "$(cat patch-issues.yml)"
deployment.apps/issues patched
$ 
$ 
$ 
$ kubectl get pods
NAME                      READY   STATUS        RESTARTS   AGE
issues-79d8bf7cdf-rpkkl   0/2     Terminating   0          64s
issues-7fd66f98f6-bj7p2   2/2     Running       0          3s
$ 
$ 
$ 
$ kubectl get pods
NAME                      READY   STATUS        RESTARTS   AGE
issues-79d8bf7cdf-rpkkl   0/2     Terminating   0          83s
issues-7fd66f98f6-bj7p2   2/2     Running       0          22s
$ 
$ 
$ 
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
issues-7fd66f98f6-bj7p2   2/2     Running   0          55s
$ 
$ 
$ 
$ kubectl exec \
>     $(kubectl get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") \
>     --container issues -- cat /vault/secrets/database-config.txt
postgresql://db-readonly-username:db-secret-password@postgres:5432/wizard
```
