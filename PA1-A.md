# Decaf PA 1-A 说明

## 任务概述
在本阶段中，大家的任务是编码实现 Decaf 语言编译器的词法分析和语法分析部分，同时生成抽象语法树。

![](assets/pa1a-1.png)

首先是**词法分析**（lexical analysis）。
你需要运用“词法分析程序自动生成工具”（lexer generator）jflex 生成 Decaf 语言的“词法分析程序”（lexer a.k.a. scanner）。
词法分析程序是前端分析的第一部分，它的功能是从左至右扫描 Decaf 源语言程序，识别
出标识符、保留字、整数常量、操作符等各种单词符号，并把识别结果以终结符的形式返回
给语法分析程序。这一部分的实验目的是要掌握 jflex 工具的用法，体会正规表达式、有限
自动机等理论的应用，并对词法分析程序的工作机制有比较深入的了解。

之后是**语法分析**（parsing）。
除了词法分析程序以外，大家还需要运用“语法分析程序自动生成工具”（parser generator）jacc 生成
Decaf 语言的语法分析程序（parser）。
在 PA1-A 中， Decaf 语法分析程序的功能是对词法分析输出
的终结符串（注意不是字符串）进行自底向上的分析。这一部分的实验目的是要初步掌握
jacc 工具的用法，体会上下文无关文法、LALR(1) 分析等理论的应用。

注意 PA1-A 的两部分是密切相关的：语法分析程序并不直接对 Decaf 源程序进行处理，
而是调用词法分析程序对 Decaf 源程序进行词法分析，然后对词法分析程序返回的终结符
序列进行归约，也就是说，词法分析程序输出的结果才是语法分析的输入。

最后，当语法分析程序运行结束，语法正确时，PA1-A 会生成与 Decaf 源程序对应的
抽象语法树。

PA1-A 是整个实验的热身阶段，需要自己完成的代码量较少，主要的工作在于建立编
程环境、熟悉代码框架、熟悉 Decaf 语言以及掌握 jflex 和 jacc 的具体用法等。

本阶段的测试要求是输出结果跟标准输出完全一致。我们保留了一些测试例子没有公
开，所以请自己编写一些测试例子，测试自己的编译器是符合 Decaf 语言规范里的相应规
定。

### 本阶段涉及的文件的说明
本阶段主要涉及的文件有

| 文件 | 含义 | 你的任务 |
| --- | --- | --- |
| Lexer.l | 词法分析的规则 | 定义什么正则表达式生成什么非终结符 |
| Parser.y | 语法分析的规则 | 定义新特性的语法规则和归约动作 |
| SemValue.java | 语法分析产生的非终结符都有一个相关的 SemValue 保存必要的信息 | 如果需要，增加新字段 |
| Tree.java | 抽象语法树的结点 | 如果需要，增加对应新特性的新结点 |

## 相关知识
### lambda 表达式
一等公民的函数（function as first-class citizens）（也称 lambda
表达式）指的是，函数能像普通的 int， string 一样，
通过字面量（literal）初始化、作为参数和返回值传递。
现代语言，Rust，Haskell，Scala，甚至 C++ 和 Go 都开始支持一等公民的函数。

例如 Java 语言中函数就不是一等公民，所以只能通过将函数包装成类的方法绕开这个限制。
下面的 `IntPredicate` 虽说是个 interface，但其实说是整数到布尔值的函数类型更合适。
但是 Java 不支持一等公民的函数，所以只好用 interface 和局部类来实现。
```java
interface IntPredicate { boolean judge(int x); }

class Main {
    static IntPredicate not_eq(int y) {
        return new IntPredicate() {
            public boolean judge(int x) { return x != y; }
        };
    }

    static IntPredicate eq(int y) {
        return new IntPredicate() {
            public boolean judge(int x) { return x == y; }
        };
    }

    static void check(String msg, IntPredicate p, int x) {
        System.out.println(msg + p.judge(x));
    }

    public static void main(String[] args) {
        check("3 != 3 is ", not_eq(3), 3);
        check("3 != 4 is ", not_eq(3), 4);
        check("3 == 3 is ", eq(3), 3);
        check("3 == 4 is ", eq(3), 4);
    }
}
```

lambda 表达式的用处主要便是简化编程，让你的程序更可读、更可维护、更健壮。
比较下面的 C 代码
```c
int l = 0;
for (int i = 0; i < n; i++) {
    if (a[i] % 2 == 0) continue;
    b[l++] = a[i] * 3;
}
```
和 Haskell 代码
```haskell
let odd x = x `mod` 2 /= 0 in    map (*3) $ filter odd a
```
后者更可读，更精准的传达了程序员的意图。

### 抽象语法树
所谓的抽象语法树（Abstract Syntax Tree），是指一种只跟我们关心的内容有关
的语法树表示形式。抽象语法是相对于具体语法而言的，所谓具体语法是指针对字符串形式
的语法规则，而且这样的语法规则没有二义性，适合于指导语法分析过程。抽象语法树是一
种非常接近源代码的中间表示，它的特点是：

1. 不含我们不关心的终结符，例如逗号等（实际上只含标识符、常量等终结符）。

