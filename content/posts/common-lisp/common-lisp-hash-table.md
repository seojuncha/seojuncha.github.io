---
title: "Common Lisp의 Hash Table 이해하기"
date: 2025-03-14
tags: [Common Lisp, Common Lisp Hash Table]
description: ""
---

최근 다시 조금씩 From the transistor 프로젝트의 컴파일러 만들기를 진행하면서 C언어에서 사용하는 키워드 저장을 위해 [**해시 테이블(Hash Table)**](https://en.wikipedia.org/wiki/Hash_table)이 필요한 상황이 있었습니다. Common Lisp에서 **해시 테이블은 키-값(key-value) 쌍을 저장하는 효율적인 데이터 구조**입니다. 배열이나 리스트와 달리, 해시 테이블은 **키를 기반으로 값을 빠르게 검색**할 수 있도록 설계되었습니다.

이 글에서는 Common Lisp의 해시 테이블을 생성하고 활용하는 방법을 설명합니다.

## 1. Hash Table 생성하기 (`make-hash-table`)

Common Lisp에서 해시 테이블을 생성하려면 `make-hash-table` 함수를 사용합니다.

```lisp
(defparameter *my-table* (make-hash-table))
```

이렇게 생성된 해시 테이블은 기본적으로 **eq 테스트**를 사용하여 키를 비교합니다. 하지만 다른 비교 방법을 사용하고 싶다면 `:test` 옵션을 지정할 수 있습니다.

```lisp
(defparameter *my-table-equal* (make-hash-table :test 'equal))
```

### 비교 방법 옵션

- `eq` (기본값): 객체의 동일성(identity) 비교
- `eql`: 숫자와 문자도 비교 가능
- `equal`: 리스트나 문자열 비교
- `equalp`: 대소문자 무시 및 구조 비교

각 비교 방법에 대한 자세한 설명은 [Common Lisp에서의 Equality 이해하기](/posts/common-lisp/equality-predicate-in-common-lisp) 포스팅에서 더욱 자세하게 확인할 수 있습니다.


## 2. 데이터 검색 및 삽입
### 데이터 검색 (`gethash`)

`gethash` 를 사용하여 값을 가져올 수 있습니다.

```lisp
(gethash "key1" *my-table*) ; => "value1"
(gethash "key2" *my-table*) ; => 42
(gethash "key3" *my-table*) ; => NIL
```

`gethash` 접근자(Accessor)는 특정 키의 값을 반환하는 것 뿐아니라 키의 존재유무 및 기본값도 설정할 수 있는 기능을 제공합니다.  
키의 존재 유무를 확인하기 위해선 `gethash` 두번째 반환값 predicate를 사용할 수 있습니다. 

`multiple-value-bind` 매크로를 사용해서 두개의 반환값을 확인해 보겠습니다. 

```lisp
(multiple-value-bind 
  (value present)   ; 평가식 반환값을 바인딩할 변수
  (gethash "key1" *my-table*)  ; 평가식
  (if present   ; 바인딩 변수를 활용한 표현식
    (format t "There is ~a.~%" value)
    (format t "There isn't ~a.~%" value)))
```
하지만, `(gethash "key3" *my-table*)` 를 사용해 `key3`을 조회한 것 처럼 할당되지 않은 키값에 대해서는 `NIL`을 반환하기 때문에 단순히 *`gethash`만을 사용해서 키의 존재유무를 확인할 수 있지 않을까*라는 의문을 가질 수 있습니다.

`gethash`로 키의 존재유무를 확인할 경우의 문제점을 아래의 예제를 사용해서 비교해 보겠습니다. 

```lisp
;;; gethash를 if문의 조건식으로 사용 
(defun check-from-gethash (key tabl용)
  (if (gethash key table)
    (format t "exist~%")
    (format t "not exist~%")))

;;; multiple-value-bind와 함께 predicate를 if문의 조건식으로 사용
(defun check-from-value-bind (key table)
  (multiple-value-bind (value present) (gethash key table)
    (if present
      (format t "exist~%")
      (format t "not exist~%"))))

;;; 해시테이블 생성
(defparameter *my-table* (make-hash-table))
;;; key1에 nil할당
(setf (gethash 'key1 *my-table*) nil)
;;; key2에 nil할당
(setf (gethash 'key2 *my-table*) nil)

;;; key1과 key2, 두개의 키를 가짐
(format t "size: ~a~%" (hash-table-count *my-table*))
회
;;; 동일한 키(key1)을 각각 다른 방법으로 조회
(check-from-gethash 'key1 *my-table*)
(check-from-value-bind 'key1 *my-table*)
```

**결과**
```shell
size: 2
not exist
exist
```

`*my-table*`에는 두개의 키가 있지만 모두 `NIL`을 바인딩했습니다. 즉, 키가 존재하지 않는 것이 아니라 키는 존재하지만 해당 키와 쌍이 되는 값이 `NIL`인 것 뿐입니다. 그렇기 때문에, `gethash`의 첫번째 반값값으로는 키 자체의 존재재무를 정확히 확인할 수가 없습니다. 

### 데이터 삽입 (`setf` + `gethash`)

해시 테이블에 값을 추가하려면 `setf`와 `gethash`를 함께 사용합니다.

```lisp
(setf (gethash "key1" *my-table*) "value1")
(setf (gethash "key2" *my-table*) 42)
```
> `setf`는 [Common Lisp의 변수할당](/posts/common-lisp/variable-assignment-in-common-lisp/)에 관한 포스팅에서 다룬 `setq`의 확장된 버전입니다. 

다른 언어에 비해서 조금 복잡해보일 수도 있을 것 같습니다. 
언어적으로 해시맵을 지원하는 C++나 파이썬과 같은 언어에서는 대괄호를 사용해서 해시맵의 키에 쉽게 접근할 수 있습니다.

```python
my_dic = dict()
my_dic["key1"] = 1
my_dic["key2"] = 2
```

```c++
#include <map>
std::map<std::string, int> my_map;
my_map["key1"] = 1;
my_map["key2"] = 2;
```

`gethash`의 역할이 타 언어에서의 대괄호 역활과 유사하다고 생각하면 좋을 것 같습니다. 


## 3. 데이터 삭제

### 데이터 삭제 (`remhash`)

특정 키를 삭제하려면 `remhash`를 사용합니다.

```lisp
(remhash "key1" *my-table*)
```
---

## 4. Hash Table 순회 (`loop` 및 `maphash`)
해시 테이블의 모든 키-값을 순회하려면 `maphash` 또는 `loop`를 사용할 수 있습니다.

### `maphash` 사용

```lisp
(maphash (lambda (key value)
           (format t "~A => ~A~%" key value))
         *my-table*)
```

### `loop` 사용

반복문을 위한 `loop`의 사용법은 [이전 포스팅](/posts/common-lisp/common-lisp-loop/)에서 언급했죠. 

```lisp
(loop for key being the hash-keys of *my-table*
      using (hash-value value)
      do (format t "~A => ~A~%" key value))
```

## 5. Hash Table의 크기 및 초기화

### 크기 확인 (`hash-table-count`)

해시 테이블의 키 개수를 얻으려면 `hash-table-count`를 사용합니다.

```lisp
(hash-table-count *my-table*)
```

### 초기화 (`clrhash`)

해시 테이블을 초기화(모든 데이터 삭제)하려면 `clrhash`를 사용합니다. 

```lisp
(clrhash *my-table*)
```

> 초기화를 하면 모든 키-값이 삭제되고 크기가 0으로 초기화 될 뿐, 해시테이블에 바인딩된 변수 자체가 삭제되지는 않습니다.  
> (추천하지는 않는 방법이지만) 그렇기 때문에, 해시테이블에 바인딩되었던 변수에 다른 객체(예: 문자열, 숫자 등)을 새로 바인딩하는 것도 가능합니다.

## 결론
해시 테이블은 효율적인 데이터 저장과 빠른 검색이 필요한 경우 유용하게 사용할 수 있습니다. Common Lisp에서도 다른 현대 언어들 처럼 언어 자체적으로 지원하는 해시 테이블이 있기 때문에 쉽게 해시 테이블을 사용할 수 있습니다. `:test`옵션과 함께 키를 비교하는 방법을 직접 연습해본다면 Common Lisp의 키 비교 방식에 대해서 더욱 정확히 이해할 수 있을 것 같습니다.  
