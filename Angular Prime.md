# Angular Prime

[Angular Essential](https://www.rubypaper.co.kr/82) 요약

# 구동 흐름

1. `angular(-cli).json` 에 설정된 대로 빌드된 결과 `index` 항목으로 설정된 `src/index.html`, `main` 항목으로 설정된 `src/main.js`가 생성됨
1. 브라우저에 `src/index.html` 파일이 로딩되고, html 파일 내 `<script>`로 지정된 `src/main.js`가 로딩됨
1. `src/main.js`의 원래 내용은 `angular(-cli).json`에 설정된대로 `src/main.ts`임
1. `src/main.ts`의 전형적인 내용은 다음과 같으며 환경 설정 내용을 읽고 부트스트랩 모듈을 로딩

    ```typescript
    import { enableProdMode } from '@angular/core';
    import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

    import { AppModule } from './app/app.module';
    import { environment } from './environments/environment';

    if (environment.production) {
      enableProdMode();
    }

    platformBrowserDynamic().bootstrapModule(AppModule);
    ```

1. 부트스트랩 모듈인 `AppModule`은 `src/app/app.module.ts`에 있으며, 다음과 같이 `bootstrap` 항목으로 지정한 루트 컴포넌트를 로딩

    ```typescript
    @NgModule({
      declarations: [AppComponent],
      imports: BrowserModule,
      providers: [],
      bootstrap: [AppComponent]
    })
    export class AppModule {
    }
    ```

1. `src/app/app.component.ts`에 있는 `AppComponent`가 렌더링하는 내용이 `src/index.html`에 있는 루트 컴포넌트 디렉티브 `<app-root>`에 표시됨


# 웹 컴포넌트

재사용 가능한 HTML 커스텀 요소를 만드는 컴포넌트의 모음으로, 다음 네 가지 스펙을 기반으로 한다.

- Custom Elements
- Shadow DOM
- ES Modules
- HTML Template

## 구성

앵귤라는 일반적으로 다음의 네 가지 파일이 하나의 컴포넌트를 구성한다.

- `컴포넌트이름.component.ts`: 모델 및 화면 로직
- `컴포넌트이름.html`: 템플릿
- `컴포넌트이름.scss`: 스타일

## 데이터 및 이벤트 교환

컴포넌트와 템플릿의 데이터 및 이벤트 교환은 다음과 같다.

> 프로퍼티 바인딩으로 `component.ts`에 있는 모델 데이터를 템플릿에 전달
>
> 이벤트 바인딩으로 `html` 템플릿의 이벤트를 `component.ts`에 전달

## 포함하는 컴포넌트 vs 포함되는 컴포넌트

A 컴포넌트는 C 컴포넌트를 포함하고 B 컴포넌트도 C 컴포넌트를 포함한다고 하면,

- A, B 컴포넌트의 `A.component.ts` 파일과 `B.component.ts` 파일은 C 컴포넌트의 존재를 모른다.
   - 단지 프로퍼티 바인딩을 통해 `A.component.ts`는 `A.html`에, `B.component.ts`는 `B.html`에 데이터를 전달해줄 뿐
   - 필요한 경우 `@ViewChild` 데코레이터를 써서 C 컴포넌트의 존재를 인식하고 사용할 수도 있다.
 - A, B 템플릿 파일은 C 컴포넌트의 존재를 알고 있으며, `<c>...</c>`와 같은 형식으로 C 컴포넌트를 명시적으로 사용한다.


# 모듈

- 다른 모듈에 import되어 재사용 될 수 있는 단위
- 컴포넌트는 모듈에 포함되고, 해당 모듈이 import 되어야만 모듈 내의 컴포넌트도 다른 모듈에 import 되어 사용될 수 있다.
- Angular 앱의 최상위 모듈은 `src/app/app.module.ts`에 명시되어 있으며, `@NgModule`의 `imports` 항목에 명시된 모듈만 사용될 수 있다.
- 직접 작성한 모듈(아래 예에서는 UserListComponent)이 제대로 import 되지 않으면 브라우저 콘솔에 다음과 같은 에러가 발생한다.

    ```
    Uncaught Error: Component UserListComponent is not part of any NgModule or the module has not been imported into your module.
    ```

- 외부 모듈을 제대로 import 하지 않으면 브라우저 콘솔에 다음과 같은 에러가 발생한다.

    ```
    compiler.js:1021 Uncaught Error: Template parse errors:
    'mat-card-title' is not a known element:
    1. If 'mat-card-title' is an Angular component, then verify that it is part of this module.
    2. If 'mat-card-title' is a Web Component then add 'CUSTOM_ELEMENTS_SCHEMA' to the '@NgModule.schemas' of this component to suppress this message. ("<div class="page-layout">
      <mat-card class="page-layout-content">
        [ERROR ->]<mat-card-title>
          <div class="page-layout-header">
            <div class="col-lg-8" style="font-wei"): ng:///UserListModule/UserListComponent.html@2:4
    'mat-card-content' is not a known element:
    1. If 'mat-card-content' is an Angular component, then verify that it is part of this module.
    2. If 'mat-card-content' is a Web Component then add 'CUSTOM_ELEMENTS_SCHEMA' to the '@NgModule.schemas' of this component to suppress this message. ("
          </div>
        </mat-card-title>
        [ERROR ->]<mat-card-content class="report-content col-md-12">
          <div class="report-content__tab-content" st"): ng:///UserListModule/UserListComponent.html@46:4
    'mat-card' is not a known element:
    1. If 'mat-card' is an Angular component, then verify that it is part of this module.
    2. If 'mat-card' is a Web Component then add 'CUSTOM_ELEMENTS_SCHEMA' to the '@NgModule.schemas' of this component to suppress this message. ("<div class="page-layout">
      [ERROR ->]<mat-card class="page-layout-content">
        <mat-card-title>
          <div class="page-layout-header">
    "): ng:///UserListModule/UserListComponent.html@1:2
        at syntaxError (compiler.js:1021)
        at TemplateParser.push../node_modules/@angular/compiler/fesm5/compiler.js.TemplateParser.parse (compiler.js:14851)
        at JitCompiler.push../node_modules/@angular/compiler/fesm5/compiler.js.JitCompiler._parseTemplate (compiler.js:24708)
        at JitCompiler.push../node_modules/@angular/compiler/fesm5/compiler.js.JitCompiler._compileTemplate (compiler.js:24695)
        at compiler.js:24638
        at Set.forEach (<anonymous>)
        at JitCompiler.push../node_modules/@angular/compiler/fesm5/compiler.js.JitCompiler._compileComponents (compiler.js:24638)
        at compiler.js:24548
        at Object.then (compiler.js:1012)
        at JitCompiler.push../node_modules/@angular/compiler/fesm5/compiler.js.JitCompiler._compileModuleAndComponents (compiler.js:24547)
    ```

## 개별 모듈 작성 사례

`***.module.ts` 파일에 다음과 같이 선언적인 내용이 담겨있다.

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Routes } from '@angular/router';
import { TraitTargetScheduleResultModule } from './history/trait-target-schedule-result.module';
...

