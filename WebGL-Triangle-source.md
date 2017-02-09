<!DOCTYPE html>
<html lang="ko">
<head>
<title> WebGL 삼각형 그리기 </title>
<meta charset="UTF-8">
<script id="vertexShaderSource" type="x-shader/x-vertex">
// type에는 text/javascript가 아닌 임의의 값을 주면 됨.
//     javascript로 해석되지 않도록 하는 것이 목적

attribute vec3 aVertexPosition;
// attribute : 셰이더 변수 종류(attribute | uniform | varying)
// vec3 : 배열 타입, vec3는 3행 배열
// aVertexPosition : 배열 이름, 한 개의 버텍스를 나타낸다. 관습적으로 맨 앞에 a를 붙여서 attribute임을 표시

void main() {
// 버텍스 하나 마다 실행되는 버텍스 셰이더 실행부
//     버퍼에 있는 배열 데이터는 gl.vertexAttribPointer()의 파라미터에 있는 정보를 기준으로 여러 개의 버텍스로 분리되고,
//     여러 개의 버텍스는 여러 개의 코어에서 이 main() 함수에 의해 한 번에 병렬 처리된다.

    gl_Position = vec4(aVertexPosition, 1.0);
    // vec4() : 네 개의 원소를 파라미터로 받아서 4행 벡터를 반환하는 함수
    // aVertexPosition : 한 개의 버텍스를 나타낸다.
    // 1.0 : 버텍스는 점 이므로 1.0(동차좌표계 내용 참고)
    // gl_Position : 버텍스 셰이더의 내장 변수. 동차좌표계를 사용하므로 vec4 타입만 받는다.
    // 별도의 return 문 없이도 gl_Position에 할당된 값이
    //     파이프라인 상에서 버텍스 셰이더의 다음 단계(primitive 조립)의 입력값으로 전달된다.
    // 이 예제에서는 단순히 버텍스 하나를 그대로 gl_Position에 할당할 뿐이지만,
    //     일반적인 경우 버텍스에 여러가지 변환 연산을 적용한 후에 gl_Position에 값을 할당한다.
}
</script>
<script id="fragmentShaderSource" type="x-shader/x-fragment">
// type에는 text/javascript가 아닌 임의의 값을 주면 됨.
//     javascript로 해석되지 않도록 하는 것이 목적

precision mediump float;
// precision : 데이터의 정밀도 지정
// mediump : 정밀도 수준(highp | mediump | lowp) 대부분의 경우 mediump를 사용
// float : 부동소수형 데이터

void main() {
// 프래그먼트 하나마다 실행되는 프래그먼트 셰이더 실행부
//     프래그먼트 셰이더는 파이프라인 상에서 래스터라이징의 바로 다음 단계에 있으며,
//     래스터라이징의 결과물인 프래그먼트 하나하나마다 이 main() 함수가 실행되는데,
//     여러 개의 코어에서 이 main() 함수가 각기 다른 프래그먼트 정보를 기준으로 한 번에 병렬 실행된다.

    gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
    // 1.0, 1.0, 1.0, 1.0 : r, g, b, a 값
    // gl_FragColor : 프래그먼트셰이더의 내장 변수. rgba 값을 사용하므로 vec4 타입만 받는다.
    // 별도의 return 문 없이도 gl_FragColor에 할당된 값이
    //     파이프라인 상에서 프래그먼트 셰이더의 다음 단계(가위 테스트)의 입력값으로 전달된다.
}
</script>
<script type="text/javascript">
function startup() {
    // WebGL 컨텍스트 생성
    var canvas = document.getElementById('webGLCanvas');
    var gl = createGLContext(canvas);

    // 셰이더 준비
    var shaderProgram = setupShaders(gl);

    // 버퍼 준비
    var vertexBuffer = setupBuffers(gl);

    // 화면 준비
    setupViewport(gl, canvas);

    // 그리기
    draw(gl, shaderProgram, vertexBuffer);
}

/**
 * WebGL 컨텍스트 생성
 *     canvas에서 WebGL 컨텍스트를 가져온다.
 *
 * @canvas WebGL 컨텍스트를 가지고 있는 HTML 캔버스 요소
 */
function createGLContext(canvas) {
    var glNames = [ "webgl", "experimental-webgl" ]; // experimental-webgl은 WebGL이 정식으로 지원되지 않을 때 사용되던 이름
    var context;

    for (var i = 0, l = glNames.length; i < l ; i++) {
        // try 문은 canvas.getContext() 실행 중 에러가 나더라도
        // 스크립트 실행이 정지되지 않고 glNames의 다음 요소로 다시 시도하게 한다.
        try {
            context = canvas.getContext(glNames[i]);
        } catch (e) {}

        if (context)
            return context;
    }

    // 컨텍스트가 없으면 종료
    if (!context) {
        alert("Fail to get WebGL Context");
        return null;
    }
}

