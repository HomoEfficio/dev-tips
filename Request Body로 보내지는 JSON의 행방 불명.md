# Request Body로 보내지는 JSON의 행방 불명

## 문제

api 테스트 시 Postman을 자주 사용하는데, 다음과 같이 JSON을 서버에 보내면,

![Imgur](http://i.imgur.com/XVxcdns.png)

서버의 Request에서 JSON 정보를 찾을 수 없다.

![Imgur](http://i.imgur.com/zdLoEy0.png)

## 해결

하지만 동일한 json 데이터를 postman이 아니라 다음과 같이 실제로 ajax로 호출하는 코드를 통해 서버로 보내면 서버의 Request에서 JSON 정보가 잘 나온다.

```html
<html>
<body>
<button id='tester'>JSON Request Test</button>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.12.4/jquery.min.js"></script>
<script>
var tester = document.getElementById('tester');
tester.addEventListener('click', function(e) {
	(function($) {
	    $.ajax({
	        url: 'http://---------/v1/logs/view/33/55',
//	        contentType: 'application/json',
	        method: 'POST',
	        crossDomain: true,
	        data: {
	            url: "++++++++++++++++++++++++++++++++++++",
	            ref: "====================================",
	            items: [
	                {id: "1198708089",
	                title:"나이키 에어맥스",
	                image:"%%%%%%%%%%%%%%%%%%%%%/1197829883_B_V2.jpg",
	                price: 12000,
	                currency:"KRW",
	                catalog: ["catalog_id"],
	                is_now_delivery: 0,
	                adult_only: 0,
	                is_hidden: 0}
	            ],
	            user: { gender: "F", birthyear:"1980", mid:"8371920381"}
	        }
	    });
	})(jQuery);
});
</script>
</body>
</html>
```

둘의 결정적인 차이는 ContentType 헤더의 값이다.

Postman에서 보낼 떄는 ContentType을 `application/json`으로 보냈고, 그 때문에 request에서 찾을 수 없었다.

jQuery의 ajax로 보낼 때 ContentType을 명시하지 않으면 디폴트로 `application/x-www-form-urlencoded; charset=UTF-8`로 지정되며([여기 참고](http://api.jquery.com/jQuery.ajax/)), Request 요청을 브라우저에서 살펴봐도 아래와 같이 Request Body에 포함되고,

![Imgur](http://i.imgur.com/3ZAtT6n.png)

서버의 Request에서도 확인이 가능하다.

![Imgur](http://i.imgur.com/tH0DcQG.png)

## 호기심

참고로 jQuery ajax로 보내더라도, ContentType을 강제로 `application/json`으로 주면 요청은 아래와 Form Data가 아니라 Payload로 날라가므로,

![Imgur](http://i.imgur.com/MTE7wEJ.png)

서버에서는 아래와 같이 `request.getParameterMap()`으로는 읽을 수 없고,

![Imgur](http://i.imgur.com/sbVbEB9.png)

대신 아래와 같이 `request.getReader()`로는 읽을 수 있다. 

![Imgur](http://i.imgur.com/jYT2dAG.png)

하지만 이걸 사용하려면 또 parsing을 해야하고, stream에서는 한 번만 읽을 수 있는 등 불편한점이 많아서 실무적이지 않다.

## 실무에서는?

실제로는 request를 직접 핸들링하는 방식 대신에 DTO를 만들고 request 안에 json으로 넘어온 정보를 Jackson 같은 라이브러리를 써서 DTO에 집어넣는 방식을 택한다.

이때 프론트쪽에서 주의해야할 것이 있는데, jQuery의 ajax는 데이터를 문자열화 해주지 않기 때문에 아래와 같이 보내면,

```javascript
$.ajax({
    url: 'http://---------/v1/logs/view/33/55',
    contentType: 'application/json',
    method: 'POST',
    crossDomain: true,
    data: {
        url: "++++++++++++++++++++++++++++++++++++",
        ref: "====================================",
        items: [
            {id: "1198708089",
            title:"나이키 에어맥스",
            image:"%%%%%%%%%%%%%%%%%%%%%/1197829883_B_V2.jpg",
            price: 12000,
            currency:"KRW",
            catalog: ["catalog_id"],
            is_now_delivery: 0,
            adult_only: 0,
            is_hidden: 0}
        ],
        user: { gender: "F", birthyear:"1980", mid:"8371920381"}
    }
});
```

서버에서는 대략 아래와 같은 에러가 발생한다.

```
org.springframework.http.converter.HttpMessageNotReadableException: Could not read document: Unrecognized token 'url': was expecting ('true', 'false' or 'null')
```

이는 `url`이 JSON 객체 내의 key로 인식되지 않기 때문에 발생하는 문제로서, 이를 방지하려면 다음과 같이 `JSON.stringify()`를 통해 `url`이 key로 인식될 수 있도록 `"url"`와 같이 문자열로 만들어서 보내줘야 한다.

```javascript
$.ajax({
    url: 'http://---------/v1/logs/view/33/55',
    contentType: 'application/json',
    method: 'POST',
    crossDomain: true,
    data: JSON.stringify({
        url: "++++++++++++++++++++++++++++++++++++",
        ref: "====================================",
        items: [
            {id: "1198708089",
            title:"나이키 에어맥스",
            image:"%%%%%%%%%%%%%%%%%%%%%/1197829883_B_V2.jpg",
            price: 12000,
            currency:"KRW",
            catalog: ["catalog_id"],
            is_now_delivery: 0,
            adult_only: 0,
            is_hidden: 0}
        ],
        user: { gender: "F", birthyear:"1980", mid:"8371920381"}
    })
});
```
서버에서도 Spring을 예로 들면, controller 메서드의 파라미터에 JSON으로 받을 데이터를 매핑할 수 있는 DTO를 추가하고 앞에 `@RequestBody`를 꼭 붙여줘야 데이터가 DTO에 정상적으로 입력된다.