const routes: Routes = [
  {path: '', component: TraitTargetJobListComponent},
];

@NgModule({
  imports: [
    RouterModule.forChild(routes),
    CommonModule,
    MatCardModule, MatInputModule, MatButtonModule, MatIconModule,
    DataTableModule,
    TraitTargetScheduleResultModule
  ],
  exports: [TraitTargetJobListComponent],
  declarations: [TraitTargetJobListComponent],
  providers: [TraitTargetJobAdminService, DmpAuthInfoService, ToastrService],
})
export class TraitTargetJobListModule {

}
```

- imports: 본 모듈에서 import 해서 사용할 외부 모듈 리스트
- exports: 외부에 import 되어 사용될 수 잇는 본 모듈의 컴포넌트 리스트
- declarations: 본 모델에서 정의된 컴포넌트 리스트
- providers: 본 모듈에서 사용될 외부 서비스 리스트

## 다른 컴포넌트를 상속해서 만든 컴포넌트와 모듈

예를 들어 A 모듈의 B 컴포넌트를 상속해서 새로 만든 C 컴포넌트가 있다고 하자.  
C 컴포넌트의 물리적 파일 위치는 C 컴포넌트를 사용하는 D 모듈의 폴더 안에 있더라도, C 컴포넌트는 `a.module.ts`의 `declarations`와 `exports`에 명시되어야 하고, D 모듈은 A 모듈을 import 해야 D 모듈 내에 있는 E 컴포넌트의 템플릿이 C 컴포넌트의 템플릿을 인식해서 C 컴포넌트를 사용할 수 있게 된다.

정리하면 상속해서 만든 자식 컴포넌트는 자식 컴포넌트를 사용하는 모듈에 포함되는 것이 아니라, 부모 컴포넌트가 포함된 모듈에 함께 포함된다.


# 바인딩

컴포넌트와 템플릿은 바인딩을 통해 상호 작용

## 프로퍼티 바인딩

컴포넌트 -> 템플릿 단방향 바인딩

> <element [property]="expression">...\</element>

`{{expression}}` 형태로 사용하는 interpolation은 프로퍼티 바인딩의 편리한 표기방식

## 애트리뷰트 바인딩

컴포넌트 -> 템플릿 단방향 바인딩

><element [attr.attribute-name]='expression'>...\</element>

### 프로퍼티와 애트리뷰트의 차이

- 프로퍼티는 DOM 트리에 속한 노드 객체에 속하는 속성으로 동적으로 변한다.
- 애트리뷰트는 HTML 문서에 존재하며 불변이다.

예를 들어, `<input id="nickname" type="text" value="angular">`가 있을 때, 다음과 같이 `value`의 값을 바꾸고,

`document.getElementById("nickname").value = '앵귤라'`

`document.getElementById("nickname").value`를 출력해보면 `앵귤라`가 표시되지만,  
`document.getElementById("nickname").getAttribute('value')`를 출력해보면 원래 값인 `angular`가 표시된다.

## 이벤트 바인딩

템플릿 -> 컴포넌트 단방향 바인딩

><element (event)="statement">...\</element>  
><element (click)="aPublicMethodOfComponent('angular')">...\</element>  
><input type="text" [value]="alias" (input)="setAlias($event)">...\</element>  

1. 템플릿에서 이벤트가 발생하면 `statement`로 지정된 이벤트 핸들러가 호출되고,
1. 컴포넌트에 정의되어 있는 이벤트 핸들러가 컴포넌트의 프로퍼티 값을 변경하면,
1. 프로퍼티 바인딩에 의해 변경된 값이 템플릿에 반영된다.

## 양방향 바인딩

컴포넌트 <--> 템플릿 양방향 바인딩

결과적으로 양방향 바인딩처럼 동작하지만 실제로는 프로퍼티 바인딩과 양방향 바인딩의 조합으로 구현

><element [(name)]="expression">...\</element>  
><element [(ngModel)]="name">...\</element>  
><input [value]="userName" (input)="userName = $event.target.value"/>
><input [(ngModel)]="userName"/>

`ngModel`은 폼 바인딩 편리 기능을 제공하는 특수 디렉티브

## 클래스 바인딩

컴포넌트 -> 템플릿 단방향 바인딩

><element [class]="class-name">...\</element>  
><element [class.class-name]="booleanExpression">...\</element>

`booleanExpression`의 값이 참이면 `class-name`이 `class` 애트리뷰트의 값으로 추가된다. 이미 동일한 `class-name`이 있으면 따로 추가되지는 않으며, 이미 동일한 `class-name`이 있는데 `booleanExpression`이 거짓이면, 기존에 있는 `class-name`이 제거된다.

`class-name`에는 직접 클래스 이름이 올 수도 있고 이 때는 단항 클래스 바인딩이며, 여러 클래스 이름을 문자열로 가지고 있는 변수가 올 수도 있고 이를 다항 클래스 바인딩이라고 한다.

## 스타일 바인딩

컴포넌트 -> 템플릿 단방향 바인딩

><element [style.style-property]="expression">...\</element>  
><div [style.font-size.px]="'16'">


# 디렉티브

- DOM의 모든 것(모양, 동작 등)을 관리하기 위한 명령
- 애플리케이션 전역에서 사용할 수 있는 공통 관심사를 컴포넌트에서 분리해서 구현한 것으로 HTML 요소 또는 애트리뷰트 형태로 사용될 수 있음

## Built-in Structural 디렉티브

- 조건, 반복 처리 담당
- `[]` 안에 디렉티브를 쓰지 않고 디렉티브 앞에 `*` 표시
- 하나의 호스트 요소에 하나의 구조 디렉티브만 사용 가능
- `ngIf`, `ngFor`, `ngSwitch`

### ngIf

><element \*ngIf="booleanExpression">...\</element>
>
>아래의 코드로 변환된다.
>
><ng-template [ngIf]="booleanExpression">  
>  \<element>...\</element>  
>\</ng-template>

1. 별도의 ng-template 사용하지 않음

   >\<div \*ngIf="mySkill==='HTML'; else elseBlock">HTML\</div>  
   >\<ng-template #elseBlock>\<div>CSS\</div>\</ng-template>

1. 별도의 ng-template 사용

   >\<div \*ngIf="mySkill==='HTML'; then thenBlock_1 else elseBlock_1">\</div>  
   >\<ng-template #thenBlock\_1>\<div>HTML\</div>\</ng-template>  
   >\<ng-template #elseBlock>\_1>\<div>CSS\</div>\</ng-template>

### ngFor


### ngSwitch


# 개발 팁

## html 에서 typescript enum 접근 하기

getter를 통해 html에 참조를 제공할 수 있다. `UserAdminAction`이라는 enum이 있다고 하면 component.ts 파일에서 다음과 같이 getter를 제공하면,

```typescript
  get userAdminAction() {
    return UserAdminAction;
  }
