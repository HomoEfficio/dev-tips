# docker-compose 명령

```
docker exec -i -t [아이디 or 컨테이너 이름] /bin/bash

mysql -u [아이디] -p[비밀번호]
```



```
docker-compose build --force-rm: docker-compose 에 포함된 서비스의 이미지 생성, --force-rm 은 중간 이미지 항상 삭제
docker-compose up -d : docker-compose 에 포함된 서비스의 이미지에서 컨테이너 생성 및 실행. -d: daemon
docker-compose logs -f -t 컨테이너ID : -d 로 실행돼서 볼 수 없는 로그를 호스트 화면에 표시
docker-compose stop : docker-compose 로 실행된 컨테이너 중지
docker-compose down : docker-compose stop + 컨테이너 삭제
docker-compose up -d --scale {{서비스이름}}=3: {{서비스이름}} 인스턴스 3개 띄움
```

