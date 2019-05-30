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

.. compound::

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

这样一切都迎刃而解。在实践中，我们建议使用者始终使用IRBuilder来生成代码。IRBuilder的使用是没有额外的"语法开销"的（不需要在编译器的每个地方进行常量检查）
并且还可以显著减少在某些情况（尤其是需要宏预处理或使用很多常量的语言）下生成的LLVM IR的代码量。

.. compound::

另一方面， IRBuilder也会受到一定的限制：IRBuilder会在构建时进行代码\ `内联`__\。比如我们使用一个稍微复杂的例子：

-- https://en.wikipedia.org/wiki/Inline_expansion
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

.. compound::

在这种情况下，LHS和RHS相乘的值是相同的。但是我们更希望IRBuilder生成"tmp = x + 3; result = tmp * tmp"而不是生成两次"x+3"。

.. compound::

令人遗憾的是，基本上没有本地分析可以检测并优化这个问题。解决这个问题需要两个转换：重新关联表达式（使加法表达式相同）和使用\ `公共表达式消除`__\来删除冗余的加法指令。
幸运的是LLVM用"Passes"提供了多种方式进行优化。

-- https://en.wikipedia.org/wiki/Common_subexpression_elimination


LLVM优化-Passes（LLVM Optimization Passes）
=============================================





