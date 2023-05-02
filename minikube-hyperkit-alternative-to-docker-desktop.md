# Docker desktop 의 대안 = minikube + hyperkit

Docker Desktop 이 유료화되어 대체재가 필요해졌다.

맥에서 어떤 대체제가 있나 찾아보니 minikube + hyperkit 조합이 편리해보인다.

(추가) Mi Mac에서는 hyperkit을 설치할 수 없어서 이 조합을 사용할 수 없다. 대신에 colima를 사용해야 한다. [여기](https://github.com/HomoEfficio/dev-tips/blob/master/colima-alternative-to-docker-desktop-on-m1-mac.md)를 참고하자.

minikube 는 로컬에서 k8s 클러스터를 만들고 사용할 수 있게 해주는 도구다.

hyperkit 는 맥에서 사용할 수 있는 hypervisor 도구다.


## 간단 개요

Docker Desktop 이 유료화 된 것일 뿐 Docker CLI 와 Docker Engine는 여전히 오픈 소스다.

Docker CLI는 맥에서도 그대로 사용할 수 있는데, Docker Engine 은 리눅스에서만 실행되므로 맥에서는 대체재가 필요하다.

결국 맥에서 Docker Engine 을 실행할 수 있게 해주고, 이 Docker Engine과 Docker CLI가 통신할 수 있게 해주는 것이 Docker Desktop을 대체하는 작업이라고 할 수 있다.

이 작업을 편리하게 수행할 수 있는 방법 중 하나가 minikube + hyperkit 이다.

참고: https://dhwaneetbhatt.com/blog/run-docker-without-docker-desktop-on-macos


## docker cli 설치

>brew install docker

```
xxx 🦑🍕🍺 ❯ brew install docker                                                                                           ⏎
Updating Homebrew...
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/core).
==> New Formulae
actionlint           clickhouse-cpp       gotests              libavif              neovim-qt            pkgconf
apache-pulsar        git-cliff            gtop                 lunzip               openssl@3            red-tldr
bottom               gomodifytags         lastz                lziprecover          pkg-config-wrapper   viddy
==> Updated Formulae
Updated 611 formulae.
==> Deleted Formulae
ocamlsdl                                                         vavrdiasm

==> Downloading https://ghcr.io/v2/homebrew/core/docker/manifests/20.10.8
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/docker/blobs/sha256:6f5af0ff463a59e5e11343f34a6a9f5d194cc8cde56cc9a3a5b70ecdf22
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:6f5af0ff463a59e5e11343f34a6a9f5d194cc8cde56
######################################################################## 100.0%
==> Pouring docker--20.10.8.catalina.bottle.tar.gz
==> Caveats
zsh completions have been installed to:
  /usr/local/share/zsh/site-functions
==> Summary
🍺  /usr/local/Cellar/docker/20.10.8: 12 files, 58.8MB
```

docker 명령을 실행해보면 아래와 같이 docker daemon 을 찾지 못한다.

```
xxx 🦑🍕🍺 ❯ docker container ls
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

이제 docker daemon 이 실행되도록 minikube 와 hyperkit 을 설치한다.


## minikube 설치

docker daemon 실행을 위해 minikube를 설치하는 이유는 간단하다. minikube가 내부적으로 Docker Engine 을 daemon 으로 실행하기 때문이다.

>brew install minikube

```
xxx 🦑🍕🍺 ❯ brew install minikube                                                                                         ⏎
==> Downloading https://ghcr.io/v2/homebrew/core/minikube/manifests/1.23.2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/minikube/blobs/sha256:faa91becd0ebb93d8af0ce849ff57516f3cb090a2c350b0b180025072
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:faa91becd0ebb93d8af0ce849ff57516f3cb090a2c3
######################################################################## 100.0%
==> Pouring minikube--1.23.2.catalina.bottle.tar.gz
==> Caveats
zsh completions have been installed to:
  /usr/local/share/zsh/site-functions
==> Summary
🍺  /usr/local/Cellar/minikube/1.23.2: 9 files, 68.7MB

xxx 🦑🍕🍺 ❯ minikube version
minikube version: v1.23.2
commit: 0a0ad764652082477c00d51d2475284b5d39ceed
```


## hyperkit 설치

hyperkit 는 아직까지는 맥에서만 실행되는 hypervisor다.

>brew install hyperkit

```
xxx 🦑🍕🍺 ❯ brew install hyperkit
==> Downloading https://ghcr.io/v2/homebrew/core/hyperkit/manifests/0.20200908
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/hyperkit/blobs/sha256:26a203b17733ff5166d8c31069e3ecd5af15c74448a51d8b682689cb0
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:26a203b17733ff5166d8c31069e3ecd5af15c74448a
######################################################################## 100.0%
==> Pouring hyperkit--0.20200908.catalina.bottle.tar.gz
🍺  /usr/local/Cellar/hyperkit/0.20200908: 5 files, 4.3MB
```


## hyperkit를 사용해서 minikube 클러스터 실행

>minikube start --driver hyperkit

```
xxx 🦑🍕🍺 ❯ minikube start --driver hyperkit                                                                              ⏎
* Darwin 10.15.7 의 minikube v1.23.2
  - KUBECONFIG=:/Users/user/.kube/config/xxxxx.config
* 유저 환경 설정 정보에 기반하여 hyperkit 드라이버를 사용하는 중
* 드라이버 docker-machine-driver-hyperkit 다운로드 중 :
    > docker-machine-driver-hyper...: 65 B / 65 B [----------] 100.00% ? p/s 0s
    > docker-machine-driver-hyper...: 10.53 MiB / 10.53 MiB  100.00% 2.47 MiB p
* The 'hyperkit' driver requires elevated permissions. The following commands will be executed:

    $ sudo chown root:wheel /Users/user/.minikube/bin/docker-machine-driver-hyperkit 
    $ sudo chmod u+s /Users/user/.minikube/bin/docker-machine-driver-hyperkit 


Password:
* 가상 머신 부트 이미지 다운로드 중 ...
    > minikube-v1.23.1.iso.sha256: 65 B / 65 B [-------------] 100.00% ? p/s 0s
    > minikube-v1.23.1.iso: 225.22 MiB / 225.22 MiB [] 100.00% 5.57 MiB p/s 41s
* minikube 클러스터의 minikube 컨트롤 플레인 노드를 시작하는 중
* 쿠버네티스 v1.22.2 을 다운로드 중 ...
    > preloaded-images-k8s-v13-v1...: 511.84 MiB / 511.84 MiB  100.00% 9.30 MiB
* hyperkit VM (CPUs=2, Memory=4000MB, Disk=20000MB) 를 생성하는 중 ...
* 쿠버네티스 v1.22.2 을 Docker 20.10.8 런타임으로 설치하는 중
  - 인증서 및 키를 생성하는 중 ...
  - 컨트롤 플레인이 부팅...
  - RBAC 규칙을 구성하는 중 ...
* Kubernetes 구성 요소를 확인...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* 애드온 활성화 : storage-provisioner, default-storageclass
* 끝났습니다! kubectl이 "minikube" 클러스터와 "default" 네임스페이스를 기본적으로 사용하도록 구성되었습니다.


xxx 🦑🍕🍺 ❯ kubectx
minikube
...

xxx 🦑🍕🍺 ❯ kubectl get pods
No resources found in default namespace.
```

## docker 설정

위와 같이 minikube + hyperkit 로 k8s 클러스터를 시작해도 아직 docker 를 사용할 수는 없다.

```
xxx 🦑🍕🍺 ❯ docker container ls
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

이제 Docker CLI가 minkube 로 실행한 k8s 클러스터 안에 있는 docker daemon 과 통신할 수 있게 설정해주면 된다. 아래 명령으로 설정값을 확인할 수 있다. 설정을 위해 실행해야 하는 명령까지 친절하게 보여준다.

>minikube docker-env

```
xxx 🦑🍕🍺 ❯ minikube docker-env                                                                                           ⏎
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.64.2:2376"
export DOCKER_CERT_PATH="/Users/user/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"

# To point your shell to minikube's docker-daemon, run:
# eval $(minikube -p minikube docker-env)
```

안내받은 명령을 실행한 후 다시 docker 명령을 실행하면 이번에는 제대로 실행이 된다.

```
xxx 🦑🍕🍺 ❯ eval $(minikube -p minikube docker-env)
xxx 🦑🍕🍺 ❯ docker container ls
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS     NAMES
df5f4c0902c0   6e38f40d628d           "/storage-provisioner"   3 minutes ago   Up 3 minutes             k8s_storage-provisioner_storage-provisioner_kube-system_719b9024-cd89-46e7-b2ca-3019a1ba4b6a_0
417d47433d17   8d147537fb7d           "/coredns -conf /etc…"   3 minutes ago   Up 3 minutes             k8s_coredns_coredns-78fcd69978-7697t_kube-system_bba50d5d-b99e-4623-b8e4-6802a4220a03_0
a4c544ef08a5   873127efbc8a           "/usr/local/bin/kube…"   3 minutes ago   Up 3 minutes             k8s_kube-proxy_kube-proxy-kwwbf_kube-system_2156a888-2388-40e0-ae30-5a2080b10cfb_0
1480494b0881   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_coredns-78fcd69978-7697t_kube-system_bba50d5d-b99e-4623-b8e4-6802a4220a03_0
2a7ebe4ef887   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_storage-provisioner_kube-system_719b9024-cd89-46e7-b2ca-3019a1ba4b6a_0
5e35a8bfe9d7   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_kube-proxy-kwwbf_kube-system_2156a888-2388-40e0-ae30-5a2080b10cfb_0
fd6c2cb3b851   b51ddc1014b0           "kube-scheduler --au…"   3 minutes ago   Up 3 minutes             k8s_kube-scheduler_kube-scheduler-minikube_kube-system_97bca4cd66281ad2157a6a68956c4fa5_0
cf47c0b48406   004811815584           "etcd --advertise-cl…"   3 minutes ago   Up 3 minutes             k8s_etcd_etcd-minikube_kube-system_911c6f255a373704eb5467fbcba00e71_0
418fb9a26299   e64579b7d886           "kube-apiserver --ad…"   3 minutes ago   Up 3 minutes             k8s_kube-apiserver_kube-apiserver-minikube_kube-system_6f3a88bb8d2ad4abe06ffbeb687149ba_0
1d9bb6ed0b57   5425bcbd23c5           "kube-controller-man…"   3 minutes ago   Up 3 minutes             k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_20326866d8a92c47afe958577e1a8179_0
9aa23091476c   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_kube-scheduler-minikube_kube-system_97bca4cd66281ad2157a6a68956c4fa5_0
fa21579b0327   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_kube-controller-manager-minikube_kube-system_20326866d8a92c47afe958577e1a8179_0
6c6df84564c9   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_kube-apiserver-minikube_kube-system_6f3a88bb8d2ad4abe06ffbeb687149ba_0
bba607dd87ac   k8s.gcr.io/pause:3.5   "/pause"                 3 minutes ago   Up 3 minutes             k8s_POD_etcd-minikube_kube-system_911c6f255a373704eb5467fbcba00e71_0
```

이제 Docker Desktop 이 있을 때와 마찬가지로 docker 를 사용할 수 있다.

## minikube 종료

아래와 같이 minikube 클러스터를 종료하면 docker daemon 도 함께 종료된다.

```
xxx 🦑🍕🍺 ❯ minikube stop --all
* Stopping node "minikube"  ...
* 1개의 노드가 중지되었습니다.
```

이제 docker 명령을 실행하면 docker daemon 을 찾지 못한다. 

```
xxx 🦑🍕🍺 ❯ docker images

Cannot connect to the Docker daemon at tcp://192.168.64.2:2376. Is the docker daemon running?
```

다만 금방 타임아웃되지 않고 명령 종료시까지 약 1분 정도 대기해야 하므로 불편하다.

앞에서 eval 명령으로 지정한 환경변수를 해제하거나 터미널을 종료하고 다른 터미널을 사용하면 된다.

환경변수는 아래와 같이 제거할 수 있다.

>unset DOCKER_TLS_VERIFY DOCKER_HOST DOCKER_CERT_PATH MINIKUBE_ACTIVE_DOCKERD

```
xxx 🦑🍕🍺 ❯ unset DOCKER_TLS_VERIFY DOCKER_HOST DOCKER_CERT_PATH MINIKUBE_ACTIVE_DOCKERD                                  
xxx 🦑🍕🍺 ❯ docker images
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```


