+++
date = '2025-02-25'
title = "Common Lisp에서의 Equality 이해하기"
tags = ["common lisp", "common lisp equality"]
description = "Common Lisp의 eq, eql, equal, equalp 차이를 이해하고, 적절한 동등성 비교 연산자를 선택하는 방법을 알아보세요."
+++

## 들어가며
**Equality(동등성) 비교**는 모든 프로그래밍 언어에서 중요한 개념이지만, **Common Lisp**에서는 다양한 **동등성 비교 연산자\(equality predicates\)**가 존재하기 때문에 이를 올바르게 사용하는 것이 중요합니다. 이 글에서는 **Common Lisp에서 제공하는 네 가지 주요 동등성 비교 연산자**를 살펴보고, 각각의 특징과 사용 사례를 설명하겠습니다.

## Common Lisp의 Equality 종류

Common Lisp은 다음과 같은 네 가지 주요 비교 연산자를 제공합니다:

1. **`eq`** – 객체 참조 비교(Reference Equality)
2. **`eql`** – `eq`에 숫자와 문자 비교 기능 추가
3. **`equal`** – 리스트와 문자열의 구조적 동등성 비교
4. **`equalp`** – 대소문자 무시 및 숫자 타입 유연한 동등성 비교

각각의 연산자가 어떻게 동작하는지 자세히 알아보겠습니다.

---

### `eq` – 참조 동일성(Reference Equality)
```lisp
(eq 'a 'a) ;; ⇒ T
(eq (list 1 2) (list 1 2)) ;; ⇒ NIL
(eq 100 100) ;; ⇒ 구현에 따라 다름, sbcl은 T
(eq "hello" "hello") ;; ⇒ 구현에 따라 다름, sbcl은 NIL
```
- `eq`는 두 값이 **동일한 메모리 주소**를 가리키는지 확인합니다.
- **심볼\(Symbol\)**은 Lisp이 **인터닝\(interning\)**을 수행하기 때문에 `eq` 비교가 신뢰할 수 있습니다.
- 하지만 **숫자와 문자열의 비교 결과는 구현마다 다를 수 있습니다**.

> **인터닝(interning)이란?**
> 
> 컴파일러나 런타임 시스템이 동일한 값을 가진 객체를 하나의 메모리 위치에 저장하고, 해당 위치를 공유하도록 하는 최적화 기법입니다. 즉, **같은 이름**의 심볼(Symbol)은 하나의 메모리에 저장되기 때문에 각각 동일한 메모리 위치를 가지게 됩니다. 

---

### `eql` – `eq` + 숫자 및 문자 비교 지원
```lisp
(eql 100 100) ;; ⇒ T
(eql #\A #\A) ;; ⇒ T
(eql (list 1 2) (list 1 2)) ;; ⇒ NIL
```
- `eql`은 `eq`와 동일하게 작동하지만, **숫자와 문자(Character)** 비교를 정확하게 수행합니다.
- **동일한 값과 타입을 가진 숫자**는 `eql`로 비교할 수 있습니다.
- **같은 문자인 경우** `eql`이 `T`를 반환합니다.

---

### `equal` – 리스트와 문자열의 구조적 동등성 비교
```lisp
(equal '(1 2 3) '(1 2 3)) ;; ⇒ T
(equal "hello" "hello") ;; ⇒ T
(equal #(1 2 3) #(1 2 3)) ;; ⇒ NIL (다른 배열 인스턴스)
```
- `equal`은 **리스트를 재귀적으로 비교**합니다.
- **문자열을 한 글자씩 비교**하여 동등성을 판단합니다.
- 하지만 **배열(Array)과 숫자 타입이 다른 경우에는 `equal`이 `NIL`을 반환할 수 있습니다**.

---

### `equalp` – 대소문자 무시 및 숫자 타입 차이 허용
```lisp
(equalp 1 1.0) ;; ⇒ T
(equalp "hello" "HELLO") ;; ⇒ T
(equalp #(1 2 3) #(1 2 3)) ;; ⇒ T
```
- `equalp`는 **문자열을 대소문자 구분 없이 비교**합니다.
- `1`과 `1.0`처럼 **다른 타입이지만 값이 동일한 숫자**를 동일하다고 판단합니다.
- `equal`과 달리 **배열(Array)도 요소별로 비교하여 동등성을 판단**합니다.

---

### 적절한 Equality Predicate 선택법

| 연산자  | 사용 사례 |
|------------|----------|
| `eq`       | 두 객체가 동일한 메모리를 가리키는지 확인 (특히 심볼 비교에 적합). |
| `eql`      | `eq`와 유사하지만 숫자와 문자 비교를 정확하게 수행. |
| `equal`    | 리스트와 문자열을 구조적으로 비교. |
| `equalp`   | 대소문자를 무시한 문자열 비교, 숫자 타입 차이를 허용한 비교. |

> 표의 위에 있는 연산자일수록 더욱 엄격한 비교를 수행합니다. 

---
### 참고: 각 연산자별 반환값 확인을 위한 코드
```lisp
(defun check (func)
  (let ((ret func))
    (format t "return: ~a~%" ret)))

(format t "eq~%")
(check (eq 'a 'a))
(check (eq 100 100))
(check (eq 100 100.0))
(check (eq #\a #\a))
(check (eq #\a #\A))
(check (eq "aa" "aa"))
(check (eq (list 1 2) (list 1 2)))

(format t "eql~%")
(check (eql 'a 'a))
(check (eql 100 100))
(check (eql 100 100.0))
(check (eql #\a #\a))
(check (eql #\a #\A))
(check (eql "aa" "aa"))
(check (eql (list 1 2) (list 1 2)))


(format t "equal~%")
(check (equal 'a 'a))
(check (equal 100 100))
(check (equal 100 100.0))
(check (equal #\a #\a))
(check (equal #\a #\A))
(check (equal "aa" "aa"))
(check (equal (list 1 2) (list 1 2)))

(format t "equalp~%")
(check (equalp 'a 'a))
(check (equalp 100 100))
(check (equalp 100 100.0))
(check (equalp #\a #\A))
(check (equalp "aa" "aa"))
(check (equalp (list 1 2) (list 1 2)))
```

## 결론
Common Lisp의 여러 동등성 비교 연산자를 이해하는 것은 **예측 가능한 코드 작성과 성능 최적화**에 필수적입니다. 

- **객체 참조 비교**는 `eq`를 사용하세요.
- **숫자 및 문자 비교**는 `eql`이 적절합니다.
- **리스트 및 문자열 비교**는 `equal`을 사용하세요.
- **대소문자 무시, 숫자 타입 차이 허용 비교**는 `equalp`가 유용합니다.

각 연산자의 특성을 잘 활용하면 보다 **안정적이고 효율적인 Lisp 프로그램**을 작성할 수 있습니다.