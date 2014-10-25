title: 分析C++中结构体对齐
id: 25
categories:
  - 技术
  - 未分类
date: 2010-06-30 23:30:03
tags:
---

很多C++或者驱动程序或者嵌入式设计者都会对结构体对齐问题深有感触。错与对之间竟然就是这么单纯，有没有经历过驱动调试蓝屏？除了分析蓝屏码之外是不是会无意识推理到到参数长度匹配问题？或许是布局又出问题了？同样的，作为.NET开发人员，如果通过互操作跟本地代码进行交互话，出现这种问题的情况就更多了，如果想知道是否是结构体封送出现了问题？Marshals.Sizeof()一下就知道了。

结构体在被编译器编译的时候会被自动的按照某种方式对齐排序，其目的是减少CPU读取数据的次数以提高访问效率。不同操作系统会有不同的对齐策略，本文讨论的是Windows操作系统对齐策略。结构体对齐分为自然对齐和指定对齐方式。

### 先谈自然对齐

有结构体：
`struct X
{
bool A;
int B;
int C;
byte D;
double E;
bool F;
};`
在Windows操作系统中默认的对齐方式下的字节空间是多少哪?  当然绝对不可能是19，而是sizeof(x) = 32.

总结一下Windows默认的严格自然对齐方式是这样的:
_1.各类型成员在结构体中的地址能被自己长度整除.__ 且对齐的大小标准按照结构体内最长的那个成员为准._
如：bool类型变量地址能被1整出;int 类型变量地址能被4整除; double类型变量能被8整除. 因此对齐的严格度 bool&lt;Int&lt;double. 这个时候我们可以分析上面的结构体按照double变量长度对齐_._

_2.整个结构体大小能被其中字节长度最长的那个字节长度整除。_
如:上面整个结构体长度大小必须能够被double类型变量长度整除。这是因为结构体数组也会顺序排列的原因吧。

按照上面的对齐准则结构体X按照8个字节对齐，
E的地址能被8整除，
A和B可以填充到前8个字节中，
C和D可以填充到后8个字节中，
E独占8个字节，
F独占8个字节。如图：

![](http://pic002.cnblogs.com/img/huomm23/201006/2010060323385089.jpg)
上图中我们也看到了结构体只需要19个字节大小的内存空间，但实际上系统申请的地址空间确是32个字节，13个字节空间被浪费掉。这种现象在平时应用开发感觉不到什么，但是在类似Wince等资源限制严格的环境下这种布局是无法容忍的，特别如果我们需要构造超长的结构体数组时必将吃掉大量内存。说到这里我们自然而然的开始重构结构体布局了：

现在调整代码布局如下：
`struct X
{
double E;
int B;
int C;
bool A;
byte D;
bool F;
}`
更改后的布局图如下：

![](http://pic002.cnblogs.com/img/huomm23/201006/2010060323392738.jpg)
显而易见稍微调整了一下布局，就减少了8个字节空间。

总结一下减小内存可以有下面一些规则：
_1.尽量使每一个对象成员都连续存放_
_2.尽量不要让布局中间留下填充空洞，如果没有办法阻止填充的话宁愿填充到最后。_
_3.如果实现以上规则可以按照字节大小顺序从前到后进行声明数据成员，如果DOUBLE类型声明在最前面，bool类型生命在最后面。_

### 在最优化布局的前提下尽力充分的利用地址空间。

再看一个例子：
<div>struct Y
{
int A;
short B;
byte C;
};</div>
此结构体对象按照自然对齐会占用8个字节，其中最后一个字节自动填充的。那么我们考虑下变量C，如果没有特别的意义为什么不把他声明为ushort类型哪？不但充分利用了空间而且还便于应对今后对数据的有限扩展。

### 排序从大到小也不是必须的，具体问题具体分析。

看一个例子：
<div>struct Z
{
ushort A;
bool B;
ushort C[1]
}</div>
分析上面布局，普遍的想法是： B和C切换，或者按顺序C,A,B 更好一些，可能这种分析大部分情况下是没有问题的。

但是具体需求一出来就不这样了比如：此结构体是用来构造一个动态对象的，即：在结构体对象中C数组的大小在运行时是不确定，这样在实际应用中我就可以动态的在运行时扩充C的大下，这在驱动程序中是很常见的。那么如果有这种需求的话，从大到小排序的规则就需要好好考虑了，否则应用起来问题是一定会出现的。而且很严重。

### 自定义对齐

以上讲到的都是自然对齐，很多时候特别驱动开发中沿用的是自定义对齐，这种对齐方式相对于编译器自动对齐读起来更为明确，如果是针对接口编程的结构体声明，那么这种方式更便于调用的稳定性。可以在代码中使用预编译指令显示声明对齐方式：
我们可以显式指定的对齐方式有 1，2，4，8，16，下面声明的结构体显式指定为8个字节对齐：
_#pragma pack(push,8)_
_结构体_
_#pragma pack(pop)_

### 总结一下

前面列举的是简单类型的结构体，实际上复杂类型的结构体布局是一样的，最简单的分析方式是将内部结构体分解，并按照以上方式分析就可以了。

再次总结下分析结构体对齐的一些原则：
_1.判断大小，找出自然对其中最严格（长度最大的那个成员变量）如果是复杂类型，先拆后分析或者嵌套分析。_
_2.进行布局计算，各个成员变量地址都能被本身大小整除_
_3.整个结构体对象的大小也能够被要求最严格的对齐大小整出（数组原因）_
_4.调整对齐布局时一般按大小排序，但是遇到数组等具体问题时需要具体分析。_
以上讨论的是普通基于Windows操作系统的编译布局，如果是Linux其沿用的对齐策略可能是两个字节类型变量按照两字节对齐，大于两个的按照4个字节对齐。