---
title: Raw String 的巨坑
date: 2020-02-27 19:47:02
tags: 
    - Python 
    - string 
    - 编译原理 
    - 踩坑记录
---
# Raw String 的巨坑

众所周知, 在编程语言中`\`字符有着特别的意义. 在一个字符串中, `\`字符会 "转义" 紧接着他的一个字符. 如, `\n` 会被转义为换行符, `\t` 会被转义为制表符等. 这种做法极大地方便了编程, 给予了我们一种方便的在字符串字面量 (源代码中的固定的量) 中表示一个无法输入的或者会破坏语法结构的字符的方式.

但是, 在有大量`\`的情况下我们不希望字符串中的 `\` 字符被转义, 如在表示地址或正则表达式的时候. 因此, 很多语言提供了 Raw String 字面量, 在 Raw string 字面量中 `\` 字符被视为普通的字符而不是转义符. 正如 Python 文档中描述的: 

> Both string and bytes literals may optionally be prefixed with a letter 'r' or 'R'; such strings are called raw strings and treat backslashes as literal characters. 

> 字符串与字节字面量都可以以一个 r 或者 R 前缀来表示原始字符串. 在原始字符串中, \ 字符被当做普通字符而不是转义符来处理.

正如, 在Python中:
```python
>>> print(r"asd\nsd\nsd")
asd\nsd\nsd
```
\n 会保持原样输出而不是被替换为换行符. 

那么, 问题来了: 下面的代码会输出什么?
```python
print(r"\")
```
可能看到语法高亮的颜色不太对劲就会让你意识到, 在Python中输出一个这样的字符串会报错而不是输出一个 \ 字符. 在Python文档中这样写道:

> Even in a raw literal, quotes can be escaped with a backslash, but the backslash remains in the result; for example, r"\"" is a valid string literal consisting of two characters: a backslash and a double quote; r"\" is not a valid string literal (even a raw string cannot end in an odd number of backslashes). Specifically, a raw literal cannot end in a single backslash (since the backslash would escape the following quote character). Note also that a single backslash followed by a newline is interpreted as those two characters as part of the literal, not as a line continuation.

> 即使在一个原始字面量中, 引号仍然会被 \ 字符转义, 只不过 \ 字符仍然保留在结果中. 如: r"\"" 是一个合法的字符串而 r"\" 会被认为缺少了一个引号.

这是因为, 在原始字面量的处理中, 并不是去掉了转义. 转义操作仍然在进行, 只不过正常情况下 '\n' 会转移成换行符, 但是在原始字面量中会转移为字符串\n. 在分析那些字符需要转义的时候, 编译器(或者说解释器)并不清楚这个字符是否在一个 Raw String 中. 编译器会把他标记为需要转义, 在之后的步骤中再完成具体的转移操作. 但是对于 r"\" 这个字符串, 在第一步的时候编译器会认为 \ 符号转义了后面的", 导致这个字符串的引号数量不匹配, 爆出语法错误. 

(~~所以锅还是得丢给编译器~~)

注: 上文中编译器指广义上的编译器或解释器等.

感谢群里的小伙伴们, 没有你们的帮助和讨论就不会有这篇文章. 