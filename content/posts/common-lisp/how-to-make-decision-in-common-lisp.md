+++
date = '2025-02-13'
title = "Common Lisp 조건문 완벽 가이드: when, unless, if, cond 활용법"
tags = ["common lisp", "common lisp if", "common lisp if-else", "common lisp cond", "common lisp when", "common lisp unless"]
description = "Lisp에서 조건문을 활용하는 방법을 알아봅니다. when, unless, if, cond 등의 분기문을 사용하여 논리 흐름을 제어하고, 반환 값을 활용하는 방식까지 자세히 설명합니다. 이를 통해 가독성 높은 코드 작성과 오류 방지를 위한 최적의 조건문 사용법을 익혀보세요."
+++

오늘은 프로그래밍 논리 흐름을 제어하기 위한 **분기문의 사용법**을 알아보겠습니다. 분기문(Branch Statements)은 특정 조건의 **참 혹은 거짓** 여부를 판단하여 다음 행동을 결정합니다. Common Lisp(이하, Lisp)에서도 조건(Condition)검사를 위한 다양한 방식을 제공하고 있기 때문에, 각각의 활용법을 익히고 적절하게 사용하면 **코드 가독성 향상**과 **오류를 최소화**하는데 도움이 됩니다.

## 참(True) 혹은 거짓(False): `t` & `nil`
대부분의 프로그래밍 언어와 같이, 조건 검사는 참/거짓을 판단하는 것 입니다.  
예를 들어, 파이썬은 아래와 같이 `True` 혹은 `False`를 사용하여 참/거짓을 구분합니다.

```python
is_true = True
is_false = False
```

Lisp 또한 참/거짓을 구분할 수 있는데, 다른 언어와의 차이점은 명시적인 `bool` 타입을 가지지 않는다는 것 입니다.  
대신에, 참을 표현하는 `t`와 거짓, 그리고 null 객체,를 표현하기 위한 `nil`이 있습니다.

```lisp
(defparameter *is-true* t)
(defconstant +empty+ nil)
```

## 하나의 조건 만족 검사하기: `when`, `unless`
가장 간단한 조건 검사부터 살펴보겠습니다. Lisp에서는 단 하나의 조건 만족 여부를 검사하기 위해 `when`과 `unless`라는 매크로를 제공합니다. 

- [`when`|`unless`] *test-form* form* => result*

`test-form`은 참 혹은 거짓으로 판별되는 표현식이고, `form*`은 조건을 만족할 때 실행하는 표현식 입니다.

예를 들어, 아래와 같이 사용할 수 있습니다.
```lisp
(defconstant +pi+ 3.14)
(defparameter *my-value* +pi+)

;; (= *my-value* +pi+)는 참(t) => while실행 조건 만족
(when (= *my-value* +pi+)
  (format t "my-value is PI!~%"))

;; 전역변수 *my-value*값 변경
(setq *my-value* 3)

;; (= *my-value* +pi+)는 거짓(nil) => unless실행 조건 만족
(unless (= *my-value* +pi+) 
  (format t "my-value is not PI!~%"))
```

> Lisp의 다른 키워드 심볼들 처럼 `when`과 `unless` 역시 영어문장으로 표현했을 때 자연스로운 문장이 되도록 읽을 수 있어 직관적입니다.  

파이썬에서는 `if` 혹은 `if not`을 사용한 방식과 동일합니다.

```python
pi = 3.14
my_value = pi

if pi == my_value:
  print("my_value is PI!")

my_value = 3
if not pi == my_value:
  print("my_value is not PI!")
```


### 기본적인 숫자 비교 연산자: `=`, `/=`, `>`, `<`, `>=`, `<=`
다음으로 넘어가기 전에, 기본적인 숫자 비교 연산자들을 간단히 알아보겠습니다.   
Lisp에서는 숫자 비교를 위한 6개의 연산자를 제공합니다.
- `=`
  - 두 숫자가 같은 값인지 확인
- `/=`
  - 두 숫자가 서로 다른 값인지 확인
- `>`, `<`, `>=`, `<=`
  - 두 숫자의 크기를 비교

위 연산자들을 사용할 때 주의해야 할 점은, 모두 **수학적 의믜**의 동일성과 크기비교를 한다는 것 입니다.

```lisp
(= 10 10)    ; t
(= 10 10.0)  ; t
(eql 10 10.0) ; nil (정수와 실수는 다른 타입)
(/= 4 8/3)   ; nil, 4와 8/3(=2.666..)은 같지 않음
```
즉, `=`은 수학적 동등성을 비교하지만 `eql`은 데이터 타입까지 고려한 비교를 수행합니다.

> `eql` 이나 `equalp`, 혹은 문자나 문자열을 비교하기 위한 연산자들은 다른 포스팅에서 알아보겠습니다.

## 조건 만족/불만족 함께 검사하기: `if`

파이썬의 `if-else` 문과 동일한 동작을 합니다.

