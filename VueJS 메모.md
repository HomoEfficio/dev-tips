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

  다음과 같이 함수 호출이 아니라 `data`의 속성인 것처럼 참조해서 사용할 수 있다.

  ```html
  <body>
  <div id="example">
  2배하면: {{ mul2 }}
  <hr/>
  <ul>
    <li v-for="(item, index) in mul2" :key="index">{{ item }}</li>
  </ul>
  </div>
  </body>
  ```
  
  `data`에 사용된 속성의 이름과 `computed`의 속성이 겹치면 에러가 발생한다.
  
  ```
  vue.js:597 [Vue warn]: The computed property "mul2" is already defined in data.

  (found in <Root>)
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


## watch properties

- computed properties와 마찬가지로 계산이 포함된 함수를 마치 `data`의 속성인 것처럼 참조해서 사용할 수 있으나
  - `data`의 속성에 포함된 이름을 `watch`의 속성 이름으로 사용해야 해당 속성의 값 변경이 감지될 수 있다.
    - 반면에 `data`의 속성에 포함된 이름을 `computed`의 속성 이름으로 사용하면 에러가 발생한다.
  - 계산을 담당하는 함수가 값을 반환하지 않아도 된다.
    - 반환하지 않아도 되므로 비동기 호출 로직을 사용될 수 있다.


## 라이프 사이클 - CMUD(Create Mount Update Destroy)

### beforeCreated

Vue 객체가 생성되고 이벤트와 라이프사이클 초기화 된 직후

### Created

Vue 객체가 생성되고 데이터에 대한 관찰(watch), 계산된 속성(computed), 메서드 등 설정 완료 직후

### beforeMount

마운트(HTML 엘레먼트에 Vue 객체를 바인딩)가 시작되기 직전

### Mounted

`el`(또는 `template`)에 Vue 객체의 데이터가 마운트 된 직후

### beforeUpdate

데이터가 변경되었으나 관련 가상 DOM이 렌더링 되기 전

### Updated

데이터 변경에 의해 가상 DOM 렌더링이 완료된 직후

### beforeDestroy

Vue 인스턴스가 제거되기 직전

### Destroy

Vue 인스턴스 제거 직후





