# Docker desktop 의 대안 = colima

Docker Desktop 이 유료화되어 대체재가 필요해졌다.

M1 맥에서 어떤 대체제가 있나 찾아보니 [colima](https://github.com/abiosoft/colima)가 편리해보인다.

홈페이지를 보니 이렇게 나와 있어서 어떤 도구인지 금방 감잡을 수 있었다.

>What is with the name?
>
>Colima means Containers in Lima.
>Since Lima is aka Linux on Mac. By transitivity, Colima can also mean Containers on Linux on Mac.

결국 맥 위에 리눅스 가상머신인 Lima를 띄우고 그 위에서 실행되는 컨테이너 런타임이라고 이해하면 될 것 같다.

## 간단 개요

Docker Desktop 이 유료화 된 것일 뿐 Docker CLI 와 Docker Engine는 여전히 오픈 소스다.

Docker CLI는 맥에서도 그대로 사용할 수 있는데, Docker Engine 은 리눅스에서만 실행되므로 맥에서는 대체재가 필요하다.

결국 맥에서 Docker Engine 을 실행할 수 있게 해주고, 이 Docker Engine과 Docker CLI가 통신할 수 있게 해주는 것이 Docker Desktop을 대체하는 작업이라고 할 수 있다.

이 작업을 편리하게 수행할 수 있는 방법 중 하나가 colima 이다.


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

## colima 설치

>brew install colima

설치에 시간이 꽤 오래(약 3분 정도) 걸린다.

```
xxx 🦑🍺 ❯ brew install colima                                                                                        ⏎ ✹ ✭
==> Fetching dependencies for colima: capstone, gettext, pcre2, glib, gmp, bdw-gc, m4, libtool, libunistring, pkg-config, guile, libidn2, libtasn1, nettle, p11-kit, libnghttp2, unbound, gnutls, jpeg-turbo, libpng, libslirp, libssh, libusb, lzo, ncurses, pixman, snappy, vde, xz, qemu and lima
==> Fetching capstone
==> Downloading https://ghcr.io/v2/homebrew/core/capstone/manifests/4.0.2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/capstone/blobs/sha256:53b534ac0a71300d7e005cb7552a2778ff6d0e3bcaaa6417303c0c86112a6884
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:53b534ac0a71300d7e005cb7552a2778ff6d0e3bcaaa6417303c0c86112a6884?se=2022-12-24
######################################################################## 100.0%
==> Fetching gettext
==> Downloading https://ghcr.io/v2/homebrew/core/gettext/manifests/0.21.1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/gettext/blobs/sha256:356b52e24b883af3ef092d13b6727b76e0137154c2c9eb42fe7c272bb7d3edec
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:356b52e24b883af3ef092d13b6727b76e0137154c2c9eb42fe7c272bb7d3edec?se=2022-12-24
######################################################################## 100.0%
==> Fetching pcre2
==> Downloading https://ghcr.io/v2/homebrew/core/pcre2/manifests/10.42
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/pcre2/blobs/sha256:23ce93cf86bd4816b7d039efa0a5d68c751bce3f552a8cbf41762518b4be199e
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:23ce93cf86bd4816b7d039efa0a5d68c751bce3f552a8cbf41762518b4be199e?se=2022-12-24
######################################################################## 100.0%
==> Fetching glib
==> Downloading https://ghcr.io/v2/homebrew/core/glib/manifests/2.74.3
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/glib/blobs/sha256:2476e85f3c4fd1648bb09efbdf812aba44b4af1723205e2376ad5ff787d2d60d
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:2476e85f3c4fd1648bb09efbdf812aba44b4af1723205e2376ad5ff787d2d60d?se=2022-12-24
######################################################################## 100.0%
==> Fetching gmp
==> Downloading https://ghcr.io/v2/homebrew/core/gmp/manifests/6.2.1_1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/gmp/blobs/sha256:a43a2ae4c44d90626b835a968a32327c8b8bbf754ec1d2590f8ac656c71dace9
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:a43a2ae4c44d90626b835a968a32327c8b8bbf754ec1d2590f8ac656c71dace9?se=2022-12-24
######################################################################## 100.0%
==> Fetching bdw-gc
==> Downloading https://ghcr.io/v2/homebrew/core/bdw-gc/manifests/8.2.2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/bdw-gc/blobs/sha256:01693d25c01c27b4ae2fc7c176f57c1c46849c24440f1da484df9a2e99074594
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:01693d25c01c27b4ae2fc7c176f57c1c46849c24440f1da484df9a2e99074594?se=2022-12-24
######################################################################## 100.0%
==> Fetching m4
==> Downloading https://ghcr.io/v2/homebrew/core/m4/manifests/1.4.19
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/m4/blobs/sha256:8e9fa0d7d946f7c38e1a6f596aab3169d2440fccd34ec321b9a032d903ec951c
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:8e9fa0d7d946f7c38e1a6f596aab3169d2440fccd34ec321b9a032d903ec951c?se=2022-12-24
######################################################################## 100.0%
==> Fetching libtool
==> Downloading https://ghcr.io/v2/homebrew/core/libtool/manifests/2.4.7
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libtool/blobs/sha256:5f92327e52a1196e8742cb5b3a64498c811d51aed658355c205858d2a835fdc9
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:5f92327e52a1196e8742cb5b3a64498c811d51aed658355c205858d2a835fdc9?se=2022-12-24
######################################################################## 100.0%
==> Fetching libunistring
==> Downloading https://ghcr.io/v2/homebrew/core/libunistring/manifests/1.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libunistring/blobs/sha256:b8b2f6fe30eefd002bf0dbb5fc0e5c6dc0d5f9b9219f4d6fcddc48e3bc229b23
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:b8b2f6fe30eefd002bf0dbb5fc0e5c6dc0d5f9b9219f4d6fcddc48e3bc229b23?se=2022-12-24
######################################################################## 100.0%
==> Fetching pkg-config
==> Downloading https://ghcr.io/v2/homebrew/core/pkg-config/manifests/0.29.2_3
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/pkg-config/blobs/sha256:2af9bceb60b70a259f236f1d46d2bb24c4d0a4af8cd63d974dde4d76313711e0
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:2af9bceb60b70a259f236f1d46d2bb24c4d0a4af8cd63d974dde4d76313711e0?se=2022-12-24
######################################################################## 100.0%
==> Fetching guile
==> Downloading https://ghcr.io/v2/homebrew/core/guile/manifests/3.0.8_2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/guile/blobs/sha256:56fc54551418481510668be3665501ebae56e681856c761d2246117760e18b7a
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:56fc54551418481510668be3665501ebae56e681856c761d2246117760e18b7a?se=2022-12-24
######################################################################## 100.0%
==> Fetching libidn2
==> Downloading https://ghcr.io/v2/homebrew/core/libidn2/manifests/2.3.4
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libidn2/blobs/sha256:38eed5a97aaddeebf0c510ed609466c2c0d1fdc996452e380a4ed9366000fe5e
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:38eed5a97aaddeebf0c510ed609466c2c0d1fdc996452e380a4ed9366000fe5e?se=2022-12-24
######################################################################## 100.0%
==> Fetching libtasn1
==> Downloading https://ghcr.io/v2/homebrew/core/libtasn1/manifests/4.19.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libtasn1/blobs/sha256:cf95a18e2fabf1675d77ec8a1abb41fdb091cef689dec3318a420ad2f25beb76
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:cf95a18e2fabf1675d77ec8a1abb41fdb091cef689dec3318a420ad2f25beb76?se=2022-12-24
######################################################################## 100.0%
==> Fetching nettle
==> Downloading https://ghcr.io/v2/homebrew/core/nettle/manifests/3.8.1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/nettle/blobs/sha256:f2fa03ad5664fdcf8475c1490a22f66d26056779911fd92ae2cb0d36998319a4
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:f2fa03ad5664fdcf8475c1490a22f66d26056779911fd92ae2cb0d36998319a4?se=2022-12-24
######################################################################## 100.0%
==> Fetching p11-kit
==> Downloading https://ghcr.io/v2/homebrew/core/p11-kit/manifests/0.24.1_1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/p11-kit/blobs/sha256:092795b583a9f4e529eca159e4fbadfb4c92b4af1b62174e0e7882f8a7961908
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:092795b583a9f4e529eca159e4fbadfb4c92b4af1b62174e0e7882f8a7961908?se=2022-12-24
######################################################################## 100.0%
==> Fetching libnghttp2
==> Downloading https://ghcr.io/v2/homebrew/core/libnghttp2/manifests/1.51.0-1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libnghttp2/blobs/sha256:0449303373bd44645f0ff77464a4f99f600b9059212b0b31aa906580074ee3fc
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:0449303373bd44645f0ff77464a4f99f600b9059212b0b31aa906580074ee3fc?se=2022-12-24
######################################################################## 100.0%
==> Fetching unbound
==> Downloading https://ghcr.io/v2/homebrew/core/unbound/manifests/1.17.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/unbound/blobs/sha256:cd06e5b7f62103ad750fab0d5cfdb933c93fc1e40c7769605697b4c8777986b6
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:cd06e5b7f62103ad750fab0d5cfdb933c93fc1e40c7769605697b4c8777986b6?se=2022-12-24
######################################################################## 100.0%
==> Fetching gnutls
==> Downloading https://ghcr.io/v2/homebrew/core/gnutls/manifests/3.7.8
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/gnutls/blobs/sha256:2de64828679245123f641ecdc5b166b444f24586184d0d5717b4ac446406009f
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:2de64828679245123f641ecdc5b166b444f24586184d0d5717b4ac446406009f?se=2022-12-24
######################################################################## 100.0%
==> Fetching jpeg-turbo
==> Downloading https://ghcr.io/v2/homebrew/core/jpeg-turbo/manifests/2.1.4
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/jpeg-turbo/blobs/sha256:c9dbfe3df4b1c8cd4ac7ef18a3643c923c8081e6acdf9936ebff79b7514f14cd
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:c9dbfe3df4b1c8cd4ac7ef18a3643c923c8081e6acdf9936ebff79b7514f14cd?se=2022-12-24
######################################################################## 100.0%
==> Fetching libpng
==> Downloading https://ghcr.io/v2/homebrew/core/libpng/manifests/1.6.39
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libpng/blobs/sha256:c437aaaf373f369e94825937854374a0b17bf965c1ba2a0faf22818111372038
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:c437aaaf373f369e94825937854374a0b17bf965c1ba2a0faf22818111372038?se=2022-12-24
######################################################################## 100.0%
==> Fetching libslirp
==> Downloading https://ghcr.io/v2/homebrew/core/libslirp/manifests/4.7.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libslirp/blobs/sha256:dbfc3fabef1a14eb7807e97bac7e318dbf0ca0ac631cb949cd165ca79c57d16d
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:dbfc3fabef1a14eb7807e97bac7e318dbf0ca0ac631cb949cd165ca79c57d16d?se=2022-12-24
######################################################################## 100.0%
==> Fetching libssh
==> Downloading https://ghcr.io/v2/homebrew/core/libssh/manifests/0.10.4
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libssh/blobs/sha256:3932e63ecd8236e305a43d1aa27c98957b8c513171873d21fd9858671c1b4a5d
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:3932e63ecd8236e305a43d1aa27c98957b8c513171873d21fd9858671c1b4a5d?se=2022-12-24
######################################################################## 100.0%
==> Fetching libusb
==> Downloading https://ghcr.io/v2/homebrew/core/libusb/manifests/1.0.26
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libusb/blobs/sha256:ab90516396d8dc99f96d31615bcbddfcfd2082fcc7494dabb9d22b275628e800
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:ab90516396d8dc99f96d31615bcbddfcfd2082fcc7494dabb9d22b275628e800?se=2022-12-24
######################################################################## 100.0%
==> Fetching lzo
==> Downloading https://ghcr.io/v2/homebrew/core/lzo/manifests/2.10
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/lzo/blobs/sha256:e16072e8ef7a8810284ccea232a7333a2b620367814b133a455217d22e89ae8e
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:e16072e8ef7a8810284ccea232a7333a2b620367814b133a455217d22e89ae8e?se=2022-12-24
######################################################################## 100.0%
==> Fetching ncurses
==> Downloading https://ghcr.io/v2/homebrew/core/ncurses/manifests/6.3
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/ncurses/blobs/sha256:a1aabfa5d0fd9b2735b3d83d5378a447049190a34d85c3df9f2983beecbf83d5
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:a1aabfa5d0fd9b2735b3d83d5378a447049190a34d85c3df9f2983beecbf83d5?se=2022-12-24
######################################################################## 100.0%
==> Fetching pixman
==> Downloading https://ghcr.io/v2/homebrew/core/pixman/manifests/0.42.2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/pixman/blobs/sha256:1e4026e8980666338f1a49cc61a3b6e968a744d92a67aeacfe918f8e8266d8ce
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:1e4026e8980666338f1a49cc61a3b6e968a744d92a67aeacfe918f8e8266d8ce?se=2022-12-24
######################################################################## 100.0%
==> Fetching snappy
==> Downloading https://ghcr.io/v2/homebrew/core/snappy/manifests/1.1.9
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/snappy/blobs/sha256:8259999a686e6998350672e5e67425d9b5c3afaa139e14b0ad81aa6ac0b3dfa9
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:8259999a686e6998350672e5e67425d9b5c3afaa139e14b0ad81aa6ac0b3dfa9?se=2022-12-24
######################################################################## 100.0%
==> Fetching vde
==> Downloading https://ghcr.io/v2/homebrew/core/vde/manifests/2.3.3
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/vde/blobs/sha256:0cd674a5b677c8e4deb2735884366f6a384b0867aa7483e7293e361cbaab350e
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:0cd674a5b677c8e4deb2735884366f6a384b0867aa7483e7293e361cbaab350e?se=2022-12-24
######################################################################## 100.0%
==> Fetching xz
==> Downloading https://ghcr.io/v2/homebrew/core/xz/manifests/5.4.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/xz/blobs/sha256:1872953eda5b6724d90376a449d18745e5daf8ebe760f02c2f7a4236847176fc
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:1872953eda5b6724d90376a449d18745e5daf8ebe760f02c2f7a4236847176fc?se=2022-12-24
######################################################################## 100.0%
==> Fetching qemu
==> Downloading https://ghcr.io/v2/homebrew/core/qemu/manifests/7.2.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/qemu/blobs/sha256:62bea721ff6fa3ee15ed53ee8215fea42b2cd07e8fb39b77ce8e39f30287576f
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:62bea721ff6fa3ee15ed53ee8215fea42b2cd07e8fb39b77ce8e39f30287576f?se=2022-12-24
######################################################################## 100.0%
==> Fetching lima
==> Downloading https://ghcr.io/v2/homebrew/core/lima/manifests/0.14.2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/lima/blobs/sha256:f1390f3133d929dd5b24c0b1449aec981e55eaadfcb97a56056f800fdd763629
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:f1390f3133d929dd5b24c0b1449aec981e55eaadfcb97a56056f800fdd763629?se=2022-12-24
######################################################################## 100.0%
==> Fetching colima
==> Downloading https://ghcr.io/v2/homebrew/core/colima/manifests/0.5.1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/colima/blobs/sha256:2f550554789f2c861bf2ecfe821c8592a0b6251dbea925016193302cc920d60e
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:2f550554789f2c861bf2ecfe821c8592a0b6251dbea925016193302cc920d60e?se=2022-12-24
######################################################################## 100.0%
==> Installing dependencies for colima: capstone, gettext, pcre2, glib, gmp, bdw-gc, m4, libtool, libunistring, pkg-config, guile, libidn2, libtasn1, nettle, p11-kit, libnghttp2, unbound, gnutls, jpeg-turbo, libpng, libslirp, libssh, libusb, lzo, ncurses, pixman, snappy, vde, xz, qemu and lima
==> Installing colima dependency: capstone
==> Pouring capstone--4.0.2.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/capstone/4.0.2: 25 files, 14.8MB
==> Installing colima dependency: gettext
==> Pouring gettext--0.21.1.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/gettext/0.21.1: 1,983 files, 20.9MB
==> Installing colima dependency: pcre2
==> Pouring pcre2--10.42.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/pcre2/10.42: 230 files, 6.2MB
==> Installing colima dependency: glib
==> Pouring glib--2.74.3.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/glib/2.74.3: 448 files, 21.9MB
==> Installing colima dependency: gmp
==> Pouring gmp--6.2.1_1.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/gmp/6.2.1_1: 21 files, 3.2MB
==> Installing colima dependency: bdw-gc
==> Pouring bdw-gc--8.2.2.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/bdw-gc/8.2.2: 73 files, 1.7MB
==> Installing colima dependency: m4
==> Pouring m4--1.4.19.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/m4/1.4.19: 13 files, 742.5KB
==> Installing colima dependency: libtool
==> Pouring libtool--2.4.7.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libtool/2.4.7: 75 files, 3.8MB
==> Installing colima dependency: libunistring
==> Pouring libunistring--1.0.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libunistring/1.0: 56 files, 5.0MB
==> Installing colima dependency: pkg-config
==> Pouring pkg-config--0.29.2_3.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/pkg-config/0.29.2_3: 11 files, 676.5KB
==> Installing colima dependency: guile
==> Pouring guile--3.0.8_2.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/guile/3.0.8_2: 846 files, 62.4MB
==> Installing colima dependency: libidn2
==> Pouring libidn2--2.3.4.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libidn2/2.3.4: 79 files, 1.3MB
==> Installing colima dependency: libtasn1
==> Pouring libtasn1--4.19.0.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libtasn1/4.19.0: 61 files, 717.9KB
==> Installing colima dependency: nettle
==> Pouring nettle--3.8.1.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/nettle/3.8.1: 91 files, 2.9MB
==> Installing colima dependency: p11-kit
==> Pouring p11-kit--0.24.1_1.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/p11-kit/0.24.1_1: 67 files, 3.9MB
==> Installing colima dependency: libnghttp2
==> Pouring libnghttp2--1.51.0.arm64_monterey.bottle.1.tar.gz
🍺  /opt/homebrew/Cellar/libnghttp2/1.51.0: 13 files, 738.0KB
==> Installing colima dependency: unbound
==> Pouring unbound--1.17.0.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/unbound/1.17.0: 58 files, 5.7MB
==> Installing colima dependency: gnutls
==> Pouring gnutls--3.7.8.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/gnutls/3.7.8: 1,288 files, 11.1MB
==> Installing colima dependency: jpeg-turbo
==> Pouring jpeg-turbo--2.1.4.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/jpeg-turbo/2.1.4: 44 files, 2.5MB
==> Installing colima dependency: libpng
==> Pouring libpng--1.6.39.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libpng/1.6.39: 27 files, 1.3MB
==> Installing colima dependency: libslirp
==> Pouring libslirp--4.7.0.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libslirp/4.7.0: 11 files, 385.0KB
==> Installing colima dependency: libssh
==> Pouring libssh--0.10.4.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libssh/0.10.4: 23 files, 1.3MB
==> Installing colima dependency: libusb
==> Pouring libusb--1.0.26.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/libusb/1.0.26: 22 files, 595KB
==> Installing colima dependency: lzo
==> Pouring lzo--2.10.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/lzo/2.10: 31 files, 565.6KB
==> Installing colima dependency: ncurses
==> Pouring ncurses--6.3.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/ncurses/6.3: 3,968 files, 9.6MB
==> Installing colima dependency: pixman
==> Pouring pixman--0.42.2.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/pixman/0.42.2: 11 files, 842.5KB
==> Installing colima dependency: snappy
==> Pouring snappy--1.1.9.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/snappy/1.1.9: 18 files, 158.7KB
==> Installing colima dependency: vde
==> Pouring vde--2.3.3.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/vde/2.3.3: 66 files, 1.3MB
==> Installing colima dependency: xz
==> Pouring xz--5.4.0.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/xz/5.4.0: 95 files, 1.6MB
==> Installing colima dependency: qemu
==> Pouring qemu--7.2.0.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/qemu/7.2.0: 163 files, 508.7MB
==> Installing colima dependency: lima
==> Pouring lima--0.14.2.arm64_monterey.bottle.tar.gz
🍺  /opt/homebrew/Cellar/lima/0.14.2: 72 files, 37.7MB
==> Installing colima
==> Pouring colima--0.5.1.arm64_monterey.bottle.tar.gz
==> Caveats
zsh completions have been installed to:
  /opt/homebrew/share/zsh/site-functions
==> Summary
🍺  /opt/homebrew/Cellar/colima/0.5.1: 8 files, 5.2MB
==> Running `brew cleanup colima`...
Disable this behaviour by setting HOMEBREW_NO_INSTALL_CLEANUP.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).
==> Upgrading 1 dependent of upgraded formulae:
Disable this behaviour by setting HOMEBREW_NO_INSTALLED_DEPENDENTS_CHECK.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).
python@3.10 3.10.8 -> 3.10.9
==> Fetching python@3.10
==> Downloading https://ghcr.io/v2/homebrew/core/python/3.10/manifests/3.10.9
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/python/3.10/blobs/sha256:a9b28161cec6e1a027f1eab7576af7c650e283f6c3dc492bc520eae969da8431
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:a9b28161cec6e1a027f1eab7576af7c650e283f6c3dc492bc520eae969da8431?se=2022-12-24
######################################################################## 100.0%
==> Upgrading python@3.10
  3.10.8 -> 3.10.9 

==> Pouring python@3.10--3.10.9.arm64_monterey.bottle.tar.gz
==> /opt/homebrew/Cellar/python@3.10/3.10.9/bin/python3.10 -m ensurepip
==> /opt/homebrew/Cellar/python@3.10/3.10.9/bin/python3.10 -m pip install -v --no-deps --no-index --upgrade --isolated --target=/opt/homebrew/lib/python3.10/site-p
🍺  /opt/homebrew/Cellar/python@3.10/3.10.9: 3,110 files, 57.1MB
==> Running `brew cleanup python@3.10`...
Removing: /opt/homebrew/Cellar/python@3.10/3.10.8... (3,116 files, 57.3MB)
==> Checking for dependents of upgraded formulae...
==> No broken dependents found!
==> Caveats
==> colima
zsh completions have been installed to:
  /opt/homebrew/share/zsh/site-functions
```

`colima help` 명령을 실행하면 다음과 같이 간결한 도움말이 나온다.
```
xxxx 🦑🍺 ❯ colima help                                                                                                  ✹ ✭
Colima provides container runtimes on macOS with minimal setup.

Usage:
  colima [command]

Available Commands:
  completion  Generate completion script
  delete      delete and teardown Colima
  help        Help about any command
  kubernetes  manage Kubernetes cluster
  list        list instances
  nerdctl     run nerdctl (requires containerd runtime)
  ssh         SSH into the VM
  ssh-config  show SSH connection config
  start       start Colima
  status      show the status of Colima
  stop        stop Colima
  template    edit the template for default configurations
  version     print the version of Colima

Flags:
  -h, --help             help for colima
  -p, --profile string   profile name, for multiple instances (default "default")
  -v, --verbose          enable verbose log
      --very-verbose     enable more verbose log

Use "colima [command] --help" for more information about a command.
```

맨 위에 colima가 뭐하는 도구인지 간명하게 설명해주고 있어 너무 마음에 든다.

설치만으로 docker daemon을 실행할 수 있는 것은 아니고 colima 를 시작해야 한다.

>colima start

첫 시작이라 그런지 QEMU 등 이것저것 다운로드해서 설치하는데 이번에도 1분 가까이 시간이 걸린다.

```
xxx 🦑🍺 ❯ colima start                                                                                               ⏎ ✹ ✭
INFO[0000] starting colima                              
INFO[0000] runtime: docker                              
INFO[0000] preparing network ...                         context=vm
INFO[0000] creating and starting ...                     context=vm
INFO[0060] provisioning ...                              context=docker
INFO[0060] starting ...                                  context=docker
INFO[0066] done   
```

이제 드디어 docker 명령을 실행할 수 있다!

```
xxx 🦑🍺 ❯ docker images                                                                                                ✹ ✭
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE
```

더 이상 docker 명령을 실행할 필요가 없으면 다음과 같이 colima를 종료하면 된다.

>colima stop

```
xxx 🦑🍺 ❯ colima stop                                                                                                  ✹ 
INFO[0000] stopping colima                              
INFO[0000] stopping ...                                  context=docker
INFO[0001] stopping ...                                  context=vm
INFO[0006] done                                         
```

docker 명령을 실행하면 이전처럼 실패한다.

```
xxx 🦑🍺 ❯ docker images                                                                                                ✹ 
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
