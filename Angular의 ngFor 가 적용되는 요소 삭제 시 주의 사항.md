# *ngFor가 적용되는 요소 삭제 시 주의 사항

아래와 같이 `*ngFor`에 의해 렌더링되는 엘레먼트(아래 예에서는 `<li>`)는 `*ngFor`의 근간이 되는 배열(아래 예에서는 `baselines`)의 원소가 변경(pop, push, shift, unshift, slice 등) 되면 DOM에서 완전히 삭제된 후에 다시 생성된다. 이런 특징은 문서로 확인한 것은 아니고 `document.getElementById()`로 확인한 것이다.

```html
<span *ngFor="let baseline of baselines; index as i">
  <li class="rulset-input-item">
    <span>{{baseline.name}}</span>
    <span style="cursor: pointer;"><i class="tiny material-icons" (click)="deleteBaselineItem(i)">delete</i></span>
    <input type="text" class="form-control"
           id="baselineRecency{{i}}"
           formControlName="baselineRecency{{i}}"
           placeholder="1~365"
           size="5">
    <input type="text" class="form-control"
           id="baselineFrequency{{i}}"
           formControlName="baselineFrequency{{i}}"
           placeholder="1~999"
           size="5">
  </li>
</span>
```

![Imgur](https://i.imgur.com/bCp2daS.png)

위 화면에서 두 번째 아이템인 `Woman, 20, 1`을 삭제할 때, `baselines.splice(1, 1)`를 사용하면 `*ngFor`의 동작 특징에 의해 `input`이 새로 렌더링 되므로 `{{i}}`가 포함된 `formControlName`의 값은 새로 바뀌는데, `FormControl`의 `name`은 바뀌지 않으므로 일관성이 깨진다.

구체적으로 말하면 삭제 전에는 위 세 원소의 인덱스는 0, 1, 2이고 `FormControl`의 이름도 0, 1, 2로 끝나게 되는데, 여기에서 `splice`로 1을 지우면, 두 원소의 인덱스는 0, 1이 되는데, `FormControl`의 끝자리는 바뀌지 않고 0, 2로 남아있어 일관성이 깨진다는 얘기다.

물론 이를 맞추기 위해 `FormControl`을 하나씩 땡겨서 새로 만들어도 되지만 너무 불편한 일이다.

### 따라서 `*ngFor`에 사용된 배열의 원소 중 `FormControl`과 연동된 원소를 삭제할 때는 `array.splice(pos, counts)`를 사용하면 안 된다.

대신에 **`delete array[index]`를 사용해서 해당 원소를 삭제하는 대신 `undefined`로 만들어서 배열의 길이와 배열 내 원소의 위치를 그대로 보존**해서 `FormControl`과의 일관성이 깨지지 않게 해야한다. 

화면쪽에서는 아래와 같이 `*ngIf="baseline !== undefined"`을 추가해서 화면에서 렌더링 되지 않게 하면 된다.

```html
<span *ngFor="let baseline of baselines; index as i">
  <li class="rulset-input-item" 이 부분 추가!!! *ngIf="baseline !== undefined">
    <span>{{baseline.name}}</span>
    <span style="cursor: pointer;"><i class="tiny material-icons" (click)="deleteBaselineItem(i)">delete</i></span>
    <input type="text" class="form-control"
           id="baselineRecency{{i}}"
           formControlName="baselineRecency{{i}}"
           placeholder="1~365"
           size="5">
    <input type="text" class="form-control"
           id="baselineFrequency{{i}}"
           formControlName="baselineFrequency{{i}}"
           placeholder="1~999"
           size="5">
  </li>
</span>
```

그리고 최종적으로 서버에 전송할 때 배열 원소 중 `undefined`인 것만 `splice`로 삭제한 후 전송하면 된다.

아마도 장바구니 구현할 때 이런 방식이 사용될 것 같다.
