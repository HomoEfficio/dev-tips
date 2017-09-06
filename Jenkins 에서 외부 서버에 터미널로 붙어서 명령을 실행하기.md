# Jenkins 에서 외부 서버에 터미널로 붙어서 명령을 실행하기

Job의 구성 > Build > Execute shell 에서 다음과 같은 형식으로 접속 및 명령 실행

```
ssh -i ~/.ssh/어쩌구인증파일 id@my.test.com "cd /path/to/target_dir; execute something.py"
```
