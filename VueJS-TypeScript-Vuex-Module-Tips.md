# VueJS TypeScript Vuex Module 이슈 모음

## module 은 getModule() 로 가져와야..


## getModule() 로 가져오려면 라우터에서 컴포넌트를 lazy 방식으로 가져와야..


## 컴포넌트들이 로딩되기 전에 외부에서 데이터를 가져오려면..

main.ts 생성 시 다음과 같이 Vuex Root Store의 액션을 await 호출하는 async created() 를 추가한다.

```typescript
// main.ts
Vue.config.productionTip = false;

new Vue({
  router,
  store,
  i18n,
  render: (h) => h(App),
  async created() {
    await this.$store.dispatch('fetchAllRecruitParts');
  },
}).$mount('#app');
```

컴포넌트에서는 `this.$store.state.<<STATE_NAME>>` 로 읽어서 사용 가능

이렇게 `main.ts의 created() + Root Store`로 하지 않고 `개별 컴포넌트의 mouted() 등 + 개별 Vues 모듈`로 로딩하면 컴포넌트 로딩/렌더링과 데이터 로딩 타이밍이 겹쳐서 제대로 사용할 수 없게됨

## State 변수는 **반드시** null 로라도 초기화해야 한다.

```typescript
@Module({
  namespaced: true,
  name: 'overview',
})
export default class ModuleOverview extends VuexModule {

  // 이렇게 하면 this.context 아래에 _field1 이 생기지도 않으며, 따라서 나중에 @Mutation에서 _field1 에 값을 할당해도 외부 컴포넌트에서 _field1 을 참조하면 계속 undefined 를 돌려받는다.
  private _field1?: MyType1;
  
  // 이렇게 null 로라도 초기화를 해줘야 this.context 아래에 _field1 이 생기고, 나중에 @Mutation 에서 할당한 값이 그대로 외부 컴포넌트에도 전달된다.
  private _field1?: MyType1|null = null;
```
