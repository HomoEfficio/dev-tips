# Angular5 개발

개발 순서

- NgModule 생성
- NgModule 등록
- 라우팅 등록
- 메뉴 항목 등록

## NgModule 생성

NgModule은 Java의 package와 비슷한 개념으로 말 그대로 개발할 하나의 모듈을 의미

모듈도 위계를 가질 수 있으며 컨테이너 역할을 하는 상위 모듈은 폴더로만 존재

실제 내용을 가진 하위 모듈에는 보통 다음의 파일이 세트로 존재

- 하위모듈이름.component.ts  // 컴포넌트 로직
- 하위모듈이름.module.ts     // 모듈 정보
- 하위모듈이름.html          // 템플릿
- 하위모듈이름.scss          // 스타일

### module 폴더 생성

`src/app` 아래에 `/추가할모듈이름` 폴더 생성. 여러 개의 하위 모듈이 있다면 `/추가할모듈이름/하위모듈이름01`, `/추가할모듈이름/하위모듈이름02`와 같이 하위 폴더도 생성한다.

### component 파일 작성

`src/app/추가할모듈이름/추가할모듈이름.component.ts` 파일 생성. 파일의 구조는 대략 아래와 같다.

```typescript
import {Component, OnDestroy, OnInit} from '@angular/core';

import { ToastrService } from 'ngx-toastr';

import { BaseComponent } from '../../shared/base-component/base.component';

@Component({
  selector: 'model-list',
  templateUrl: 'model-list.html',
  styleUrls: ['./model-list.scss'],
  providers: []
})
export class ModelListComponent extends BaseComponent implements OnDestroy, OnInit {

  constructor(toastService: ToastrService) {
    super(toastService);
  }

  ngOnDestroy(): void {
  }

  ngOnInit(): void {
  }
}
```


### module 파일 작성

`src/app` 아래에 `/추가할모듈이름` 추가 후 `/추가할모듈이름`에 `추가할모듈이름.module.ts` 작성

대략 아래와 같이 구성된다.

- import
- 하위 Route 지정
- @NgModule 선언
- export module 클래스

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Routes } from '@angular/router';
import { FormsModule } from '@angular/forms';
import { ModelListComponent } from './model-list/model-list.component';

const routes: Routes = [
  {path: '', component: ModelListComponent},
];

@NgModule({
  imports: [RouterModule.forChild(routes), CommonModule, FormsModule],
  providers: [],
  declarations: [ModelListComponent],
  exports: [ModelListComponent]
})
export class ExpansionModule {
}

```

### @NgModule

#### declarations

@NgModule 내부의 local scope에 존재하는 컴포넌트로서 package내에서만 접근 가능

외부에서 사용 가능하게 하려면 export해야함

#### providers

@NgModule 내의 컴포넌트에서 사용하는 외부 서비스로서 보통 global scope



## NgModule 등록

src/app/app.module.ts 의 import 문과 @NgModule의 imports에 모듈 추가해서 Application에서 사용할 수 있도록 모듈 등록

```typescript
import { NgModule } from '@angular/core';
import ...;

import { ExpansionModule } from './expansion/expansion.module';  // 여기에 추가

@NgModule({
  declarations: [
    ...
  ],
  imports: [
    ...,
    ExpansionModule  // 여기에 추가
  ],
  providers: [...],
  bootstrap: [AppComponent]
})
export class AppModule { }

```

## 라우팅 추가

src/app/app.routing.module.ts 의 @NgModule의 imports에 라우팅 추가

```typescript
import { NgModule } from '@angular/core';
import ...;

const routes: appRoutes = [
  {path: '', component: HomeComponent},
  ...,
  {
    path: 'expansion',
    canActivate: [AuthGuard, SiteGuard],
    loadChildren: 'app/expansion/expansion.module#ExpansionModule'
  },
  {path: '**', component: NotFoundComponent}
];

@NgModule({
  exports: [RouterModule],
  imports: [
    RouterModule.forRoot(appRoutes)
  ]
})
export class AppRoutingModule { }
```


## 상단 메뉴 추가

dashboard.toolbar.components.ts 에 추가

