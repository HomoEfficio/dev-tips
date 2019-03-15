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

앵귤라는 일반적으로 다음의 네 가지 파일이 하나의 컴포넌트를 구성한다.

- `컴포넌트이름.component.ts`: 모델 및 화면 로직
- `컴포넌트이름.html`: 템플릿
- `컴포넌트이름.module.ts`: 모듈 단위 export 정보
- `컴포넌트이름.scss`: 스타일

컴포넌트와 템플릿의 데이터 및 이벤트 교환은 다음과 같다.

> 프로퍼티 바인딩으로 `component.ts`에 있는 모델 데이터를 템플릿에 전달
>
> 이벤트 바인딩으로 `html` 템플릿의 이벤트를 `component.ts`에 전달


# 모듈

- 다른 모듈에 import되어 재사용 될 수 있는 단위
- 컴포넌트는 모듈에 포함되고, 해당 모듈이 import 되어야만 모듈 내의 컴포넌트도 import 되어 사용될 수 있다.

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
- providers: 본 모듈에서 사용할 외부 서비스 리스트

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

