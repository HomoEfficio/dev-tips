# TDD 연습

박재성님의 [우아한 테크 세미나(20190425) 영상](https://youtu.be/bIeqAlmNRrA) 요약

내용 순서는 우아한 테크 코스 사례로 시작하는 영상과는 조금 다르게 편집

# 마음 가짐

- TDD, 리팩터링 == 운동
  - 평생 연습


# 연습 대상

- 토이 프로젝트
  - 언어 기본 API 사용
    - String, HashMap, CompletableFuture, ..
- 업무
  - Input과 Output이 명확한 클래스 (ex: Util 성격의 메서드)


# TDD, 리팩터링 실패 이유

- 처음부터 '레거시 애플리케이션에 테스트 코드 추가해 리팩터링' 같은 높은 난이도로 시작하기 때문


# 연습 원칙

- 문제 해결이 아니라 TDD 연습이 목적
  - 어려운 문제 X, 쉽거나 익숙한 문제로 시작
- UI(웹, 모바일), DB 등에 의존 관계를 갖지 않는 대상으로 연습
- Fail -> Pass -> Refactor

  1. 다음과 같이 실패하는 assert 부터 작성
    
      ```java
      @Test
      public void null_or_empty() {
          assertThat(xxx.yyy(null)).isEqualTo(0);
          assertThat(xxx.yyy("")).isEqualTo(0);
      }
      ```

  1. 테스트 대상 실제 구현
  1. 테스트 통과
  1. 리팩터링
- 한 번에 여러 원칙을 지키려 하지 말고 명확하게 한 가지 목표를 대상으로 연습


# 지켜야 할 제약 사항

정량적인 기준을 정해야 피드백과 동기부여가 명확

## 1단계

>- indent 2 이하
>- 함수를 최대한 작게, 15라인 이하
>- else 예약어 사용 안 하기

## 2단계

>- 함수 10 라인 이하
>- indent 가능한 1 이하 
>- 함수 인자 3 이하


# 보편적 원칙

## 객체지향 생활 체조 원칙 9가지

The ThroughtWorks Anthology by Martin Fowler

1. 한 메서드에 indent는 한 단계만
1. else 사용 금지
1. 모든 원시값과 문자열을 포장
1. 한 줄에 점을 하나만
1. 이름에 축약어를 사용 금지
1. 모든 엔티티를 작게 유지
1. 클래스의 인스턴스 변수 3개 이하
1. 일급 컬렉션 사용(?)
1. getter/setter/property 사용 금지(?)

## 클린 코드

1. 메서드 인자 개수는 가능한 2개 이하, 3개 허용, 4개 이상 금지


# 리팩터링 기법

- 위 1, 2단계 제약 사항
- compose method 패턴: 메서드를 동등한 수준의 여러 단계로 나누어 조합
  
    ```java
    // 동등한 수준의 여러 단계로 분할
    String[] splitted = text.split(",|":);
    int[] ints = toInts(splitted);
    int result = sum(ints);

    // 인라이닝으로 조합
    return sum(toInts(text.split(",|:")));
    ```
- Wrapper 사용
  - primitive 변수지만 비즈니스 적 제약 사항이 존재한다면 wrapper 객체를 만들어서 사용
    - 양의 정수 ex: int 대신 

      ```java
      class LuckyNumber { 
          private int luckyNumber;
          public LuckyNumber(int num) {
              if (num < 0 || num > 45)
                  throw new IllegalArgumentException();
              this.luckyNumber = num;
          }
          public LuckyNumber(String value) {
              this(Integer.parseInt(value));
          }
          // getter
      }
      ```
    - Collection ex: Set<E> 대신

      ```java
      class Lotto {
          private static final int LOTTO_NUM_COUNT = 6;
          private Set<LuckyNumber> luckyNumbers = new LinkedHashSet<>(LOTTO_NUM_COUNT);
          public Lotto(Set<LuckyNumber> luckyNumbers) {
              if (luckyNumbers.size != LOTTO_NUM_COUNT)
                  throw new IllegalArgumentException();
              this.luckyNumbers.addAll(luckyNumbers);
          }
      }
      ```


# 성장 하기

## 개인 단계

- 처음부터 남과 함께 하려하지 말고 혼자 시작
  - 변화를 능동적으로 받아들이는 사람이나 팀은 많지 않음
  - 내가 작성한 코드에 리팩토링
  - 내가 작성할 코드를 TDD
- 자뻑 또는 동료의 관심에서 작은 성취감
- (먼저 전도하지 말고) 관심있는 사람이 생기면 전파
- 리더가 하지 말라면 그만둔다. TDD/리팩터링이 아니라 그 회사를.
  - 성장했으니 갈 곳 많다. 손해 보는 장사 아니다.

## 리더 단계

- 개인 수준에서 실행/전파해보고 경력이 쌓이면 리더 단계에서 시도
- 1:1 공략
  - 문제를 찾아 지적하고 해법을 알려주는 방식 지양
  - 1:1 면담으로 개선점을 함께 찾고, 해법 제시 대신 반문으로 생각 유도
- 팀 단위로 작은 성취감
- 사이클
  - 면담/회고 -> practice 선택 -> 한 가지에 집중 -> 작은 성공 -> 면담/회고
- 어떤 practice를 하느냐보다 나아지고 있다는 방향성이 중요
- 작은 성공을 쌓아 큰 성공으로
  - 토이 프로젝트, 내가 맡은 업무에 TDD 적용, 작은 성취
  - 동료와 짝 프로그래밍으로 TDD 전파, 작은 성취
  - 주변에 더 많이 전파하며 큰 성취
- 실패해도 나는 성장한다.
  - 실패의 책임을 물으면 그만둔다. TDD/리팩터링이 아니라 그 회사를.
