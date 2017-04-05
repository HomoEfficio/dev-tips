# AngularJS의 Select Box 기본 선택 설정

AngularJS에서는 2-way 데이터 바인딩을 통해 모델 데이터를 Select Box의 Option을 비교적 쉽게 설정할 수 있다.

아래의 예는 AngularJS Doc 페이지에서 가져와서 살짝 단순화했다.

- id as name for option in list 소스

위와 같이 설정하면 예상과는 달리 Select Box가 컨트롤러에서 설정한 카푸치노로 기본 선택이 되지 않으며, 비어 있는 option이 자동으로 하나 추가되고 기본으로 선택된다.

비어 있는 option이 추가되지 않게하고, 컨트롤러에서 지정한 선택 값이 기본 선택으로 설정되게 하려면 다음과 같이 `ng-options`에 `track by`를 사용하면 된다.

```javascript
(function(angular) {
  'use strict';
angular.module('defaultValueSelect', [])
  .controller('ExampleController', ['$scope', function($scope) {
    $scope.todayCoffees = [
        {id: '1', name: '카푸치노'},
        {id: '2', name: '헤이즐넛'},
        {id: '3', name: '카페모카'}
    ];

    $scope.selectedCoffee = $scope.todayCoffees[0];
 }]);
})(window.angular);

/*
Copyright 2017 Google Inc. All Rights Reserved.
Use of this source code is governed by an MIT-style license that
can be found in the LICENSE file at http://angular.io/license
*/
```

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Example - example-select-with-default-values-production</title>
  <script src="//code.angularjs.org/snapshot/angular.min.js"></script>
  <script src="app.js"></script>
</head>
<body ng-app="defaultValueSelect">
  <div ng-controller="ExampleController">
  <form name="myForm">
    <label for="mySelect">Make a choice:</label>
    <select name="mySelect" id="mySelect"
      ng-options="option.name for option in todayCoffees track by option.id"
      ng-model="selectedCoffee"></select>
  </form>
  <hr>
  <tt>option = {{selectedCoffee}}</tt><br/>
</div>
</body>
</html>

<!-- 
Copyright 2017 Google Inc. All Rights Reserved.
Use of this source code is governed by an MIT-style license that
can be found in the LICENSE file at http://angular.io/license
-->
```

