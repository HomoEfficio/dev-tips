# java 8 ByteBuffer 관련 이슈

정확한 원인은 모르지만 java 8 환경에서 아래와 같이 ByteBuffer의 메서드가 없다며 NoSuchMethodError가 나는 경우가 있다.

![Imgur](https://i.imgur.com/qGwvE7k.png)

동일한 코드를 java 11, 14에서 해보니 에러가 나지 않는다.

물론 늘 발생하는 오류는 아니고 나면 참 골치 아프다. gradle, intellij 설정 아무리 맞춰도 잘 사라지지 않는다.

https://www.morling.dev/blog/bytebuffer-and-the-dreaded-nosuchmethoderror/ 여기에 비교적 상세히 나와있다.

이제는 java 8 벗어나서 그냥 자바 버전 올리는 쪽으로 하는 게 좋을 듯

