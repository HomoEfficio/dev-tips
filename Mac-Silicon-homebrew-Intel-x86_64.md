# Mac M# 에서 Intel x86_64 패키지 설치하기

Silicon(M#) 칩 Mac에서 `brew install` 명령으로 패키지를 설치하면 아래와 같이 x86_64 용 패키지는 설치할 수 없다는 에러가 날 때가 있다.

```bash
~ 🦑🍺 ❯ brew install making/tap/rsc
Running `brew update --auto-update`...
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/core).

You have 12 outdated formulae installed.
You can upgrade them with brew upgrade
or list them with brew outdated.

rsc: The x86_64 architecture is required for this software.
Error: rsc: An unsatisfied requirement failed this build.
```

이럴 때는 어떻게 해야할까?

Resetta 2를 설치하고, Intel x86_64 용 homebrew를 따로 설치하면 Intel x86_64 용 패키지를 설치할 수 있다.

## Rosetta 2 설치

인텔 기준 명령어로 만들어진 패키지도 Rosetta 2를 통해 Mac 실리콘 칩 시스템에서 실행될 수 있다. https://support.apple.com/ko-kr/guide/security/secebb113be1/web

```bash
~ 🦑🍺 ❯ /usr/sbin/softwareupdate --install-rosetta
I have read and agree to the terms of the software license agreement. A list of Apple SLAs may be found here: http://www.apple.com/legal/sla/
Type A and press return to agree: A
2022-12-04 10:42:37.043 softwareupdate[48281:139852780] Package Authoring Error: 002-66270: Package reference com.apple.pkg.RosettaUpdateAuto is missing installKBytes attribute
Install of Rosetta 2 finished successfully
```

## Intel x86_64 용 homebrew 설치

`/usr/local/homebrew`에 인텔용 homebrew를 따로 설치한다.

```bash
~ 🦑🍺 ❯ cd ~/Downloads 
Downloads 🦑🍺 ❯ mkdir homebrew
Downloads 🦑🍺 ❯ curl -L https://github.com/Homebrew/brew/tarball/master | tar zx --strip 1 -C homebrew
Password:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 2909k  100 2909k    0     0  1238k      0  0:00:02  0:00:02 --:--:-- 1383k
Downloads 🦑🍺 ❯ sudo mv homebrew /usr/local/homebrew/
```

`.zshrc` 파일에 인텔용 homebrew에 대한 alias를 추가한다.

```bash
alias a86brew="arch -x86_64 /usr/local/homebrew/bin/brew"
```

인텔 용 패키지를 설치할 때는 `brew install` 대신 `a86brew install` 명령을 사용하면 되고, `/usr/local/homebrew/Cellar`에 설치된다.

