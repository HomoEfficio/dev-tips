# K6 Load Test + InfluxDB + Grafana

k6 + InfluxDB + Grafana 를 활용한 부하 테스트 및 시각화

참고: https://k6.io/docs/results-visualization/influxdb-+-grafana

## k6

- [k6](https://k6.io) 활용
- 설치, 실행 등 세부 사항은 https://k6.io/docs/ 참고

### 설치

>brew install k6

### 스크립트 작성

POST 요청 부하 테스트 용 js 파일 작성

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 100,  // Virtual Users (동시 사용자)
  duration: '10s',
};

const randomStrGen = () => Math.random().toString(36).substring(2);
const randomPhoneNumGet = () => {
  const phoneNum = Math.floor(1000000000 + Math.random() * 9000000000).toString();
  return phoneNum.substring(0, 3) + '-' + phoneNum.substring(3, 6) + '-' + phoneNum.substring(6)
}

export default function() {
  const url = 'http://localhost:8080/v1/sellers';
  const body = JSON.stringify({
    "name": randomStrGen(),
    "email": `${randomStrGen()}@test.com`,
    "phone": `${randomPhoneNumGet()}`,
    "loginId": randomStrGen(),
    "password": randomStrGen()
  });
  const params = {
    headers: {
      'Content-Type': 'application/json',
    },
  };
  const res = http.post(url, body, params);
  check(res, {
    'status was 200': r => r.status === 200,
    'tx time OK': r => r.timings.duration < 30
  });
}
```

### 부하 테스트 실행

>k6 run k6-post.js

![Imgur](https://i.imgur.com/hTT14Tx.png)

- application.yml 에서 `server.tomcat.max-threads` 값을 변경하면서 테스트 해보면 재미있긔~


## InfluxDB

### InfluxDB 설치

>brew install influxdb

### InfluxDB 실행

>influxd -config /usr/local/etc/influxdb.conf

![Imgur](https://i.imgur.com/YV4Qi2L.png)

### InfluxDB 종료

CTRL+C

## Grafana

### Grafana 설치

>brew install grafana

### Grafana 실행

>brew services start grafana

![Imgur](https://i.imgur.com/fTcss5d.png)

### Grafana 접속 및 로그인

>localhost:3000

![Imgur](https://i.imgur.com/URY2kTf.png)

- default username/password: admin/admin

### Grafana DataSource 추가

![Imgur](https://i.imgur.com/HHfm7hG.png)

![Imgur](https://i.imgur.com/jfvGr9X.png)

![Imgur](https://i.imgur.com/G2Auov6.png)

![Imgur](https://i.imgur.com/dU51XUN.png)

![Imgur](https://i.imgur.com/pqjDDVp.png)

### Grafana 대시보드 추가

- 커스텀 대시보드를 만들 수도 있지만 [k6 문서에서 추천한 Preconfigured Grafana Dashboard](https://k6.io/docs/results-visualization/influxdb-+-grafana#preconfigured-grafana-dashboards) 중에서 [k m](https://grafana.com/grafana/dashboards/10660) 사용

  ![Imgur](https://i.imgur.com/9jsC10K.png)

  ![Imgur](https://i.imgur.com/lMky2Le.png)

  ![Imgur](https://i.imgur.com/CX76Qgl.png)

  ![Imgur](https://i.imgur.com/X5XweXl.png)

  ![Imgur](https://i.imgur.com/DPBuQzP.png)

  ![Imgur](https://i.imgur.com/CdLxzqz.png)

- 테스트 실행 결과를 influxdb로 전송

  >k6 run --out influxdb=http://localhost:8086/myk6db k6-post.js

- 대시보드 refresh

  ![Imgur](https://i.imgur.com/DoWqFeG.png)
