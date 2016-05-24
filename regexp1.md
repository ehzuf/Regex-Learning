# 正则表达式可以既简单又快速

Author: Russ Cox

Translator: Zhe Fu

Source: https://swtch.com/~rsc/regexp/regexp1.html

## 引言

一般地，正则表达式匹配有两种实现方式。一种应用于许多语言中，例如Perl等；另一种应用地方较少，最著名的是awk和grep。两种方法的性能特征差异很大。

![](http://ww4.sinaimg.cn/large/65076efejw1f3v58u50f2j20kc06iaan.jpg)

如上图所示，`a?3a3`表示`a?a?a?aaa`。左图是Perl实现，右图是Thompson方法。注意，右图纵轴的单位是ms，这并不是一个笔误。当匹配一个29字符的小写字符串时，Thompson方法比Perl快百万倍。图中曲线的趋势会继续，当匹配100字符的字符串时，Thompson NFA方法需要200ms，而Perl需要超过10<sup>15</sup>年的时间。（Perl是所有用相同算法的语言中最显著的例子，上图也可以是Python，PHP，Ruby等等。文章中另一幅图展示了其他实现的结果）

可能你很难相信这幅图。也许你使用过Perl，但你从未意识到正则表达式匹配会如此之慢。事实上，绝大多数情况下Perl的正则表达式匹配已经足够快了。但是，正如图上所示，我们可以写出所谓的“病态”正则表达式使得Perl匹配非常非常慢。相反，对于Thompson NFA方法，没有正则表达式能够称之为“病态”（笔者存疑，NFA最坏情况性能无法保证？）。那么，为什么Perl不使用Thompson NFA方法呢？Perl是能够使用的，这也是这篇文章所要介绍的。

从历史上看，正则表达式是计算机科学历史上用“好理论”产生好算法的最经典的例子之一。正则表达式最初作为一个简单计算模型被理论学家所提出。在为CTSS所编写的文本编辑器QED中，Ken Thompson实现了程序意义上的正则表达式匹配。Dennis Ritchie为了GE-TSS，自己学着实现了QED。Thompson和Ritchie继续创造了Unix，理所当然地将正则表达式带入了Unix。到了1970年代晚期，正则表达式已成为Unix的关键功能，应用于ed，sed，grep，egrep，awk，lex等工具中。

如今，正则表达式却变成了一个忽视“好理论”从而导致坏程序的经典例子之一。今天我们所使用的著名的工具中的正则表达式实现比那些已经有30余年历史的Unix工具要慢得多。

这篇文章回顾了“好理论”：Ken Thompson在1960年代所发明的正则表达式，自动机以及正则表达式搜索算法。这篇文章同样将理论运用到实际，描述了一个简单的Thompson算法实现。这个实现就是文章开头与Perl进行比较的方法。它的性能要好于Perl，Python，PCRE等更复杂的实现方式。

## 正则表达式

正则表达式是用来描述字符串集合的符号。当集合中的一条特定字符串被一条正则表达式所描述，我们就说这条正则表达式匹配了这个字符串。

最简单的正则表达式是单个字符。除了特殊的元字符`*+?()|`，字符匹配它本身。为了匹配元字符，需要在前面加上反斜杠`\`。

两个正则表达式可以被并接或者串接，从而形成一个新的正则表达式。如果e<sub>1</sub>匹配了s而e<sub>2</sub>匹配了t，那么e<sub>1</sub>|e<sub>2</sub>匹配s或者t，e<sub>1</sub>e<sub>2</sub>匹配st。

元字符`*+?`是重复操作符。e<sub>1</sub>*匹配e<sub>1</sub>的零次或者多次重复，e<sub>1</sub>+匹配一次或者多次重复，e<sub>1</sub>?匹配零次或者一次重复。

优先操作符，从弱到强，依次是串接，并接，重复操作符。括号可以用来强制改变含义，正如在算术表达式中的那样。一些例子：`ab|cd`等于`(ab)|(cd)`，`ab*`等于`a(b*)`等等。

以上所描述的语法是传统的Unix egrep正则表达式语法的子集。这些语法已经足够描述所有的正则语言。不严谨的讲，正则语言是用固定大小内存、能够一次性地通过文本并发生匹配的字符串集合。新的正则表达式（以Perl为代表）增加了[许多新的操作符和转义符](http://www.perl.com/doc/manual/html/pod/perlre.html)，使正则表达式更加简明，有时更加”神秘“，但并没有更加强大：这些新的正则表达式几乎总是可以用更长的传统正则表达式表示出来。

反向引用（backreferences）是一个确实提供了更强大功能的正则表达式语法扩展。一个反向引用（\1或者\2）匹配之前加上括号的表达式。例如，`(cat|dog)\1`匹配`catcat`以及`dogdog`，但没有匹配`catdog`或者`dogcat`。如果考虑到正则表达式理论，带有反向引用的正则表达式并不是正则表达式。反向引用所带来的代价很大：在最差情况下，目前所知的最好实现需要指数级别的搜索算法，正如Perl所用到的。Perl和其他语言目前不能移除反向引用的支持，但是对于没有反向引用的正则表达式，他们能够采用更加快速的算法。本文就是关于那些更加快速的算法。

## 有限自动机

有限自动机是另外一种描述字符串集合的方式。有限自动机也被叫做状态机，“automaton”与“machine”两者是可以替换的。

下图是一个正则表达式`a(bb)+a`对应的有限自动机：
![](https://swtch.com/~rsc/regexp/fig0.png)

有限自动机总是在状态中的一个，用圆圈表示。（圆圈中的数字仅仅让讨论更容易，不是自动机操作的一部分）当读入字符，一个状态跳转到另一个状态。有限自动机有两种特殊状态：起始状态s<sub>0</sub>和匹配状态s<sub>4</sub>。起始状态被孤立箭头指向，匹配状态有两个圆圈。

自动机每次读入一个字符，顺着箭头，从一个状态跳转到另一个状态。假设输入字符是`abbba`，匹配过程如下图所示。

![](https://swtch.com/~rsc/regexp/fig1.png)

自动机在匹配状态s<sub>4</sub>停止，所以它匹配了字符串。如果自动机在非匹配状态停止，它没有匹配字符串。如果在自动机执行的任何时候，当前状态没有输入字符对应的箭头，自动机提前停止。

这个自动机叫做确定性有限自动机（DFA），因为在任何状态，每一个可能的输入会通向至多一个新状态。我们同样能创造必须选择多个可能下一状态的自动机。例如，下图中的自动机等价于之前的自动机，但是是非确定的。

![](https://swtch.com/~rsc/regexp/fig2.png)

因为在状态s<sub>2</sub>，读入字符b，有多个下一状态选择。既能够回到s<sub>1</sub>，又能够前往s<sub>3</sub>。由于自动机不能提前看到剩余字符串，它不知道哪一个是正确选择。在这种情况下，让自动机总是猜正确的选择是一个有趣的事。这样的自动机叫做非确定性有穷自动机（NFA或NDFA）。如果通过某种方式，NFA能够读入字符串并且根据箭头到达匹配状态，那么就发生了匹配。

有时，让NFA拥有一些没有相应字符的箭头是一件很方便的事情。这些箭头没有打上标签。在任何时候，NFA能够根据没有标签箭头跳转到下一状态，而不用读入任何字符。下图中的NFA等价于前两者，但是没有标签的箭头使得`a(bb)+a`表述更加明晰。

![](https://swtch.com/~rsc/regexp/fig3.png)

## 将正则表达式转化为NFA

正则表达式和NFA被证明是等价的：每一条正则表达式都对应等价的NFA（匹配相同的字符串），反之亦成立。（同样，DFA与正则表达式也是等价的，之后会看到。）将正则表达式翻译成NFA有许多种方法，这里给出的方法是Thompson在他的1968 CACM论文中提出的。

正则表达式所对应的NFA由正则表达式子表达式所对应的局部NFA组成，不同的操作符有不同的构建方式。局部NFA没有匹配状态：想法，它们有一个或者多个指向空的箭头。当将这些箭头指向匹配状态，构建过程就结束了。

匹配单个字符的NFA看起来像：

![](https://swtch.com/~rsc/regexp/fig4.png)

串接e<sub>1</sub>e<sub>2</sub>的NFA将e<sub>1</sub>自动机的终止箭头指向e<sub>2</sub>的启示箭头。

![](https://swtch.com/~rsc/regexp/fig5.png)

并接e<sub>1</sub>|e<sub>2</sub>的NFA增加一个新的起始状态，选择可以是e<sub>1</sub>自动机或者e<sub>2</sub>自动机。

![](https://swtch.com/~rsc/regexp/fig6.png)

e<sub>1</sub>?的NFA将e<sub>1</sub>自动机与空路径并接。

![](https://swtch.com/~rsc/regexp/fig7.png)

e<sub>1</sub>*的NFA创造了一个环，将e<sub>1</sub>指向起始状态。

![](https://swtch.com/~rsc/regexp/fig8.png)

e<sub>1</sub>+的NFA同样创造了一个环，但是需要经过e至少一次。

![](https://swtch.com/~rsc/regexp/fig9.png)

观察图中的新状态数，我们能够发现这种方法对于正则表达式中的每一个字符或者元字符，创造了准确的一个状态（括号除外）。因此，**最终NFA的状态最多等于原始正则表达式的长度**。

正如前面NFA例子中所提到的，去除没有标签的箭头总是可能的，并且起初产生不含没有标签的箭头的NFA也总是可能的。留下没有标签的箭头使得NFA更容易理解，并且使C语言描述更加简单，因此我们留下了没有标签的箭头。

## 正则表达式搜索算法

现在我们能够测试一个正则表达式是否匹配一个字符串：将正则表达式转化成NFA，然后将字符串当做输入，运行NFA。需要记得的是，当面对下一状态选择的时候，NFA能够正确地做出猜测。用普通的计算机运行NFA，我们必须找到模拟这种猜测过程的方法。

一种模拟方式是尝试一种选择，如果它不工作，尝试另外一种。例如，考虑`abab|abbb`的NFA以及输入`abbb`：

![](https://swtch.com/~rsc/regexp/fig10.png)

![](https://swtch.com/~rsc/regexp/fig11.png)

在第0步，NFA必须做出选择：尝试匹配`abab`还是`abbb`？在图中，NFA尝试了`abab`，在第3步失败了。然后NFA尝试另一个选项，开始第4步，直到最终匹配。这种回溯（backtracking）的方法可以简单地递归实现，但是在之后会多次读入输入字符。如果字符串没有匹配，自动机必须尝试所有的可能执行路径，直到放弃。图中NFA仅尝试了两条不同路径。但是在最坏情况下，可能会有指数级别的可能路径，导致非常慢的匹配速度。

一个更有效、但是更复杂的模拟猜测方法是同事尝试所有的选择。这种方法允许自动机同时在多个状态中。为了处理每一字符，所有（激活的）状态沿着所有匹配字符的箭头前进。

![](https://swtch.com/~rsc/regexp/fig12.png)

在第1步和第2步，NFA同时在两个状态中。只有第三步状态集缩减到一个。多状态方法同时尝试多个路径，只需读入输入字符一次。在最坏情况下，NFA每一步都可能在所有状态中，但是这种结果最差导致常数倍的工作量，与输入字符长度无关。所以任意大的输入字符能够在线性时间内处理。相对于回溯法的指数级别的时间要求，这是一个巨大的改进。这种改进来自于仅仅追踪可到达的状态，而非哪一条路径通向了这个状态。**对于有n个节点的NFA，每一步最多有n个可以到达的状态，但是可能有2<sup>n</sup>条路径**。


## 实现

Thompson在他1968年的paper中介绍了多状态模拟方法。在他的描述中，NFA的状态由小的机器码序列表示，用函数调用指令序列表示可能状态的列表。本质上，Thompson将正则表达式编译成聪明的机器码。40年后，计算机速度快得多，因此机器码方法已经不是必要的了。下一部分介绍的实现由可移植的标准C语言实现。所有源代码（小于400行）和评测脚本可以在这里获得。（对C语言或者指针不熟悉的读者可以仅仅阅读描述，跳过实际的代码）

## 实现：编译成NFA

第一步是将正则表达式编译成等价的NFA。在我们的C程序中，我们可以将NFA表示成链接的状态（State）结构集合。

```
struct State
{
	int c;
	State *out;
	State *out1;
	int lastlist;
};
```

每一个State根据c的值的不同，表示下列NFA片段的一个。

![](https://swtch.com/~rsc/regexp/fig13.png)

（在执行过程中，Lastlist会被使用，这将在下一个部分介绍）

根据Thompson的paper，编译器从用后缀注释的形式表示的正则表达式构建NFA，这些后缀注释是"."，表示连接。一个单独的函数re2post将诸如`a(bb)+a`形式的中缀形式正则表达式改写成等价的后缀表现形式`abb.+.a.`。（真实实现中，"."当然表示任意字符，而不是连接符。真实实现同样可能在解析正则表达式过程中构建NFA，而不是转化成明确的后缀形式的正则表达式。然而，后缀形式版本更方便，也与Thompson的paper更加紧密。）

当编译器扫描后缀形式的表达式时，它维护着一个由计算过的NFA片段组成的栈。字符使NFA片段入栈，操作符使NFA片段出栈。例如，当编译`abb.+.a.`中的`abb`时，栈中包含有NFA片段a，b，和b。"."的编译使两个NFA片段b出栈，并将NFA片段"bb."入栈。每一个NFA片段由它的起始状态以及外出箭头所组成：

```
struct Frag
{
	State *start;
	Ptrlist *out;
};
```

Start指针指向NFA片段的启示状态，out是指向没有连接任何东西的State*指针的指针列表。他们既是NFA片段中孤立的箭头。

一些操作指针列表的辅助功能函数：


```
Ptrlist *list1(State **outp);
Ptrlist *append(Ptrlist *l1, Ptrlist *l2);

void patch(Ptrlist *l, State *s);
```

List1创造了一个新的指针列表，包含单指针outp。Append连接了两个指针列表，返回结果。Patch将指针列表l中的孤立箭头指向状态s：对于l中的每一个指针outp，使*outp=s。

考虑到这些基本单元以及片段栈，编译器实际上是一个在后缀形式正则表达式上的简单循环。最后留下一个片段：补充填上一个匹配状态，完成了整个NFA。


```
State*
post2nfa(char *postfix)
{
	char *p;
	Frag stack[1000], *stackp, e1, e2, e;
	State *s;

	#define push(s) *stackp++ = s
	#define pop()   *--stackp

	stackp = stack;
	for(p=postfix; *p; p++){
		switch(*p){
		/* compilation cases, described below */
		}
	}
	
	e = pop();
	patch(e.out, matchstate);
	return e.start;
}
```

对应于之前介绍了各个翻译步骤，相应的编译代码如下。

文字字符：

![](https://swtch.com/~rsc/regexp/fig14.png)

```
default:
	s = state(*p, NULL, NULL);
	push(frag(s, list1(&s->out));
	break;
```

串接：

![](https://swtch.com/~rsc/regexp/fig15.png)

```
case '.':
	e2 = pop();
	e1 = pop();
	patch(e1.out, e2.start);
	push(frag(e1.start, e2.out));
	break;
```

并接：
![](https://swtch.com/~rsc/regexp/fig16.png)

```
case '|':
	e2 = pop();
	e1 = pop();
	s = state(Split, e1.start, e2.start);
	push(frag(s, append(e1.out, e2.out)));
	break;
```

零或一：
![](https://swtch.com/~rsc/regexp/fig17.png)

```
case '?':
	e = pop();
	s = state(Split, e.start, NULL);
	push(frag(s, append(e.out, list1(&s->out1))));
	break;
```

零或者更多：
![](https://swtch.com/~rsc/regexp/fig18.png)

```
case '?':
	e = pop();
	s = state(Split, e.start, NULL);
	push(frag(s, append(e.out, list1(&s->out1))));
	break;
```

一个或者更多：
![](https://swtch.com/~rsc/regexp/fig19.png)

```
case '+':
	e = pop();
	s = state(Split, e.start, NULL);
	patch(e.out, s);
	push(frag(e.start, list1(&s->out1)));
	break;
```

## 实现：模拟NFA

现在NFA已经构建完毕了，我们需要模拟（simulate）之。模拟需要追踪状态（State）集合，可以用简单的数组列表存储。

```
struct List
{
	State **s;
	int n;
};
```

模拟过程使用两个列表：clist是当前NFA所在的状态的集合，nlist是处理完当前字符后，NFA将在的状态的集合。执行的循环中，首先初始化clist，使之仅包含开始状态，然后每次运转自动机一步。

```
int
match(State *start, char *s)
{
	List *clist, *nlist, *t;

	/* l1 and l2 are preallocated globals */
	clist = startlist(start, &l1);
	nlist = &l2;
	for(; *s; s++){
		step(clist, *s, nlist);
		t = clist; clist = nlist; nlist = t;	/* swap clist, nlist */
	}
	return ismatch(clist);
}
```

为了避免每一次都需要分配内存，match函数使用两个提前分配好的列表了l1和l2作为clist和nlist，每一步之后交换两者。

如果最终状态集合包括匹配状态，那么字符串就匹配了。

```
int
ismatch(List *l)
{
	int i;

	for(i=0; i<l->n; i++)
		if(l->s[i] == matchstate)
			return 1;
	return 0;
}
```

Addstate向一个列表中添加一个状态，但是如果此列表中已经有了就不添加。每次add操作扫描整个列表是很低效的，因此，我们用变量listid作为列表产生数。当addstate向列表中添加一个状态，它在`s->lastlist`中记录`listid`。如果两者已经相同，那么s已经在列表中了。addstate同样与没有标签的箭头相关：如果s是Split状态，有两个没有标签的箭头指向新状态，那么addstate将这些状态加入列表中，而不是s。

```
void
addstate(List *l, State *s)
{
	if(s == NULL || s->lastlist == listid)
		return;
	s->lastlist = listid;
	if(s->c == Split){
		/* follow unlabeled arrows */
		addstate(l, s->out);
		addstate(l, s->out1);
		return;
	}
	l->s[l->n++] = s;
}
```

Startlist创造了初始状态列表，仅仅加入起始状态。

```
List*
startlist(State *s, List *l)
{
	listid++;
	l->n = 0;
	addstate(l, s);
	return l;
}
```

最后，对于每一个输入字符，step将NFA前进一步，用当前状态列表clist计算下一个列表nlist。

```
void
step(List *clist, int c, List *nlist)
{
	int i;
	State *s;

	listid++;
	nlist->n = 0;
	for(i=0; i<clist->n; i++){
		s = clist->s[i];
		if(s->c == c)
			addstate(nlist, s->out);
	}
}
```

## 性能

以上C语言的实现并没有刻意考虑性能问题。即便如此，一个线性时间算法的慢实现可以轻松超过指数时间算法的快实现，在指数很大的情况下。用“病态的”正则表达式测试不同的著名的正则表达式引擎，可以很好地得出这个结论。

考虑正则表达式a?<sup>n</sup>a<sup>n</sup>，当a?没有匹配任何字母时，它匹配字符串a<sup>n</sup>。回溯法实现对于?，首先尝试一次，然后一次。如果有n次选择，那么总共有2<sup>n</sup>种可能性。只有最后一种选择，既所有的?均为0时，会发生匹配。因此，回溯法需要O(2<sup>n</sup>)时间复杂度，当n=25时无法扩展。

相反，Thompson的实现维护了长度大约为n的状态列表，对于长度为n的字符串，时间复杂度是O(n<sup>2</sup>)。（运行时间是超线性的，因为当输入字符变多时，正则表达式的长度并没有不变。实际上，对于长度为m的正则表达式，以及长度为n的字符串，时间复杂度为O(mn)）。

下图展示了当检测a?<sup>n</sup>a<sup>n</sup>是否匹配a<sup>n</sup>的所需时间图。

![](https://swtch.com/~rsc/regexp/grep1p.png)

注意到，为了能在一张图上表示更多的数据，y轴是对数坐标。

从这张图上，我们能清楚地看到，Perl，PCRE，Python和Ruby都是用的递归回溯查找的方法。当n=23时，PCRE因为超过了最大递归迭代步数而停止了。对于Perl 5.6，它的正则处理引擎[据说](http://perlmonks.org/index.pl?node_id=502408)能够记录递归回溯查找的结果，这应该能够用一定的内存代价避免指数级别的内存消耗，除非使用的反向引用。正如评测结果所展示出的，这种说法并不完全：即便没有反向引用，Perl的运行时间依旧是指数级别增长。另外，虽然评测没有提到，Java同样使用了回溯的实现方法。事实上，因为任意的Java代码能够替换成匹配路径（？？？），java.util.regex借口需要回溯方法的实现。

图中的粗蓝线既是Thompson方法的C语言实现版本。**Awk，Tcl，GNU grep，以及GNU awk构建了DFA**，要么通过提前计算的方式，要么通过下一章将会讲到的即时构建的方式。

有些人可能认为评测对回溯实现方法不公平，因为仅仅关注了不常见的边界情况。这种说法漏掉了关键一点：如果从一个对于任意输入运行时间可预测的、为常数时间的快的实现，与一个通常很快但在一些输入下会需要几年或者更多的CPU时间的实现中选择一个，答案就很明白了。同样，尽管这种“悲剧”的例子很少发生，包括像用来划分5个空格分隔的字段“(.\*) (.\*) (.\*) (.\*) (.\*)”，以及在通常情况没有列出的情况下使用并接这样的例子的确会发生。因此，程序员通常要学习哪种构造代价大从而避免之，或者是转向所谓的[优化](http://search.cpan.org/~dankogai/Regexp-Optimizer-0.15/lib/Regexp/Optimizer.pm)。使用Thompson的NFA方法并不需要这样的适应：没有代价大的正则表达式。

## 缓存NFA构造DFA

注意到，DFA的执行过程要比NFA更有效，因为DFA在任意时刻只会有一个状态激活：从来不会有多个下一状态的选择问题。**任何NFA都能转化成等价的DFA**，其中DFA的状态对应于NFA的一组状态。

例如，下图是正则表达式`abab|abbb`对应的NFA，加上了状态序号：

![](https://swtch.com/~rsc/regexp/fig20.png)

等价的DFA如下：

![](https://swtch.com/~rsc/regexp/fig21.png)

DFA中的每一个状态对应于NFA中的一组状态。

某种意义上，Thompson的NFA模拟方法就是在执行等价的DFA：每一个列表对应于一些DFA状态，step函数根据给定的列表以及下一个字符，计算下一个进入的DFA状态。Thompson的方法用过重新构建所需要的DFA的状态来实现DFA的模拟。为了避免将来可能的重复计算，我们可以计算必要的等价DFA，缓存与空闲的内存中。本章将介绍和实现这样一种方法。从上一部分的NFA实现开始，我们需要添加少于100行代码来构建DFA实现。

为了实现DFA缓存，我们首先介绍一个新的用来表示DFA状态的数据类型：

```
struct DState
{
	List l;
	DState *next[256];
	DState *left;
	DState *right;
};
```

DState是状态列表l的缓存拷贝。next数组包含对于每一个可能的输入指向下一个状态的指针：如果当前状态是d，下一个输入是c，那么d->next[c]就是下一个状态。如果d->next[c]是空，说明下一个状态还未完成。对于一个给定的状态和输入字符，Nextstate计算、记录并返回下一个状态。

正则表达式匹配不断重复d->next[c]，在需要的时候调用nextstate计算新状态。


```
int
match(DState *start, char *s)
{
	int c;
	DState *d, *next;
	
	d = start;
	for(; *s; s++){
		c = *s & 0xFF;
		if((next = d->next[c]) == NULL)
			next = nextstate(d, c);
		d = next;
	}
	return ismatch(&d->l);
}
```

所有已经计算过的DStates被存储在一个可以让我们通过它的List查询DState的结构中。为了实现这样的目的，我们将他们排列入一个二叉树，用排序好的List作为key。对于给定的List，dstate函数返回DState，如果必要的话分配一个。


```
DState*
dstate(List *l)
{
	int i;
	DState **dp, *d;
	static DState *alldstates;

	qsort(l->s, l->n, sizeof l->s[0], ptrcmp);

	/* look in tree for existing DState */
	dp = &alldstates;
	while((d = *dp) != NULL){
		i = listcmp(l, &d->l);
		if(i < 0)
			dp = &d->left;
		else if(i > 0)
			dp = &d->right;
		else
			return d;
	}
	
	/* allocate, initialize new DState */
	d = malloc(sizeof *d + l->n*sizeof l->s[0]);
	memset(d, 0, sizeof *d);
	d->l.s = (State**)(d+1);
	memmove(d->l.s, l->s, l->n*sizeof l->s[0]);
	d->l.n = l->n;

	/* insert in tree */
	*dp = d;
	return d;
}
```

Nextstate运行NFA的step函数，返回相对应的DState：

```
DState*
nextstate(DState *d, int c)
{
	step(&d->l, c, &l1);
	return d->next[c] = dstate(&l1);
}
```

最后，DFA的起始状态对应NFA的起始状态列表：


```
DState*
startdstate(State *start)
{
	return dstate(startlist(start, &l1));
}
```

（在NFA模拟中，l1是提前分配的List）

DStates对应于DFA的状态，但是DFA只有在需要的时候才会构建：如果一个DFA状态没有在搜索过程中遇到，那么它就不会在缓存中存在。另外一种方法是一次性计算整个DFA。这样可以使匹配更快一点儿，通过移除可能的分支，但是代价是启动时间以及内存大小的增加。

有人可能会担心实时构建DFA所带来的内存边界问题。由于DState只是step函数的缓存，如果缓存增长过大，dstate的实现可以选择丢弃目前的整个DFA。这种缓存替换策略仅仅需要在dstate和nextstate中增加几行代码，另外加上大约50行的内存管理代码。实现可以在[这里](http://swtch.com/~rsc/regexp/)获得。([Awk](http://cm.bell-labs.com/cm/cs/awkbook/)使用了相似的缓存大小限制策略，固定限制32个缓存状态，这也可以解释性能图中当n=28时曲线的不连续。）


从正则表达式得到的NFA展现出了良好的局部性：在绝大部分文本上，它们一直访问同样的状态，进行同样的跳转。这就使得缓存很有价值：当箭头第一册被访问时，NFA模拟中的下一个状态必须被计算，但是之后的转移仅仅是一次内存访问。真实实现中基于DFA的方法可以利用额外的优化使得运行得更快。一篇相关的文章（还没有写出）将更详细地探索基于DFA的正则表达式匹配实现方法。

## 真实世界的正则表达式

真实程序中使用的正则表达式比上面描述的实现更加复杂。本章将简要地介绍常见的“并发症”，完全处理任何一种情况都超过了本篇文章的范围。

*字符类*。字符集，例如 [0-9] 或 \w 或 . (dot)，就是并接的简洁表现形式。字符集可以在编译过程中扩展成并接形式，尽管用一种新的NFA类型明确地表示它们会更加有效。[POSIX](http://www.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap09.html)定义了特殊的字符集诸如[[:upper:]] ，可以根据当前的位置改变含义。但是这部分困难的地方是确定它们的意思，而不是将意思编码如NFA。

*转义序列*。实际的正则表达式语法需要处理转义序列，不仅需要匹配元字符(\(, \), \\, 等)，而且需要指定很难输入的字符，例如\n。

*重复计数*。许多正则表达式实现提供了重复计数操作符{n}，匹配准确的n次字符串重复。{n,m}表示匹配至少n次至多m次重复，{n,}表示匹配n次或者更多。递归回溯方法的实现可以通过循环的方式实现重复计数，而基于NFA或者DFA的方法必须扩展重复：e{3}扩展成eee；e{3,5}扩展成eeee?e?，e{3,}扩展成eee+，等等。

*子串抽取*。当正则表达式被用作分隔或者解析字符串时，如果能找到输入字符串的哪一部分被每条正则表达式匹配将会非常有用。当一条诸如`([0-9]+-[0-9]+-[0-9]+) ([0-9]+:[0-9]+)`的正则表达式匹配了字符串（比如说时间和日期），许多正则表达式引擎能够得到被每一个加上括号的正则表达式匹配的文本。例如，在Perl中可以写如下：

```
if(/([0-9]+-[0-9]+-[0-9]+) ([0-9]+:[0-9]+)/){
	print "date: $1, time: $2\n";
}
```

字串边界抽取被绝大多数的计算机科学理论所忽略，并且这是使用递归回溯法的最新引人的证据。但是，Thompson风格的代码能够在不放弃高效性能的前提下实现追踪子串边界。早在1985年，在第八版的Unix regexp(3)库中已经实现了这样的算法，但是正如下面解释道的那样，没有被广泛的使用甚至是被注意到。

*非锚定的匹配*。这篇文章假定正则表达式匹配整个输入字符。在文章中，人们通常希望找到输入中的一个字串匹配正则表达式。传统的Unix工具返回输入中从最左开始的最长的子串。对于`e`的非锚定的搜索是子串抽取的特例：就像搜索`.*(e).*`，第一个`.*`被限制为匹配尽可能的短的字符串。

*非贪婪操作符*。传统的Unix正则表达式中，重复操作符`?`，`*`和`+`定义为匹配尽可能多的字符串，同时仍然允许整个正则表达式的匹配：当用`(.+)(.+)`匹配`abcd`，第一个`(.+)`将匹配`abc`，第二个将会匹配`d`。这些操作符被称作**贪婪的**。Perl引入了`??`，`*?`和`+?`，作为非贪婪的版本，在保持所有匹配的基础上匹配尽可能少的字符串：当用`(.+?)(.+?)`匹配`abcd`，第一个`(.+?)`将仅匹配`a`，第二个将会匹配`bcd`。根据定义，一个操作符是否是贪婪的并不会影响正则表达式对整个特定字符串的匹配；它只会影响子串边界的选择。回溯法的实现允许非贪婪操作符的简单实现：在长串之前尝试短串。例如，在标准的回溯法实现中，`e?`首先尝试用`e`，然后尝试不使用它；`e??`用另外一种顺序。Thompson算法的子串追踪变种可以适应非贪婪的操作符。

*断言*。传统的正则表达式元字符`^`和`$`可以被当做它们周围字符的**断言**：`^`表示之前的字符是新行（或者是字符串的开头），`$`表示下一个字符是新行（或者是字符串的结尾）。Perl添加了更多了断言，例如单词边界`\b`，表示之前的字符是字母数字组成的，但是下一个不是；反之亦然。Perl同样能够支持任意的通用条件的断言`(?=re)`，表示当前输入位置之后的文本匹配了`re`，但是并不会使输入前进；`(?!re)`类似，但是表示文本不匹配`re`。这样的断言还有`(?<=re)`和`(?<!re)`，表示在当前输入位置之前的字符。像`^`、`$`和`\b`这样的简单断言用NFA可以很容易的实现，为了断言推迟匹配1个字符。通用的断言对于NFA很难处理，但是原理上依旧能被NFA编码。

*反向引用*。正如之前提到的，目前还不知道如何高效地实现带有反向引用的正则表达式，尽管还没人证明这是不可能的。（[这个问题是NP完全的](http://perl.plover.com/NPC/NPC-3SAT.html)，意味着如果有人能够找到高效的实现方法，将会成为计算机科学家的大新闻，并且获得[百万美元的奖励](http://www.claymath.org/Popular_Lectures/Minesweeper/)）。原始的awk和egrep所采用的最简单、最高效的实现就是**不实现**。这种方法不再切合实际了：用户变得依赖偶尔需要的反向引用的功能，并且反向引用成为[POSIX正则表达式标准](http://www.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap09.html)之一。即便如此，用Thompson的NFA模拟方法处理绝大多数正则表达式是合理的，只在需要的时候采用回溯。更聪明的方法是将两者结合起来，只在需要处理后向引用的时候采用回溯。

*带记忆的递归*。Perl采用记忆的方法避免可能的回溯中的指数爆炸，这是一个好方法。至少在理论上，这可以让Perl的正则表达式表现更像NFA而不是回溯。但是，记忆方法并没有完全解决这个问题：记忆本身需要与文本乘以正则表达式大小相当的内存需求。记忆同样没有解决递归所用的栈空间带来的内存问题，这和输入文本大小是线性相关的：匹配长的字符串可能导致回溯实现耗尽栈空间：

```
$ perl -e '("a" x 100000) =~ /^(ab?)*$/;'
Segmentation fault (core dumped)
$
```

*字符集*。现代的正则表达式实现方法必须能够处理大型的非ASCII字符集，例如Unicode。[Plan 9正则表达式库](http://swtch.com/plan9port/unix/)通过将单个Unicode字符作为每一步的输入运行NFA，纳入了Unicode。这个库将输入解码与NFA运行分离开来，所以对于[UTF-8](http://plan9.bell-labs.com/sys/doc/utf.html)和宽字符输入可以使用同样的正则表达式匹配代码。


## 历史和引用

Michael Rabin和Dana Scott在1959年介绍了非确定性有穷自动机和非确定论[7]，说明NFA可以用DFA（可能大得多）模拟，其中每一个DFA状态对应于一组NFA状态。（他们因为在论文中介绍了非确定论，在1976年获得图灵奖）

R. McNaughton和H. Yamada[4]，以及Ken Thompson[9]因为第一个给出正则表达式到NFA的构建方法而声名远扬，尽管没有一篇论文提到了之后刚出现的NFA的方法。McNaughton和Yamada的构建方法创造了DFA，Thompson的构建方法创造了IBM 7094机器码，但是通过阅读源代码能够得到潜在的NFA构建方法。正则表达式到NFA的转化仅仅在如何编码NFA的选择上有所区别。文中采用的方法学习了Thompson的方法，将选择编码成确定的选择状态节点（上面的Split节点）以及非标签的箭头。另外一种方法，归功于McNaughton和Yamada，避免了非标签的箭头，而是让NFA在同样的标签下有多个输出箭头。McIlroy[3]用Haskell优雅地实现了这种方法。

Thompson的正则表达式实现是为了在IBM 7094运行的操作系统CTSS[10]上的文本编辑器QED。编辑器的拷贝可以在CTSS源代码[5]中找到。L. Peter Deutsch和Butler Lampson[1]首先开发了QED，但是Thompson的重新实现是第一个采用正则表达式的。Dennis Ritchie，另一个QED的实现的作者，在QED编辑器的早期历史建立了文档[8]。（Thompson，Ritchie和Lampson之后因为与QED或者有限自动机不相关的工作获得了图灵奖）

Thompson的论文标志着正则表达式历史的开端。Thompson在Unix第一版（1971）的ed编辑器，或者之后在Unix第四版（1973）的grep中，并没有选择他的算法。相反，这些古老的Unix工具采用了回溯递归的方法。回溯法是合理的，因为当时的正则表达式语法相当有限：它忽略了分组括号和`|``?``+`操作符。Al Aho的egrep，第一次在Unix第七版（1979）出现，是第一个支持完整正则表达式语法的Unix工具，采用了提前计算的DFA。到了第八版（1985），egrep采用了在线计算的方式，正如之前给出的实验一样。

1980年代早期，Rob Pike在写sam[6]文本编辑器时写了新的正则表达式匹配实现，Dave Presotto将其抽取出来，放入Unix第八版中。Pike的实现用NFA模拟有效地实现了字串匹配，但是正如第八本源代码剩余的那样，并没有被广泛地分发。Pike自己也没有认识到他的技术有任何新颖的地方。Henry Spencer从头重新实现了第八版库，但是使用了回溯的方法，并[公开了他的实现](http://arglist.com/regex/)。它被广泛使用，最终成为了之前提到的慢的正则表达式实现（Perl，PCRE，Python等等）的基础。（在他的辩解中，Spencer知道该方法可能比较慢，而且他不知道更有效的方法的存在。他甚至在文档中做出了警告，“许多用户发现速度是足够的，尽管讲egrep的内部代码改成这种方法的代码可能是一种错误。”）Pike的正则表达式实现扩展到支持Unicode，在[1992年晚期](http://groups.google.com/group/comp.os.research/msg/f1783504a2d18051)的sam中可以得到，但是更有效的正则表达式搜索算法变得无人注意的。这份代码目前可以在许多论坛中拿到：[sam的一部分](http://plan9.bell-labs.com/sources/plan9/sys/src/cmd/sam/)，[Plan 9正则表达式库](http://plan9.bell-labs.com/sources/plan9/sys/src/libregexp/)，或者是[Unix单独包](http://swtch.com/plan9port/unix/)。Ville Laurikari在1999年独立发现了Pike的算法，同样发展形成了理论基础[2]。

最后，如果不提到Jeffrey Friedl的著作*掌握正则表达式（Mastering Regular Expressions）*，也许是当今程序员最著名的参考，那么关于正则表达式的讨论就是不完整的。Friedl的书如何最好地**使用**目前的正则表达式匹配实现，但是没有讲如何最好地**实现**正则表达式匹配。与实现相关的一丁点文字使得人们普遍认为递归回溯是模拟NFA的唯一途径。Friedl使这一点很清楚：他[既不懂也不尊重](http://regex.info/blog/2006-09-15/248)背后的理论。

## 总结

使用已经知道数十年的基于有限自动机的方法，正则表达式匹配可以既简单又快速。相反，Perl，PCRE，Python，Ruby，Java，以及许多其他语言基于回溯递归的方法实现正则表达式匹配，这种方法简单，但是可能难以忍受的慢。除了反向引用外，基于缓慢的回溯方法的实现的正则表达式功能都可以被基于自动机的方法提供，并且速度适中如一的快。

这个系列的下一篇文章，”[正则表达式匹配：虚拟机方法](https://swtch.com/~rsc/regexp/regexp2.html)“，讨论了基于NFA方法的子串抽取。第三篇文章，”[自然环境中的正则表达式匹配](https://swtch.com/~rsc/regexp/regexp3.html)“，检查了正则表达式的生产实现。第四篇文章，”[带有卦索引的正则表达式匹配](https://swtch.com/~rsc/regexp/regexp4.html)“，解释了Google代码搜索是如何实现的。


## 致谢

Lee Feigenbaum, James Grimmelmann, Alex Healy, William Josephson, and Arnold Robbins read drafts of this article and made many helpful suggestions. Rob Pike clarified some of the history surrounding his regular expression implementation. Thanks to all.

## 引用

[1] L. Peter Deutsch and Butler Lampson, “An online editor,” Communications of the ACM 10(12) (December 1967), pp. 793–799. http://doi.acm.org/10.1145/363848.363863

[2] Ville Laurikari, “NFAs with Tagged Transitions, their Conversion to Deterministic Automata and Application to Regular Expressions,” in Proceedings of the Symposium on String Processing and Information Retrieval, September 2000. http://laurikari.net/ville/spire2000-tnfa.ps

[3] M. Douglas McIlroy, “Enumerating the strings of regular languages,” Journal of Functional Programming 14 (2004), pp. 503–518. http://www.cs.dartmouth.edu/~doug/nfa.ps.gz (preprint)

[4] R. McNaughton and H. Yamada, “Regular expressions and state graphs for automata,” IRE Transactions on Electronic Computers EC-9(1) (March 1960), pp. 39–47.

[5] Paul Pierce, “CTSS source listings.” http://www.piercefuller.com/library/ctss.html (Thompson's QED is in the file com5 in the source listings archive and is marked as 0QED)

[6] Rob Pike, “The text editor sam,” Software—Practice & Experience 17(11) (November 1987), pp. 813–845. http://plan9.bell-labs.com/sys/doc/sam/sam.html

[7] Michael Rabin and Dana Scott, “Finite automata and their decision problems,” IBM Journal of Research and Development 3 (1959), pp. 114–125. http://www.research.ibm.com/journal/rd/032/ibmrd0302C.pdf

[8] Dennis Ritchie, “An incomplete history of the QED text editor.” http://plan9.bell-labs.com/~dmr/qed.html

[9] Ken Thompson, “Regular expression search algorithm,” Communications of the ACM 11(6) (June 1968), pp. 419–422. http://doi.acm.org/10.1145/363347.363387 (PDF)

[10] Tom Van Vleck, “The IBM 7094 and CTSS.” http://www.multicians.org/thvv/7094.html

