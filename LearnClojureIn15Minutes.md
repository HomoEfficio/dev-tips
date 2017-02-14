# Learn Clojure In 15 Minutes

https://adambard.com/blog/clojure-in-15-minutes/ 에 있는 내용을 번역, 추가 및 재구성했다.

```clojure
; 클로저에서 주석은 세미콜론으로 시작한다.
;
; 클로저는 괄호와 괄호 안에서 공백으로 구분되는 인자로 구성되는 form이라는 형태로 작성된다
; (인자0 인자1 인자2 ...)
;
; form 내부의 첫번째 인자는 보통 함수이거나 매크로이고, 나머지 인자는 그 함수나 매크로의 인자다.

; 다음과 같은 form은 namespace를 test라고 설정한다.
(ns test)

;======================
; 감을 잡기 위한 기본 예제
;======================

; 기본 산술 연산
(+ 1 2) ; 3
(- 2 1) ; 1
(* 3 2) ; 6
(/ 4 2) ; 2


; 인자가 여러 개 일 수도 있다.
(+ 1 2.5 3) ; 6.5
(- 4 5 6) ; -1
(* 2 3 4) ; 24
(/ 2 3 4) ; 1/6 클로저에는 비율(분수) 타입이 있다.
(+ 0 1 nil) ; java.lang.NullPointerException nil은 null 비슷하다.


; 논리 연산
(and true true) ; true
(and false true) ; false
(and true false) ; false
(and true true false) ; false 인자가 3개 이상일 수도 있다.
(and true true true) ; true
(or true true) ; true
(or false true) ; true
(or true false) ; true
(or false false) ; false
(or nil false) ; false nil은 논리연산에서는 NullPointerException이 발생하지 않고 false로 평가된다.
(or false false true) ; true 인자가 3개 이상일 수도 있다.
(not true) ; false
(not false) ; true
(not true true) ; clojure.lang.ArityException: Wrong number of args (2) passed to: core$not


; 비교 연산
(= 1 1) ; true
(= 1 2) ; false
(= 1 1 1) ; true
(= 1 1 2) ; false 인자가 3개 이상일 수도 있다.
(= 2 1 1) ; false
(not= 1 1) ; false
(not= 1 2) ; true
(not= 1 2 1 3) ; true. (and (not= 1 2) (not= 2 1) (not= 1 3))와 같다.
(> 3 2) ; true
(< 3 2) ; false
(>= 3 2) ; true
(>= 3 3) ; true
(<= 3 2) ; false
(<= 3 3) ; true
(> 3 2 1 0) ; true. (and (> 3 2) (> 2 1) (> 1 0))와 같다.
(> 3 1 2) ; false. (and (> 3 1) (> 1 2))와 같다.
(>= 3 2.2 2.1 1) ; true. (and (>= 3 2.2) (>= 2.2 2.1) (>= 2.1 1))
(> 0 nil) ; java.lang.NullPointerException nil은 비교 연산에서도 NullPointerExcpetion 발생


; 중첩
(+ 1 (- 1 4)) ; = 1 + (1 - 4) = -2
(/ 6 (- 1 4 -1)) ; = 6 / (1 - 4 - 1) = -3
(/ 6 (- 1 4 -1) 2) ; = (6 / (1 - 4 - 1)) / 2 = -3/2




;======================
; 타입
;======================

; 클로저는 불리언(booean), 문자열(string), 숫자(number)는 자바의 데이터 타입을 그대로 사용

; class 함수로 타입 확인 가능
(class 1) ; java.lang.Long
(class 1.) ; java.lang.Double
(class "") ; java.lang.String 문자열은 언제나 쌍따옴표
;(class '') ; 클로저에서 '는 다른 의미로 사용
(class false) ; java.lang.Boolean
(class nil); 클로저에서는 nil 이 자바의 null과 비슷하다.

(class +) ; clojure.core$_PLUS_
(class class) ; clojure.core$class
(class inc) ; clojure.core$inc
(class def) ; Macro를 class의 인자로 쓰면 에러. clojure.lang.Compiler$CompilerException: java.lang.RuntimeException: Can't take value of a macro

(class (def a)) ; clojure.lang.Var
(class :a) ; clojure.lang.Keyword


; Collections
(class '(1 2 3)) ; clojure.lang.PersistentList
(class [1 2 3]) ; clojure.lang.PersistentVector
(class #{1 2 3}) ; clojure.lang.PersistentHashSet
(class {:1 1 :b 2}) ; clojure.lang.PersistentArrayMap
(class {:1 1, :b 2}) ; clojure.lang.PersistentArrayMap 쉼표는 써도 되고 안 써도 되지만 맵에서만 관례적으로 쉼표로 구분한다.

(coll? '(1 2 3)) ; 리스트는 collection인가? true
(coll? [1 2 3]) ; 벡터는 collection인가? true
(coll? #{1 2 3}) ; 셋은 collection인가? true
(coll? {:1 1, :b 2}) ; 맵은 collection인가? true

;======================
; 컬렉션 주요 함수
;======================
(count '(0 1 2 3)) ; 4
(count [0 1 2 3]) ; 4
(count #{0 1 2 3}) ; 4
(count {:0 0, :1 1, :2 2, :3 3}) ; 4 맵은 키-값 쌍의 개수를 반환한다.


(cons 4 '(0 1 2 3)) ; (4 0 1 2 3)
(cons 4 [0 1 2 3]) ; (4 0 1 2 3)
(cons 4 #{0 1 2 3}) ; (4 0 1 2 3)
(cons {:3 3} {:1 1, :2 2}) ; ({:3 3} [:1 1] [:2 2])
(cons {:0 0} {:1 1, :2 2}) ; ({:0 0} [:1 1] [:2 2])
(cons {:-1 -1} {:1 1, :2 2}) ; ({:-1 -1} [:1 1] [:2 2])
(cons '(4 5) '(0 1 2 3)) ; ((4 5) 0 1 2 3)
(cons '(4 5) [0 1 2 3]) ; ((4 5) 0 1 2 3)
(cons [4 5] '(0 1 2 3)) ; ([4 5] 0 1 2 3)
(cons [4 5] [0 1 2 3]) ; ([4 5] 0 1 2 3)
(cons [4 5] #{0 1 2 3}) ; ([4 5] 0 1 2 3)
(cons '(4 5) {:0 0, :1 1, :2 2}) ; ((4 5) [:0 0] [:1 1] [:2 2])
(cons 3 4 '(0 1 2)) ; clojure.lang.ArityException: Wrong number of args (3) passed to: core$cons
; (cons 원소A 컬렉션B)은 
;   컬렉션B를 시퀀스C로 만든 후에 원소A를 시퀀스C의 맨 앞에 추가한 새로운 시퀀스D를 반환한다.
;     컬렉션B가 맵이라면 하나의 키-값 쌍은 하나의 벡터E로 변환되고, 맵 전체는 벡터E를 원소로 하는 시퀀스C로 변환된다. {:0 0, :1 1, :2 2} -> ([:0 0] [:1 1] [:2 2])
;     원소A가 컬렉션이더라도 A를 분해하지 않고, 시퀀스로 만들지도 않고 그대로 하나의 원소로 취급한다.
;   2개의 인자만 허용한다.


(conj '(0 1 2 3) 4) ; (4 0 1 2 3)
(conj [0 1 2 3] 4) ; [0 1 2 3 4]
(conj #{0 1 2 3} 4) ; #{0 1 2 3 4}
(conj #{0 1 2 3} -1) ; #{0 -1 1 2 3}
(conj {:1 1, :2 2} {:3 3}) ; {:3 3, :1 1, :2 2}
(conj {:1 1, :2 2} {:0 0}) ; {:0 0, :1 1, :2 2}
(conj {:1 1, :2 2} {:-1 -1}) ; {:-1 -1, :1 1, :2 2} conj는 컬렉션의 타입이 보존됨. 맵은 맨 앞에 데이터가 추가된다.
(conj '(0 1 2 3) '(4 5)) ; ((4 5) 0 1 2 3)
(conj '(0 1 2 3) [4 5]) ; ([4 5] 0 1 2 3)
(conj [0 1 2] '(3 4)) ; [0 1 2 (3 4)]
(conj [0 1 2] [3 4] '(5 6)) ; [0 1 2 [3 4] (5 6)]
(conj [0 1 2] 3 '(4 5) [6 7] #{8 9} {:10 10, :11 11, :12 {:13 13}}) ; 
(conj #{0 1 2} -1 3) ; #{0 -1 1 2 3}
(conj {:0 0, :1 1, :2 2} {:-1 -1} {:3 3}) ; {:3 3, :-1 -1, :0 0, :1 1, :2 2}
; (conj 컬렉션A 원소B)은 
;   컬렉션A의 타입을 그대로 보존하면서 컬렉션A의 데이터 추가 위치 특성(리스트는 맨 앞, 벡터는 맨 뒤, 셋은 해쉬값에 따라, 맵은 맨 앞)에 따라 원소B를 추가한 새로운 컬렉션C를 반환한다.
;   원소B가 컬렉션이더라도 A를 분해하지 않고 그대로 하나의 원소로 취급한다.
;   2개 이상의 인자도 허용한다.


(concat '(1 3) '(2 4)) ; (1 3 2 4)
(concat '(1 3) [2 4]) ; (1 3 2 4)
(concat [1 3] '(2 4)) ; (1 3 2 4)
(concat [1 3] [2 4]) ; (1 3 2 4)
(concat '(1 3) [2 4] #{6 5}) ; (1 3 2 4 5 6)
(concat '(1 3) [2 4] #{6 5} {:7 7, :8 8}) ; (1 3 2 4 5 6 [:7 7] [:8 8])
(concat '(1 3) [2 4] #{6 5} {:7 7, :8 8, :9 {:10 10}}) ; (1 3 2 4 5 6 [:7 7] [:8 8] [:9 {:10 10}])
(concat '(1 3) 0) ; java.lang.IllegalArgumentException: Don't know how to create ISeq from: java.lang.Long
; (concat 컬렉션A 컬렉션B 컬렉션C ...)은
;   인자로 받은 컬렉션을 각각 시퀀스a, 시퀀스b, 시퀀스c, ...로 변환한 후 
;   시퀀스a, 시퀀스b, 시퀀스c, ...의 모든 원소를 하나의 lazy 시퀀스Z에 담아 반환한다.


(seq '(1 2 3)) ; (1 2 3)
(seq? '(1 2 3)) ; true
(class (seq '(1 2 3))) ; clojure.lang.PersistentList

(seq [1 2 3]) ; (1 2 3)
(seq? [1 2 3]) ; false
(class (seq [1 2 3])) ; clojure.lang.PersistentVector$ChunkedSeq

(seq #{1 2 3}) ; (1 2 3)
(seq? #{1 2 3}) ; false
(class (seq #{1 2 3})) ; clojure.lang.APersistentMap$KeySeq

(seq {:1 1, :2 2, :3 3}) ; 
(seq? {:1 1, :2 2, :3 3}) ; false
(class (seq {:1 1, :2 2, :3 3})) ; clojure.lang.PersistentArrayMap$Seq

(coll? (seq '(1 2 3))) ; true
(coll? (seq [1 2 3])) ; true
(coll? (seq #{1 2 3})) ; true
(coll? (seq {:1 1, :2 2, :3 3})) ; true       
; 시퀀스는 출력되는 형태는 리스트와 같지만 시퀀스는 추상(abstration, 자바의 인터페이스 개념)이고, 
; 리스트는 시퀀스의 구현체 중 하나다.
; 시퀀스도 컬렉션이다.



(first '(0 1 2)) ; 0
(last '(0 1 2)) ; 2
(rest '(0 1 2)) ; (1 2)
(first (rest '(0 1 2))) ; 1
(first '()) ; nil
(last '()) ; nil
(last '(0)) ; 0
(rest '()) ; ()
(rest '(0)) ; ()

(first [0 1 2]) ; 0
(last [0 1 2]) ; 2
(rest [0 1 2]) ; (1 2)
(first (rest [0 1 2])) ; 1
(first []) ; nil
(last []) ; nil
(last [0]) ; 0
(rest []) ; ()
(rest [0]) ; ()

(first #{0 1 2}) ; 0
(last #{0 1 2}) ; 2
(rest #{0 1 2}) ; (1 2)
(first (rest #{0 1 2})) ; 1
(first #{}) ; nil
(last #{}) ; nil
(last #{0}) ; 0
(rest #{}) ; ()
(rest #{0}) ; ()

(first {:0 0 :1 1 :2 2}) ; [:0 0]
(last {:0 0 :1 1 :2 2}) ; [:2 2]
(rest {:0 0 :1 1 :2 2}) ; ([:1 1] [:2 2])
(first (rest {:0 0 :1 1 :2 2})) ; [:1 1]
(first {}) ; nil
(last {}) ; nil
(last {:0 0}) ; [:0 0]
(rest {}) ; ()
(rest {:0 0}) ; ()
; (first 컬렉션A)는 인자인 컬렉션A를 시퀀스B로 변환하고 시퀀스B의 첫번째 원소를 반환한다.
; (last 컬렉션A)는 인자인 컬렉션A를 마지막 원소를 반환한다. 컬렉션의 길이가 길수록 오래 걸린다.
;   아래에서 나오지만 벡터에서는 last를 쓰는 것보다 peek을 사용하는 것이 빠르다.
; (rest는 컬렉션A)는 인자인 컬렉션A를 시퀀스B로 변환하고 시퀀스B의 첫번째 원소를 제외한 나머지 시퀀스C를 반환한다.


; Stack 함수
(peek '(0 1 2)) ; 0
(pop '(0 1 2)) ; (1 2)
(peek '()) ; nil
(pop '()) ; java.lang.IllegalStateException: Can't pop empty list
(pop '(0)) ; ()

(peek [0 1 2]) ; 2
(pop [0 1 2]) ; [0 1]
(peek []) ; nil
(pop []) ; java.lang.IllegalStateException: Can't pop empty vector
(pop [0]) ; []

(peek #{0 1 2}) ; java.lang.ClassCastException: clojure.lang.PersistentHashSet cannot be cast to clojure.lang.IPersistentStack 
(pop #{0 1 2}) ; java.lang.ClassCastException: clojure.lang.PersistentHashSet cannot be cast to clojure.lang.IPersistentStack 

(peek {:0 0 :1 1 :2 2}) ; java.lang.ClassCastException: clojure.lang.PersistentArrayMap cannot be cast to clojure.lang.IPersistentStack 
(pop {:0 0 :1 1 :2 2}) ; java.lang.ClassCastException: clojure.lang.PersistentArrayMap cannot be cast to clojure.lang.IPersistentStack 
; peek은 원소 하나를 반환한다.
;   리스트나 큐에서는 first와 같고, 벡터에서는 last와 같은 값을 반환하지만 훨씬 효율적이다.
; pop은 peek에서 반환되는 원소 하나를 제외한 나머지 컬렉션을 새로운 컬렉션으로 반환한다.
; 셋이나 맵은 Stack 타입의 컬렉션이 아니므로 peek이나 pop을 사용할 수 없다.


(.indexOf '(3 5 7) 1) ; -1 찾는 원소가 없으면 -1 반환
(.indexOf '(3 5 7) 3) ; 0
(.indexOf [3 5 7] 5) ; 1
(.indexOf #{3 5 7} 7) ; java.lang.IllegalArgumentException: No matching method found: indexOf for class clojure.lang.PersistentHashSet
(.indexOf {:3 3 :5 5 :7 7} 7) ; java.lang.IllegalArgumentException: No matching method found: indexOf for class clojure.lang.PersistentArrayMap
(nth '(3 5 7) 0) ; 3
(nth [3 5 7] 1) ; 5
(nth #{3 5 7} 2) ; java.lang.UnsupportedOperationException: nth not supported on this type: PersistentHashSet
(nth {:3 3, :5 5, :7 7} 1) ; java.lang.UnsupportedOperationException: nth not supported on this type: PersistentArrayMap
(nth '(3 5 7) -1) ; java.lang.IndexOutOfBoundsException
(nth [3 5 7] 3) ; java.lang.IndexOutOfBoundsException
; .indexOf나 nth는 리스트와 벡터에만 사용 가능


(range) ; (0 1 2 3 ...) 0을 포함한 무한한 자연수를 원소로 하는 lazy 시퀀스
(range 3) ; (0 1 2)
(range 3 7) ; (3 4 5 6)
(range 3 11 3) ; (3 6 9) (range start end step)
(take 2 (range 3 11 3)) ; (3 6)
; range와 take는 모두 lazy 시퀀스를 반환


(inc 1) ; 2
(dec 0) ; -1
(map inc '(0 1 2)) ; (1 2 3)
(map dec [0 1 2]) ; (-1 0 1)
(map inc #{0 1 2}) ; (1 2 3)
(map inc {:0 0, :1 1, :2 2}) ; java.lang.ClassCastException: clojure.lang.MapEntry cannot be cast to java.lang.Number
; (map 함수A 컬렉션B)는 

(filter even? '(0 1 2)) ; (0 2)
(filter even? (map inc [0 1 2])) ; (2)
(map dec (filter even? #{0 1 2})) ; (-1 1)

(reduce + '(1 2 3 4)) ; 10 (+ (+ (+ 1 2) 3) 4)
(reduce - [1 2 3 4]) ; -8 (- (- (- 1 2) 3) 4)
(reduce * #{1 2 3 4}) ; 24 (* (* (* 1 2) 3) 4)
(reduce / '(1 2 3 4)) ; 1/24 (/ (/ (/ 1 2) 3) 4)
(reduce + {:1 1, :2 2, :3 3, :4 4}) ; java.lang.ClassCastException: clojure.lang.MapEntry cannot be cast to java.lang.Number
(reduce conj '() #{1 2 3}) ; (3 2 1) (conj (conj (conj '() 1) 2) 3)
(reduce conj [] '(1 2 3)) ; [1 2 3] (conj (conj (conj [] 1) 2) 3)
(reduce conj #{} [1 2 3 2 4]) ; #{1 2 3 4} (conj (conj (conj (conj (conj #{} 1) 2) 3) 2) 4)
(reduce conj {} {:1 1, :2 2, :3 3}) ; {:3 3, :2 2, :1 1} (conj (conj (conj #{} {:1 1}) {:2 2}) {:3 3})
(reduce conj #{} {:1 1, :2 2, :3 3}) ; #{[:2 2] [:3 3] [:1 1]}
(reduce conj [] {:1 1, :2 2, :3 3}) ; [[:1 1] [:2 2] [:3 3]]
(reduce conj '() {:1 1, :2 2, :3 3}) ; ([:3 3] [:2 2] [:1 1])


; do
; java의 {} 블록과 유사
; clojure의 함수는 intrinsic do를 가지고 있어서 명시적으로 do를 쓰지 않아도 do가 있는 것처럼 동작
(do (println "a")
    (println "b")) ; a 출력 후 다음 줄에 b출력
    
; if
(if true "a" "b") ; "a"
(if false "a" "b") ; "b"
(if (if false false true) "a" "b") ; "a"
(if true
  (do (println "a")
      (println "b"))
  (print "c")) ; a 출력 후 다음 줄에 b출력
(if false
  (do (println "a")
      (println "b"))
  (print "c")) ; c 출력

; when
(when true "a") ; "a"
(when false "a") ; nil

; cond
(cond true "a"
      true "b"
      true "c") ; "a"
(cond false "a"
      true "b"
      true "c") ; "b"
(cond false "a"
      false "b"
      true "c") ; "c"
(cond false "a"
      false "b"
      false "c") ; nil

; case
(case 1
    1 "a"
    2 "b"
    3 "c") ; "a"
(case 2
    1 "a"
    2 "b"
    3 "c") ; "b"
(case 3
    1 "a"
    2 "b"
    3 "c") ; "c"



;======================
; 함수
;======================

; (defn 함수이름 인자벡터 함수바디)
; 함수는 호출되면 바디를 수행하고 
; 바디의 마지막 식의 결과를 반환한다.
(defn my-func [arg1]
    (str "arg1: " arg1))
(my-func "testing arg1") ; "arg1: testing arg1"

; 가변 인자
(defn my-var-args-func [& args]
  (count args))
(my-var-args-func 1 2 3 4 5) ; 5


; 맵 destruction
; 아래와 같이 {함수내에서사용할이름1 :인자로받은맵의key1 함수내에서사용할이름2 :인자로받은맵의key2}의 형식으로 받는다.  
(defn map-destruction [{k1 :name k2 :email}]
  (str "k1: " name " k2: " email))
(map-destruction {:name "abc" :email 123}) ; "k1: abc k2: 123"

(defn map-keys-destruction [{:keys [name email]}] (str "k1: " name " k2: " email))
(map-keys-destruction {:name "def" :email 789}) ; "k1: def k2: 789"




;======================
; 동시성
;======================

; future
; (future & body)
; 새로운 쓰레드를  생성해서 & body 를 순서대로 실행하고,
; 실행 결과를 캐쉬해둔다.
; future의 실행 결과는 @ 또는 deref 를 통해 가져올 수 있다. 
(def f (future (Thread/sleep 5000) (println 'Future executed') 'My Future'))
f ; (Thread/sleep 5000) (println 'Future executed') 'My Future' 를  순차적으로 실행한다.
  ; 즉 새로운 쓰레드를 생성해서 5초 후에 'Future executed'를 출력하고,
  ; f에는 'My Future'가 할당된다.
  ; f의 값은 @f 로 읽어올 수 있으며, 최초에 1회 읽은 후 캐쉬되므로 그 이후 @f는 캐쉬된 값을 읽어온다.
  ; 새로운 쓰레드에서 5초간 sleep 하므로 원래의 쓰레드는 기본적으로 block 되지 않는다.
  ; 다만 아직 실행이 완료되지 않은 future를 @나 deref로 참조하면, 
  ;   @나 deref를 실행한 쓰레드는 대상 future가 완료될 때까지 block 된다.  
; 5초 후 'Future excuted' 출력
@f ; 'My Future'

(def f (future (Thread/sleep 5000) (println 'Future executed') 'My Future'))
f
; 5초 이내에 @f
@f ; future 실행이 완료될 때까지 blocking
   ; 완료된 후 'Future executed'가 출력되고
   ; 'My Future'가 출력된다.
  
  
  
  
  
;================================

; Functions
;;;;;;;;;;;;;;;;;;;;;

; Use fn to create new functions. A function always returns
; its last statement.
(fn [] "Hello World") ; => fn

; (You need extra parens to call it)
((fn [] "Hello World")) ; => "Hello World"

; You can create a var using def
(def x 1)
x ; => 1

; Assign a function to a var
(def hello-world (fn [] "Hello World"))
(hello-world) ; => "Hello World"

; You can shorten this process by using defn
(defn hello-world [] "Hello World")

; The [] is the list of arguments for the function.
(defn hello [name]
  (str "Hello " name))
(hello "Steve") ; => "Hello Steve"

; You can also use this shorthand to create functions:
(def hello2 #(str "Hello " %1))
(hello2 "Fanny") ; => "Hello Fanny"

; You can have multi-variadic functions, too
(defn hello3
  ([] "Hello World")
  ([name] (str "Hello " name)))
(hello3 "Jake") ; => "Hello Jake"
(hello3) ; => "Hello World"

; Functions can pack extra arguments up in a seq for you
(defn count-args [& args]
  (str "You passed " (count args) " args: " args))
(count-args 1 2 3) ; => "You passed 3 args: (1 2 3)"

; You can mix regular and packed arguments
(defn hello-count [name & args]
  (str "Hello " name ", you passed " (count args) " extra args"))
(hello-count "Finn" 1 2 3)
; => "Hello Finn, you passed 3 extra args"


; Hashmaps
;;;;;;;;;;

(class {:a 1 :b 2 :c 3}) ; => clojure.lang.PersistentArrayMap

; Keywords are like strings with some efficiency bonuses
(class :a) ; => clojure.lang.Keyword

; Maps can use any type as a key, but usually keywords are best
(def stringmap (hash-map "a" 1, "b" 2, "c" 3))
stringmap  ; => {"a" 1, "b" 2, "c" 3}

(def keymap (hash-map :a 1 :b 2 :c 3))
keymap ; => {:a 1, :c 3, :b 2} (order is not guaranteed)

; By the way, commas are always treated as whitespace and do nothing.

; Retrieve a value from a map by calling it as a function
(stringmap "a") ; => 1
(keymap :a) ; => 1

; Keywords can be used to retrieve their value from a map, too!
(:b keymap) ; => 2

; Don't try this with strings.
;("a" stringmap)
; => Exception: java.lang.String cannot be cast to clojure.lang.IFn

; Retrieving a non-present value returns nil
(stringmap "d") ; => nil

; Use assoc to add new keys to hash-maps
(assoc keymap :d 4) ; => {:a 1, :b 2, :c 3, :d 4}

; But remember, clojure types are immutable!
keymap ; => {:a 1, :b 2, :c 3}

; Use dissoc to remove keys
(dissoc keymap :a :b) ; => {:c 3}

; Sets
;;;;;;

(class #{1 2 3}) ; => clojure.lang.PersistentHashSet
(set [1 2 3 1 2 3 3 2 1 3 2 1]) ; => #{1 2 3}

; Add a member with conj
(conj #{1 2 3} 4) ; => #{1 2 3 4}

; Remove one with disj
(disj #{1 2 3} 1) ; => #{2 3}

; Test for existence by using the set as a function:
(#{1 2 3} 1) ; => 1
(#{1 2 3} 4) ; => nil

; There are more functions in the clojure.sets namespace.

; Useful forms
;;;;;;;;;;;;;;;;;

; Logic constructs in clojure are just macros, and look like
; everything else
(if false "a" "b") ; => "b"
(if false "a") ; => nil

; Use let to create temporary bindings
(let [a 1 b 2]
  (> a b)) ; => false

; Group statements together with do
(do
  (print "Hello")
  "World") ; => "World" (prints "Hello")

; Functions have an implicit do
(defn print-and-say-hello [name]
  (print "Saying hello to " name)
  (str "Hello " name))
(print-and-say-hello "Jeff") ;=> "Hello Jeff" (prints "Saying hello to Jeff")

; So does let
(let [name "Urkel"]
  (print "Saying hello to " name)
  (str "Hello " name)) ; => "Hello Urkel" (prints "Saying hello to Urkel")

; Modules
;;;;;;;;;;;;;;;

; Use "use" to get all functions from the module
(use 'clojure.set)

; Now we can use set operations
(intersection #{1 2 3} #{2 3 4}) ; => #{2 3}
(difference #{1 2 3} #{2 3 4}) ; => #{1}

; You can choose a subset of functions to import, too
(use '[clojure.set :only [intersection]])

; Use require to import a module
(require 'clojure.string)

; Use / to call functions from a module
(clojure.string/blank? "") ; => true

; You can give a module a shorter name on import
(require '[clojure.string :as str])
(str/replace "This is a test." #"[a-o]" str/upper-case) ; => "THIs Is A tEst."
; (#"" denotes a regular expression literal)

; You can use require (and use, but don't) from a namespace using :require.
; You don't need to quote your modules if you do it this way.
(ns test
  (:require
    [clojure.string :as str]
    [clojure.set :as set]))

; Java
;;;;;;;;;;;;;;;;;

; Java has a huge and useful standard library, so
; you'll want to learn how to get at it.

; Use import to load a java module
(import java.util.Date)

; You can import from an ns too.
(ns test
  (:import java.util.Date
           java.util.Calendar))

; Use the class name with a "." at the end to make a new instance
(Date.) ; <a date object>

; Use . to call methods. Or, use the ".method" shortcut
(. (Date.) getTime) ; <a timestamp>
(.getTime (Date.)) ; exactly the same thing.

; Use / to call static methods
(System/currentTimeMillis) ; <a timestamp> (system is always present)

; Use doto to make dealing with (mutable) classes more tolerable
(import java.util.Calendar)
(doto (Calendar/getInstance)
  (.set 2000 1 1 0 0 0)
  .getTime) ; => A Date. set to 2000-01-01 00:00:00
```
