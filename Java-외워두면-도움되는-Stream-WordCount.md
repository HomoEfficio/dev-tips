# Java-외워두면-도움되는-Stream-WordCount

Java Stream에서 사용할 수 있는 다양한 메서드를 포함하고 있어서 외워두면 여러모로 도움된다.

```java
Map.Entry<String, Long> mostCommonEntry = 
        List.of("나 혼자 길을 걷고", "나 혼자 티비 보고", "나 혼자 취해보고", "이렇게 매일 울고 불고")
                .stream()
                .map(s -> s.split(" "))
                .flatMap(Arrays::stream)
                .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
                .entrySet().stream()
                .min(Map.Entry.comparingByValue(Comparator.reverseOrder()))
                .orElseThrow(() -> new RuntimeException("no entry error"));
                
System.out.println(mostCommonEntry.getKey() + ": " + mostCommonEntry.getValue());

// '나: 3' 출력
```