/**
 * 셰이더 준비
 *     버텍스 셰이더, 프래그먼트 셰이더의 소스를 읽고, 컴파일하고
 *     셰이더 프로그램에 두 셰이더를 추가하고, 링크한다.
 *
 * @gl WebGL 컨텍스트. 'gl.~~~()는 버스를 통해 GPU에게 무언가를 시키는 것이다.'라고 해석하자.
 */
function setupShaders(gl) {
    // 버텍스 셰이더 소스를 문자열로 담아온다.
    var vertexShaderSource = document.getElementById('vertexShaderSource').text;

    // 버텍스 셰이더를 담을 그릇을 GPU한테 받아온다.
    var vertexShader = gl.createShader(gl.VERTEX_SHADER);

    // vertexShader라는 그릇에 vertexShaderSource를 담아서 GPU에 보내고
    gl.shaderSource(vertexShader, vertexShaderSource);

    // GPU에게 vertexShader를 컴파일하도록 시킨다.
    gl.compileShader(vertexShader);

    // 컴파일이 제대로 되었는지도 GPU에게 시켜서 확인
    if (!gl.getShaderParameter(vertexShader, gl.COMPILE_STATUS)) {
        alert("Error compiling vertex shader : " + gl.getShaderInfoLog(vertexShader));
        // 컴파일이 실패했으면 GPU 메모리에서 vertexShader를 지우도록 GPU에게 시킨다.
        gl.deleteShader(vertexShader);
        return null;
    }

    // 위의 버텍스 셰이더와 똑같은 과정
    var fragmentShaderSource = document.getElementById('fragmentShaderSource').text;
    var fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
    gl.shaderSource(fragmentShader, fragmentShaderSource);
    gl.compileShader(fragmentShader);
    if (!gl.getShaderParameter(fragmentShader, gl.COMPILE_STATUS)) {
        alert("Error compiling fragment shader : " + gl.getShaderInfoLog(fragmentShader));
        gl.deleteShader(fragmentShader);
        return null;
    }

    // 셰이더 프로그램을 담을 그릇을 GPU한테 얻어온다.
    var shaderProgram = gl.createProgram();

    // 셰이더 프로그램 그릇에 컴파일 된 두 셰이더를 담는다.
    gl.attachShader(shaderProgram, vertexShader);
    gl.attachShader(shaderProgram, fragmentShader);

    // 컴파일 된 두 셰이더를 링크한다.
    // 링크할 때 버텍스 셰이더의 varying 변수와 프래그먼트 셰이더의 varying 변수가 연결된다.
    gl.linkProgram(shaderProgram);

    // 링크가 제대로 되었는지도 GPU에게 시켜서 확인
    if (!gl.getProgramParameter(shaderProgram, gl.LINK_STATUS)) {
        alert("Failed to link shaders");
        // 링크가 실패했으면 GPU 메모리에서 shaderProgram를 지우도록 GPU에게 시킨다.
        gl.deleteProgram(shaderProgram);
        return null;
    }

    // 링크까지 성공했으면 그리는데 shaderProgram을 이용하도록 GPU에게 시킨다.
    gl.useProgram(shaderProgram);

    return shaderProgram;
}

/**
 * 버퍼 준비
 *     삼각형을 그릴 정보를 버퍼에 담는다.
 *
 * @gl WebGL 컨텍스트. 'gl.~~~()는 버스를 통해 GPU에게 무언가를 시키는 것이다.'라고 해석하자.
 */
function setupBuffers(gl) {
    // 버퍼를 담을 그릇을 GPU한테 얻어온다.
    var vertexBuffer = gl.createBuffer();

    // GPU 메모리 내에 있는 gl.ARRAY_BUFFER라는 key에 vertexBuffer를 바인딩하도록 GPU에게 시킨다.
    gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);

    // 삼각형의 버텍스 정보(x, y, z 좌표값)
    // 좌표값은 WebGL이 표시되는 영역의 중심을 원점으로 -1.0 ~ 1.0 사이의 값을 쓴다.
    // 어떤 값을 -1.0 ~ 1.0 사이의 값으로 환산하여 표시하는 것을 정규화라고 한다.
    var triangleVertices = [
         0.0,  0.5,  0.0,
        -0.5, -0.5,  0.0,
         0.5, -0.5,  0.0
    ];

    // 삼각형 정보를 Array Buffer에 담고, Array Buffer의 Float32 형 View인 Typed Array를 반환한다.
    // Array Buffer, Typed Array는 3강 내용 참고
    var typedArray = new Float32Array(triangleVertices);

    // GPU 메모리 내에 있는 gl.ARRAY_BUFFER라는 key에 바인딩 되어 있는 버퍼(vertexBuffer)에
    //     삼각형 정보를 담도록 GPU에게 시킨다.
    // https://github.com/projectBS/S63-WebGL/blob/master/Day2-20150908.md#glcreatebuffer-glbindbuffer-glbufferdata 참고
    gl.bufferData(gl.ARRAY_BUFFER, typedArray, gl.STATIC_DRAW);

    // 버텍스 하나의 위치를 나타내는 정보의 개수(1 ~ 4의 값 가능. 2D context라면 2를 쓴다. 여기서는 x, y, z를 사용하므로 3)
    // 나중에 gl.vertexAttribPointer()에 사용
    // itemsPerVertex는 내장 변수 아님
    vertexBuffer.itemsPerVertex = 3;

    // 버텍스의 갯수(여기서는 삼각형이므로 3)
    // 나중에 gl.draw~~~()에 사용
    // numOfVertices는 내장 변수 아님
    vertexBuffer.numOfVertices = 3;

    return vertexBuffer;
}

