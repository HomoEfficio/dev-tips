# WebGL - 02 - Hello Triangle

## 프로그래밍 일반

### HTML

> **HTML**은 다양한 형태의 미디어를 포함할 수 있는 **컨테이너 시스템**이다.
- HTML은 태그(tag)로 되어 있고, **태그로만 되어 있다**.
- HTML의 태그는 컨테이너다. 이미지를 보여주는 **`img`**, 비디오를 보여주는 **`video`**, 오디오를 들려주는 **`audio`**, 그림을 그릴 수 있게 해주는 **`canvas`** 등 각자 상이한 시스템들이 모두 이 컨테이너에 담겨져서, **컨테이너에 담겨져야만 HTML내에 들어올 수 있다**.

### 상태(State)

> 상태는 **지속되는 변수**를 말한다.
- 상태는 **스코프(Scope)**라는 생존 기간, 생존 공간, 생존 범위 동안 그 **값을 유지**할 수 있고, 그 **값을 변경**할 수도 있다.
- **상태는 변경되기 전까지는 기존의 상태를 그대로 유지한다.**

사람은 대체로 상태라는 개념에 익숙하다. 가장 널리 사용되는 프로그래밍 언어인 C, Java, JavaScript는 모두 사람에게 익숙한 상태를 다루는 **ALGOL60** 이라는 언어에서 파생되어 발전해온 언어이다.

**사람이 상태에 익숙하기는 하지만 여러 상태를 한꺼번에 관리해야 할 때 특정 시점, 특정 상황에서 여러 상태를 정확히 파악하기는 어려운 일이다**. 상태를 파악하기 어렵기 때문에 **조건문을 통해 상태를 확인**하게 된다. **조건문이나 반복문을 통해 상태를 다루는 프로그래밍 언어를 제어형(또는 절차형) 언어**라고 한다.

조건문의 과도한 사용은 또다시 상태의 파악을 어렵게 하는 요인이 된다. **객체 지향이나 디자인 패턴은 궁극적으로는 과도한 조건문의 사용을 막는 것을 목적**으로 한다.

### 상태 머신(State Machine)

> 말 그대로 **상태를 다루는 기계**이다.
- 상태 머신은 상태를 포함하며, 외부에서 상태에 직접 접근하지 못하게 하고, **상태 머신을 통해서만 접근을 허용**한다.
- 상태 머신은 포함하고 있는 **상태가 변할 때, 상태 머신에 정의되어 있는 다른 일도 할 수 있다**.
- 상태 머신은 객체 지향에서 말하는 캡슐화와 비슷한 면이 있다.

상태 머신의 예는 매우 다양하다.

워드를 떠올려 보자.
워드에서 글자색을 빨간색으로 설정하면 그 이후에 입력하는 모든 글자는, **글자색을 다른 색으로 변경하기 전까지는  빨간색으로 화면에 표시**된다.
이때 빨간색으로 설정하는 행위는 **상태 머신이 제공하는 API를 통해서 글자색이라는 상태를 변경**하며,
빨간색으로 설정되면 글자만 빨간색으로 표시하는 것이 아니라 **상태 머신에 정의된 대로 글자색 아이콘의 색상도 빨간색으로 표시**한다. 

### 컨텍스트(Context)

> 컨텍스트는 어떤 일을 진짜로, 실제로 수행하는 담당자.
- 담당자는 **어떤 일을 수행하기 위해 필요한 정보, 어떤 일이 수행될 수 있는 환경**을 가지고 있다.

컨텍스트(담당자)의 예를 들어 보면,

- style 태그의 담당자는 Sheet 객체.
- canvas 태그의 담당자는 2D Context, WebGL Context 

하나의 HTML은 여러 개의 canvas를 가질 수 있고, canvas마다 각각 자기만의 context를 갖는다.

**GPU는 WebGL 만 처리하는 것이 아니라 OS에 의해 WebGL 외의 여러가지 일을 처리**한다. 이 과정에서 GPU 자원이 부족해지면 **임의로 WebGL 컨텍스트를 소멸**시키기도 한다. 이를 **WebGL 컨텍스트 상실**이라고 하는데, 관련 내용은 책 4장에서 다루고 있다.

## 삼각형 그리기 코드 해부

WebGL에서의 Hello, World는 기본 Primitive인 삼각형 그리기다.

### 큰 구조