```python
num = 10
if num < 10:
  print("num is less than 10")
else:
  print("num is larger than or equal to 10")
```

위 파이썬 코드를 Lisp으로 표현하면 아래와 같습니다.

```lisp
(defparameter *number* 10)
(if (< *number* 10)   ; 조건식 (< *number* 10)은 nil
  (format t "number is less than 10~%")  ; 조건식의 결과가 t일 때 실행
  (format t "num is larger than or equal to 10~%"))  ; 조건식의 결과가 nil일 때 실행
```
예제코드 처럼 Lisp에서는 `if` *연산자*를 사용했을 때 참/거짓 각각에 대한 조건을 검사할 수 있습니다.

하지만, `if`연산자의 **`else`절을 의미하는 표현식은 필수가 아닌 선택사항**입니다. 예를 들어,
```lisp
(if t (format t "true"))
```
위와 같은 코드는 `else`에 해당하는 표현식이 없기 때문에 `if-then`의 방식으로 동작합니다. 

### 홀수/짝수 판별 함수: `evenp` & `oddp`
Lisp에서 제공하는 또 다른 편의함수인 `evenp`와 `oddp`는 각각, 하나의 숫자가 짝수인지 홀수인지 판별하고 `bool`타입(`t` 혹은 `nil`)을 반환합니다.

```lisp
(let ((is-even (evenp 4))   ; 짝수이므로 t 반환
      (is-odd (oddp 7)))    ; 홀수이므로 t 반환
  (format t "is-even: ~a , is-odd: ~a~%" is-even is-odd))
```
`let`을 사용해서 `is-even`과 `is-odd`, 2개의 [지역변수를 생성](/posts/variable-assignment-in-common-lisp/)하였습니다. 숫자 `4`와 숫자 `7`은 각각 짝수와 홀수 이므로 2개의 변수에 모두 참(t)을 할당합니다.

> `evenp`와 `oddp`의 마지막 문자인 **p** 는 관례적으로 *predicate*를 표현합니다.


## 여러 조건을 검사하기: `cond`
`cond`는 다른 프로그래밍 언어에서 사용하는 일반적인 `if-else if-else`문과 동일하게 다수의 조건을 검사할 수 있습니다.

예를 들어, 특정 숫자의 범위를 확인하기 위한 파이썬 코드는 아래와 같이 작성할 수 있습니다.
```python
number = 8

if number < 0:
  print("is negative")
elif number >= 0 and number < 10:
  print("is positive and less then or equal to 10")
else:
  print("is larger than 10")
```

동일한 논리를 Lisp의 `cond`를 사용하면 아래와 같이 작성할 수 있습니다.

```lisp
(let ((number 8))
  (cond 
    ((< number 0) 
      (format t "is negative~%"))
    ((and (>= number 0) (< number 10)) 
      (format t "is positive and less then or equal to 10~%"))
    (t    ; 항상 참이므로, 모든 조건이 거짓일 때 실행
      (format t "is larger than 10~%"))))
```
`cond`에서 마지막 `(t ...)` 부분은 `else`의 역할을 하며, 위의 모든 조건이 만족하지 않을 때 실행됩니다.

## 모든 조건식은 반환값이 있다.
조건식들의 사용자체는 어렵지 않지만, 한가지 유의해야할 점은 `when`과 `if`, `cond`와 같은 모든 조건식들은 **마지막에 평가된 표현식의 값을 반환**한다는 것 입니다.

```lisp
(when t
  (format t "Hello, World!~%")
  42)  ; => 42를 반환
```
이런 특성을 사용하면 특정 조건에 따른 변수의 값을 설정하는데 활용할 수 있습니다.

```lisp
(defparameter *is-set* t)  ; 전역변수 *is-set*에 t할당
(let ((check-set (when *is-set* 99)))     ; *is-set*은 참이므로 99를 반환, check-set은 99로 초기화
  (format t "check-set: ~a~%" check-set)) ; check-set은 99를 출력
```
이런 점은 Lisp이 단순히 분기 명령을 위한 조건문이 아닌, **값 자체를 반환하는 표현식**이라는 문법적인 특성을 나타냅니다.


## 결론
Lisp에서 제공하는 다양한 조건문을 살펴보았습니다. `when`과 `unless`를 사용하면 단일 조건에 대한 간결한 표현이 가능하고, `if`는 참과 거짓에 따라 두 가지 실행 경로를 만들 수 있습니다. 또한, 여러 조건을 한 번에 평가해야 할 경우 `cond`를 활용하면 보다 깔끔한 논리 흐름을 만들 수 있습니다.

이러한 분기문을 적절히 조합하면 코드의 가독성을 높이고, 불필요한 중첩을 줄여 보다 직관적인 프로그램을 작성할 수 있습니다. 특히, Lisp에서는 `t`와 `nil`을 기반으로 참/거짓을 판단하기 때문에 조건문을 사용할 때 이러한 특성을 고려하는 것이 중요합니다.
