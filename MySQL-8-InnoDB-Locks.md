# MySQL 8 InnoDB Locks

- from https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html + AI help(https://g.co/gemini/share/060ac63490d7)
- 문서에는 분류 기준 없이 아래 8개(항목은 8개지만 Lock은 9개) Lock 이 나열돼 있음
  - Shared and Exclusive Locks
  - Intention Locks
  - Record Locks
  - Gap Locks
  - Next-Key Locks
  - Insert Intention Locks
  - AUTO-INC Locks
  - Predicate Locks for Spatial Indexes

## 배경 지식

- session
  - 클라이언트가 DB에 접속할 때부터 이탈할 때까지의 구간
  - 하나의 세션에 여러 tx가 시작/종료될 수 있다.
- transaction(tx)
  - ACID 보장 단위
  - tx마다 서로 다른 격리 수준(isolation)을 가질 수 있다.
  - 모든 DML은 tx 안에서 실행된다.
    - **명시적으로 `start transaction`을 하지 않아도 DB에 의해 묵시적으로 tx가 시작된다.**

## Shared and Excluisve Locks

- InnoDB는 두 가지 타입의 레코드 수준 락 제공
- Shared Lock(S Lock)은 공유되는 락이다. 락이 공유된다는 게 이상하지만 데이터 변경이 없는 읽기인 경우 공유되어도 데이터 정합성이 깨지지 않는다.
- Exclusive Lock(X Lock)은 배타적 락이다. 개념적으로 익숙한 락이며 데이터 변경이 있는 Update, Delete 에 사용된다.
- 동작
  >tx T1이 r행에 대한 s lock을 가지고 있는 상태에서
  >- T2가 r행에 대한 s lock을 요청하면 T1이 가지고 있던 s lock이 T2에도 즉시 부여되어 공유된다.
  >- T2가 r행에 대한 x lock을 요청하면 즉시 부여될 수 없다.(즉시는 아니지만 부여는 될 수 있다는 소린가?? -> 걍 T1의 s lock이 해제된 후에 부여된다는 얘기)
  >
  >tx T1이 r행에 대한 x lock을 가지고 있는 상태에서는
  >- T2가 r행에 대해 s lock을 요청하든 x lock을 요청하든 즉시 부여될 수 없고, T2는 T1이 x lock을 해제할 때까지 기다려야 한다.
- chatGPT: where 조건에 의해 3개의 행이 조회된다면, 3개의 행 각각에 1개씩 총 3개의 s lock이 생기나?
  - 격리 수준이 `REPEATABLE READ`이나 `SERIALIZABLE`이면 그렇다.
  - 격리 수준이 `READ COMMITEED`이면 아무 lock도 생셩되지 않지만, 쿼리에 `FOR SHARE`나 `FOR UPDATE`가 포함되면 3개의 행 각각에 1개씩 총 3개의 x lock이 생긴다
  - https://chatgpt.com/share/66f959dc-5508-8011-9e66-9d6ba83ca654

## Intention Locks

- **테이블 수준 락을 걸 때 방해가 되는 레코드 수준 락이 있는지 효율적으로 확인하기 위해 사용**하는, 일종의 마커나 플래그와 같은 **개념적인** 테이블 수준의 락
  - intention lock 은 테이블 수준에서 여러 개가 동시에 설정될 수 있다. 
  - 그래서 `일종의 마커나 플래그와 같은 개념적인` 테이블 수준의 락이라고 볼 수 있다.
- intention lock 이 없으면 `alter table` 이나 `create index`, `analyze table` 처럼 테이블 수준 락이 필요할 때 방해가 되는 레코드 수준 락이 있는지 모든 레코드에 대해 확인해야 하므로 매우 비효율적이다.
- intention shared lock(IS lock) 과 intention exclusive lock(IX lock) 이렇게 두 가지 intention lock 이 있다.
- `select ... for share` 는 테이블에 IS lock 을 설정하고, `select ... for update` 는 테이블에 IX lock 을 설정한다.
- 참고: https://g.co/gemini/share/70c7700e8a80
  ```
  Think of a library with many books (rows) on shelves (tables).

  Row-level lock (S/X on a book): You want to read a specific book (S lock) or write notes in it (X lock).
  Table-level lock (LOCK TABLES WRITE on a shelf): The librarian wants to move the entire shelf to a different room.
  Intention lock (IS/IX on a shelf):
  When you walk into the library and head towards a shelf with the intention of picking up a book to read, you're essentially setting an IS lock on that shelf.
  If someone else wants to rearrange all the books on that shelf (requiring a full shelf lock), they'd first check if anyone has an "intention to read/write" flag on it. If they see your IS flag, they know they can't just move the shelf yet.
  Multiple people can have an "intention to read" (IS) on the same shelf, and they can all go pick up different books to read concurrently.
  If someone has an "intention to write" (IX) on the shelf, it means they might be changing some books. Others can still intend to read from that shelf (IS), but a full shelf move (table X lock) would be blocked.
  ```

## Record Locks

- 말 그대로 레코드 수준에 설정하는 lock. 다만 **인덱스 레코드에 설정**된다.
- `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;`가 실행되면 `c1 = 10`인 모든 레코드(실제로는 이 레코드의 인덱스 레코드)에 lock이 설정되고, lock이 해제되기 전까지는 `c1 = 10`인 레코드를 insert, update, delete 할 수 없다.


## Gap Locks

- 동일 Tx 내에서 동일 쿼리에 대한 결과가 다르게 나오는 **Phantom Read 를 방지하기 위해 사용하는 락**
- insert 에 대해서만 락으로 동작하며, 일정 구간에 존재하는 여러 Gap 에 모두 Lock 이 설정되며, 여러 tx에 의해 중첩 설정될 수도 있다.
- Tx1 에서 `SELECT c1 FROM t WHERE c1 BETWEEN 100 AND 200 FOR UPDATE`가 실행되면, 100과 실제 데이터, 200 사이에 있는 모든 구간에 대해 Gap Lock이 설정된다.
  - 예를 들어 실제 c1 = 100, c1 = 120, c1 = 150, c1 = 170 인 레코드가 있다면
  - 100, 120, 150, 170 인 레코드에 대해 Record Lock이 설정되고,
  - (100, 120), (120, 150), (150, 170), (170, 200] 인 구간에 Gap Lock이 설정된다.
- Tx2 에서 
  - `INSERT INTO t (c1) VALUES (110)`을 실행하면 110은 (100, 120)에 설정된 Gap Lock에 막혀 insert 되지 못한다.
  - `INSERT INTO t (c1) VALUES (80)`을 실행하면 80은 Gap Lock에 막혀있지 않으므로 insert 된다. 80인 데이터는 `BETWEEN 100 AND 200`에 해당되지 않으므로 insert 되더라도 Phantom Read가 발생하지 않는다.

### Gap Locks 이 필요한 이유

- 언뜻 생각하기에 MVCC 만으로도 Phantom Read를 막을 수 있을 것 같지만, **MVCC 의 snapshot 은 쿼리 결과에 대한 스냅샷이 아니라, 쿼리 실행 시점의 데이터에 대한 스냅샷이다.**
- 그리고 동일한 쿼리를 다시 실행하면 실제 쿼리문을 수행하지만, 쿼리문 조건에 해당되는 데이터가 snapshot 에 있다면 snapshot 의 데이터를 사용한다.
  - 즉 다른 Tx에서 commit 된 update, delete 되었더라도 snapshot 의 데이터가 사용된다
- 따라서 이미 존재했던 데이터가 다른 Tx에서 변경되더라도 MVCC 가 Phantom Read를 막아줄 수 있지만,
- 존재하지 않던 데이터가 insert 되어 나타나는 Phantom Read 는 MVCC 가 막아줄 수 없다.
- Gap Locks이 insert 에 의한 Phantom Read를 막아주며, Gap Locks이 insert에 대해서만 락으로 작동하는 이유이기도 하다.

### Phantom Read 와 Transaction Isolation Level

- SERIALIZABLE 에서는 여러 Tx가 동시에 레코드에 접근하는 것 자체가 원천적으로 불가능하므로 Phantom Read 가 발생할 수 없다.
- REPEATABLE READ 에서는 위에 설명한 것처럼 MVCC 가 update, delete 에 의한 Phantom Read 를 막아주고 Gap Lock 이 insert 에 의한 Phantom Read 를 막아준다.
- READ COMMITTED 에서는 Phantom Read 가 발생하지만, READ COMMITTED 라는 이름 자체에서 알 수 있듯이 다른 Tx에서 커밋된 내용에 의한 Phantom Read 는 이상 현상이 아니라 정상 현상이다.
- READ UNCOMMITED 에서는 다른 Tx에서 커밋되지 않은 변경도 조회되므로 Phantom Read 역시 이상 현상이 아니라 정상 현상이다.


## Next-Key Locks

- Next-Key Lock = Gap Lock + Index Record Lock
- Gap Lock은 실제 존재하는 데이터의 사이 구간에 대한 락이고, Next-Key Lock 은 Gap 의 종료 지점 바로 다음에 실제 존재하는 데이터(Next-Key)에 대한 Index Record Lock 까지 설정한다.
  - 즉 데이터가 7, 13, 18, 20 이 있다고 할 때 조회 범위 [10, 20]에 대해
    - Gap Lock: `[10, 13)`, `(13, 18)`, `(18, 20)`
    - Next-Key Lock: `[10, 13]`, `(13, 18]`, `(18, 20]`
- **Next-Key Lock 도 Gap Lock 과 마찬가지로 Phantom Read 방지가 목적**이며, Gap Lock 과 Gap 종료지점 바로 다음 데이터(Next-Key)에 대한 Index Record Lock 을 설정하므로 **REPEATABLE READ 에서는 사실 상 Next-Key Lock 이 Phantom Read를 방지**한다고 볼 수 있고, 그래서 **REPEATABLE READ 에서는 Next-Key Lock 이 기본으로 동작**한다.


## Insertion Intention Locks

- insert 하려는 데이터가 next-key lock 이 지정된 구간에 있다면, insert 할 수 없고 대신에 Insertion Intention Lock 을 획득하고 이미 존재하는 next-key lock 해제까지 기다린다.
- Insert Intention Lock 도 Intention Lock 의 일종이라서 insert를 보장한디기 보다는 insert 하려는 의도가 존재한다는 일종의 개념적인 마커 역할과 비슷하다.
- next-key lock 에 의해 잠겨있던 gap 이 해제될 때, 해당 gap 에 insert 할 의도가 있다고 마킹된 insert 연산들에 대해 insert 가 진행된다.
  - insert 되려는 데이터들의 key값들이 B-tree index 상에서 동일한 page 에 있다면 순차적으로 insert 돼야 하고, 다른 page 에 있다면 동시에 insert 될 수도 있다. 따라서 일종의 Queue 역할을 한다고 볼 수도 있다.
- Insertion Intention Lock은 table 수준도, row 수준도 아닌, 하나의 Gap 에 대해 설정되는 gap-level lock 이다.


## AUTO-INC Locks

- AUTO_INCREMENT 가 지정된 컬럼이 있는 데이터는 해당 컬럼 값이 순차적으로 증가하도록 보장해야하며 이 때 사용되는 table-level lock


## Predicate Locks for Spatial Indexes

- SPATIAL 인덱스(R-tree 인덱스 사용)가 있는 테이블의 격리 수준 보장을 위해 InnoDB 는 Predicate Lock 을 사용한다.
  - 일반적인 데이터는 B-tree 인덱스를 사용하므로 정렬 개념이 있어서 gap lock, next-key lock 을 사용할 수 있지만,
  - 공간 데이터 같은 다차원 데이터는 R-tree 인덱스를 사용하며 절대적인 정렬이라는 개념이 존재하지 않아서 Next 라는 것을 정의할 수 없으므로, Next-Key Lock을 사용할 수 없다.
  - 따라서 값 기준으로 Lock 을 지정하는 것이 아니라 조건(Predicate) 기준으로 Lock 을 지정한다.
- 사례
  - Tx1
    ```sql
    SELECT COUNT(*)
    FROM Trees
    WHERE MBRContains(ST_GeomFromText('POLYGON((10 10, 20 10, 20 20, 10 20, 10 10))'), location)
    FOR UPDATE;
    ```
  - Tx2
    - `POINT(18 12)`가 위 `POLYGON((10 10, 20 10, 20 20, 10 20, 10 10))`에 해당되므로 insert 는 Tx1 에서 설정한 Predicate Lock 이 해제될 때까지 대기
    ```sql
    INSERT INTO Trees (name, location)
    VALUES ('New Sapling', ST_GeomFromText('POINT(18 12)'));
    ```


