# MySQL shell에서 mysql 실행 시 특수문자가 포함된 비밀번호 처리

작업하다보면 shell에서 `mysql -h#.#.#.# -P#### -uXXXX -pYYYY`와 같은 형식으로 mysql 작업을 처리해야할 때가 있다.

## `!`

만약 비밀번호에 `!`가 포함되어 있다면 골때리는 현상이 발생할 수 있다. 아 물론 최신 버전의 MySQL에서는 해결되어 있을 수도 있고, 아래는 무려 MySQL 5.1 기준..

### 터미널

- `mysql -h#.#.#.# -P#### -uXXXX -pYY!YY`로 하면 인증에 실패한다.

#### 해결!!

> **`mysql -h#.#.#.# -P#### -uXXXX -p'YY!YY'`로 해야 인증에 성공한다.**

### shell

- `mysql -h#.#.#.# -P#### -uXXXX -pYY!YY`로 하면 인증에 실패한다.

- `mysql -h#.#.#.# -P#### -uXXXX -p'YY!YY'`로 하면 인증에 실패한다.

- 비밀번호를 변수(`DB_PASSWORD`)에 담아서 `mysql -h#.#.#.# -P#### -uXXXX -p${DB_PASSWORD}`로 하면 인증에 실패한다.

- 비밀번호를 변수(`DB_PASSWORD`)에 담아서 `mysql -h#.#.#.# -P#### -uXXXX -p'${DB_PASSWORD}'`로 하면 인증에 실패한다.

- 비밀번호를 변수(`DB_PASSWORD`)에 담아서 `mysql -h#.#.#.# -P#### -uXXXX '-p${DB_PASSWORD}'`로 하면 인증에 실패한다.

#### 해결!!

> **`dummy=$(mysql -h#.#.#.# -P#### -uXXXX -pYY!YY)`로 해야 인증에 성공한다.**
    
> **비밀번호를 변수(`DB_PASSWORD`)에 담아서 `dummy=$(mysql -h#.#.#.# -P#### -uXXXX -p${DB_PASSWORD})`로 해야 인증에 성공한다.**


## 기타 특수 문자

https://dba.stackexchange.com/questions/48778/mysqladmin-not-taking-inline-password 를 참고 한다.
