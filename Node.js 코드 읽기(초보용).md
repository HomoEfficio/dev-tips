# Node.js 코드 읽기(초보용)

## 관습적 표현

### function callbackA(err, result, [callbackB])

- 비동기로 호출되는 callback
- callback을 호출하는 함수가 수행 중 에러 발생 시 err에 에러 내용을 담아서 callback 호출
- callback을 호출하는 함수가 정상 수행되면 result에 수행 내용을 담아서 callback 호출
- callbackB를 인자로 받으면 callbackA 수행 결과에 따라 callbackB에 새로운 err이나 새로운 result를 callbackB에 인자로 전달하면서 callbackB를 호출

### function myFun(arg1, arg2, callback)

- myFun은 arg1, arg2, ... 을 토대로 자신의 함수 내용을 실행하고 그 결과 또는 에러를 callback에 파라미터로 전달하며 callback 호출
- callback은 myFun의 인자로 전달되지만, callback의 내용이 myFun의 수행에 입력값으로 사용된다기 보다 myFun이 수행되고 나서 결과적으로 callback이 수행됨

### return callback(err, result)

- callback()의 결과를 반환한다는 의미가 아니라, callback 실행 후 해당 함수에서의 실행을 종료하고 제어를 반환한다는 의미
    - 경우에 따라 return을 안 붙여주면 Callback already executed 비슷한 에러가 발생할 수 있음
- 따라서 callback에서 무엇을 반환하는지 굳이 들여다보지 않아도 됨

## async

### async.waterfall(fnArray, [callback])

참고: http://hatemogi.com/holiday-project-day-10/

fnArray는 [fn1, fn2, fn3] 이라고 하면

#### fn1

- fn1은 보통 다음과 같은 형태로 정의됨

    ```javascript
    function fn1(cb) {
        ...
        cb(null, arg1, arg2);  // cb는 fn2임
    }
    ``` 
- 즉, ... 를 수행하고 fnArray에 fn1 다음에 수행되도록 정해진 fn2를 호출


#### fn2

- fn2는 fn1에서 호출되는 모양새에 맞게 정의되어야 함

    ```javascript
    function fn2(err, arg1, arg2, cb) {
        ... // arg1, arg2를 사용해서 뭔가를 수행하고
        cb(err, arg1); // cb는 fn3임
    }
    ```
- 즉, ... 를 수행하고 fnArray에 fn2 다음에 수행되도록 정해진 fn3을 호출

#### fn3

- fn3는 fn2에서 호출되는 모양새에 맞게 정의되어야 함

    ```javascript
    function fn3(err, arg1, cb) {
        ... // arg1를 사용해서 뭔가를 수행하고
        cb(err, arg1); // cb는 callback 임
    }
    ```
- 즉, ... 를 수행하고 waterfall의 마지막 인자인 callback을 호출


#### callback
- waterfall의 두번째 인자인 callback은 finally와 비슷함
- fnArray에 정의된 함수가 
   - 모두 cb(null, result) 와 같이 에러 없이 수행되면, 마지막 fnX가 callback을 호출함
   - 수행 중에 에러가 발생해서 cb(nonNull, result)와 같이 호출하면, 그다음 fnX는 호출되지 않고, calllback(nonNull, result)가 호출됨

