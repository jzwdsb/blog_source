---
layout: post
title: pl0 compiler
date: 2018-01-26
description: structure and expand feature
category: compiler
---

The code can be found in this [repositoy](https://github.com/jzwdsb/pl0_compiler)

The general compiler structure can divide into three parts ignore the difference of language.

1. Lexer
2. Parser
3. Target Code Generator

The three parts divide the work like blows.
![structure](/downloads/basic-data-structure.png)

The basic data flows likes the picture belows

![data_flows](/downloads/basic-data-flow.png)

### Lexer

　　The Lexer transform the input file in to a token stream, it should be at least have
two interface.

- getToken
- nextToken

　　The getToken method should get one token from the tokenstream, in order to consume the tokens
which belongs to the current nonterminal symbol.<br>
　　The nextToken method should allow the parser to look ahead one token in order to select the
different rlues to move on.

In the current finished version in cpp, the Lexer has two parts

1. **Scanner**
2. **Lexer**

　　Scanner transform the source file into char stream, Lexer transform the char stream into token
Stream.

### Parser

　　The parser handles the core steps of the compile process, in this simple pl0 compiler,
we mainly used recursive descent method.<br>
　　For each nonterminal symbol, we need to write function which handles
the parsing process for corresponding nonterminal symbol.

```c++
    void program();
    void block();
    void char_declaration();
    void const_declaration();
    void var_declaration();
    void array_declaration();
    void procedure_declaration();
    void statement();
    void condition();
    void expression();
    void term();
    void factor();
```

　　The start symbol of pl0 language according to it's EBNF form is program, follow with
a terminal symbol '.', which indicate the end of the parsing process.

　　It calls the block and judge the next symbol is '.' or not.<br>
　　The block function will consume the tokens which belong to the current block.

　　Then next thing is the block will call other function according to the EBNF
that compose the block.

### Target Code Generator

　　The Target Code Generator just like the Lexer, both of them are called by the
Parser when it needed.

　　Generally, The Parser calls the code generator when it meets a terminal symbol, such
as '+', '-', '*', '/' etc. Since the pl0 runtime model is a stack, the Parser transform
the expression into Anti Polish Notationm, The target code of operator must be generated
after it's two operand were pushed into stack.

General form like this

```c++
    term();
    /** meets +*/
    term();
    /** + code*/
    generate_code(...);
```

　　We need to define the process on the noterminal symbol handling.

　　Take block for example,we need to add jump instruction to the position where
the statement started after it's variable declaration so that interpreter won't
run the code which is not in the current procedure.After the statement is over,
we need add return instruction so the current procedure can finished and release
the memory it used. When the top procedure is returned, the program is finished.

## Expand Feature Content

　　Some feature are easy to expand, such as +=, -=, *=, /= etc. Just treat the left
operand as a factor and push it onto the stack, do the operator assignment then store
the value on the top of the stack to the address of left variable. Necessary information
is known in the parsing process.

　　To implement array feature, We need to add two instruction to the set.

1. lda
2. sta

　　The two instructions above give the interpret the ability to find an element in
the runtime.

　　They are basically the same expect the number of operands.

　　lda has three operands. The L, level subs. The M, The start address of the 
Another implicit operand is the number on the top of the stack. This number indicates
the offset of the element in the array. To load a array element, First Thing is to find
the start address of the array by the L and M. Then compute the address by plus the M and
the number on the top of the stack. Before push it on the stack, we have to pop the offset
out of the stack so that it won't have infulence to later operation;

　　sta is basically the same but it has four operands. The diffence is that the top of stack
is the number to store in an array element. The second top of the stack is the offset.

　　Due to The lda instruction pop the offset out of the stack, the array element can't be both
factor and assign variable. The operator that have side effect and do operations can't apply on
array elements. Such as ++, -\-, +=, -=, etc. The offset value can't be consumed twice.

　　Both sto, sta we won't define operation on the number to store which is on the top of stack.
Whatever do to it, it won't have infulence to later operation.