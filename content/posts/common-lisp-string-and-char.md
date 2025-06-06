+++
date = '2025-02-17'
title = "Common Lisp 문자와 문자열 완벽 가이드 – 문자 비교, 변환, 문자열 조작 방법 정리"
tags = ["common lisp", "common lisp string", "common lisp character"]
description = "Common Lisp의 문자와 문자열을 다루는 방법을 자세히 설명합니다. 문자 표현, ASCII 변환, 문자열 비교, 부분 문자열, 숫자 변환, 문자열 조작까지 다양한 기능을 예제와 함께 알아보세요."
+++

Common Lisp에서 **문자는 `#\` 접두사를 사용**하여 표현되며, ASCII 및 유니코드 문자 집합을 포함하여 다양한 문자를 다룰 수 있습니다. 문자는 **self-evaluating 객체**이므로 `quote` 없이도 사용 가능하며, **공백(#\space)과 줄바꿈(#\newline) 같은 특수 문자도 제공**합니다. 문자 비교를 위한 다양한 함수가 제공되며, 대소문자 구분 여부에 따라 `char=`와 `char-equal`을 선택할 수 있습니다. 또한, 문자를 ASCII 코드로 변환하거나, 반대로 숫자를 문자로 변환하는 기능도 지원합니다.

문자열은 단순히 문자의 배열로, **큰따옴표(" ")로 감싸서 표현**됩니다. 문자열 비교는 문자 비교와 유사한 방식으로 이루어지며, 대소문자 구분 여부에 따라 적절한 함수를 선택해야 합니다. **문자열을 다루기 위한 다양한 함수(`length`, `subseq`, `concatenate`, `string-upcase`, `string-downcase`, `search` 등)가 제공**되며, `coerce`를 활용하여 **문자열과 리스트 간 변환**도 가능합니다. 또한, `parse-integer`를 사용하여 문자열을 숫자로 변환하거나, `princ-to-string`을 사용하여 숫자를 문자열로 변환할 수 있습니다.

## Common Lisp의 문자 표현
Common Lisp의 문자 앞에 `#\`를 붙여서 문자 객체를 나타냅니다. `#\`와 함께 아래의 94개 문자를 표현할 수 있습니다.

```
! " # $ % & ' ( ) * + , - . / 0 1 2 3 4 5 6 7 8 9 : ; < = > ? 
@ A B C D E F G H I J K L M N O P Q R S T U V W X Y Z [ \ ] ^ _ 
` a b c d e f g h i j k l m n o p q r s t u v w x y z { | } ~
```
Common Lisp에서 문자 또한 숫자와 동일하게 self-evaluating 객체입니다. 그렇기 때문에 `REPL`에서 괄호를 사용하지 않고 `#\a`를 입력해서 오류가 발생하지 않고, `quote`를 사용하지 않아도 됩니다.

### 공백과 줄바꿈
Common Lisp에서는 공백(Whitespace)과 줄바꿈(Newline)을 표현하기 위한 특별한 방법을 사용합니다.

- `#\space`
- `#\newline`

> `#\tab`, `#\linefeed`등의 특수 문자도 사용할 수 있지만, 모든 Common Lisp구현에서 제공할 의무는 없습니다. 따라서, 사용하는 Lisp구현체에 따라서 제공하는 것을 확인하고 사용해야 합니다!

C언어에서 Escape Sequence와 특정 문자 하나(예: `\t`, `\n`)을 조합해 사용하는 것과는 달리 명시적으로 어떤 동작을 하는지 정의하고 있습니다.

예를 들어, 아래와 같이 공백을 표준출력(stdout)에 출력할 수 있습니다.

```lisp
(format t "~a~a" #\space #\a)
```
**코드 실행 결과**
```
 a
NIL
```
`format`함수에서 두개의 포맷 지정자(Specifier)를 사용했을 때, 첫번째 `~a`에는 공백 문자, `#\space`로 인하여 공백 뒤에  `a`를 출력합니다.

> `#\space`와 `#\newline`의 space와 newline은 case-sensitive하지 않습니다. 즉, `#\space`와 `#\spACe`는 동일하게 공백으로 처리됩니다.

