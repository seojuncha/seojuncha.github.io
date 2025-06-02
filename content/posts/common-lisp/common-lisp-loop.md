+++
date = '2025-02-24'
title = "Common Lisp의 기초적인 loop 활용"
tags = ["common lisp", "common lisp loop"]
description = "Common Lisp에서 loop 문을 활용하여 반복문을 쉽게 구현하는 방법을 알아봅니다. 기본적인 사용법부터 리스트 순회, 범위 지정, 조건문과 함께 사용하는 방법까지 다양한 예제를 통해 익혀보세요."
+++

Common Lisp에서 반복문을 작성하는 방법은 여러 가지가 있지만, 그중에서도 `loop` 문은 강력하면서도 직관적인 구문을 제공합니다. 이번 글에서는 Common Lisp의 `loop` 문을 활용하는 기초적인 방법을 예제와 함께 살펴보겠습니다.

## 기본적인 `loop` 사용법
`loop`문은 다양한 하위 조항(subclause)들을 조합하여 반복횟수, 지역변수 할당, 조건 분기 등의 동작이 가능하지만,
가장 간단한 형태의 `loop`문은 무한 루프 입니다.

```lisp
(loop
  (format t "this is loop!~%"))
```

출력:
```shell
this is loop!
this is loop!
this is loop!
...
# 종료: Ctrl+c
```

### 반복 횟수 지정하기 (`repeat`)
무한대의 횟수로 반복하는 것이 아니라 특정 횟수만큼만 반복하기 위해선 `repeat`을 사용합니다.

**문법**
- `loop repeat` *반복횟수* `do`

**예제**
```lisp
(loop repeat 5 do
  (format t "this is loop!~%"))
```
**출력**
```shell
this is loop!
this is loop!
this is loop!
this is loop!
this is loop!
```

## 리스트 순회 (`for...in`)
**문법**
- `loop for` *리스트 아이템 변수* `in` *리스트* `do`

**예제**
```lisp
(loop for item in '(1 2 3) do
  (format t "~a~%" item))
```

**출력**
```shell
1
2
3
```

## 숫자 범위 반복 (`for...from/to/by`)
**문법**
- `loop for` *변수* `from` *변수의 시작 값* `to` *변수의 종료 값* `do`

**예제**
```lisp
(loop for i from 0 to 5 do
  (format t "~d~%" i))
```
**출력**
```shell
0
1
2
3
4
5
```

`downto`를 사용하여 숫자의 증가 뿐아니라 감소도 가능합니다.

```lisp
(loop for i from 5 downto 0 do
  (format t "~d~%" i))
```

출력:
```shell
5
4
3
2
1
0
```
### 증가/감소 값 정의 (`by`)
리스트 순회 값의 증가/감소는 기본적으로 1씩 증가 혹은 감소합니다. 하지만 `by` 키워드를 사용하여 증가/감소 값을 정의할 수 있습니다.

```lisp
(loop for i from 0 to 5 by 2 do
  (format t "~d~%" i))
```
출력:
```shell
0
2
4
```
## 종료 조건 지정하기 (`while`, `always`, `thereis`)
앞서 살펴본 `repeat`은 특정 횟수만큼 반복한다는 의미를 가지지만, 결국 종료 조건을 지정하는 것과 같습니다.
`loop` 문에서 사용할 수 있는 종료조건에는 여러 가지가 있으며, 대표적인 몇 개를 살펴보겠습니다.

### 특정 조건을 만족하는 동안 반복 (`while`)
일반적으로 우리가 알고 있는 `while`문의 동작과 동일합니다. 조건을 만족할 때 반복하고, 조건을 만족하지 않으면(`NIL`을 반환하면) 반복을 종료합니다.

**문법**
  - (loop while <조건> do <반복 실행 코드>)

**설명**
  - <조건>이 NIL일 때, 반복문 종료

**예제**
```lisp
(defparameter *number* 5)   ; 전역변수 *number*에 5 할당
(loop while (< *number* 10) do   ; *number*가 10 이상이면 종료
  (format t "~d~%" *number*)
  (incf *number*))   ; *number* 1 증가
```
**출력**
```shell
5
6
7
8
9
```

### 모든 반복에서 조건 검사 (`always`)
앞서 살펴본 반복구문과 달리, `do`를 함께 사용하지 않습니다. 그렇기 때문에, 매 반복마다 조건 만족 여부를 검사하고
모든 조건이 통과 되었을 때, `T`를 반환하고 그렇지 않으면 `NIL`을 반환하는 동작입니다. 

**문법**
- (loop always <조건>)

**예제**
```lisp
(loop for i in '(2 4 6 8 10)
  always (evenp i))
```

### 특정 값 찾기 (`thereis`)
반복문에서 특정 값을 찾고, 찾는 즉시 해당 값을 반환할 때 사용합니다.

**문법**
- (loop thereis <값>)

**예제**
```lisp
(let ((check-ret (loop for i in '(1 3 5 7 8 9)
                    thereis (if (evenp i) i))))
  (format t "return : ~d~%" check-ret))
```
**출력**
```lisp
return: 8
```

## 조건문과 함께 사용하기 (`when`, `if`)
[이전 포스팅](/posts/common-lisp/how-to-make-decision-in-common-lisp/)에서 다루었던 조건문을 사용하여 반복문 내에서 특정 조건을 만족할 때만 동작하도록 할 수 있습니다.

**`when` 예제**
```lisp
(loop for i from 0 upto 6   ; i값이 0부터 6까지 반복
  when (evenp i) do    ; i가 짝수일 경우,
    (format t "~d is even~%" i))  ; <i값> is even 출력
```
**출력**
```shell
0 is even
2 is even
4 is even
6 is even
```
**`if` 예제**
```lisp
(loop for i from 1 to 10 do
  (if (oddp i)
    (format t "~A is odd.~%" i)
    (format t "~A is even.~%" i)))
```
**출력**
```shell
1 is odd.
2 is even.
3 is odd.
4 is even.
5 is odd.
6 is even.
7 is odd.
8 is even.
9 is odd.
10 is even.
```

## 결론
Common Lisp의 `loop`문은 반복문을 보다 직관적으로 사용할 수 있도록 다양한 기능을 제공합니다.
- `repeat`로 특정 횟수만큼 반복
- `for...in`으로 리스트 순회
- `for...from/to/by`로 범위 기반 반복
- `when`,`if`를 사용한 조건 분기

하지만, `C` 나 `Python`과 같은 언어에서 제공하는 `for`나 `while`기반의 반복문을 주로 사용해왔다면 익숙해지는데 조금 시간이 걸릴지도 모릅니다. [HyperSpec문서](https://www.lispworks.com/documentation/HyperSpec/Body/m_loop.htm#loop)를 참고하여 다양한 반복문을 조합하는 연습을 해보세요!
