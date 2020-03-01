# ResultSet Mock Test

테스트하다 보면 DB 처리 결과를 담고 있는 ResultSet에 대한 가짜 Mock ResultSet을 사용해야 할 때도 있다.

실제로 어떻게 Mock을 할 수 있을지 알아보자.

예를 들어, ResultSet이 다음과 같다면,

seg_id | uid | uid_type
--- | --- | ---
1001 | 10096b0c | gaid
1001 | 10096B0C | idfa
1001 | 20096B0C | idfa
1002 | 30096B0C | idfa
1002 | 20096b0c | gaid
1003 | 30096b0c | gaid
1004 | 40096b0c | gaid
1005 | 50096b0c | gaid

다음과 같이 **컬럼 별로 Mock 처리하면 되는데, 특이한 것은 실제 데이터 뿐만 아니라 `rs.next()`에 대한 Mock 처리도 추가로 해줘야 한다**는 점이다.

```java
@RunWith(MockitoJUnitRunner.class)
public class SegIdBasedResultSetToFileWriterTest {

    @Mock
    private ResultSet rs;
    
    @Test
    public void ResultSet_test_1() throws Exception {
    
        // Given
        // 아래와 같이 rs.next() 도 처리해줘야함
        given(rs.next())
            .willReturn(true, true, true, true, true, true, true, true, 
                    false); // 마지막에 false 반환 추가

        given(rs.getString("seg_id"))
                .willReturn(
                        "1001",
                        "1001",
                        "1001",
                        "1002",
                        "1002",
                        "1003",
                        "1004",
                        "1005"
                );
                
        given(rs.getString("uid"))
                .willReturn(
                        "10096b0c",  // seg1
                        "10096B0C",  // seg1
                        "20096B0C",  // seg1
                        "30096B0C",  // seg2
                        "20096b0c",  // seg2
                        "30096b0c",  // seg3
                        "40096b0c",  // seg4
                        "50096b0c"   // seg5
                );
                
        given(rs.getString("uid_type"))
                .willReturn(
                        "gaid",
                        "idfa",
                        "idfa",
                        "idfa",
                        "gaid",
                        "gaid",
                        "gaid",
                        "gaid"
                );

        ...
        
        // When
        // ResultSet를 사용하는 곳에 Mock 처리된 rs 전달
        ...
        
        // Then
        // Mock 처리된 rs 기준으로 예상되는 값 assert
```
