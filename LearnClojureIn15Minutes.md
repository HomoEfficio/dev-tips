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

;;;;;;;;;;;;;;;;;;;;;;
; 감을 잡기 위한 기본 예제
;;;;;;;;;;;;;;;;;;;;;;

; 기본 산술 연산
(+ 1 2) ; 3
(- 2 1) ; 1
(* 3 2) ; 6
(/ 4 2) ; 2


; 인자가 여러 개 일 수도 있다.
(+ 1 2.5 3) ; 6.5
(- 4 5 6) ; -1
(* 2 3 4) ; 24
(/ 2 3 4) ; 1/6 클로저에는 분수 타입이 있다.


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

(coll? '(1 2 3)) ; 리스트는 collection인가? true
(coll? [1 2 3]) ; 벡터는 collection인가? true
(coll? #{1 2 3}) ; 셋은 collection인가? true
(coll? {:1 1 :b 2}) ; 맵은 collection인가? true

; 컬렉션 주요 함수
(count '(0 1 2 3)) ; 4

(cons 4 '(0 1 2 3)) ; (4 0 1 2 3)
(cons 4 [0 1 2 3]) ; (4 0 1 2 3) cons의 결과는 언제나 리스트. 리스트는 언제나 맨 앞에 데이터가 추가된다.
(cons 4 #{0 1 2 3}) ; (4 0 1 2 3) cons의 결과는 언제나 리스트. 리스트는 언제나 맨 앞에 데이터가 추가된다.
(cons {:3 3} {:1 1, :2 2}) ; ({:3 3} [:1 1] [:2 2]) cons의 결과는 언제나 리스트. 리스트는 언제나 맨 앞에 데이터가 추가된다.
(cons {:0 0} {:1 1, :2 2}) ; ({:0 0} [:1 1] [:2 2]) cons의 결과는 언제나 리스트. 리스트는 언제나 맨 앞에 데이터가 추가된다.
(cons {:-1 -1} {:1 1, :2 2}) ; ({:-1 -1} [:1 1] [:2 2]) cons의 결과는 언제나 리스트. 리스트는 언제나 맨 앞에 데이터가 추가된다.

(conj '(0 1 2 3) 4) ; (4 0 1 2 3) conj는 컬렉션의 타입이 보존됨. 리스트는 언제나 맨 앞에 데이터가 추가된다.
(conj [0 1 2 3] 4) ; [0 1 2 3 4] conj는 컬렉션의 타입이 보존됨. 벡터는 언제나 맨 뒤에 데이터가 추가된다.
(conj #{0 1 2 3} 4) ; #{0 1 2 3 4} conj는 컬렉션의 타입이 보존됨.
(conj #{0 1 2 3} -1) ; #{0 -1 1 2 3} 셋은 키의 해쉬값에 따라 정렬됨.
(conj {:1 1, :2 2} {:3 3}) ; {:3 3, :1 1, :2 2} conj는 컬렉션의 타입이 보존됨. 맵은 맨 앞에 데이터가 추가된다.
(conj {:1 1, :2 2} {:0 0}) ; {:0 0, :1 1, :2 2} conj는 컬렉션의 타입이 보존됨. 맵은 맨 앞에 데이터가 추가된다.
(conj {:1 1, :2 2} {:-1 -1}) ; {:-1 -1, :1 1, :2 2} conj는 컬렉션의 타입이 보존됨. 맵은 맨 앞에 데이터가 추가된다.

(cons 3 4 '(0 1 2)) ; clojure.lang.ArityException: Wrong number of args (3) passed to: core$cons

(conj '(0 1 2) 3 4) ; (4 3 0 1 2)
(conj [0 1 2] 3 4) ; [0 1 2 3 4]
(conj #{0 1 2} -1 3) ; #{0 -1 1 2 3}
(conj {:0 0 :1 1 :2 2} {:-1 -1} {:3 3}) ; {:3 3, :-1 -1, :0 0, :1 1, :2 2}

(concat '(1 3) '(2 4)) ; (1 3 2 4)
(concat '(1 3) [2 4]) ; (1 3 2 4)
(concat [1 3] '(2 4)) ; (1 3 2 4)
(concat [1 3] [2 4]) ; (1 3 2 4)


(first '(0 1 2)) ; 0
(rest '(0 1 2)) ; (1 2)
(first (rest '(0 1 2))) ; 1

(peek '(0 1 2)) ; 0 리스트나 큐에서는 first는 peek과 같다.
(pop '(0 1 2)) ; (1 2) 리스트나 큐에서는 pop은 첫번째 아이템을 제외한 나머지 아이템을 리스트나 큐로 반환

(first '()) ; nil
(rest '()) ; ()
(rest '(0)) ; ()

(peek '()) ; nil
(pop '()) ; java.lang.IllegalStateException: Can't pop empty list
(pop '(0)) ; ()


(first [0 1 2]) ; 0
(rest [0 1 2]) ; (1 2)
(first (rest [0 1 2])) ; 1

(peek [0 1 2]) ; 0 리스트나 큐에서는 first는 peek과 같다.
(pop '(0 1 2)) ; (1 2) 리스트나 큐에서는 pop은 첫번째 아이템을 제외한 나머지 아이템을 리스트나 큐로 반환

(first '()) ; nil
(rest '()) ; ()
(rest '(0)) ; ()

(peek '()) ; nil
(pop '()) ; java.lang.IllegalStateException: Can't pop empty list
(pop '(0)) ; ()



(.indexOf '(3 5 7) 1) ; -1
(.indexOf '(3 5 7) 3) ; 0(rest '()) ; ()
(.indexOf '(3 5 7) 5) ; 1
(nth '(3 5 7) 0) ; 3
(nth '(3 5 7) 2) ; 7
(nth '(3 5 7) -1) ; java.lang.IndexOutOfBoundsException
(nth '(3 5 7) -3) ; java.lang.IndexOutOfBoundsException






; Collections & Sequences
;;;;;;;;;;;;;;;;;;;

; Vectors and Lists are java classes too!
(class [1 2 3]); => clojure.lang.PersistentVector
(class '(1 2 3)); => clojure.lang.PersistentList

; A list would be written as just (1 2 3), but we have to quote
; it to stop the reader thinking it's a function.
; Also, (list 1 2 3) is the same as '(1 2 3)

; Both lists and vectors are collections:
(coll? '(1 2 3)) ; => true
(coll? [1 2 3]) ; => true

; Only lists are seqs.
(seq? '(1 2 3)) ; => true
(seq? [1 2 3]) ; => false

; Seqs are an interface for logical lists, which can be lazy.
; "Lazy" means that a seq can define an infinite series, like so:
(range 4) ; => (0 1 2 3)
(range) ; => (0 1 2 3 4 ...) (an infinite series)
(take 4 (range)) ;  (0 1 2 3)

; Use cons to add an item to the beginning of a list or vector
(cons 4 [1 2 3]) ; => (4 1 2 3)
(cons 4 '(1 2 3)) ; => (4 1 2 3)

; Use conj to add an item to the beginning of a list,
; or the end of a vector
(conj [1 2 3] 4) ; => [1 2 3 4]
(conj '(1 2 3) 4) ; => (4 1 2 3)

; Use concat to add lists or vectors together
(concat [1 2] '(3 4)) ; => (1 2 3 4)

; Use filter, map to interact with collections
(map inc [1 2 3]) ; => (2 3 4)
(filter even? [1 2 3]) ; => (2)

; Use reduce to reduce them
(reduce + [1 2 3 4])
; = (+ (+ (+ 1 2) 3) 4)
; => 10

; Reduce can take an initial-value argument too
(reduce conj [] '(3 2 1))
; = (conj (conj (conj [] 3) 2) 1)
; => [3 2 1]

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
