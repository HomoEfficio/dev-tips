# CSV 파일 엑셀 한글 깨짐 BOM

브라우저를 통해 다운로드 받은 CSV 파일을 텍스트 에디터나 다른 앱에서 열면 괜찮은데 유독 엑셀에서 열면 한글이 깨질 때가 있다.

원인은 한 마디로 엑셀이 CSV 파일을 열 때 UTF-8로 디코딩 하지 않기 때문이다.
그래서 다운로드 받은 CSV 파일이 UTF-8로 인코딩 돼 있으면 엑셀로 열 때 한글이 깨져 보인다.

해결 방법은 두 가지다.

1. UTF-8로 인코딩 된 CSV 파일을 ANSI 인코딩 된 파일로 변경 저장
2. 엑셀 파일을 열 때 UTF-8로 디코딩

1은 아무래도 번거로우니 어쩔 수 없을 때나 최후의 방법으로 쓰는 것이 좋겠고, 개발자 관점에서는 2를 통해 해결하는 것이 바람직하다.


# 엑셀 파일을 열 때 UTF-8로 디코딩

왠지 모르지만(알고 싶지도 않..) 엑셀은 UTF-8로 디코딩하지 않으며 UTF-8로 디코딩 하려면 두 가지 방법이 있다.

## 텍스트 마법사 사용

OS 따라 다르지만 Mac 기준으로 '데이터 > 외부 데이터 가져오기 > 텍스트에서'를 선택해서 결국에는 아래와 같이 '텍스트 마법사'에서 CSV 파일이 어떤 방식으로 인코딩 됐는지 지정해서 그에 맞게 디코딩하게 만들 수 있다.

![Imgur](https://i.imgur.com/8IuY0u2.png)

그런데 이 방법도 번거롭기는 1과 마찬가지다.

## BOM 추가

CSV 파일을 다운로드 해줄 때 BOM(Byte Order Mark)를 파일 맨 앞에 추가하면 엑셀이 이를 인지하고 알아서 UTF-8로 디코딩한다.

BOM은 유니코드로 `'\ufeff'`이며 바이트로는 `0xEF`, `0xBB`, `0xBF` 이렇게 3바이트짜리 값이다.

BOM을 추가하는 방법은 사실 간단하며 예제는 https://mkyong.com/java/java-how-to-add-and-remove-bom-from-utf-8-file/ 여기에 아주 잘 나와있다.

한 가지 예를 들면 다음과 같다.

```kotlin
fun dataToCSV(lines: List<List<String>>, csvMetaInfo: CsvMetaInfo): ByteArrayInputStream {

    val csvFormat: CSVFormat = CSVFormat.Builder.create().setQuoteMode(QuoteMode.MINIMAL).build()
    try {
        ByteArrayOutputStream().use { out ->
            val pw = PrintWriter(out, false, Charsets.UTF_8)
            pw.write(0xfeff)  // 여기!!

            CSVPrinter(pw, csvFormat).use { csvPrinter ->
                if (!csvMetaInfo.header.isNullOrEmpty()) {
                    csvPrinter.printRecord(csvMetaInfo.header)
                }
                for (line in lines) {
                    csvPrinter.printRecord(line)
                }
                csvPrinter.flush()
                return ByteArrayInputStream(out.toByteArray())
            }
        }
    } catch (e: IOException) {
        throw XxxException(
            // ...
        )
    }

}
```

BOM이 붙어 있는 파일을 hexdump로 열어보면 다음과 같이 `ef bb bf`로 시작한다.

```
~ 🦑🍺 ❯ hexdump -n 20 -C sample.csv
00000000  ef bb bf 69 64 2c 6e 61  6d 65 0d 0a 6d 6f 6e 65  |﻿id,name..mone|
00000010  79 2c eb a8                                       |y,?|
00000014

```

이 파일을 엑셀에서 열면 별다른 조치 없이도 한글이 깨지지 않고 잘 표시된다.

끝!

인 것 같지만 이렇게 BOM을 추가해도 브라우저에서 다운로드 받은 파일을 hexdump로 열어보면 BOM이 추가돼 있지 않아 엑셀로 열면 한글이 계속 깨진다. 왜 그럴까?

이유는 **브라우저가 파일을 저장하면서 BOM을 제거하기 때문**이었다!

똑같은 URL로 다운로드 할 수 있는 CSV 파일을 브라우저를 통하지 않고 cURL 같은 도구로 다운로드 하면 BOM이 추가돼 있음을 확인할 수 있었다.

그렇다면 브라우저로 다운로드 할 때도 BOM이 유지되게 하려면 어떻게 해야할까?

다른 방법도 있을 수 있겠지만, 이래저래 하다가 알게된 방법은 걍 아래와 같이 BOM을 하나 더 추가하면 된다능..

```kotlin
        ByteArrayOutputStream().use { out ->
            val pw = PrintWriter(out, false, Charsets.UTF_8)
            pw.write(0xfeff)  // 여기!!
            pw.write(0xfeff)  //// 하나 더 추가!!
```

물론 이렇게 하면 브라우저로 다운로드 하지 않고 cURL로 다운로드 하면 BOM이 두 개가 들어있게 되지만 글자는 깨지지 않으며, 무엇보다 사실 상 거의 대부분 브라우저로 다운로드 하는 현실을 감안하면, CSV 파일을 엑셀로 열어야 하는 상황에서 괜찮은 편법이라고 할 수 있다.






