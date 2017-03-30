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

하지만 이걸 사용하려면 또 parsing을 해야하고, stream에서는 한 번만 읽을 수 있는 등 불편한점이 많으므로, JSON을 보낼 떄는 ContentType을 `application/x-www-form-urlencoded; charset=UTF-8`로 하고, 서버에서는 `request.getParameterMap()`을 이용해서 처리하는 것이 좋겠다.

----

- @RequestMapping(consumes="application/json")로 하고 controller 메서드에 해당 DTO 객체를 주입시키면 스프링이 알아서 DTO에 넣어줄까?




