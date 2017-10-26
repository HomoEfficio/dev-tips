# Python Map Reduce 훑어보기

파이썬 MapReduce는 보통 MRJob 클래스를 상속받아서 다음과 같은 모양새로 구현된다.

```python
# 파일명: mr_word_count_mapper_reducer.py
from mrjob.job import MRJob
from mrjob.job import MRStep


class MRWordFrequencyCount(MRJob):

    def steps(self):
        return [
            MRStep(mapper=self.my_mapper,
                   reducer=self.my_reducer)
        ]

    def my_mapper(self, _, line):
        yield "chars", len(line)
        yield "words", len(line.split())
        yield "lines", 1

    def my_reducer(self, key, values):
        yield key, sum(values)

if __name__ == '__main__':
    MRWordFrequencyCount.run()
```

워드 카운트의 대상이 되는 텍스트 파일(word-count.txt)이 다음과 같다면,

```
고요한 내 가슴에 나비처럼 날아와서
사랑을 심어놓고 나비처럼 날아 간 사람
내 가슴에 지울 수 없는 그리움 주고 간 사람
그리운 내 사연을 뜬 구름아 전해 다오
아아아 아아아아아
사랑은 얄미운 나비인가봐
```

위 파이썬 MR의 실행 결과는 다음과 같다.

```
python3 mr_word_count_mapper_reducer.py word-count.txt

No configs found; falling back on auto-configuration
Creating temp directory /var/folders/v2/7ykd39_d60zc410c_6slt53r0000gn/T/mr_word_count_mapper_reducer.1003604.20170628.022753.626414
Running step 1 of 1...
Streaming final output from /var/folders/v2/7ykd39_d60zc410c_6slt53r0000gn/T/mr_word_count_mapper_reducer.1003604.20170628.022753.626414/output...
"chars"	108
"lines"	6
"words"	32
Removing temp directory /var/folders/v2/7ykd39_d60zc410c_6slt53r0000gn/T/mr_word_count_mapper_reducer.1003604.20170628.022753.626414...
```

신기하다. 

하지만 내부 호출 구조가 직관적으로 눈에 들어오지 않는다. 

## Mapper

그래서 mapper를 다음과 같이 바꿔서 어떻게 도는지 봤다.

```python
    def my_mapper(self, _, line):
        print("chars: ", line, len(line))
        yield "chars", len(line)
        print("words: ", line, len(line.split()))
        yield "words", len(line.split())
        print("lines: ", line, 1)
        yield "lines", 1
```

실행해보면 다음과 같이 나온다.

```
python3 mr_word_count_mapper_reducer.py word-count.txt

No configs found; falling back on auto-configuration
Creating temp directory /var/folders/v2/7ykd39_d60zc410c_6slt53r0000gn/T/mr_word_count_mapper_reducer.1003604.20170628.023054.864351
Running step 1 of 1...
chars:  고요한 내 가슴에 나비처럼 날아와서 19
words:  고요한 내 가슴에 나비처럼 날아와서 5
lines:  고요한 내 가슴에 나비처럼 날아와서 1
chars:  사랑을 심어놓고 나비처럼 날아 간 사람 21
words:  사랑을 심어놓고 나비처럼 날아 간 사람 6
lines:  사랑을 심어놓고 나비처럼 날아 간 사람 1
chars:  내 가슴에 지울 수 없는 그리움 주고 간 사람 25
words:  내 가슴에 지울 수 없는 그리움 주고 간 사람 9
lines:  내 가슴에 지울 수 없는 그리움 주고 간 사람 1
chars:  그리운 내 사연을 뜬 구름아 전해 다오 21
words:  그리운 내 사연을 뜬 구름아 전해 다오 7
lines:  그리운 내 사연을 뜬 구름아 전해 다오 1
chars:  아아아 아아아아아 9
words:  아아아 아아아아아 2
lines:  아아아 아아아아아 1
chars:  사랑은 얄미운 나비인가봐 13
words:  사랑은 얄미운 나비인가봐 3
lines:  사랑은 얄미운 나비인가봐 1
Streaming final output from /var/folders/v2/7ykd39_d60zc410c_6slt53r0000gn/T/mr_word_count_mapper_reducer.1003604.20170628.023054.864351/output...
"chars"	108
"lines"	6
"words"	32
Removing temp directory /var/folders/v2/7ykd39_d60zc410c_6slt53r0000gn/T/mr_word_count_mapper_reducer.1003604.20170628.023054.864351...
```

yield로 반환되는 `chars`, `words`, `lines` 가 행 단위로 반복되며 실행되는 것을 알 수 있다.

위 결과로 유추할 수 있는 것은 아래와 같다.

>- mapper 내부의 코드는 행 단위로 실행된다.
>- mapper 내부에 있는 yield는 몇 개가 있더라도 행 단위로 모두 실행 된다.

이것만 알아도 파이썬 MR을 대충 짤 수 있을 것 같다.

하지만 위와 같이 호출되는 이유는 여전히 모른다. generator인 이 mapper를 누가 어떻게 호출하는지 모르므로 `yield`가 어떻게 호출되는지 모르기 때문이다.

MRJob의 소스(python3.6/site-packages/mrjob/job.py)에 답이 있다. 관련있는 부분의 코드는 다음과 같다.

