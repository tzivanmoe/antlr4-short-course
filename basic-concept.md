# 基本概念

一门语言由有效的句子组成，一个句子由短语组成，一个短语由子短语和词汇符号组成。要实现一门语言，我们必须构建一个能读取句子以及对发现的短语和输入符号作出适当反应的应用。

这样的应用必须能识别特定语言的所有有效的句子、短语和子短语。识别一个短语意味着我们能确定短语的各种组件并能指出它与其它短语的区别。例如，我们把输入a=5识别为赋值语句，这就意味着我们知道a是赋值目标以及5是要存储的值。识别赋值语句a=5也意味着应用认为它是明显不同于，比如说，a+b语句的。在识别后，应用将执行适当的操作，例如performAssignment("a", 5)或者translateAssignment("a", 5)。

识别语言的程序被称为语法分析器。语法指代控制语言成员的规则，每条规则都表示一个短语的结构。为了更容易地实现识别语言的程序，通常我们会把识别语言的语法分析拆解成两个相似但不同的任务或阶段。

把字符组成单词或符号（记号）的过程被称为词法分析或简单标记化。我们把标记输入的程序称为词法分析器。词法分析器能把相关的记号组成记号类型，例如INT（整数）、ID（标志符）、FLOAT（浮点数）等。当语法分析器只关心类型的时候，词法分析器会把词汇符号组成类型，而不是单独的符号。记号至少包含两块信息：记号类型（确定词法结构）和匹配记号的文本。

第二阶段是真正的语法分析器，它使用这些记号去识别句子结构，在本例中是赋值语句。默认情况下，ANTLR生成的语法分析器会构建一个称为语法分析树或语法树的数据结构，它记录语法分析器如何识别输入句子的结构和它的组件短语。下图阐明了语言识别器的基本数据流：

