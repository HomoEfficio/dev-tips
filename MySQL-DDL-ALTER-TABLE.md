# MySQL ALTER TABLE

MySQL(8 기준)에서 테이블 구조를 변경하는 ALTER TABLE 관련 알아야 할 내용 정리

## 개요

테이블 이름 변경, 컬럼 추가/삭제/이름변경/타입변경/순서변경, 인덱스 추가/삭제/이름변경 등 테이블 구조 변경 작업은 데이터 사용에 영향을 거의 미치지 않는 작업도 있고 심대한 영향을 미치는 작업도 있다. 따라서 ALTER TABLE 을 사용할 때는 이를 잘 알아보고 수행해야 운영에 문제를 일으키지 않는다.

ALTER TABLE 도 DDL의 일부인데, **DDL에 의한 운영 영향은 알고리듬(algorithm)과 락(lock) 두 가지 개념을 통해 알 수 있다.**


## Algorithm

아래와 같이 세 가지 알고리듬을 통해 DDL 작업 방식이 결정된다.

- COPY: 테이블 데이터 복사 필요. 오랜 시간이 걸리며 복사하는 도중 발생한 데이터 변경은 대기하고 있다가 복사 완료 및 구조 변경 적용 완료 후에 적용됨
- INPLACE: 테이블 데이터 복사는 발생하지 않지만 테이블 rebuild가 발생할 수도 있다. 준비 단계 및 실행 단계에서 테이블 메타데이터에 대해 순간적인 메타데이터 락이 걸릴 수 있다. 특별한 상황이 아닌 한 작업 도중 데이터 변경 가능
- INSTANT: 데이터 딕셔터리에 있는 메타데이터만 수정. 준비 단계, 실행 단계에서 아무런 락도 발생하지 않고 테이블 데이터에 미치는 영향도 없어서 작업이 금방 수행되며 작업 도중 데이터 변경 허용(8.0.12에 도입)

쉽게 말해 INSTANT를 지원하는 작업은 운영에 영향을 미치지 않고, INPLACE를 지원하는 작업은 운영에 거의 영향을 미치지 않고, COPY만 지원하는 작업은 운영에 심대한 영향을 미친다.

데이터 복사가 발생하지 않아 운영에 미치는 영향이 적은 INSTANT와 INPLACE를 지원하는 DDL 작업을 Online DDL이라고 부른다.

DDL 작업 별로 지원되는 알고리듬이 정해져 있는데 이건 조금 이따 다룬다.


## Lock

아래와 같이 네 가지 락을 통해 DDL 도중 발생하는 동시성을 제어한다.

- DEFAULT: 알고리듬이 지원하는 가장 관대한 동시성 허용
- NONE: 알고리듬에서 지원한다면 동시 읽기/쓰기 모두 허용. 알고리듬이 동시 읽기/쓰기 모두 허용하지 않는데 NONE을 지정하면 DDL 실행 전 에러 발생
- SHARED: 알고리듬에서 지원한다면 동시 읽기는 허용하지만 동시 쓰기는 불허. 즉 알고리듬에서 동시 쓰기를 지원하더라도 명시적으로 동시 쓰기를 허용하지 않고 싶을 때 사용. 알고리듬이 동시 읽기를 허용하지 않는데 SHARED를 지정하면 DDL 실행 전 에러 발생
- EXCLUSIVE: 알고리듬에서 지원하더라도 동시 읽기/쓰기 모두 명시적으로 허용하지 않고 싶을 때 사용


## 실제 사용

DDL 작업별로 알고리듬이 정해져 있고, 알고리듬에 따라 사용 가능한 Lock도 정해지므로, 결국 **어느 작업에 어떤 알고리듬이 적용되는지 알아야 한다**.

관련 내용은 https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html 여기에 잘 정리돼 있으니 반드시 확인해보자.


## 미리 확인

