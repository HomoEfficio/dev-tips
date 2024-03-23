# MongoDB collection copy

- mongoexport, mongoimport 를 파이프(`|`)로 조합해서 복사할 수 있다.

```
mongoexport -u 계정이름 -p 비밀번호 \
--authenticationDatabase admin \
-h='소스 DB host 이름:포트, 쉼표 구분 복수 가능' \
-d=소스DB이름 -c=소스컬렉션이름 \
| mongoimport -u 계정이름 -p 비밀번호 \
--authenticationDatabase admin \
-h='타겟 DB host 이름:포트, 쉼표 구분 복수 가능' \
-d=타겟DB이름 -c=타겟컬렉션이름
```

