---
layout: post
title: 为什么要使用模板元?
categories: moderncpp
---

## 1. 起因

先引用维基百科上的定义：
> Template metaprogramming (TMP) is a metaprogramming technique in which templates are used by a compiler to generate temporary source code, which is merged by the compiler with the rest of the source code and then compiled. The output of these templates include compile-time constants, data structures, and complete functions. The use of templates can be thought of as compile-time execution. 

大致的意思翻译一下。模板元编程是一种元编程的技巧。它让编译器利用模板来生成临时代码，与其他源代码合并后，再完成编译工作。这些模板会输出编译期常量，类型还有纯函数。使用这些模板可以看作是编译期的执行。

在笔者自己学习C++，看到这个定义的时候，内心是崩溃的。我问自己的第一个问题就是，编译期计算，虽然不明觉厉，但是它到底意义何在？这和人们认知一个事物的逻辑顺序是有所偏差的。What, why & how. 在考察what部分的时候，我就对这个事物的合理性有了巨大的质疑。加之C++ template的代码往往不太直观，也不常见。于是乎，模板狂人，奇淫巧技还有黑魔法等，一堆帽子就扣过来了。所以，对于TMP而言why比what重要许多，也应该放在what之前了解。结果就有了这篇鄙文。