ALTER TABLE을 포함한 DDL 작업은 운영에 영향을 미칠 수 있으므로 다음과 같이 임시 테이블에서 미리 확인해보는 것이 좋다.

>1. 테이블 구조를 복제해서 임시 테이블을 만들고
>2. 임시 테이블에 소량의 데이터만 집어넣고
>3. 알고리듬과 락을 명시해서 DDL 수행

예를 들어 사용 안 하는 컬럼 삭제가 운영에 영향 없이 가능한지 알아보려면 1, 2를 한 후에 다음 명령을 실행한다.

```
alter table tmptmp
  drop column unused_col,
  algorithm = INSTANT
;
```

실행하면 다음과 같은 에러 메시지를 볼 수 있으며, drop column 은 INSTANT를 지원하지 않는다는 사실을 알 수 있다.

```
[0A000][1845] ALGORITHM=INSTANT is not supported for this operation. Try ALGORITHM=COPY/INPLACE.
```

INPLACE 로 지정해서 실행하면 성공한다.
```
alter table tmptmp
  drop column unused_col,
  algorithm = INPLACE
;
```

컬럼 데이터 타입 변경은 어떨까? bigint를 varchar(255)로 INSTANT 알고리듬으로 변경해보면,

```
alter table tmptmp
    modify col_big_int varchar(255) null,
    algorithm = INSTANT
;

[0A000][1846] ALGORITHM=INSTANT is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY/INPLACE.
```

컬럼 타입 변경은 INPLACE로도 안 되는데 INSTANT를 썼으니 안 된다는 얘기다. 그러면 COPY로만 가능하다는 얘기 같은데 마지막에는 애매하게 ALGORITHM=COPY/INPLACE 로 해보라고 한다. 무슨 뜻일까?

실험해보니 varchar(255) 였던 컬럼을 varchar(1000) 으로 크기를 늘릴 때는 INPLACE 도 성공한다.

하지만 varchar(1000) 였던 컬럼을 varchar(255) 로 줄일 때는 다음과 같이 COPY로만 가능하다는 에러가 메시지가 표시된다. 아마 varchar 크기를 줄일 때는 기존에 저장된 값이 축소된 컬럼 길이에 맞게 잘릴 수도 있고, 이 경우 값이 변경돼야 하므로 COPY로만 가능한 것으로 보인다.

```
[0A000][1846] ALGORITHM=INPLACE is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

에러 메시지뿐 아니라 성공 메시지에서도 여러 정보를 미리 알아낼 수 있다.

예를 들어 다음과 같은 성공 메시지가 나오면,

```
Query OK, 0 rows affected (0.07 sec)
```

`0 rows affected` 에서 데이터 복사가 발생하지 않음을 알 수 있고, `0.07 sec`라는 짧은 수행시간에서 테이블 재구성(rebuild)도 발생하지 않음을 알 수 있다. 운영에 거의 영향을 미치지 않을 것임을 알 수 있다.

다음과 같은 성공 메시지가 나오면,

```
Query OK, 0 rows affected (21.42 sec)
```

데이터 복사는 발생하지 않지만 `21.42 sec`라는 긴 수행시간은 테이블 재구성(rebuild)가 발생한다는 사실을 알 수 있다. 테이블 재구성이 발생하더라도 해당 작업에서 지원하는 락에 따라 재구성 동안에도 해당 테이블을 사용할 수도 있다.

다음과 같은 성공 메시지가 나오면, 

```
Query OK, 1671168 rows affected (1 min 35.54 sec)
```

데이터 복사가 발생한다는 사실을 알 수 있다.

따라서 이런 동작을 미리 알아두고 상황에 맞게 계획을 수립하고 진행하는 것이 좋다.


## 참고

- https://dev.mysql.com/doc/refman/8.0/en/alter-table.html
- https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html
- https://severalnines.com/database-blog/alter-table-mysql-friend-or-foe 여기에도 마이그레이션 관련 좋은 팁이 설명돼있다.