### 문자 비교 방법
문자를 사용해서 할 수 있는 것 중 하나는 역시 문자 비교입니다. 임의의 문자가 다른 임의의 문자보다 크거나 작거나 같음을 확인하는 것은 흔하게 발생할 수 있는 경우죠. 다만, Common Lisp에서는 [숫자비교](/posts/how-to-make-decision-in-common-lisp/)를 위해 사용했던 연산자(예: `=`등)를 동일하게 사용할 수 없습니다. 

하지만, 결국 같거나, 크거나, 작거나,를 확인해야 하는 것은 숫자와 동일합니다. 하나의 차이점은 **문자는 숫자와 달리 대소문자 구분 여부가 필요**합니다. 

| number | case-sensitive | case-insensitive |
|:-:|:-|:-|
|`=`|`char=`|`char-equal`|
|`/=`|`char/=`|`char-not-equal`|
|`>`|`char>`|`char-greaterp`|
|`<`|`char<`|`char-lessp`|
|`>=`|`char>=`|`char-not-lessp`|
|`<=`|`char<=`|`char-not-greaterp`|

예를 들어, 두 문자가 같은지 확인하기 위해선 아래의 두가지 방법을 사용할 수 있습니다.
```lisp
(char= #\a #\A)       ; NIL (`a`와 `A`는 다름)
(char-equal #\a #\A)  ; T
```

