# MySQL Big Number 처리

식별자로 사용하고 있는 큰 숫자들이 MySQL에 insert 될 때 어떻게 들어가는지 실험해봤다.

## node.js

```javascript
"INSERT INTO `recopick`.`TMP_TEST` " +
          "(`varchar(100)`, `int`, `integer`, `bigint`, `float`, `double`) " +
          "VALUES " +
          "(?,?,?,?,?,?)",
          [123456789012345678901234567890,
           2147483647,
           2147483647,
           922337203685477580,
           123456789012345678901234567890.123456789012345678901234567890,
           123456789012345678901234567890.123456789012345678901234567890],
```

결과는 다음과 같다.

![Imgur](http://i.imgur.com/7bBtaxK.png)

- `123456789012345678901234567890`를 `varchar(100)`에 넣으면 `1.2345678901234568e29`로 들어간다.
- unsigned의 `int`나 `integer`의 MAX는 `2147483647`이므로 이를 초과하는 값을 넣으면 아래와 같이 에러가 발생한다.
    
    ```
    { [Error: ER_WARN_DATA_OUT_OF_RANGE: Out of range value for column 'int' at row 1]
      code: 'ER_WARN_DATA_OUT_OF_RANGE',
      errno: 1264,
      sqlState: '22003',
      index: 0 }
    ```
- unsigned의 `bigint`의 MAX는 `9223372036854775807`인데, 이상하게도 `9223372036854775807`을 insert하면 `Out of range value for column 'bigint' at row 1`에러가 발생한다. 그래서 마지막 7을 제외한 채 insert 했다.
    - 에러가 나는 이유는 JavaScript에서 `9223372036854775807`를 `9223372036854776000`로 반올림 하기 때문이다. 역시 JavaScript에서는 `Number.MAX_SAFE_INTEGER`인 `9007199254740991`를 넘는 값을 사용하는 상황을 피하는 것이 좋다.
- `float`와 `double`에는 소수점이 사라지면서 적당한 위치에서 반올림 된 정수형 데이터가 들어간다. 

>결론적으로 Node.js 환경에서 정수형 데이터는 `9007199254740991` 이하, **자리수로만 말하면 15자리 이하로 하는 것이 안전**하다.