![](http://codemany.com/uploads/basic-data-flow.png)

语法分析树的内部节点是分组和确认它们子节点的短语名字。根节点是最抽象的短语名字，在本例中是prog（“program”的缩写）。语法分析树的叶子节点永远是输入记号。

通过生成语法分析树，语法分析器给应用的其余部分提供了方便的数据结构，它们含有关于语法分析器如何把符号组成短语的完整信息。树是非常容易处理的，并且也能被程序员很好的理解。更好的是，语法分析器能自动地生成语法分析树。

通过操作语法分析树，需要识别相同语言的多个应用能重用同一个语法分析器。当然，你也可以选择直接在语法中嵌入特定应用的代码片段，这是语法分析器生成器传统的做法。ANTLR v4仍然允许这样做，但是语法分析树有助于更简洁更解耦的设计。

语法分析树对于需要多次树遍历的转换也是非常有用的，因为在计算依赖关系的阶段通常会需要前一个阶段的信息。相比于在每个阶段都要准备输入字符，我们只需要遍历语法分析树多次，更具有效率。

因为我们用一套规则指定短语，语法分析树子树根节点对应于语法规则名。这里的语法规则对应于上图中assign子树的第一层：

```
assign : ID '=' expr ;    // 匹配赋值语句像"a=5"
```

明白ANTLR如何把这些规则转换为人类可读的语法分析代码是使用和调试语法的基础，因此让我们深入地挖掘语法分析是如何工作的。

## 实现语法分析器

ANTLR工具根据语法规则，例如我们刚才看到的assign，生成递归下降语法分析器。递归下降语法分析器只是递归方法的一个集合，每个规则一个方法。下降这个术语指的是分析从语法分析树的根开始向着叶子进行（记号）。我们首先调用的规则，即prog符号，成为语法分析树的根。那也就意味着对前面部分的语法分析树来说需要调用方法prog()。这类分析更通用的术语是自顶向下分析：递归下降语法分析器仅仅是自顶向下语法分析器实现的一种。

要了解递归下降语法分析器是什么样子，这里是ANTLR为规则assign生成的方法（稍微整理）：

```
// assign : ID '=' expr ;
void assign() {    // 根据规则assign生成的方法
    match(ID);     // 比较ID和当前输入符号然后消费
    match('=');
    expr();        // 通过调用expr()匹配表达式
}
```

递归下降语法分析器最酷的部分是通过调用方法prog()、assign()和expr()跟踪出的调用关系图反映了内部的语法分析树节点。match()的调用对应语法分析树叶子。为了在一个手工构建的语法分析器中手动构建一颗语法分析树，我们需要在每个规则方法的开始处插入“添加新子树根”操作，以及给match()一个“添加新叶子节点”操作。

方法assign()只是检查确保所有必要的记号存在且以正确的顺序。当语法分析器进入assign()时，它不必在多个选项之间进行选择。选项是规则定义右边的选择之一。例如，调用assign的prog规则可能有其它类型的语句。

```
/** 匹配起始于当前输入位置的任何语句 */
prog
    : assign    // 第一个选项（'|'是选项分隔符）
    | ifstat    // 第二个选项
    | whilestat
    ...
    ;
```

prog的分析规则看起来像一条switch语句：

```
void prog() {
    switch ( «current input token» ) {
        CASE ID : assign(); break;
        CASE IF : ifstat(); break;    // IF是关键字'if'的记号类型
        CASE WHILE : whilestat(); break;
        ...
        default : «raise no viable alternative exception»
    }
}
```

方法prog()必须通过检查下一个输入记号作出分析决定或预测。分析决定预判哪一个选项将会成功。在本例中，当看到WHILE关键字时会预判是规则prog的第三个选项。规则方法prog()然后就会调用whilestat()。你以前可能听说过术语预读记号，那只是下一个输入记号。预读记号可以是语法分析器在匹配和消费它之前嗅探的任何记号。

有时候，语法分析器需要一些预读记号去预判哪个选项会成功。它甚至必须考虑从当前位置直到文件结尾的所有的记号！ANTLR默默地为你处理所有的这些事情，但是对决策过程有个基本的了解是有帮助的，可以让调试生成的语法分析器更容易。

为更好地理解分析决定，想象有个单一入口和单一出口的迷宫，有单词写在地板上。每个沿着从入口到出口路径的单词序列表示一个句子。迷宫的结构与定义一门语言的语法规则类似。为测试一个句子在一门语言中的成员身份，我们在穿越迷宫时把句子的单词和沿着地板的单词作比较。如果通过句子的单词我们能到达出口，那么句子是有效的。

为了通过迷宫，我们必须在每个岔口选择一条有效路径，正如我们必须在语法分析器中选择选项。我们必须决定该走哪条路，通过把我们句子中下一个单词（们）和沿着来自每个岔口的每条路径上可见的单词比较。我们能从岔口看到的单词与预读记号类似。当每条路径以唯一的单词开始时决定是相当容易的。在规则prog中，每个选项从唯一的记号开始，因此prog()可以通过查看第一个预读记号识别选项。

当单词从一个岔口重叠部分开始每条路径时，语法分析器需要继续往前看，扫描可以识别选项的单词。ANTLR根据需要为每个决定自动上下调节预读数量。如果预读的结果是多条同样的到出口的路径，即当前的输入短语有多种解释。解决这样的二义性将是我们的下一个主题。

## 二义性

一个模棱两可的短语或句子通常是指它有不止一种解释。换句话说，短语或句子能适配不止一种语法结构。要解释或转换一个短语，程序必须要能唯一地确认它的含义，这意味着我们必须提供无歧义的语法，以便生成的语法分析器能用明确的一个方法匹配每个输入短语。

在这里，让我们展示一些有歧义的语法以便让二义性的概念更具体。如果你以后在构建语法时陷入二义性，你可以参考本节的内容。

一些明显有歧义的语法：

```
assign
    : ID '=' expr    // 匹配一个赋值语句，例如f()
    | ID '=' expr    // 前面选项的精确复制
    ;

expr
    : INT ;
```

大多数时候二义性是不明显的，如同以下的语法，它能通过规则stat的两个选项匹配函数调用：

```
stat
    : expr          // 表达式语句
    | ID '(' ')'    // 函数调用语句
    ;

expr
    : ID '(' ')'
    | INT
    ;
```

这里是两个输入f()的解释，从规则stat开始：

![](http://codemany.com/uploads/fn-parse-tree.png)

左边的语法分析树显示f()匹配规则expr。右边的语法分析树显示f()匹配规则stat的第二个选项。

因为大部分语言它们的语法都被设计成无歧义的，有歧义的语法类似于编程缺陷。我们需要识别语法以便为每个输入短语提交单一选择给语法分析器。如果语法分析器发现一个有歧义的短语，它必须选一个可行的选项。ANTLR通过选择涉及决定的第一个选项解决二义性。在本例中，语法分析器将选择与左边的语法分析树有关的f()的解释。

二义性可以发生在词法分析器中也能发生在语法分析器中，但ANTLR可以自动地解决它们。ANTLR通过使输入字符串和语法中第一个指定的规则匹配来解决词法二义性。为了明白这是如何工作的，让我们看看对大部分编程语言都很普遍的二义性：在关键字和标志符规则中的二义性。关键字begin（后面有个非字母）也是标志符，至少词法上，因此词法分析器可以匹配b-e-g-i-n到两者中的任何一个规则。

```
BEGIN : 'begin' ;    // 匹配b-e-g-i-n序列，即把二义性解析为BEGIN
ID    : [a-z]+ ;     // 匹配一个或多个任意小写字母
```

注意，词法分析器会试着为每个记号尽可能匹配最长的字符串，这意味着输入beginner将仅匹配规则ID。词法分析器不会把beginner匹配成BEGIN随后ID匹配输入ner。

有时候语言的语法就明显有歧义，没有任何的语法重组能改变这个事实。例如，算术表达式的自然语法可以用两种方式解释输入像1+2*3这样，要么执行运算符从左到右，要么像大部分语言那样按优先级顺序。

C语言展示了另一种二义性，但我们可以使用上下文信息比如标志符如何被定义来解决它。考虑代码片段i*j;。在语法上，它看起来像是一个表达式，但它的含义或者语义依赖i是类型名还是变量。如果i是类型名，那么这个片段不是表达式，而是一个声明为指向类型i的指针变量j。

## 语法分析树

为制作语言应用，我们必须为每个输入短语或子短语执行一些适当的代码，那样做最简单的方法是操作由语法分析器自动创建的语法分析树。

早些时候我们已经学习了词法分析器处理字符和把记号传递给语法分析器，然后语法分析器分析语法和创建语法分析树的相关知识。对应的ANTLR类分别是CharStream、Lexer、Token、Parser和ParseTree。连接词法分析器和语法分析器的管道被称为TokenStream。下图说明了这些类型的对象如何连接到内存中其它的对象。

![](http://codemany.com/uploads/basic-data-structure.png)

这些ANTLR数据结构分享尽可能多的数据以便节省内存的需要。上图显示在语法分析树中的叶子（记号）节点含有在记号流中记号的点。记号记录开始和结束字符在CharStream中的索引，而不是复制子串。这里没有与空格字符有关的记号，因为我们假设我们的词法分析器扔掉了空格。

下图显示的是ParseTree的子类RuleNode和TerminalNode以及它们所对应的子树根节点和叶子节点。RuleNode包含有方法如getChild()和getParent()等，但RuleNode并不专属于特定语法所有。为支持更好地访问在特定节点中的元素，ANTLR为每个规则生成一个RuleNode子类。下图为我们显示了赋值语句例子的子树根节点的特定类，它们是ProgContext，AssignContext和IntContext：

![](http://codemany.com/uploads/parse-tree-node.png)

因为它们记录了我们知道的通过规则对短语识别的每件事，所以这些被称为上下文对象。每个上下文对象知道被识别短语的开始和结束记号以及提供对所有短语的元素的访问。例如，AssignContext提供方法ID()和INT()去访问标志符节点和表达式子树。

给出了具体类型的描述，我们可以手工写代码去执行树的深度优先遍历。当我们发现和完成节点时我们可以执行任何我们想要的动作。典型的操作是诸如计算结果，更新数据结构，或者生成输出。相比每次为每个应用写同样的树遍历样板代码，我们可以使用ANTLR自动生成的树遍历机制。

## 监听器和访问者

ANTLR在它的运行库中为两种树遍历机制提供支持。默认情况下，ANTLR生成一个语法分析树Listener接口，在其中定义了回调方法，用于响应被内建的树遍历器触发的事件。

在Listener和Visitor机制之间最大的不同是：Listener方法被ANTLR提供的遍历器对象调用；而Visitor方法必须显式的调用visit方法遍历它们的子节点，在一个节点的子节点上如果忘记调用visit方法就意味着那些子树没有得到访问。

让我们首先从Listener开始。在我们了解Listener之后，我们也将看到ANTLR如何生成遵循Visitor设计模式的树遍历器。

### 语法分析树Listener

在Calc.java中有这样两行代码：

```
ParseTreeWalker walker = new ParseTreeWalker();
walker.walk(new DirectiveListener(), tree);
```

类ParseTreeWalker是ANTLR运行时提供的用于遍历语法分析树和触发Listener中回调方法的树遍历器。ANTLR工具根据Calc.g中的语法自动生成ParseTreeListener接口的子接口CalcListener和默认实现CalcBaseListener，其中含有针对语法中每个规则的enter和exit方法。DirectiveListener是我们编写的继承自CalcBaseListener的包含特定应用代码的实现，把它传递给树遍历器后，树遍历器在遍历语法分析树时就会触发DirectiveListener中的回调方法。

![](http://codemany.com/uploads/calc-listener-hierachy.png)

下图左边的语法分析树显示ParseTreeWalker执行了一次深度优先遍历，由粗虚线表示，箭头方向代表遍历方向。右边显示的是语法分析树的完整调用序列，它们由ParseTreeWalker触发调用。当树遍历器遇到规则assign的节点时，它触发enterAssign()并且给它传递AssignContext语法分析树节点。在树遍历器访问完assign节点的所有子节点后，它触发exitAssign()。

![](http://codemany.com/uploads/listener-call-sequence.png)

Listener机制的强大之处在于所有都是自动的。我们不必要写语法分析树遍历器，而且我们的Listener方法也不必要显式地访问它们的子节点。

### 语法分析树Visitor

有些情况下，我们实际想要控制的是遍历本身，在那里我们可以显式地调用visit方法去访问子树节点。选项-visitor告诉ANTLR工具从相应语法生成Visitor接口和默认实现，其中含有针对语法中每个规则的visit方法。

下图是我们熟悉的Visitor模式操作在语法分析树上。左边部分的粗虚线表示语法分析树的深度优先遍历，箭头方向代表遍历方向。右边部分指明Visitor中的方法调用序列。

![](http://codemany.com/uploads/visitor-call-sequence.png)

下面是Calc.java中的两行代码：

```
EvalVisitor eval = new EvalVisitor();
// To start walking the parse tree
eval.visit(tree);
```

我们首先初始化自制的树遍历器EvalVisitor，然后调用visit()去访问整棵语法分析树。ANTLR运行时提供的Visitor支持代码会在看到根节点时调用visitProg()。在那里，visitProg()会把子树作为参数调用visit方法继续遍历，如此等等。

![](http://codemany.com/uploads/calc-visitor-hierachy.png)

ANTLR自动生成的Visitor接口和默认实现可以让我们为Visitor方法编写自己的实现，让我们避免必须覆写接口中的每个方法，让我们仅仅聚焦在我们感兴趣的方法上。这种方法减少了我们学习ANTLR必须要花费的时间，让我们回到我们所熟悉的编程语言领域。