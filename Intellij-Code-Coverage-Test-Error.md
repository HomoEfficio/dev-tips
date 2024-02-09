# Intellij Code Coverage Test Error

코드 커버리지 확인을 위해 `Run XXX with Coverage` 명령으로 테스트 실행 시 다음과 같은 오류가 발생하기도 한다.

```
Caused by: java.lang.IllegalArgumentException: Attempt to get [I field "org.hibernate.hql.internal.antlr.HqlTokenTypes.__$hits$__" with illegal data type conversion to int
    at java.base/jdk.internal.reflect.UnsafeFieldAccessorImpl.newGetIllegalArgumentException(UnsafeFieldAccessorImpl.java:69)
    at java.base/jdk.internal.reflect.UnsafeFieldAccessorImpl.newGetIntIllegalArgumentException(UnsafeFieldAccessorImpl.java:132)
    at java.base/jdk.internal.reflect.UnsafeQualifiedStaticObjectFieldAccessorImpl.getInt(UnsafeQualifiedStaticObjectFieldAccessorImpl.java:58)
    at java.base/java.lang.reflect.Field.getInt(Field.java:601)
    at org.hibernate.hql.internal.ast.util.ASTUtil.generateTokenNameCache(ASTUtil.java:404)
    at org.hibernate.hql.internal.ast.util.ASTPrinter.<init>(ASTPrinter.java:44)
```

주로 JPA Repository 조회 메서드를 실행할 때 발생하는 것으로 보인다.

## 원인

HqlTokenTypes 클래스에는 대략 아래와 같이 HQL 관련 모두 int 타입인 키워드가 포함돼 있는데,

```java
public interface HqlTokenTypes {
    int EOF = 1;
    int NULL_TREE_LOOKAHEAD = 3;
    int ALL = 4;
    int ANY = 5;
    int AND = 6;
    // 중략
    int HEX_DIGIT = 138;
    int EXPONENT = 139;
    int FLOAT_SUFFIX = 140;
}
```

코드 커버리지 테스트를 실행하면 위 에러 메시지에 나온 것처럼 원래에는 없던 `int[]` 타입의 `__$hits$__` 필드가 코드 커버리지를 위해 필요한 [instrument 과정에서 추가](https://github.com/search?q=repo%3AJetBrains%2Fintellij-coverage%20__%24hits%24__&type=code)되고,
테스트 실행 과정에서 하이버네이트 로깅에 사용되는 것으로 보이는 `ASTUtil.generateTokenNameCache()` 함수 내에서 `int[]` 타입의 `__$hits$__` 필드에 대해 `getInt()`를 호출하다가 에러가 발생한다.

```java
    public static String[] generateTokenNameCache(Class tokenTypeInterface) {
        final Field[] fields = tokenTypeInterface.getFields();
        //We try to guess the right size from what we know ANTLR will do;
        //"guessing" is safe as the three interfaces this is used on are static,
        //and this is all run at boot so at worst would fail fast.
        final String[] names = new String[ fields.length + 2 ];
        for ( final Field field : fields ) {
            if ( Modifier.isStatic( field.getModifiers() ) ) {
                int idx = 0;
                try {
                    idx = field.getInt( null );  // 여기에서 field 값이 `__$hits$__` 일 때 타입이 `int[]` 인데 `getInt()`를 호출하면서 에러 발생
                }
                catch (IllegalAccessException e) {
                    throw new HibernateError( "Initialization error", e );
                }
                String fieldName = field.getName();
                names[idx] = fieldName;
            }
        }
        return names;
    }
```

관련해서 이미 YouTrack에 두 가지 이슈가 올라와 있다.

- [org.hibernate.hql.internal.antlr.HqlTokenTypes Error when running IntelliJ Coverage](https://youtrack.jetbrains.com/issue/IDEA-287458)
  - 댓글에 `org.hibernate.hql.internal.antlr.HqlTokenTypes, org.hibernate.hql.internal.antlr.SqlTokenTypes, org.hibernate.sql.ordering.antlr.GeneratedOrderByFragmentRendererTokenTypes` 네 가지 클래스를 exclude 하면 된다고 나와있지만 그렇게 해도 마찬가지 오류가 발생한다.
- [Some frameworks do not expect coverage engine to add extra fields during instrumentation](https://youtrack.jetbrains.com/issue/IDEA-290288/)



## 해결

사실 우리가 작성하지도 않은 코드인 HqlTokenTypes 클래스의 코드 커버리지까지 확인할 필요는 없다. 

따라서 `Edit Configuration...` 팝업에서 다음과 같이 우리가 작성한 코드만 코드 커버리지 측정 범위에 포함하도록 명시적으로 지정해주면,  
HqlTokenTypes 클래스에 불필요한 instrumentation 이 발생하지 않고 결과적으로 위와 같은 에러도 발생하지 않는다.

![Imgur](https://i.imgur.com/d8D4ECm.png)

이런 걸 수동으로 할 게 아니라 인텔리제이 안에서 자체적으로 설정해주면 좋을텐데~

