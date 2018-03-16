`@Parameterized.Parameters`에 `name` 속성으로 테스트 이름을 지정할 수 있어 편리하다.

```java
@RunWith(Parameterized.class)
public class InvalidDynamicQueryParamMapBuilderTest {

    private DynamicQueryParamMapBuilder builder;

    private Map<String, String> paramMap;

    private String testCaseName;

    public InvalidDynamicQueryParamMapBuilderTest(String testCaseName, String columnName, String operatorValue) {
        this.testCaseName = testCaseName;
        HashMap<String, String> map = new HashMap<>();
        map.put(columnName, operatorValue);
        this.paramMap = map;
    }

    @Parameterized.Parameters(name = "{index}: {0}, <{1}, {2}>")
    public static Collection input() {
        return Arrays.asList(new Object[][] {
                {"컬럼값이 없는 경우",       "col001", ""}
                , {"연산자만 있는 경우",     "col002", "="}
                , {"splitter만 있는 경우",  "col003", MySqlOperators.SPLITTER.toString()}
                , {"값만 있는 경우",        "col003", "%NBA%"}
                , {"연산자와 splitter만 있는 경우", "col003", "=" + MySqlOperators.SPLITTER.toString()}
                , {"splitter와 값만 있는 경우",    "col003", MySqlOperators.SPLITTER.toString() + "%NBA%"}
                , {"연산자와 값만 있는 경우",        "col003", "like%NBA%"}
                , {"입력된 연산자가 SQL 연산자가 아닌 경우",              "col001", "!==" + MySqlOperators.SPLITTER.toString() + "멀티"}
                , {"다중 값이 와야할 곳에 값은 없고 ||| 구분자만 있는 경우", "col001", "in" + MySqlOperators.SPLITTER.toString() + MySqlOperators.VALUE_DELIMITER.toString()}
                , {"like 검색에서 검색어는 없고 %만 있는 경우",           "col001", "like" + MySqlOperators.SPLITTER.toString() + "%%%"}
        });
    }

    @Before
    public void before() {
        this.builder = new DynamicQueryParamMapBuilder();
    }

    @Test(expected = InvalidSearchConditionParameterException.class)
    public void 컬럼값이_비정상적이면_예외던짐() throws Exception {

        LinkedHashMap<String, String> dynamicQueryParamerMap =
                this.builder.buildDynamicQueryParamerMap(
                        this.paramMap,
                        "col\\d{3}",
                        MySqlOperators.SPLITTER.toString());
    }
}
```
