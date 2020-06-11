# Angular Prime

[Angular Essential](https://www.rubypaper.co.kr/82) 요약 + 개발하다 알게된 것들

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
```

html 에서는 다음과 같이 참조할 수 있다.

```
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

### 값에 해당하는 enum 구하기

`Enum클래스이름[값]`으로 enum 을 구할 수 있다. 값에는 string, number 모두 올 수 있다.

```typescript
this.authKeyForm.get('authKeyType').patchValue(AuthKeyType[$event.value]);
```

## RadioButton 사용 시 주의 사항

https://www.ngxfoundation.com/components/buttons 여기보고 따라하면 되는데 잊지 말아야 할 것은 링크의 Usage에 나와있는 것처럼 `ButtonsModule.forRoot()`를 모듈의 import에 추가해야 한다는 점이다. 추가하지 않으면 다음과 같은 에러가 난다.

```
No value accessor for form control with name:
```

https://github.com/valor-software/ngx-bootstrap/issues/1648 참고

또한 `btnRadio`에 단순 텍스트만 지정할 수 있는 것 같지만 아래와 같이 `{{ }}`로 변수를 넣어주면 변수를 사용할 수도 있다. `btnRadio`의 값은 항목마다 달라야 한다.

```typescript
<div class="btn-group" btnRadioGroup [(ngModel)]="companySite.userRoleId">
        <label *ngFor="let userRole of userRoles; index as j" class="btn btn-success"
               btnRadio="{{ userRole }}" tabindex="0" role="button" [(ngModel)]="companySite.userRoleId">{{ userRole }}</label>
      </div>
</div>
```

## Reactive Form Group의 데이터에서 모델 객체 생성

Reactive Form Group에 속한 폼 컨트롤러 들의 formControllerName 과 모델 객체의 프로퍼티 이름이 같다면 다음과 같이 `Partial`과 `Object.assign()`을 활용해서 쉽게 모델 객세 생성 가능

```typescript
// 모델
export class UserAdminDetail {

  public id: string;
  public email: string;
  public name: string;
  public password1: string;
  public password2: string;
  public company: Company;
  public userSites: UserSite[];
  public deptName: string;
  public jobPosition: string;
  public job: string;
  public userId: string;

  public constructor(init?: Partial<UserAdminDetail>) {
    Object.assign(this, init);
  }
}

// 컴포넌트
const userDetail = new UserAdminDetail(this.userForm.value);  // userForm이 Form Group 타입의 Reactive Form Group 임
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


## Reactive Form Group 방식으로 외부 데이터 조회 후 배열인 변수에 데이터 설정

회사(company) 목록을 ng-select selectbox 에 담아서 보여주는 화면을 생각해보자. selectbox에 바인딩 되는 데이터는 다음과 같이 배열이다.

```typescript
this.companies: Company[] = [];
```

constructor에서 아래와 같은 initForm 메서드를 호출해서 Reactive Form Group을 설정한다.

```typescript
private initForm() {
  this.userForm = this.formBuilder.group({
    email: ['', [Validators.required]],
    name: ['', [Validators.required]],
    password1: ['', [Validators.required]],
    password2: ['', [Validators.required]],
    companyName: [''],  // <-- 배열을 품고 있는 selectbox
    deptName: [''],
    jobPosition: [''],
    job: ['']
  });
}
```
companyName의 Form은 다음과 같다.

```html
<ng-select formControlName="companyName"  id="companyList"
          [items]="companies"
          bindLabel="companyName"
          bindValue="companyId"
          (change)="selectCompany($event)">
  <ng-template ng-option-tmp let-item="item">{{item.companyName}}</ng-template>
</ng-select>
```

ngOnInit에서 company의 목록을 외부에서 다음과 같이 조회해서 `companies`에 push 메서드를 통해 데이터를 채우면,

```typescript
this.companyService.getCompanies()
  .do((response) => {
    if (response['code'] !== PocAPIStatusCode.OK) {
      this.toastrService.error('회사 리스트 정보를 가져오는데 실패하였습니다. 다시 시도해주세요.');
    }
  })
  .map((data) => data['result'])
  .map(result => result['companies'])
  .subscribe((companies: Company[]) => {
    companies.forEach(company => this.companies.push(new Company(company)));    
    console.log('this.companies:', this.companies);
  });
```

화면의 selectbox에 회사 목록이 표시되지 않는다.

push로 이미 바인딩 된 배열에 데이터를 채우지 말고, 다음과 같이 아예 새로운 배열을 할당하면 selectbox에 회사 목록이 표시된다.

```typescript
this.companyService.getCompanies()
  .do((response) => {
    if (response['code'] !== PocAPIStatusCode.OK) {
      this.toastrService.error('회사 리스트 정보를 가져오는데 실패하였습니다. 다시 시도해주세요.');
    }
  })
  .map((data) => data['result'])
  .map(result => result['companies'])
  .subscribe((companies: Company[]) => {
    const tmpCompanies: Company[] = [];
    companies.forEach(company => tmpCompanies.push(new Company(company)));
    this.companies = tmpCompanies;  // <-- 새로 할당!!
    console.log('this.companies:', this.companies);    
  });
```

push를 통해 이미 바인딩 되어 있는 배열에 원소를 채워도 ng-select에서 변경 감지를 못하는 것으로 보인다.

## ReactiveForms에 있는 데이터를 서버에 전송하는 방법

- 엄밀하게 TypeScript를 쓴다면 서비스 메서드에 전달할 때 `new XXX(this.formGroup.value)`로 생성하고, FormGroup 안에 중첩되어 있는 객체도 모두 `new YYY(this.formGroup.get('yyy').value)`와 같이 생성하고 전달해야 정확한 타입 정보가 유지되지만,
- 서비스 메서드 이후로는 그냥 서버에 전송하는 일 밖에 없으므로 사실 상 타입 정보가 필요 없고, 서버에 전송될 때는 어차피 직렬화되어 전송되므로,
- 서비스 메서드 호출 시 그냥 `this.formGroup.value`으로 전달해도 나쁘지 않을 것 같다.

## JSON 객체를 클래스 인스턴스로 Mapping

- https://github.com/typestack/class-transformer 참고
- `npm install class-transformer --save`
- `npm install reflect-metadata --save`
- 소스에서 다음과 같이 사용
    ```typescript
    import { plainToClass } from 'class-transformer';
    ...
    const jobInfoResponse = plainToClass(JobInfoResponse, result);
    ```

## `mat-checkbox` 사용

- 기본으로 체크되어 있도록: `<mat-checkbox [checked]=true (change)="filterChart(ExecutionType.BATCH, $event)">배치 작업</mat-checkbox>`
- `(click)`이 아니라 `(change)` 이벤트 핸들러를 등록해야 `MatCheckboxChange` 이벤트가 인자로 넘겨지며 `.checked`로 쉽게 체크 여부를 읽을 수 있다.


## `@ViewChild`로 지정한 자식 컴포넌트의 사용 가능 시점

- `@ViewChild tree: TreeComponent` 와 같이 `@ViewChild`로 지정한 변수 `tree`는 일반적으로 `ngAfterViewInit` 라이프사이클 이전에 초기화 된다.
- 즉, 일반적인 경우에는 `ngOnChanges()`나 `ngOnInit()` 훅 메서드 내에서는 `tree`가 undefined 이지만, `ngAfterViewInit()` 훅 메서드 내에서 `tree`를 참조하면 undefined가 아닌 제대로 된 값이 들어가있다.


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

## ngx-datatable 3.1.3 사용 시 overflow: hidden 해제하기

테이블 만들 때 [ngx-datatable](https://github.com/swimlane/ngx-datatable) 을 사용하면 아주 편리하다. 그런데 한 가지 골치 아픈 문제가 있는데 아래와 같이 모든 cell에 `overflow-x: hidden`이 먹어서 드랍다운 박스 같은 걸 쓸 수가 없다는 점이다.

![Imgur](https://i.imgur.com/XZvlE34.png)

[공식 문서](https://github.com/swimlane/ngx-datatable/blob/master/demo/basic/css.component.ts)를 참고해서 시도해봤지만, 최종 생성되는 html 파일에서 ngx-datatable이 만들어내는 css 파일이 항상 최하단에 위치하기 때문에 내가 지정한 custom css는 ngx-datatable의 css에 의해 덮어써져서 효과가 없다.

이 문제는 https://github.com/swimlane/ngx-datatable/issues/937 에도 이슈로 올라가 있는데 아직 해결되진 않은 것 같아서 일단 임시스러운 해결책을 올렸다.

https://github.com/swimlane/ngx-datatable/issues/937#issuecomment-484879030

아래와 같이 `ngAfterInit()` 훅을 이용해서 DOM을 직접 수정해서 해결은 했지만 좋은 방법은 아닌 것 같다.

```html
<ngx-datatable-column name="good" cellClass="overflow-visible">
...
</ngx-datatable-column>
```
```typescript
  public ngAfterViewInit() {
    this.cellOverflowVisible();
  }

  private cellOverflowVisible() {
    const cells = document.getElementsByClassName('datatable-body-cell overflow-visible');
    for (let i = 0, len = cells.length; i < len; i++) {
      cells[i].setAttribute('style', 'overflow: visible !important');
    }
  }
```

어쨌든 DOM 직접 수정해서 다음과 같이 제대로 나오게 했다.

![Imgur](https://i.imgur.com/jgnJfp3.png)

## ModalComponent 관련

### Modal 창 띄우기

TODO

### ModalComponent에서 ModalComponent를 호출한 Component의 메서드 호출하기

편의상 ModalComponent를 호출한 Component를 parent라고 하고 ModalComponent를 modal이라고 하자.  
예를 들어 다음과 같이 사용자 목록 화면이 있고, '신규 등록'이나 '수정'을 클릭하면 회원 정보를 편집할 수 있는 Modal창을 띄우는 상황을 생각해보자. 

![Imgur](https://i.imgur.com/drsztKo.png)

이런 UI에서 '신규 등록'이나 '수정' 화면을 별도의 페이지로 가져가지 않고 팝업으로 처리해서 얻는 장점은, 별도의 페이지로 만들면 회원 정보 변경 후에 parent의 검색 결과, 페이지 이동 결과가 유지되지 않지만(유지되게 하려면 여러가지 처리가 필요), 팝업으로 처리하면 간단하게 그대로 유지할 수 있다는 점이다.

하지만 '신규 등록' 후에는 새로 등록한 회원의 정보를 서버에서 받아서 목록에 표시해야하므로, 서버에서 회원 정보를 받아오는 역할을 담당하는 parent의 메서드를 호출해야 한다.  
그런데 parent와 modal은 그냥 BsModalService로 호출될 수 있을 뿐, modal의 컴포넌트를 parent의 html에서 표시하지 않으므로 이벤트로 전달할 수도 없다. 어떻게 하면 modal에서 parent의 메서드를 호출할 수 있을까?

다음과 같이 parent에서 modal을 호출할 때 config에 parent의 메서드를 화살표 함수를 통해 modal에 전달하면,

```typescript
// parent 쪽 코드

  public openUserEditModal(userId: number) {
    const initialState = {
      userId: +userId,
      closeModal: () => {  // 이렇게 화살표 함수로 전달
        this.closeUserEditModal();
      },
      refreshUserList: () => {  // 이렇게 화살표 함수로 전달
        this.initUsers();
      }
    };
    this.bsModalRef = this.modalService.show(
      UserEditModalComponent,
      { initialState, class: 'modal-lg' });
  }

  public closeUserEditModal() {
    this.bsModalRef.hide();
  }
```

modal에서는 다음과 같이 parent의 메서드를 호출할 수 있다.

```typescript
// modal 쪽 코드

  // parent로부터 전달받은 변수
  public userId: string;
  public closeModal;  // parent에서 전달한 화살표 함수
  public refreshUserList;  // parent에서 전달한 화살표 함수

  ...

  this.closeModal();

  ...

  this.refreshUserList();
```

## module '***.module.ts' is not a module

대략 다음과 같은 에러가 날 때가 있다.

```
ERROR in Source file not found: '/Users/XXX/gitRepo/my-app/src/app/member/user-agree/user-agree.component.ts'.
ℹ ｢wdm｣: Failed to compile.
ERROR in src/app/app.module.ts(72,33): error TS2306: File '/Users/XXX/gitRepo/my-app/src/app/member/user-agree/user-agree.module.ts' is not a module.
src/app/dashboard/dashboard.module.ts(9,33): error TS2306: File '/Users/XXX/gitRepo/my-app/src/app/member/user-agree/user-agree.module.ts' is not a module.
```

아버지를 아버지라 부를 수 없고 모듈이 모듈이 아니라는 소린데..  

실제 물리적 위치가 잘못돼 있다면 정정하면 되고, 물리적 위치가 맞다면, 실행 중이던 `npm watch`를 종료하고 재실행하면 에러가 사라진다.


## ReactiveForms, formGroup, formControl 관련

항상도 아니고 가끔 다음과 같은 에러가 발생한다.

`Cannot find control with path: ... -> ...`

![Imgur](https://i.imgur.com/Y6Rj6I9.png)

이유는 정확히 모르지만, 라이프사이클과 관련이 있는 것으로 추정된다.

```typescript
this.targetForm = this.formBuilder.group({
    ...
});
```
위와 같이 form을 초기화 하는 로직을 `ngOnInit()` 훅 내에서 하면 위와 같은 에러가 간혹 발생한다.

form 초기화 로직을 `ngOnInit()`에 앞서 실행되는 `constructor()` 내부에서 실행하면 위와 같은 에러가 발생하지 않는다.

## mat-radio Readonly 처리

radio는 HTML 수준에서 readonly를 지원하지 않는다.

`disable`을 붙이면 변경은 안 되지만, 값이 아예 전달이 안 되므로 의미없는 짓이다.

`onclick="return false;"`로도 가능하지만 조건에 따라 readonly 를 주고 싶을 때는 이 방법으로도 안 된다.

그럴 때는 다른 방법이 없다. 기존 선택값을 보관하고 있다가, 변경이 발생하면 강제로 기존 선택값으로 되돌리는 방법으로 readonly 같은 효과를 낼 수 밖에..

```javascript
oldValue = getInitialOldValueFromServerOrWhatEver();

onChange(event: MatRadioChange, oldValue: someType) {
  if (someCondition) {
    this.yourForm.get('yourRadioGroupFormControlName').patchValue(event.value);
  } else {
    this.yourForm.get('yourRadioGroupFormControlName').patchValue(oldValue);
    // provide some info
    alert('This can not be changed');    
  }
}
```

```html
<mat-radio-group formControlName="yourRadioGroupFormControlName" (change)="onChange($event, oldValue)">
```

StackOverflow에 답을 다 달아보네.. ㅋㅋ https://stackoverflow.com/a/62325965/11747632