2. 不具体体现语法分析的细节步骤，例如对于 `List ::= List Elem | Elem` 这样的规
则，按照语法分析的细节步骤来记录的话应该是一棵二叉树，但是在抽象语法树中我们
只需要表示成一个链表，这样更便于后续处理。

3. 可能在结构上含有二义性，例如加法表达式在抽象语法中可能是 `Expr -> Expr + Expr`，
但是这种二义性对抽象语法树而言是无害的——因为我们已经有语法树了。

4. 体现源程序的语法结构。

使用抽象语法树表示程序的最大好处是把语法分析结果保存下来，后面可以反复利用。
在面向对象的语言中描述抽象语法树是非常简单的：我们只需要为每种非终结符创建一
个类。

在我们的代码框架中我们已经为你定义好各种符号在 AST 中对应的数据结构。请在动
手实现之前大致了解一下 `Tree.java` 中所包含的各个类。

## 实验内容
本次实验给出了基础的 Decaf 框架，它完成了[《Decaf语言规范》](https://decaf-lang.gitbook.io/workspace/spec)。
本次实验你的任务是，在这个框架的基础上，完成新特性的词法语法分析。

### 新特性 1：抽象类。
加入 `abstract` 关键字，用来修饰类和成员函数。例如，
```
abstract class Abstract {
    abstract void abstractMethod();
}
```

### 新特性 2：局部类型推断
加入 `var` 关键字，用来修饰**局部变量**。例如
```
class Main {
    static void main() {
        // int i = 0;
        var i = 0; // identical as above
    }
}
```

### 新特性 3：Lambda 表达式

PA1-A 中你只需考虑词法和语法分析的问题。具体地，你需要支持

* **类型可能出现在括号中了**，即
```
type ::= ...
       | '(' type ')'
```

* **lambda 类型（即函数类型）**：
语法如
```
type ::= ...
       | type '(' type* ')'
```
括号左边的是返回值的类型，括号内的是诸参数的类型。


* **lambda 表达式**：有两种。
```
type ::= ...
       | 'fun' '(' ( type id )* ')' '=>' expr
       | 'fun' '(' ( type id )* ')' block

```
箭头 `'=>'` 不能分开，左边的是参数列表，右边是返回值。
如果右边是一个 block，那么 block 里面的 return 语句表示返回值（当然，没有 return 语句的话返回类型 void）。

* **函数调用**：原来只支持函数是成员函数（即 `MethodDef`），现在函数可以是任意表达式了。

### lambda 表达式的例子
对部分同学，lambda 表达式还是个新东西。所以下面是一些例子，帮助你理解。

* lambda 类型
```
void()               // 没有参数，返回类型 void

int(int)             // 接受一个 int 类型的参数，返回 int

string(int, int)     // 接受两个 int 类型的参数，返回 string

class Main(int, int) // 接受两个 int 类型的参数，返回 Main 的一个实例

int(int(int))        // 参数是一个 int 到 int 的函数，返回值是 int

(int(int))(int)      // 参数是一个 int，返回一个 int 到 int 的函数（返回值还可以继续接受参数）
```

* lambda 表达式
```
(int x) => x + 1                                      // 类型是 int(int)

(int x, int y) => x + y                               // 类型是 int(int, int)

(int x) => { if (x == 0) return "no"; return "yes"; } // 类型是 string(int)

(int x) => (int y) => x + y                           // 类型是 (int(int))(int).
// 上面一个函数是所谓 curry function 的例子。其实它就是 x+y，但是它不要求你把两个参数一次性都提供了。
// 而是先接受第一个参数 x，然后返回一个函数。被返回的函数还能接受参数 y。
// 比如这个函数接受参数 x=2 之后，返回新的 (int y) => 2 + y。再接受 y=3，返回 2+3=5。
```

* 函数调用
```
(int(int))(int) f = (int x) => (int y) => x + y;
Print(f(2)(3));                                     // 输出 5。这里有两次函数调用（不含 Print）！
```

## 实验评分和实验报告
实验评分分两部分
* 评测结果：80%。这部分是机器检查，要求你的输出和标准输出**一模一样**。我们会有未公开的测例。
* 实验报告：20%。用中文简要叙述你的工作内容。并且**回答以下问题**

1. 我们用 `Tree` 里面的嵌套类表示抽象语法树的结点。但这些嵌套类间还有继承关系。
  如果 A 继承了 B，那么语法上会不会 A 和 B 有什么关系？限用 100 字符内一句话说明。

2. Tree 的嵌套类和 SemValue 似乎有很多类似的地方，但他们有什么区别？限用 60 字符内的一句话说明。

3. 你如何保证 `=>` 是右结合的？

4. PA1-A 在概念上，如下图所示。输入程序 lex **做完**得到一个终结符序列，然后在**构建出**具体语法树，最后从具体语法树构建抽象语法树。
  这个概念模型和我们的代码有什么区别？我们的具体语法树在哪里？限用 120 字符内说明。
```
作为输入的程序：char[]
    --> lexer --> 终结符序列：Token[]
    --> parser --> 具体语法树：Tree<CSTNode>
    --> 一通操作 --> 抽象语法树：Tree<ASTNode>
```
