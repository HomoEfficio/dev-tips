# jq

jq를 활용하면 터미널에서 JSON 을 가공해서 필요한 값만 확인할 수 있다.

https://xxx.yyy.zzz/aaa api 가 다음과 같은 JSON을 반환한다면

```json
{
	"category": {
		"keyword": "fashion",
		"categories": [
			{
				"keyword": "pick",
				"displayName": "For You",
				"categories": [
					{
						"keyword": "pick_gl",
						"displayName": "MD's Pick",
						"categories": [],
						"cards": [
							"card-001",
							"card-003",
							"card-108"
						]
					}
				]
			}
		]
	}
}
```

아래와 같이 curl 과 jq 를 조합해서 cards 만 추출해서 표시할 수 있다.

>curl https://xxx.yyy.zzz/aaa -s -b | jq '.category.categories[] | select(.keyword == "pick") | .categories[] | select(.keyword == "pick_gl") | .cards'

httpie를 사용할 수도 있다.

>http -b https://xxx.yyy.zzz/aaa -s -b | jq '.category.categories[] | select(.keyword == "pick") | .categories[] | select(.keyword == "pick_gl") | .cards'

로컬 파일에 있는 JSON 파일에 대해서도 가능하다.

>jq '.category.categories[] | select(.keyword == "pick") | .categories[] | select(.keyword == "pick_gl") | .cards' <로컬 파일 경로>
