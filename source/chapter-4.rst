.. role:: unsure

.. _chapter-3:

************************
第四章 添加JIT和优化支持
************************

:原文: `Adding JIT and Optimizer Support <http://llvm.org/docs/tutorial/LangImpl04.html>`_

本章简介
========

欢迎进入"用LLVM开发新语言"教程的第四章。在前1-3章中，我们主要介绍了如何实现一门简单的语言（译者加 ：Kaleidoscope）以及加入LLVM IR的生成。在本章我们
将介绍两个新的技术：在Kaleidoscope语言中加入优化，以及\ `JIT编译器`__\。这些新技术将向你展现如何为Kaleidoscope语言生成更加优质和高效的代码。

__ https://en.wikipedia.org/wiki/Just-in-time_compilation

琐碎常数折叠（Trivial Constant Folding）
=======================================

.. compound::

    虽然我们在第三章实现的示例代码优雅并且易于扩展，但是却不能生成十分满意的代码。尽管如此，在编译简单的代码时IR Buidler还是做了一些显而易见的优化。

    ::

        ready> def test(x) (1+2+x)*(x+(1+2));
        ready> Read function definition:
        define double @test(double %x) {
        entry:
                %addtmp = fadd double 3.000000e+00, %x
                %addtmp1 = fadd double %x, 3.000000e+00
                %multmp = fmul double %addtmp, %addtmp1
                ret double %multmp
        }

     以上翻译生成的LLVM IR代码并不是经过我们之前所做词法分析、语法分析后构建的AST生成的（即是被LLVM IRBuider优化过后生成的LLVM IR， 此处注释为译者加）。以下生成的LLVM IR为未优化的结果：

     ::

             ready> def test(x) 1+2+x;
             Read function definition:
             define double @test(double %x) {
             entry:
                     %addtmp = fadd double 2.000000e+00, 1.000000e+00
                     %addtmp1 = fadd double %addtmp, %x
                     ret double %addtmp1
             }

     根据上面的对比，我们可以发现\ `常数折叠`__\（在方法test里面LLVM IR翻译后 "1+2+x"的形式被优化为 "3+x"的形式）是一种很常见、重要的优化方式：因此很多语言都会在其AST中实现常数折叠。

     __ https://en.wikipedia.org/wiki/Constant_folding
.. compound::

     但是当我们使用LLVM时，我们是不需要在AST中实现常数折叠的。这是因为所有LLVM IR的生成都是通过调用LLVM IR buider 进行的，而builder会在被调用时检查是否可以进行常数折叠。如果可以，builder会折叠常数并返回折叠后的新常数，但在折叠过程中不再创建新的指令。

.. compound::


LLVM优化器 Passes（LLVM Optimization Passes）
=============================================


