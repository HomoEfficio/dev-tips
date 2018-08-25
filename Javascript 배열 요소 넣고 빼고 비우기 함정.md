# Javascript 배열 요소 넣고 빼고 비우기 함정

배열 요소를 지우는 것과 관련해서 Javascript에는 `delete`와 `splice()`가 있다.

`delete`는 배열 요소를 지우되, 지워진 자리에 `undefined`를 채워넣는다. 결국 전체 배열의 길이는 줄어들지 않는다.

`splice()`는 배열 요소를 배열에서 완전히 제거한다. 결국 전체 배열의 길이도 줄어든다.

프론트 쪽 개발을 하다보면 화면에 배열로 구성된 컴포넌트를 `splice()`로 지우면 dom과 메모리의 불일치가 발생해서 짝이 맞지 않는 경우가 생긴다.

이럴 때는 화면에서 지울 때는 `delete`로 지우고, 서버에 보낼 때 `splice()`로 배열에서 `undefined`를 지우면 된다. 아무 생각 없이 구현하면 이 부분은 다음과 같이 구현할 수 있다.

```javascript
const emptyRemover = arr => {
    for (let i = 0; i < arr.length; i++) {
        if (arr[i] === undefined) {
            arr.splice(i, 1);
        }
    }
}
```

그런데 이 때 살짝 함정이 기다리고 있다. 실제로 다음과 같이 배열의 모든 요소를 `delete` 한 후에 `emptyRemover`를 적용해보면 최종 결과가 `[]`이 아니라 `[undefined]`가 된다. 왜냐하면 `splice()`에 의해 배열 길이가 줄면서 마지막 남은 하나를 처리하지 못하기 때문이다.

```javascript
let arr = [0, 1, 2, 3];

delete arr[1];

delete arr[3];

// arr은 [0, undefined, 2, undefined] 가 되어 있다.

emptyRemover(arr);

// arr은 [0, 2] 가 되어 있다. 여기까지는 OK

delete arr[0];

delete arr[1];

// arr은 [undefined, undefined] 가 되어 있다.

emptyRemover(arr);

// arr이 [] 이어야 할 것 같지만 [undefined] 가 된다.
```

emptyRemover는 다음과 같이 구현해줘야 한다.

```javascript
const emptyRemover = a => {
    for (let i = 0; i < a.length; i++) {
        if (a[i] === undefined) {
            a.splice(i--, 1);  // 여기!!!
        }
    }
}
```

하지만 기존 배열에 `splice()`를 적용해서 복잡하게 컨트롤하는 것보다, 아예 새로 구성한 배열을 반환하게 하는 편이 훨씬 낫다.

```javascript
const filterOutEmpty = arr => arr.filter(e => e !== undefined);
```
