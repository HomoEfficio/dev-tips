# Back to the Essence - I/O

I/O가 뭔지 어떻게 동작하는지 아주 옛날에 잠시 찾아봤을 뿐이고, 그 후로 꽤 오랫동안 별 생각없이 그냥 언어나 라이브러리에서 제공해주는 API를 사용해오기만 하다보니, 어디선가 약간의 로우 레벨 내용이 나오면 잘 모르겠더라능..

그래서 저수준의 내용을 다루는 글을 보더라도 어색하지 않도록 이쯤에서 되짚어보려 한다.

찾다보니 [How Java I/O Works Internally at Lower Level?](https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/) 이 글만한 것이 없는 것 같다. 한땀한땀 원문 그대로 번역하는 것이 귀찮아서 그냥 서술했지만, 대부분의 내용은 이 글에서 가져왔다고 봐도 무방하다.

## I/O란?

조금 추상적일 수 있지만 다음 표현이 적절한 것 같다.

>I/O(Input/Output, 입/출력)은 하나의 컴퓨터와 그 컴퓨터를 제외한 세상 모든 것과의 인터페이스다.
>
>또는
>
>I/O는 하나의 프로그램과 그 프로그램이 실행되는 컴퓨터의 나머지 모든 것과의 인터페이스다.

(출처: https://howtodoinjava.com/java/io/difference-between-standard-io-and-nio/)

좀 없어보이기는 해도 쉽게 말하면 서로 다른 주체 사이의 통신이라고 봐도 크게 틀리지 않을 것 같다.

이제 I/O에 등장하는 몇 가지 로우 레벨 등장인물을 짧게 살펴보자.

## Buffer

일반적으로 버퍼는 물리적 충격을 완화하는 완충장치를 의미한다. 전산에서는 물리적 충격이 아니라 양이나 속도의 차이를 완화하는 역할을 하는 것을 버퍼라고 부른다.

I/O가 `서로 다른 주체 사이`의 `통신`이라고 했는데, `서로 다른 주체 사이`에는 처리할 수 있는 데이터의 양이나 속도에 차이가 있다. 그래서 I/O가 효율적으로 원활히 처리되려면 둘 사이의 차이를 완화해 줄 수 있도록 데이터를 임시로 담아둘 수 있는 버퍼가 필수다. 양이나 속도에 본연적인 차이가 존재하는 I/O에서 데이터는 대부분 버퍼를 통해서 오간다. `통신`을 데이터의 이동이라고 본다면 결국 버퍼에 데이터를 넣고 빼는 것이 I/O 처리의 시작이라고 볼 수 있다.

## Device Controller, Device Driver, Micro Controller

디바이스 컨트롤러(Device Controller)는 디바이스를 사용하는 컴퓨터 쪽에서 디바이스를 제어하는데 사용되는 하드웨어로 컴퓨터의 마더보드에 장착되어 있으며, USB 포트 등을 통해 디바이스와 연결된다. 그리고 디바이스 컨트롤러에는 버퍼가 있다.

디바이스 드라이버(Device Driver)는 OS와 디바이스 컨트롤러 사이의 의미 교환을 도와주는 소프트웨어로서, OS마다 그리고 디바이스마다 다르다.

마이크로 컨트롤러(Micro Controller)는 범용적으로 사용되는 용어지만 여기에서는 디바이스 자체에 내장되어 디바이스를 운용하는 소형 컴퓨터라고 볼 수 있다.

## 디스크 데이터 읽어오기

이제 이 등장인물들로 디스크의 데이터를 읽어오는 장면을 그려보면 다음과 같다.

![Imgur](https://i.imgur.com/uS3Ykxn.png)





----

이건 따로 정리하려면 일이 너무 커질 것 같아서 잘 정리된 문서 링크 모아두는 걸로 쫑내자..

https://cs.nyu.edu/~gottlieb/courses/2000s/2002-03-fall/os/lectures/lecture-13.html

https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/

https://howtodoinjava.com/java7/nio/java-nio-2-0-working-with-buffers/

https://howtodoinjava.com/java-io-tutorial/

https://www.javacodegeeks.com/2012/08/io-demystified.html

https://slideplayer.com/slide/10013778/



