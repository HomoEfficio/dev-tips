# 리눅스 - foreground로 실행 중인 프로세스를 background로 보내기

가끔 얼마 안 걸릴 줄 알고 실행했는데 예상 외로 오래 걸릴 때가 있다. 게다가 remote 터미널이라면 무심코 신경 쓰지 않다가 연결 끊어지면서 실행했던 명령도 중단되는 불상사가..

이미 foreground로 실행 중인 프로세스를 background로 보내는 방법과 예시는 다음과 같다.


1. 해당 명령을 foreground로 실행하고 있는 터미널에서 CTRL+Z

    ```
    [homo.efficio@heaven] ~
    🍺  $ mysql -p --default-character-set=utf8 -P3306 -hDB_HOST -uUSER DB_NAME < MYSQLDUMPFILE 2>> err.log
    Enter password:
    ^Z
    [1]+  Stopped                 mysql -p --default-character-set=utf8 -P3306 -hDB_HOST -uUSER DB_NAME < MYSQLDUMPFILE 2>> err.log
    ```

1. 터미널에 `bg` 입력

    ```
    [homo.efficio@heaven] ~
    🍺  $ bg
    [1]+ mysql -p --default-character-set=utf8 -P3306 -hDB_HOST -uUSER DB_NAME < MYSQLDUMPFILE 2>> err.log &
    [homo.efficio@heaven] ~
    ```

여기까지만 해줘도 백그라운드로 돌려지지만, `disown`으로 아예 `jobs` 리스트에서 빼둘 수도 있다.

이 방법은 `mysql -p` 명령을 수행할 때처럼, 명령 실행하면 그냥 끝까지 실행되는게 아니라 사용자가 비밀번호를 입력해줘야 실행이 되는 상황에서 특히 유용하다. `mysql -p 옵션들 &`로 실행하면 비밀번호를 입력받지 못하기 때문에 실제로는 실행이 되지 않는다.

이럴 때 위의 사례와 같이 `CTRL+Z`와 `bg` 콤보를 쓰면 마음 편히 백그라운드에서 실행시키고 퇴근할 수 있다.
