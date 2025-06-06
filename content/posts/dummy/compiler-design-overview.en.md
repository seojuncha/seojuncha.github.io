---
title: ""
date: 2025-03-14
tags: []
description: ""
draft: true
---

## The Structure of Compiler
There are two parts: *analysis* and *synthesis*

*analysis part (front end)*
- break up the source program into constituent pieces
- create an intermediate representation
- syntactically ill formed
- semantically unsound
- collects information about the source program
- stores the information in a data structure called a *symbol table*

*synthesis part (back end)*
- constructs the desired target program
- from intermediate representation and the informatino in the symbol table


## Phase of Compiler

Lexical Analyzer

Syntax Analyzer

Semantic Analyzer

Intermediate Code Generator

Machine-Independent Code Optimizer

Code Generator

Machine-Dependent COde Optimizer


(with Symbol Table)

## The First Phase: Lexical Analysis(Scanning)

## The Second PHase: Syntax Analysis(Parsing)
Create a tree-like intermediate representation: syntax tree


## The Third Phase: Semantic Analysis
An important part of semantic analysis is *type checking*.


## Intermediate Code Generator

## Code Optimization

## Code Generation
The code generator takes as input an intermediate representation of the source program and maps it into the target language.
The intermediate instructinos are translated into sequences of machine instructinos that perform the same task.
A crucial part is the judicious assignment of registers to hold variables.

