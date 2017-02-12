# Clojure - flatten 구현하기

http://www.4clojure.com/problem/28 에 나온 문제다.

```clojure
;; 아래를 만족시키는 함수 __를 구현하시오

(= (__ '((1 2) 3 [4 [5 6]])) '(1 2 3 4 5 6))

(= (__ ["a" ["b"] "c"]) '("a" "b" "c"))

(= (__ '((((:a))))) '(:a))
```

간신히 풀었는데, 문제의 난이도는 easy..

```clojure
(fn my-flattener [arg]
  ((fn flattener [seqt result]
    (if (= (count seqt) 0)
      result
      (if (sequential? (first seqt))
        (concat (flattener (first seqt) result) (flattener (rest seqt) result))
        (cons (first seqt) (flattener (rest seqt) result))))) arg '()))
```

이거 실제로는 어떻게 구현되었을까 궁금해서 [clojure에서 구현된 `flatten`의 소스](https://github.com/clojure/clojure/blob/master/src/clj/clojure/core.clj#L7011)를 보니 다음과 같았다.

```clojure
(defn flatten [x]
  (filter (complement sequential?)
          (rest (tree-seq sequential? seq x))))
```

일단 어떤 시퀀스를 만들기 위해 `cons`, `conj`, `concat` 등을 동원하지 않아도, `filter`로도 시퀀스를 만들 수 있다는 점을 배웠는데,

놀라운 건, 이 문제를 풀기 위해 시퀀스의 원소를 분해하고 합치는 작업을 수행하는 것이 아니라, `tree-seq`라는 함수를 활용해서 시퀀스 내에 중첩된 시퀀스를 non-sequential한 값으로 모두 펼쳐낸 후에, `filter`로 non-sequential한 원소만을 추려내는 식으로, **원소의 뽑아서 이어붙이는 방식이 아니라 자료 구조를 활용해서 해결** 한다는 점이다.

구체적으로 살펴보면 `tree-seq`를 아래와 같이 사용하면,
```clojure
(tree-seq sequential? seq '(1 2 (3 (4) [5]) 6))

; ((1 2 (3 (4) [5]) 6) 1 2 (3 (4) [5]) 3 (4) 4 [5] 5 6)
```
이렇게 원소 중에서 `sequential?`이 참인 애들은 그 안의 원소를 꺼내서 다시 펼쳐낸다.

따라서 이 중에서 `sequential?`이 거짓인 애들만 골라내면(`(filter (compliment sequential?))`) 결과적으로 flatten이 된다!!

참고로, [여기](https://mwfogleman.github.io/posts/20-12-2014-flatcat.html)에 flatten에 대한 몇 가지 해법과 좋은 설명이 있다.