``

html 에서는 다음과 같이 참조할 수 있다.

```html
  <button class="btn btn-block btn-danger"
          type="button"
          (click)="action({'action': userAdminAction.DELETE}, row, $event)">삭제</button>
```

### `*ngFor` 에서 enum 사용

`*ngFor`는 `let .. of ..` 구문으로 사용되는데 enum 은 `let .. of ..` 가 아니라 `let .. in ..`으로 iterate 할 수 있다. 따라서 `*ngFor`에서는 직접적으로 enum 을 사용할 수 없고 다음과 같이 array 로 변환한 값을 사용하면 된다.

```typescript
public options = Object.values(UserAdminAction);
```

```html
<select>
    <option *ngFor="let option of options" [value]="option">{{option}}</option>
</select>
```


## Reactive 방식으로 외부 데이터 조회 후 Child에 데이터 전달

Child 컴포넌트에 데이터를 전달하려면,

- Parent HTML에서 `<child [child에서 사용할 변수]="parent에 정의된 변수">`로 전달하고,
- Child의 컴포넌트에서 `@Input('child에서 사용할 변수') public 변수명`으로 받으면 된다.

그런데 Parent가 외부에서 Reactive 방식으로 받아온 데이터를 Child에 전달하면, 외부에서 데이터를 받아오는 동안에 Child가 이미 렌더링되고, 이 시점에서는 Parent에서 받아온 값이 없으므로 Child의 `@Input` 변수는 `undefined`상태가 된다.

