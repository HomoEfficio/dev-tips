# Angular6 - Reactive Forms - Select

Reactive Forms 방식으로 Angular6에서 `<select>` 목록 중 특정 값을 선택한 상태로 화면에 표시하는 간단한 코드 쪼가리


## 모듈 설치

아래 명령으로 [ng-select](https://github.com/ng-select/ng-select)를 설치한다.

```
npm install --save @ng-select/ng-select
```

또는

```
yarn add @ng-select/ng-select
```

`@NgModule`의 `imports`에 `FormsModule`, `ReactiveFormsModule`, `NgSelectModule` 추가

```typescript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { SelectComponent } from './select/select.component';
import { FormsModule, ReactiveFormsModule } from "@angular/forms";
import { NgSelectModule } from "@ng-select/ng-select";

@NgModule({
  declarations: [
    AppComponent,
    SelectComponent
  ],
  imports: [
    BrowserModule, FormsModule, ReactiveFormsModule, NgSelectModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }

```

## 데이터 모델

`select`에 표시될 데이터 모델

```typescript
export const coinNames = ['BITCOIN', 'ETHEREUM', 'EOS', 'IOTA'];

export const coinModels = [
  { id: 1, name: 'Bitcoin' },
  { id: 2, name: 'Ethereum' },
  { id: 3, name: 'EOS' },
  { id: 4, name: 'IOTA' },
];
```

## 컴포넌트

`FormBuilder`를 통해 생성하는 `FormGroup`에 포함되는 `FormControl`의 초기값을 지정해주면, `select`에서 선택된 상태로 표시할 수 있다.

```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';

import { coinModels, coinNames } from './data-model';


@Component({
  selector: 'app-select',
  templateUrl: './select.component.html',
  styleUrls: ['./select.component.css']
})
export class SelectComponent implements OnInit {

  coinNames = coinNames;
  coinModels = coinModels;

  formGroup: FormGroup;

  constructor(private fb: FormBuilder) {
    this.setupFormGroup();
  }

  ngOnInit() {

  }

  private setupFormGroup() {
    this.formGroup = this.fb.group({
      select: coinNames[2],    // 초기값 지정
      ngSelect: coinModels[1].id  // 템플릿 파일의 bindValue 항목으로 지정한 id 값을 넣어줘야 한다.
    });
  }
}
```

## 템플릿 파일

```html
<h2>Reactive Forms - select</h2>
<form [formGroup]="formGroup">
  <div>
    <label class="center-block">Select:
      <select formControlName="select">
        <option *ngFor="let coinName of coinNames" [value]="coinName">{{coinName}}</option>
      </select>
    </label>
  </div>
  <div>
    <label class="center-block">NgSelect:
      <ng-select formControlName="ngSelect"
                 [items]="coinModels"
                 bindLabel="name"
                 bindValue="id">
      </ng-select>
    </label>
  </div>
</form>
```

## 스타일 파일

`ng-select`는 다음과 같이 스타일을 지정해주지 않으면 화면에 제대로 표시되지 않는다.

```css
@import "~@ng-select/ng-select/themes/default.theme.css";
```

## 결과

![Imgur](https://i.imgur.com/um9XpvS.png)

## 프로젝트 전체 파일

https://github.com/HomoEfficio/ng6-project
