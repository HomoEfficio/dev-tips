# ssh 로그인 오래 걸리는 문제

`ssh 로그인@서버호스트이름`으로 ssh 로그인을 시도하면, 거의 1분이나 지난 후에 비밀번호를 입력하라는 프롬프트가 나온다.

`ssh -vvv 로그인@서버호스트이름`으로 실행하면 디버그 로그가 나오는데 대략 다음과 같은 곳에서 한참을 머문다.

```
debug3: load_hostkeys: loaded 1 keys from server01
debug3: hostkeys_foreach: reading file "/Users/1003604/.ssh/known_hosts"
debug3: record_hostkey: found key type ECDSA in file /Users/1003604/.ssh/known_hosts:37
debug3: load_hostkeys: loaded 1 keys from 192.168.56.101
debug1: Host 'server01' is known and matches the ECDSA host key.
debug1: Found key in /Users/1003604/.ssh/known_hosts:40
debug2: set_newkeys: mode 1
debug1: SSH2_MSG_NEWKEYS sent
debug1: expecting SSH2_MSG_NEWKEYS
debug2: set_newkeys: mode 0
debug1: SSH2_MSG_NEWKEYS received
debug1: SSH2_MSG_SERVICE_REQUEST sent
debug2: service_accept: ssh-userauth
debug1: SSH2_MSG_SERVICE_ACCEPT received
debug2: key: /Users/1003604/.ssh/id_rsa (0x7fbb99700270),
debug2: key: /Users/1003604/.ssh/id_dsa (0x0),
debug2: key: /Users/1003604/.ssh/id_ecdsa (0x0),
debug2: key: /Users/1003604/.ssh/id_ed25519 (0x0), <-- 여기서 한참 대기..
```

# 해결

찾아보니 ssh 서버의 `/etc/ssh/sshd_config` 파일에서 `GSSAPIAuthentication`을 `no`로 하라는 답이 많은데, 내 경우는 그래도 안되었다.

그래서 또 찾다보니 https://ubuntuforums.org/showthread.php?t=2246365&s=0476b667fccdca112aeca9369b26dffe&p=13133268#post13133268 에서 답을 찾았는데, 

>ssh 서버의 `/etc/ssh/sshd_config` 파일에서 `UseDNS`를 명시적으로 `no`로 지정해주면 된다.