```python
#... 생략 ...

    def run_mapper(self, step_num=0):
        #... 생략 ...

        # run the mapper on each line
        for key, value in read_lines():    # outer-for
            for out_key, out_value in mapper(key, value) or ():    # inner-for
                write_line(out_key, out_value)

        #... 생략 ...
        
#... 생략 ...

```

`read_lines()`를 실행하는 `outer-for`는 행 단위의 반복이라는 것은 쉽게 알 수 있다.

`inner-for`에서는 `read_lines()`의 실행 결과인 `key, value`를 인자로 해서 mapper 함수를 `inner-for` 문으로 반복해서 호출 한다. 

따라서 mapper 함수가 `inner-for`에 의해 반복 실행될 때마다 my_mapper 내부의 `yield`가 호출되며, 마지막 `yield`가 호출된 후에는 `inner-for` 의 반복 단위 하나가 종료되고, `outer-for`로 넘어가서 다음 행을 대상으로 다시 반복 된다.

결국 `outer-for`는 행 단위, `inner-for`는 `yield` 단위로 반복이 실행되는 것이고, 이제 왜 아래와 같이 출력되는지 알 수 있게 되었다.

```
chars:  고요한 내 가슴에 나비처럼 날아와서 19
words:  고요한 내 가슴에 나비처럼 날아와서 5
lines:  고요한 내 가슴에 나비처럼 날아와서 1
chars:  사랑을 심어놓고 나비처럼 날아 간 사람 21
words:  사랑을 심어놓고 나비처럼 날아 간 사람 6
lines:  사랑을 심어놓고 나비처럼 날아 간 사람 1
chars:  내 가슴에 지울 수 없는 그리움 주고 간 사람 25
words:  내 가슴에 지울 수 없는 그리움 주고 간 사람 9
lines:  내 가슴에 지울 수 없는 그리움 주고 간 사람 1
chars:  그리운 내 사연을 뜬 구름아 전해 다오 21
words:  그리운 내 사연을 뜬 구름아 전해 다오 7
lines:  그리운 내 사연을 뜬 구름아 전해 다오 1
chars:  아아아 아아아아아 9
words:  아아아 아아아아아 2
lines:  아아아 아아아아아 1
chars:  사랑은 얄미운 나비인가봐 13
words:  사랑은 얄미운 나비인가봐 3
lines:  사랑은 얄미운 나비인가봐 1
```

정리하면 mapper의 파라미터로 넘겨지는 한 행의 레코드에 대해 mapper 안에 있는 yield이 모두 실행되며, 이는 mapper뿐만 아니라, combiner나 reducer도 마찬가지다.

## Reducer

reducer도 조금 더 살펴보면, 일단 다음과 같이 reducer는 mapper와는 다르게 key 하나에 여러 개의 value가 연결된다는 걸 유추할 수 있다.

```
    def my_mapper(self, _, line):
        yield "chars", len(line)
        yield "words", len(line.split())
        yield "lines", 1
        
    #... 생략 ...

    def my_reducer(self, key, values):  # key와 함께 value가 아니라 value's'가 온다.
        yield key, sum(values)
```

그래서 maper 관련 python3.6/site-packages/mrjob/job.py 파일을 보면 다음과 같이 되어 있다.

```
  def run_reducer(self, step_num=0):
  
  #... 생략 ...
  
        # group all values of the same key together, and pass to the reducer
        #
        # be careful to use generators for everything, to allow for
        # very large groupings of values
        for key, kv_pairs in itertools.groupby(read_lines(),
                                               key=lambda k_v: k_v[0]):
            values = (v for k, v in kv_pairs)
            for out_key, out_value in reducer(key, values) or ():
                write_line(out_key, out_value)

    #... 생략 ...
```

즉, mrjob 라이브러리 수준에서 먼저 `itertools.groupby(read_lines(), key=lambda k_v: k_v[0])`로 key를 기준으로 key에 해당하는 값만을 values라는 Iterable로 만든 후에, 개발자가 작성한 reducer 함수를 `reducer(key, values)`와 같이 호출한다.

다시 정리하면, 개발자가 작성한 mapper에서 key, value로 반환한 여러 행의 데이터를 mrjob 라이브러리가 아래와 같이 key 기준으로 그룹화 한 후에 개발자가 작성한 reducer를 호출한다.

```
# mapper가 반환한 여러 행의 데이터
key0, value0-0
key1, value1-0
key1, value1-1
key0, value0-1
key1, value1-2
...

# reducer에 파라미터로 넘겨지는 데이터
<key0, {value0-0, value0-1, value0-2, ...}>
<key1, {value1-1, value1-1, value1-2, ...}>
```

key, value's'에 있어서는 combiner도 reducer와 같다.

## 정리

>- 하나의 레코드를 <k, v>로 표현하면
>- mapper는 
>    - <k0, v0-0>
>    - <k1, v1-0>
>    - <k1, v1-1>
>    - <k0, v0-1>
>    - <k0, v0-2>
>    - <k1, v1-2>
>    - ... 단위의 레코드 단위로 실행되고,
>- combiner와 reducer는 
>    - <k0, {v0-0, v0-1, v0-2, ...}>
>    - <k1, {v1-0, v1-1, v1-2, ...}>
>    - ... 단위의 레코드 단위로 실행된다.
>
>- mapper, combiner, reducer 내부에 존재하는 다수의 yield문은 파라미터로 전달받은 하나의 레코드에 대해 모두 실행된다.