> 문자의 크기 비교는 다른 언어와 동일하게 [ASCII 테이블](https://commons.wikimedia.org/wiki/File:ASCII-Table-wide.svg) 값을 기준으로 정합니다.


### 문자와 숫자의 변환
문자의 비교와 함께 흔히 사용되는 또 하나의 상황은 문자와 숫자의 상호 변환입니다. 

#### 유니코드(ASCII)값으로 변환
C언어가 일반적인 `char`타입의 문자를 처리하는 방식과 동일합니다. `char`타입의 변수는 해당 문자의 ASCII코드값과 동일한 의미를 가집니다.

예를 들어, C언어에서 문자 `A`는 ASCII코드 값으로 `65`를 나타냅니다. 
```c
printf("%d", 'A');    // 65 출력
```

하지만, Common Lisp에서는 숫자객체와 문자객체를 다른 유형의 객체로 구분하고 있기 때문에, `char-code`함수를 사용하여 명시적으로 변환할 수 있습니다.
```lisp
(char-code #\A)    ; 65 출력
```

아래와 같이 외우면 편할 것 같습니다.
```
(char-code)
char : Characeter
-    : to
code : ASCII code
```

#### 숫자를 문자로 변환하기
이번에는 반대로 숫자(숫자객체)를 문자로 변환하는 방법을 알아보겠습니다. 숫자를 문자로 변환할 때는 `code-char`함수를 사용합니다.

> `(char-code)`와 정확히 반대인 함수명입니다!

```lisp
(code-char 65)    ; #\A 출력
```

#### 숫자 문자(0~9)를 숫자로 변환
특정 문자에 대한 ASCII코드 값이 아닌 숫자문자 자체를 숫자로 변환하는 경우도 흔하게 발생하는 시나리오 입니다.
Common Lisp에서는 C언어의 `atoi`함수와 유사한 `digit-char-p`함수를 제공합니다.

```lisp
(digit-char-p #\1)  ; 1 출력
(digit-char-p #\a)  ; NIL 출력, 문자 #\a는 숫자문자가 아님
```
다만, 앞서 사용한 `char-code`나 `code-char`와 다르게 prefix `p`가 나타내듯이 숫자 문자가 아닌경우에는 `NIL`을 반환합니다. 그렇기 때문에, 숫자 문자를 숫자로 변환하는 것 뿐 아니라 특정 문자가 숫자인지를 확인하는 용도로도 사용가능하죠.

```lisp
(let ((is-digit (digit-char-p #\a)))
  (if is-digit 
    (format t "is digit~%")
    (format t "is not digit~%")))
;; is not digit 출력
```

> 숫자를 숫자 문자로 변환하는 방법은 아래에 문자열 설명에 이어서 나올 예정입니다!

## Common Lisp의 문자열 표현
Common Lisp의 문자열은 다른 언어에서의 문자열 표현과 동일하게 단일 문자들의 배열입니다. 문자열 리터럴을 나타내기 위해 큰따옴표 `"`를 사용하고 `\`를 사용한 이스케이프(Escape)를 지원합니다.

```lisp
(defparameter *my-string* "this is string")
(format t "~a~%" *my-string*)  ; this is string

(format t "~a~%" "foo\\bar")  ; foo\bar
(format t "~a~%" "\"foo\"")   ; "foo"
```

### 문자열 비교 방법
문자열을 비교하기 위한 함수는 문자 비교 함수(예: `char=`, `char-equal`)들과 매우 유사합니다. 
단순히, 문자 비교 함수명에 포함된 `char`를 `string`으로 변경해서 사용할 수 있습니다.

| number | case-sensitive | case-insensitive |
|:-:|:-|:-|
|`=`|`string=`|`string-equal`|
|`/=`|`string/=`|`string-not-equal`|
|`>`|`string>`|`string-greaterp`|
|`<`|`string<`|`string-lessp`|
|`>=`|`string>=`|`string-not-lessp`|
|`<=`|`string<=`|`string-not-greaterp`|

단일 문자 비교 방법과 유의해야 할 점은 각 함수들의 반환값입니다. 

`char`관련 함수들은 반환값으로 참/거짓을 나타내는 `t`혹은 `nil`을 반환합니다. 하지만, 문자열 비교 함수는 동일한 문자열인지 검사하는 `string=`과 `string-equal`만 `t`혹은 `nil`을 반환하고 다른 비교 함수들은 `t` 대신에 **mismatch-index를 반환**합니다. 

```lisp
(string= "ab" "ab")   ; T
(string/= "ab" "aa")  ; 1
(string< "aaab" "aaac")   ; 3
```
`mismatch-index`는 두개의 문자열이 서로 다를 때, 왼쪽에서 몇번째 인덱스부터 불일치하는지를 나타냅니다.
따라서, `(string< "aaab" "aaac")`의 경우, 처음으로 불일치하는 문자, `b`와 `c`의 인덱스를 반환합니다.
```
       aaab
       aaac
------------
index: 0123
          ^
     mismatch-index
```


### 문자열 기본 함수
문자열을 조작하기 위한 몇 가지 기본적인 함수들은 아래와 같습니다.

| 기능 | 함수 |
|:-|:-|
|문자열 길이|`length`|
|문자열 인덱싱|`char`|
|문자열 연결|`concatenate`|
|부분 문자열|`subseq`|
|대소문자 변환|`string-upcase` `string-downcase`|
|문자열 찾기|`search`|
|문자열과 리스트 변환|`coerce`|
|문자열과 숫자 변환|`parse-integer` `princ-to-string`|

문자열 전용 함수들도 있지만, 문자열은 결국 문자들의 집합인 시퀀스 타입이기 때문에 많은 함수들이 리스트와 벡터 같은 다른 시퀀스 타입의 객체에도 적용할 수 있습니다.

위 각 함수들을 예제와 함게 살펴보겠습니다.

#### 문자열 길이
```lisp
(length "abc")   ; 3
(length "")      ; 0

(defparameter *my-string* "hello")
(length *my-string*)   ; 5
```

#### 문자열 인덱싱
```lisp
(char "hello" 4)   ; 4번째 인덱스의 문자 `#\o` 반환
```

#### 문자열 연결
```lisp
(concatenate 'string "hello " "world")   ; "hello world" 반환
```
사실 `concatenate`함수는 문자열만 결합하기 위한 함수는 아닙니다. 문자열을 포함한 다양한 시퀀스(Sequence)타입(예: `vector`, `list`)을 변환하고 결합할 수 있는 유연한 기능을 제공합니다.

함수의 첫번째 인자로 사용한 `'string`은 함수의 반환 결과가 문자열이라는 것을 나타냅니다.
만약, `'string`이 아닌 `'list`를 사용하는 경우,
```lisp
(concatenate 'list "hello " "world")
```
최종 반환은 각 문자들의 리스트 형태인 `(#\h #\e #\l #\l #\o #\  #\w #\o #\r #\l #\d)`를 반환합니다.

`concatenate`함수에 대한 자세한 설명은 [공식문서](https://www.lispworks.com/documentation/HyperSpec/Body/f_concat.htm#concatenate)를 참고 바랍니다.


#### 부분 문자열
```lisp
(subseq "hello world" 0 5)   ; "hello"
```
부분 문자열을 가져오기 위한 `subseq`함수의 두번째 인자는 시작 인덱스, 세번째 인자는 부분 문자열의 길이를 나타냅니다.


#### 대소문자 변환
- 소문자를 대문자로 변환
  - `string-upcase`
- 대문자를 소문자로 변환
  - `string-downcase`

```lisp
(string-upcase "hello")    ; "HELLO"
(string-downcase "WORLD")  ; "world"
```

#### 문자열 찾기
```lisp
(search "store" "fruit store")  ; 6
(search "nice" "hello world")   ; NIL
```
`search`함수는 서브스트링의 시작 위치를 반환하고 찾을 수 없으면 `nil`을 반환합니다.


#### 문자열과 리스트 변환
```lisp
(coerce "hello" 'list) ; (#\h #\e #\l #\l #\o)
```
`coerce`함수는 강제적인 형변환을 위한 함수라고 생각하면 편합니다. 함수의 마지막 인자는 형변환 목적 타입을 명시합니다.
물론, 모든 인자값이 가능한 것은 아니므로 가능한 값은 [공식문서](https://www.lispworks.com/documentation/HyperSpec/Body/f_coerce.htm#coerce)를 참고 바랍니다.

#### 문자열과 숫자 변환
- 문자열->숫자 변환
  - `parse-integer`
- 숫자->문자열 변환
  - `princ-to-string`

Common Lisp은 단일 문자 한개를 숫자로 변환하는 직접적인 방법을 제공하지는 않고 문자열을 사용해서 숫자로 변환할 수 있습니다.

> 단, `parse-integer`라는 함수명에서 알 수 있듯이, 정수형만 변환이 가능합니다.

```lisp
(parse-integer "123")  ; 123, 3반환
(parse-integer "3.14") ; 오류. 소수점 변환 불가
```

조금 더 유용하게 사용하기 위해선 `parse-integer`함수가 제공하는 다양한 옵션들을 활용해야 합니다.
- 시작-끝 정의
  - `:start`, `:end`
- 진법 정의
  - `:radix`
- 숫자가 아닌 문자 허용 여부
  - `:junk-allowed`

예를 들어, `0x`로 시작하는 16진수 문자열을 숫자로 변환한다고 가정해보겠습니다.
```lisp
(parse-integer "0xFF" :radix 16 :start 2)  ; 255, 4 반환
```
위 예에서는 16진수 문자열 `0xFF`의 2번째 인덱스(3번째 문자)에서부터 16진수로 파싱한다는 의미이므로, `FF`의 10진수 값인 255를 반환합니다.

다른 하나의 예는 숫자와 알파벳이 혼용되어 있는 경우가 있습니다.
```lisp
(parse-integer "123int" :junk-allowed t)
```
단, `:start`가 따로 지정되지 않은 경우, **문자열의 앞부분에 숫자가 포함**되어야 합니다. 그렇기 때문에, `123int`는 허용되지만, `int123`은 허용되지 않습니다.

반대로, 숫자를 숫자 문자열로 변환할 수 도 있습니다. 조금 더 정확하게는 단순히 숫자객체만 문자열로 변환하는 방법이 아닌 객체 자체를 문자열로 변환하는 방법을 제공합니다. `princ-to-string`함수 이외에도 `write-to-string`, `prin1-to-string`를 사용할 수 있고, 각각은 출력하는 문자열의 형태에 따라서 구분해서 사용할 수 있습니다.

사실, 단순히 숫자객체를 문자열로 변경하기 위해서는 3개의 함수 중 어떤 것을 사용해도 상관없습니다.
```lisp
(princ-to-string 123)   ; "123"
(write-to-string 123)   ; "123"
(prin1-to-string 123)   ; "123"
```
각각 함수에 대한 자세한 내용은 [공식문서](https://www.lispworks.com/documentation/HyperSpec/Body/f_wr_to_.htm#princ-to-string)를 참고 바랍니다.

## 결론
Common Lisp에서는 문자를 독립적인 객체로 취급하며, 이를 활용한 문자 비교 및 변환 기능이 강력하게 제공됩니다. 문자열 역시 풍부한 내장 함수를 통해 다양한 조작이 가능하며, 리스트와의 변환도 지원하여 유연성을 제공합니다. 문자열을 다룰 때는 `coerce`, `parse-integer`, `concatenate` 등 적절한 함수를 활용하면 더욱 효율적으로 프로그래밍할 수 있습니다. Common Lisp의 문자 및 문자열 처리 기능을 숙지하면 보다 직관적이고 강력한 텍스트 기반 프로그래밍이 가능해질 것입니다.