이 때는 Child 컴포넌트를 다음과 같이 `OnChanges`를 구현하게 하고, `ngOnChanges(changes: SimpleChanges): void {}` 안에서 changes를 통해 데이터를 받아서 사용하면 된다.

```typescript
export class UserSiteRoleComponent extends BaseComponent implements OnInit, OnChanges {

  @Input('companySites')
  public companySites: CompanySite[];

  @Input('userSites')
  public userSites: UserSite[];
  
  ngOnChanges(changes: SimpleChanges): void {
    console.log('in user-site-role.ngOnChanges, changes:', changes);
    console.log('in user-site-role.ngOnChanges, companySites:', this.companySites);
    console.log('in user-site-role.ngOnChanges, userSites:', this.userSites);
  }
```



# 기타 이슈


## Circular dependency detected

상속받을 부모 컴포넌트(아래 예에서는 `DataTableComponent`)를 다음과 같이 축약형으로 import하면 발생한다.

```typescript
import { Component, Input, OnDestroy, OnInit } from '@angular/core';
import { Site } from '../../../../site/shared/site';
//import { DataTableComponent } from '../../../../shared/data-table';  // <-- 여기!!
// 아래와 같이 명시해줘야 함
import { DataTableComponent } from '../../../../shared/data-table/data-table.component';

@Component({
  selector: 'company-site-table',
  templateUrl: 'company-site-table.html',
  styleUrls: ['company-site-table.scss']
})
export class CompanySiteTableComponent extends DataTableComponent {

  @Input()
  public itemList: Site[];

  public count = 10;

  public selectRow(item: any) {
    super.selectRow(item);
  }
}

```

