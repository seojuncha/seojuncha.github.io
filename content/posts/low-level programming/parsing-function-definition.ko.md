---
title: ""
date: 2025-03-14
tags: []
description: ""
draft: true
---

```c
int main(void) { return 1; }
```

int : <type-specifier>
main(void) : <declarator>
  main : <declarator> -> <identifier>
  (void) : "(" <parameter-type-list> ")"
{} : <compound-statement>

ast-function-definition
 :return-type : <type-specifier>
 :name : <declarator> -> <identifier>
 :param : <parameter-type-list> -> <parameter-list> -> <parameter-declaration> -> <declaration-specifiers> <declarator>
 :body : <compound-statement> -> <block-item-list> -> <block-item> -> <statement> -> <jump-statement> -> <expression> -> ... -> <primary-expression> -> <constant> -> <integer-constant>


ast-function-definition
  :return-type = 'int
  :name = (ast-identifier "main")
  :param = '()
  :body = (ast-jump-statement :type 'return :expr (ast-literal 1))


parse-declarator
- main(void)

<declarator> -> main(void)

<declarator> -> <direct-declarator>

<direct-declarator>
  - <direct-declarator>
      - <identifier> -> main
  - "(" <parameter-type-list> ")"
      - '()