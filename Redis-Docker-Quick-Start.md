# Redis Docker Quick Start

## Redis 설치

### 도커 이미지 pull

```
~ 🦑🍺 ❯ docker pull redis:alpine
alpine: Pulling from library/redis
f97344484467: Pull complete 
491979375772: Pull complete 
fa2ce6183da4: Pull complete 
dc0978d3f333: Pull complete 
3a4053039d94: Pull complete 
c2bf5e631951: Pull complete 
Digest: sha256:f23b1e963e2122ce4de6c40ffd105b60ccfa62bf0134585d3109f5caf691b5b3
Status: Downloaded newer image for redis:alpine
docker.io/library/redis:alpine
~ 🦑🍺 ❯ docker images
REPOSITORY   TAG       IMAGE ID       CREATED      SIZE
nginx        latest    fd2d3e51789e   2 days ago   135MB
redis        alpine    9c5d9fbede14   2 days ago   28.2MB
```

### Redis 네트워크 생성

```
~ 🦑🍺 ❯ docker network create redis-net
242ce7f9c45b1f320867ab941b44ed772b4d4d6cfcf05705bd3267d3c951ccdc
~ 🦑🍺 ❯ docker network ls              
NETWORK ID     NAME        DRIVER    SCOPE
d88bba93ff0e   bridge      bridge    local
a894f76022b2   host        host      local
e3a9951f3a26   none        null      local
242ce7f9c45b   redis-net   bridge    local
```

### Redis 실행

```
~ 🦑🍺 ❯ docker run --rm --name omw-redis \
                    -p 6379:6379 \
                    --network redis-net \
                    -v /Users/user/dev-infra/redis/data \
                    -d redis:alpine redis-server --appendOnly yes
2f940dd8931dabe87df33210a366ed406f65d5996ed82798bb809c6834190899
~ 🦑🍺 ❯ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                       NAMES
2f940dd8931d   redis:alpine   "docker-entrypoint.s…"   4 seconds ago   Up 2 seconds   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   omw-redis
```

### redis-cli 로 Redis 서버에 접속

```
~ 🦑🍺 ❯ docker run --rm -it --network redis-net redis:alpine redis-cli -h omw-redis     
omw-redis:6379> keys *
(empty array)
omw-redis:6379> 
```

