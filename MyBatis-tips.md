# MyBatis 자잘한 팁 모음

## `#{}` 와 `${}` 의 차이

- `#{}` 은 값에 맞게 형 변환(문자열 값이면 따옴표로 감싸주기)을 해서 쿼리에 넣어준다.
- `${}` 은 형 변환 없이 값 그대로 쿼리에 넣어준다.
- 그래서 다음과 같이 값이 들어갈 자리에는 `#{}` 를 사용해야 하고, 컬럼명 등 쿼리문의 요소를 대체할 때는 `${}` 를 사용해야 한다.
  ```xml
      <select id="selectCursorPagedWithPoint" parameterType="map" resultMap="xxxWithPointMap">
          /* YOUR_REPOSITORY_FULL_PATH.METHOD_NAME */
          SELECT xpoint.top_point, xpoint.new_point, x.*
          FROM (
              SELECT * FROM xxx_point
              WHERE created = #{pointLastCreated}
              <if test="pointCursor != null and oidCursor != null">
                  AND (${orderColumn} &lt; #{pointCursor} OR (${orderColumn} = #{pointCursor} AND xxx_oid &lt; #{oidCursor} ))
              </if>
              ORDER BY ${orderColumn} DESC
              LIMIT #{limit}
          ) xpoint
          STRAIGHT_JOIN (
              SELECT * FROM xxx
              WHERE status = #{status.code}
          ) x on xpoint.xxx_oid = x.oid
      </select>
  ```
- 이걸 모르고 잘못 사용하면(ex: 컬럼명 자리에 `${}`가 아니라 `#{}` 사용), 로그에 찍힌 쿼리를 애플리케이션이 실행한 결과와 DB Client에서 실행한 결과가 서로 다르게 나오는 아찔한 경험을 할 수 있다.

## `<where>`

- 동적 쿼리를 작성하다보면 다음과 같이 `AND`로 시작하는 조건절을 동적으로 붙이게 되는데, 첫 조건이 `AND`로 시작하면 다음과 같이 쿼리 문법 오류가 발생한다.
  ```sql
  SELECT * FROM A
  WHERE AND col1 = val1
  ```
- 이를 방지하기 위해 다음과 같이 하는 것보다는
  ```sql
  SELECT * FROM A
  WHERE 1 = 1
  AND col1 = val1
  ```
- 다음과 같이 `<where>`를 사용하면 MyBatis 가 알아서 `AND` 처리를 해준다.
  ```xml
  SELECT * FROM A
  <where>
    <if test="val1 != null">
    AND col1 = val1  <!-- 첫 조건은 AND 를 제외해준다 -->
    </if>
    <if test="val2 != null">
    AND col2 = val2
    </if>
  </where>
  ```

## JOIN과 `<association>`

- JOIN 한 결과 여러 테이블의 컬럼을 포함하는 레코드를 매핑하려면 어떻게 해야할까?
- 먼저 다음과 같이 Book, Author, Publisher 세 엔티티가 있다고 하자.

  ```java
  public class Book {
  
      private String oid;
      private String title;
      private String authorOid;
      private String publisherOid;
  }
  
  public class Author {
  
      private String oid;
      private String name;
  }
  
  public class Publisher {
  
      private String oid;
      private String name;
  }
  ```

- 이에 대한 Mapper 는 각각 다음과 같다.

  ```xml
  <resultMap id="bookMap" type="a.b.c.domain.Book">
    <id property="oid" column="oid"/>
    <result property="title" column="title"/>
    <result property="authorOid" column="author_oid"/>
    <result property="publisherOid" column="publisher_oid"/>
  </resultMap>
  
  <resultMap id="authorMap" type="a.b.c.domain.Author">
    <id property="oid" column="oid"/>
    <result property="name" column="name"/>
  </resultMap>
  
  <resultMap id="publisherMap" type="a.b.c.domain.Publisher">
    <id property="oid" column="oid"/>
    <result property="name" column="name"/>
  </resultMap>
  ```

- 세 테이블을 조인해서 다음과 같이 조회한다면,

  ```
  select b.oid, b.title, b.author_oid, a.name as author_name, b.publisher_oid, p.name as publisher_name
  from book b
  inner join author a on b.author_oid = a.oid
  inner join publisher p on b.publisher_oid = p.oid
  ```

- 이에 대한 엔티티와 Mapper 는 다음과 같이 만들면 된다.

  ```java
  public class BookWithInfo {
  
      private String oid;
      private String title;
      private String authorOid;
      private String authorName;
      private String publisherOid;
      private String publisherName;
  }
  ```
  
  ```xml
  <resultMap id="bookWithInfoMap" type="a.b.c.domain.BookWithInfo">
    <id property="oid" column="oid"/>
    <result property="title" column="title"/>
    <result property="authorOid" column="author_oid"/>
    <result property="authorName" column="author_name"/>
    <result property="publisherOid" column="publisher_oid"/>
    <result property="publisherName" column="publisher_name"/>
  </resultMap>
  ```

- 그리 어렵지 않고 간단하다. 하지만 현실의 엔티티는 훨씬 더 많은 컬럼을 가지고 있다. 조인할 때마다 저렇게 개별 엔티티의 컬럼을 모두 나열하는 방식은 지겹고 누락에 의한 에러도 발생하기 쉽다.
- 이럴 때 `<assotiation>`을 사용해보자. 이미 존재하는 엔티티와 Mapper를 재사용 할 수 있어서 작업량과 에러 위험을 모두 줄여준다.

  ```java
  public class BookWithInfo {
  
      private Book book
      private String authorName;
      private String publisherName;
  }
  ```
  
  ```xml
  <resultMap id="bookWithInfoMap" type="a.b.c.domain.BookWithInfo">
    <id property="oid" column="oid"/>
    <result property="authorName" column="author_name"/>
    <result property="publisherName" column="publisher_name"/>
    <association property="book" resultMap="bookMap"/>  <!-- MyBatis xml 규약 상 association 은 result 다음에 와야 한다 -->
  </resultMap>
  ```
  
  ```xml
  <select id="myQuery" parameterType="map" resultMap="bookWithInfoMap">
  select b.oid, b.title, b.author_oid, b.publisher_oid, a.name as author_name, p.name as publisher_name
  from book b
  inner join author a on b.author_oid = a.oid
  inner join publisher p on b.publisher_oid = p.oid
  </select>
  ```

- 조인하는 테이블의 컬럼명이 겹치지 않는다면 아예 이래와 같이 할 수도 있다.
  - 하지만 예제에서는 Author.name, Publisher.name 이 겹치므로 아래와 같이 할 수 없다.

    ```java
    public class BookWithInfo {
    
        private Book book
        private Author author;
        private Publisher publisher;
    }
    ```
    
    ```xml
    <resultMap id="bookWithInfoMap" type="a.b.c.domain.BookWithInfo">
      <id property="oid" column="oid"/>
      <association property="book" resultMap="bookMap"/>
      <association property="author" resultMap="authorMap"/>
      <association property="publisher" resultMap="publisherMap"/>
    </resultMap>
    ```
