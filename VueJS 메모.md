# vue.js 메모

간단한 것 말고 좀 기억해둬야할 필요가 있는 것 메모

## computed properties

- 실제로는 계산을 적용할 수 있는 함수인데 마치 `data`의 속성값을 읽어오듯이 속성처럼 사용할 수 있어서 computed properties 라고 부른다.

  ```javascript
  const vm = new Vue({
    el : "#example",
    data : { num : 0 },
    computed : {   
      mul2 : function() {
        return this.num * 2;
      }
    }
  });
  ```

  ```html
  <body>
  <div id="example">
  2배하면: {{mul2}}
  </div>
  </body>
  ```

- computed properties에 함수를 직접 할당하지 않고 getter/setter가 있는 객체를 할당하면 읽기뿐 아니라 쓰기도 가능

  ```javascript
  const vm = new Vue({
    el : "#example",
    data : { num : 0 },
    computed : {   
      rw : {
        get: function() {
          return this.num * 2
        },
        set: function(v) {
          this.num = v;
        }
      }
    }
  });

  // Console 에서 다음과 같이 rw에 set 하면 get을 통해 {{rw}} 에 6이 출력된다.
  vm.$data.rw = 3;  // html
  ```

- computed properties가 읽기로 사용되면 computed properties에 할당된 함수는 항상 값을 반환해야 하는데, 이로 인해 `computed` 옵션을 비동기 처리에 사용할 수 없다.
  - 비동기 처리에는 값을 반환하지 않는 함수를 사용할 수 있는 `watch` 옵션을 써서 wachted properties를 사용해야 한다.