## Multiple assets emit to the same filename common.js

다른 모듈에 존재하는 Service를 `provider`로 지정하고 가져와서 사용하면 발생

아래와 같이 `SiteService`를 `providers`에 지정하고 생성자 주입을 통해 사용하면,

```typescript
@Component({
  selector: 'company-edit-dialog',
  templateUrl: 'company-edit.html',
  styleUrls: ['./company-edit.scss'],
  //providers: [SiteService]  // <-- 여기!!
})
export class CompanyEditComponent extends BaseComponent implements OnInit {

  constructor(private companyEditService: CompanyEditService,
              private toastrService: ToastrService,
              private siteService: SiteService,  // <-- 여기!!
              @Inject(Http) private http: Http) {
    super(toastrService);
  }

```

다음과 같이 `Conflict: Multiple assets emit to the same filename common.js` 에러 발생으로 컴파일 실패

```
ℹ ｢wdm｣: Compiling...

Date: 2019-03-20T05:47:05.870Z - Hash: 7c4933b3a48c63da5b33 - Time: 33753ms
38 unchanged chunks
chunk {main} main.js, main.js.map (main) 4.17 MB [initial] [rendered]
chunk {runtime} runtime.js, runtime.js.map (runtime) 7.98 kB [entry] [rendered]
chunk {35} 35.js, 35.js.map () 14.7 kB  [rendered]
chunk {15} 15.js, 15.js.map () 25.8 kB  [rendered]
chunk {21}  (common)  [rendered]

ERROR in chunk common
common.js
Conflict: Multiple assets emit to the same filename common.js
ℹ ｢wdm｣: Failed to compile.
```

~좋은 해결방법이 아니지만, 일단 `SiteService`에 있는 메서드 중 사용할 메서드를 `CompanyEditComponent` 안에 인라인화하고 `SiteService`를 `providers`와 생성자에서 제거하면 컴파일 성공~

`SiteService`를 가져올 때 다음과 같이 `@Inject`를 사용해서 가져오면 컴파일 에러는 발생하지 않지만,

```typescript
@Component({
  selector: 'company-edit-dialog',
  templateUrl: 'company-edit.html',
  styleUrls: ['./company-edit.scss'],
})
export class CompanyEditComponent extends BaseComponent implements OnInit {

  constructor(private companyEditService: CompanyEditService,
              private toastrService: ToastrService,
              @Inject(SiteService) private siteService: SiteService,  // <-- 여기!!
              @Inject(Http) private http: Http) {
    super(toastrService);
  }

```

다음과 같이 런타임에 에러 발생

```
ERROR Error: Uncaught (in promise): Error: StaticInjectorError(AppModule)[CompanyEditComponent -> SiteService]: 
  StaticInjectorError(Platform: core)[CompanyEditComponent -> SiteService]: 
    NullInjectorError: No provider for SiteService!
Error: StaticInjectorError(AppModule)[CompanyEditComponent -> SiteService]: 
  StaticInjectorError(Platform: core)[CompanyEditComponent -> SiteService]: 
    NullInjectorError: No provider for SiteService!
```

`@Component`의 `providers`에 따로 지정해줘야 정상 동작함

```typescript
@Component({
  selector: 'company-edit-dialog',
  templateUrl: 'company-edit.html',
  styleUrls: ['./company-edit.scss'],
  providers: [SiteService]  // <-- 여기!!
})
export class CompanyEditComponent extends BaseComponent implements OnInit {

  constructor(private companyEditService: CompanyEditService,
              private toastrService: ToastrService,
              @Inject(SiteService) private siteService: SiteService,  // <-- 여기!!
              @Inject(Http) private http: Http) {
    super(toastrService);
  }

```

그런데 충격적인 것은 이제 `@Inject`를 제외해도 컴파일 에러가 발생하지 않는다.. 뭥미..





