# AWS의 APIGateway-Lambda를 통해 Slack으로 알림 보내기

## 이걸 왜?

- Server의 물리적 상황은 AWS에서 감지 가능하지만, Application 수준에서의 이상 상황은 AWS에서 감지 불가(최근 새로 나온 L7 로드 밸런서인 Application LoadBalancer로는 감지 가능)
- 따라서 Application 수준에서의 이상 상황을 감지하는 로직을 만들고, 그 로직에서 AWS의 APIGateway-Lambda를 통해 Slack으로 알림을 발송하는 기능은 의미가 있다.

## 큰 흐름

1. AWS에서 날라온 알림 메시지를 받을 수 있도록 Slack에 WebHook을 만든다.
1. Application에서의 이상 상황 발생 시 요청을 보낼 대상인 API Gateway를 만든다.
1. Slack의 WebHook으로 메시지를 보내는 Lambda를 만든다.
1. API Gateway의 특정 endpoint에 앞에서 만든 Lambda를 연동한다.

## Slack WebHook 생성




## API Gateway 생성

### IAM Role 생성

먼저 API Gateway를 사용할 수 있는 IAM Role을 생성한다.

![Imgur](http://i.imgur.com/2VkBvNx.png)

![Imgur](http://i.imgur.com/yby2Peo.png)

![Imgur](http://i.imgur.com/GXhotsR.png)

![Imgur](http://i.imgur.com/UwCBvYf.png)

![Imgur](http://i.imgur.com/J91pbgf.png)

![Imgur](http://i.imgur.com/8BxNO30.png)

### API Gateway 생성




## Lambda 생성




## API Gateway - Lambda 연동
