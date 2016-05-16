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