/**
 * 화면 준비
 *     GPU에게 화면의 viewport 범위를 지정하도록 시키고,
 *     viewport 범위를 싹 칠해버릴 색깔을 지정하도록 시킨다.
 *
 * @gl WebGL 컨텍스트. 'gl.~~~()는 버스를 통해 GPU에게 무언가를 시키는 것이다.'라고 해석하자.
 * @canvas  viewport 범위 정보를 가지고 있는 캔버스
 */
function setupViewport(gl, canvas) {
    // GPU에게 캔버스의 전체를 viewport 로 지정하도록 시킨다.
    gl.viewport(0, 0, canvas.width, canvas.height);

    // GPU에게 viewport 범위를 싹 칠해버릴 색깔을 지정하도록 시킨다.
    gl.clearColor(0.0, 0.0, 0.2, 1.0);
}

/**
 * 그리기
 *     GPU에 각종 버퍼 정보를 전달해주고, gl.draw~~~()로 GPU에게 그리기를 시킨다.
 *
 * @gl
 * @shaderProgram
 * @vertexBuffer
 */
function draw(gl, shaderProgram, vertexBuffer) {
    // GPU에게 viewport 범위를 지정된 색으로 싹 칠하도록 시킨다.
    gl.clear(gl.COLOR_BUFFER_BIT);

    // shaderProgram 내에서 aVertexPosition에 접근할 수 있는 위치값(포인터)을 GPU한테 시켜서 받아온다.
    var indexOfVertexPositionAttrubite = gl.getAttribLocation(shaderProgram, "aVertexPosition");

    // aVertexPosition의 위치값을 이용해서 aVertexPosition에
    //     현재 gl.ARRAY_BUFFER라는 key에 바인딩 되어 있는 버퍼(vertexBuffer, 삼각형 버텍스정보를 담고 있다)를
    //     할당하도록 GPU에게 시킨다.
    gl.vertexAttribPointer(
        indexOfVertexPositionAttrubite, // index : aVertexPosition에 접근할 수 있는 위치값
        vertexBuffer.itemsPerVertex,    // size : 버텍스 하나의 위치를 나타내는 위치 정보의 개수. x, y, z라서 3
        gl.FLOAT,                       // type : vertexBuffer에 담겨있는 데이터 타입(gl.FLOAT | gl.FIXED)
        false,                          // float이 아닌 데이터를 float로 변환할 지 여부
        0,                              // stride : 버텍스 하나를 구성하는 byte 수(0 ~ 255의 값)
                                        //          stride == (위치 정보 개수 + 기타 정보 개수) * 데이터타입의 byte수
                                        //          0이면 기타 정보 개수가 0인 것으로 간주한다.
                                        //          여기서는 기타 정보 없이 위치 정보 개수가 3이므로
                                        //          stride에 0을 주는 것과 12(위치 정보 개수 3 * float의 byte 수 4)를
                                        //          주는 것은 같은 결과를 보여준다.
        0                               // offset : 추출하고자 하는 정보의 시작 위치(byte 단위)
                                        //          여기서는 기타 정보가 없으므로 0이지만
                                        //          기타 정보가 있는 경우 0이외의 값이 올 수 있음
                                        // stride와 offset은 http://stackoverflow.com/a/16888156 참고
    );

    // 파라미터로 지정된 위치에 있는 attribute 변수를 활성화 한다.
    // 활성화의 의미는 그릴 때 이 attribute 변수의 내용을 사용하도록 한다는 의미
    // 안 쓸 때는 disableVertexAttribArray(index)를 해준다.
    gl.enableVertexAttribArray(
        indexOfVertexPositionAttrubite  // index : enable할 attribute 변수의 위치값
    );

    // GPU에게 최종적으로 그리기를 시킨다.
    gl.drawArrays(
        gl.TRIANGLES, // mode : 그리는 방법을 지정. 정확하게는 버퍼 데이터로 생성할 primitive를 지정
                      //        파이프라인 상에서 버텍스 셰이더의 다음 단계인 Primitive 조립 에서
                      //        이 mode 값에 지정된 방식대로 Primitive를 조립한다.
        0,            // first : 그리기에 사용할 첫번째 버텍스의 위치(0이 아니면 안되던데..)
        vertexBuffer.numOfVertices  // count : 버텍스의 갯수
    );
}
</script>
</head>
<body onload="startup();">
    <canvas id="webGLCanvas" width="500" height="500"></canvas>
</body>
</html>
