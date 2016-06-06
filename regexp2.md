# 正则表达式匹配：虚拟机方法

Author: Russ Cox

Translator: Zhe Fu

Source: https://swtch.com/~rsc/regexp/regexp2.html

## 引言

列举一下最广泛使用的字节码解释器或者虚拟机。Sun的JVM？Adobe的Flash？.NET和Mono？Perl？Python？PHP？这些当然有名，但是有一个比这些加起来应用都要更广泛。这个字节码解释器是Henry Spencer的正则表达式库和它的许多后代。

这个系列的第一篇文章描述了两种主要的实现正则表达式匹配的方法：awk和egrep（以及目前绝大多数greps）使用的最坏情况线性时间的基于NFA和DFA的方法，以及到处都在用的、包括ed，sed，Perl，PCRE和Python等的最坏情况指数时间的回溯方法。