![](http://i.imgur.com/pExvTOx.png)

### 코드 해부

<a href="https://github.com/projectBS/S63-WebGL/blob/master/Day2-Triangle.html" target="_blank">github.com/projectBS/S63-WebGL/blob/master/Day2-Triangle.html</a>

#### 상태 머신

`gl.useProgram()`, `gl.bindBuffer()`, `gl.clearColor()` 등 상태 머신을 통해 지정된 GPU 쪽 상태의 스코프는 JavaScript의 스코프와 무관하다.

예를 들어 JavaScript 함수인 `setupBuffers()` 내에서 `gl.bindBuffer()`에 의해 지정된 상태는, `setupBuffers()`의 실행이 종료되어도 GPU 내에서 그대로 유지되어, JavaScript의 다른 함수인 `draw()` 내의 `gl.drawArrays()`가 실행될 때까지도 유효하다.

#### gl.createBuffer(), gl.bindBuffer(), gl.bufferData()

gl은 상태 머신으로 다음과 유사한 자료 구조 형식을 가지고 있다. 아래에 나오는 코드는 모두 CPU가 아니라 GPU 내에 생성되는 자료 구조를 의미한다.

```javascript
gl = {
    key0: value0,
    key1: value1,
    ...,
    ARRAY_BUFFER: 0x8892,
    ...
};
```

`vertexBuffer1 = gl.createBuffer()`를 실행하면 GPU 메모리 내에 배열을 생성한다.

```javascript
gl = {
    key0: value0,
    key1: value1,
    ...,
    ARRAY_BUFFER: 0x8892,
    ...,
    vertexBuffer1 = [],
    ...
};
```

`gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer1)`를 실행하는 것은 `gl[gl.ARRAY_BUFFER] = vertexBuffer1`를 실행하는 것과 비슷하다.

```javascript
gl = {
    key0: value0,
    key1: value1,
    ...,
    ARRAY_BUFFER: 0x8892,
    ...,
    vertexBuffer1 = [],
    ...,
    0x8892: vertexBuffer1,
    ...
};
```

`gl.bufferData(gl.ARRAY_BUFFER, typedArray, gl.STATIC_DRAW)`를 실행하면, `gl[gl.ARRAY_BUFFER]`, 즉,  `gl[0x8892]`, 즉, `vertexBuffer1`에 `typedArray`**의 데이터를 복사**한다. 

```javascript
gl = {
    key0: value0,
    key1: value1,
    ...,
    ARRAY_BUFFER: 0x8892,
    ...,
    vertexBuffer1 = [ 0.0,  0.5,  0.0, -0.5, -0.5,  0.0, 0.5, -0.5,  0.0 ],
    ...,
    0x8892: vertexBuffer1,
    ...
};
```

이 시점부터 GPU가 Array Buffer를 사용할 때는 별다른 지시 없이도 현재 `gl[gl.ARRAY_BUFFER]`, 즉,  `gl[0x8892]`, 즉, `vertexBuffer1`**에 있는 Array Buffer를 사용하게 된다.**


1. vertexBuffer2 = gl.createBuffer();
2. gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer2);

를 차례로 실행하면 아래와 같이 된다.

```javascript
gl = {
    key0: value0,
    key1: value1,
    ...,
    ARRAY_BUFFER: 0x8892,
    ...,
    vertexBuffer1 = [],
    ...,
    0x8892: vertexBuffer2,  // 2. gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer2)의 결과

    ...,
    vertexBuffer2 = [],  // 1. vertexBuffer2 = gl.createBuffer()의 결과
    ...
};
```

이 시점부터 GPU가 **Array Buffer**를 사용할 때는 별다른 지시 없이도 현재 `gl[gl.ARRAY_BUFFER]`, 즉,  `gl[0x8892]`, 즉, `vertexBuffer2`에 있는 **Array Buffer**를 사용하게 된다.

#### gl.createShader(), gl.createProgram(), gl.createBuffer()

> gl.createShader(), gl.createProgram(), gl.createBuffer()는 GPU에게 뭔가를 실어 보낼 그릇을 반환한다. 
> - 좀더 정확하게는 GPU 메모리 내에 각각 Shader 객체, Program 객체, Buffer 객체를 위한 공간을 할당하고 그 객체를, 즉 GPU 메모리 내부에 대한 포인터를 반환한다. 
> - 하지만 CPU 쪽(JavaScript)에서 이 포인터를 통해 GPU 내의 자원에 직접 접근할 수는 없다.

실제로 코드를 자세히 보면, `gl.createShader()`, `gl.createProgram()`, `gl.createBuffer()`에 의해 반환되는 `shader`, `program`, `buffer`는 `shader.어쩌구`, `program.어쩌구`, `buffer.어쩌구` 하는 식으로 **GPU의 자원에 직접 접근하는 코드는 없다.**(``vertexBuffer.itemsPerVertex = 3;``와 같은 코드에서 `itemsPerVertex`는 단순히 3이라는 값을 나중에 사용하기 위해 편의상 캐쉬해두는 사용자 변수일 뿐 GPU에 자원에 접근할 수 있는 내장 변수가 아니다.)

대신에 **언제나 `gl.method()`의 파라미터로만 사용**된다. 즉, **언제나 GPU에 무언가를 실어나르는 그릇 용도로만 사용**된다. 

 
## CPU - GPU 데이터 통신 정리

![](http://i.imgur.com/SeWlknt.png)

### CPU의 역할

> 사용자 이벤트 관련 데이터와 그리는데 필요한 값 및 메타데이터, 셰이더를 gl컨텍스트에 실어서 GPU에게 보낸다.
> - GPU 자원의 할당/해제는 언제나 gl컨텍스트를 통해 GPU에 시키는 형태로 수행된다.

### GPU의 역할

> CPU로부터 받은 셰이더를 파이프라인에 끼워 넣고, CPU로부터 받은 데이터를 다수의 코어에서 파이프라인을 통해 처리하고 결과를 Frame Buffer에 쓴다.
> - CPU로부터 받을 수 있는 데이터의 형식은 숫자와 이미지 두 가지 뿐이며, CPU에게는 데이터를 담을 그릇을 반환할 뿐 그 외의 어떤 데이터도 전달하지 않는다.



