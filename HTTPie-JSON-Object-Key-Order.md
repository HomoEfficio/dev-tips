# HTTPie JSON Object Key 순서 관련 이슈

httpie는 서버에서 반환해주는 Object Key 순서를 존중하지 않고 마음대로 순서를 바꿔서 반환한다.

`--unsorted` 옵션을 붙여주면 서버에서 반환해주는 순서가 유지된다.

https://httpie.io/docs/cli/format-options

