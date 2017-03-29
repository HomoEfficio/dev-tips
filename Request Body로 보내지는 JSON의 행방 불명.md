# Request Body로 보내지는 JSON의 행방 불명

## 문제

api 테스트 시 Postman을 자주 사용하는데, 다음과 같이 JSON을 서버에 보내면,

- postman 캡처

서버의 Request에서 JSON 정보를 찾을 수 없다.

- Spring ParameterMap 캡처

## 해결

하지만 동일한 json 데이터를 postman이 아니라 다음과 같이 실제로 ajax로 호출하는 코드를 통해 서버로 보내면 서버의 Request에서 JSON 정보가 잘 나온다.

- ajax 코드

둘의 결정적인 차이는 ContentType 헤더의 값이다.

jQuery의 ajax로 보낼 때 ContentType을 명시하지 않으면 디폴트로 `application/x-www-form-urlencoded; charset=UTF-8`로 지정되며([여기 참고](http://api.jquery.com/jQuery.ajax/)), Request 요청을 브라우저에서 살펴봐도 아래와 같이 Request Body에 포함되고,

- 브라우저 Request Body 캡처

서버의 Request에서도 확인이 가능하다.

- Spring ParameterMap 캡처

## 호기심

- @RequestMapping(consumes="application/json")로 해도 안 잡힐까?


