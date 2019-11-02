# PA2: 语义分析

## 任务概述

本阶段的内容是对 PA1-A 中生成的抽象语法树进行语义分析。

## 实验内容

本次实验将给出 decaf 基本框架，其中已经完成了[《Decaf 语言规范》](https://decaf-lang.gitbook.io/workspace/spec) 中所描述语言特征的词法、语法及语义分析。现在，你需要在前一阶段的词法语法分析(同时生成抽象语法树)的基础上，继续针对 Decaf 语言新增的语言特性进行语义分析。

基本框架的实验指导参见：

* [Java](https://decaf-lang.gitbook.io/decaf-book/java-kuang-jia-fen-jie-duan-zhi-dao/pa2-yu-yi-fen-xi)
* [Scala](https://decaf-lang.gitbook.io/decaf-book/scala-kuang-jia-fen-jie-duan-zhi-dao/pa2-yu-yi-fen-xi)
* [Rust](https://decaf-lang.gitbook.io/decaf-book/rust-kuang-jia-fen-jie-duan-zhi-dao/pa2-yu-yi-fen-xi)

简单来说，语义分析需要遍历两趟 AST，第一趟生成符号信息，第二趟进行类型检差。如果在某一趟遍历中发现语义错误，直接终止后续操作，输出编译错误；否则，格式化输出作用域和符号表信息。实验框架中统一定义了一些编译错误的类(如 Java 版在 `src/main/java/decaf/driver/error/`)，报错时选择最接近的错误类，对于新特性你可能需要添加新的错误类。

如果你做 PA1-A 时没有直接在框架整体上开发，而使用了只包含 PA1-A 的框架，我们提供了两种方式让你把原来的代码升级为 PA2：

1. 提供 PA2 相对于 PA1-A 修改部分的 patch 文件，你可使用 `git apply` / `patch` 等工具将修改应用于原来的代码。
2. 提供 PA2 的完整框架，你需要自己把 PA1-A 的工作复制到该框架上。

然后你需要充分理解基本框架中语义分析的代码结构以及功能，参考框架中对相似语言特征的语义处理过程，根据自己对语言的理解完成新增语言特性的语义分析。

相对于前两次 PA 来说，本次 PA 需要对基本框架做较多的修改，具有一定的难度，建议你合理安排时间，尽早开始动手。

### 新特性 1：抽象类

支持用 `abstract` 关键字定义抽象类，及修饰其中的抽象成员方法。首先主类 `Main` 不能是抽象的。由于 Decaf 不支持对成员变量和静态方法的重写 (override)，所以不允许出现抽象的成员变量和静态方法。此外，也不允许抽象方法重写基类中的非抽象方法，但可以重写同名且类型相同的抽象方法(虽然并没有什么用)。

新增两类错误类型：

#### 错误 1

若一个类中含有抽象成员，或者它继承了一个抽象类但没有重写所有的抽象方法，那么该类必须声明为抽象类。

* 错例 1-1：

    ```java
    class Foo {
        abstract int foo();
        abstract string bar();
    }
    ```

    报错：

    ```
    *** Error at (1,5): 'Foo' is not abstract and does not override all abstract methods
    ```

* 错例 1-2：

    ```java
    abstract class Foo {
        abstract int foo();
    }

    class Baz extends Foo {
        int baz() { return 1; }
    }
    ```

    报错：

    ```
    *** Error at (5,5): 'Baz' is not abstract and does not override all abstract methods
    ```

在实现上，可对每个类记录它所有“未被重载的抽象方法”列表。每当处理一个类时，先处理其父类，然后将该列表初始化为父类的列表。之后每遇到一个抽象方法，将方法名加入该列表；每重载了一个抽象方法，从列表中删去该方法。如果最后列表非空则该类需要是抽象的。

#### 错误 2

抽象类不能使用 `new` 进行实例化。

* 错例 1-3：

    ```java
    abstract class Foo {
        abstract int foo();
    }

    class Main {
        static void main() {
            new Foo();
        }
    }
    ```

    报错：

    ```
    *** Error at (7,13): cannot instantiate abstract class 'Foo'
    ```

### 新特性 2：局部类型推导

加入 `var` 关键字，支持编译器进行局部变量的类型推导。

#### 类型推导

* 例 2-1：

    ```java
    class A {}
    class B extends A {}

    class Main {
        static void main() {
            var i = 1;
            var s = "123";
            var a = new int[233];
            var f = fun (int x) => fun (int y) => x;
            var b = new B();
        }
    }
    ```

    对于上述程序，各变量的的类型分别推导为：

    ```
    i : int
    s : string
    a : int[]
    f : int => int => int
    b : class B
    ```

#### 报错

如果类型推导的结果为 `void` 需要报错。

* 错例 2-2：

    ```java
    class Main {
        static void main() {
            var m = main();
        }
    }
    ```

    报错：

    ```
    *** Error at (3,13): cannot declare identifier 'm' as void type
    ```

### 新特性 3：First-class Functions

这是本学期新特性中最复杂的一部分。本阶段的实现过程可以分为下面几部分：

#### 函数类型

你需要正确识别函数类型的声明，并在之后格式化输出符号表时也格式化输出其类型。下面是几个例子：

| 声明类型 | 格式化输出结果 | 说明 |
|---------|-------------|-----|
| `void()`              | `() => void`              | 没有参数，返回类型 `void` |
| `int(int)`            | `int => int`              | 接受一个 `int` 类型的参数，返回 `int` |
| `string(int, int)`    | `(int, int) => string`    | 接受两个 `int` 类型的参数，返回 `string` |
| `class Main(int, int)`| `(int, int) => class Main`| 接受两个 `int` 类型的参数，返回 `Main` 的一个实例 |
| `int(int(int))`       | `(int => int) => int`     | 参数是一个 `int` 到 `int` 的函数，返回值是 `int` |
| `int(int)(int)`       | `int => int => int`       | 参数是一个 `int`，返回一个 `int` 到 `int` 的函数（注意这里的返回值也是一个函数） |

在基础框架中已经实现了各类型的格式化输出，你无需修改。

如果一个函数类型的参数被声明为 `void` 类型，需要报错：

* 错例 3-1：

    ```java
    class Main {
        static void main() {
            int(int, void) f;
        }
    }
    ```

    报错：

    ```
    *** Error at (3,18): arguments in function type must be non-void known type
    ```

#### Lambda 表达式作用域

当定义一个 Lambda 表达式时，相当于打开了一层新的局部作用域 `LocalScope`，包含所有参数与内部声明的变量。作用域中变量的访问规则与普通局部作用域类似：

1. 内层作用域可以访问到外层作用域的所有符号。
2. 在局部作用域中声明的符号不能与与任何外层作用域的符号重名。

而不同之处在于：

1. 不能对捕获的外层作用域中的符号进行赋值。
2. 如果要将 Lambda 表达式赋值给一个符号，则 Lambda 内部作用域中的变量既不能与该符号重名，也不能访问到该符号。

例子：

* 错例 3-2：

    ```java
    class Main {
        static void main() {
            int x = 0;
            var addx = fun() {
                x = x + 1;
                int[] y = new int[10];
                var addy = fun() {
                    y[0] = y[0] + 1;
                };
            };
        }
    }
    ```

    报错：

    ```
    *** Error at (5,15): cannot assign value to captured variables in lambda expression
    *** Error at (8,22): cannot assign value to captured variables in lambda expression
    ```

* 错例 3-3：

    ```java
    class Main {
        static void main() {
            var f = fun (int x) {
                var g = f;
            };
        }
    }
    ```

    报错：

    ```
    *** Error at (4,21): undeclared symbol 'f'
    ```

* 错例 3-4：

    ```java
    class Main {
        static void main() {
            var f = fun (int x) {
                var f = fun (int y) => x + y;
            };
        }
    }
    ```

    报错：

    ```
    *** Error at (4,17): declaration of 'f' here conflicts with earlier declaration at (3,13)
    ```

在实现上，为了后续返回类型推导以及代码生成(PA3)的方便，我们建议你对 Lambda 表达式定义一种新的作用域 `LambdaScope`，定义一种新的符号 `LambdaScope`，以便对 Lambda 表达式具有的属性进行更好的管理。当然我们不会评估你是否真的做了这件事，用自己认为方便的方法实现即可。

#### Lambda 表达式返回类型

你需要正确推导出 Lambda 表达式的返回类型。

记 <: 是类型上的二元关系，它满足：

- 自反性：t <: t
- 传递性：If t1 <: t2 and t2 <: t3, then t1 <: t3
- 类继承：If c1 extends c2, then ClassType(c1) <: ClassType(c2)
- 函数：If t <: s and si <: ti for every i, then FunType([t1, t2, ..., tn], t) <: FunType([s1, s2, ..., sn], s)

对于 `fun (t1 x1, t2 x2, ...) => y`，若推导出 `y` 的类型为 `t`，那么整个 Lambda 表达式的类型为 `FunType([t1, t2, ...], t)`。

若 `y` 是一个 `BlockStmt`，如果 `BlockStmt` 的所有执行路径都没有 `return` 语句，则 `y` 的类型是 `void`；否则 `y` 的类型是所有 `return` 语句返回值类型的最小“上界”(无返回值的 `return` 语句可认为返回 `void` 类型)。定义：称 t 是类型 t1, t2, ..., tn 的上界，若 t1 <: t, t2 <: t , ..., tn <: t。若该上界不存在，或返回类型不是 `void` 但存在一条执行路径无返回值，那么需要报错。

* 例 3-5：

    ```java
    class A {}
    class B extends A {}

    class Main {
        static void main() {
            var abs = fun (int x) {
                if (x >= 0)
                    return x;
                else
                    return -x;
            };
            var print = fun (int x) {
                if (x < 0) return;
                Print(x);
            };
            var test = fun (class A a, class B b) {
                if (false)
                    return fun (class A a1, class B b1) => a;
                else
                    return fun (class B b2, class A a2) => b;
            };
        }
    }
    ```

    三个 Lambda 表达式类型分别为：

    ```
    abs : int => int
    print : int => void
    test : (class A, class B) => (class B, class B) => class A
    ```

* 错例 3-6：

    ```java
    class Main {
        static void main() {
            var cmp = fun (int x) {
                if (x > 0)
                    return 1;
                else if (x < 0)
                    return "-1";
                else
                    Print("equal");
            };
        }
    }
    ```

    报错：

    ```
    *** Error at (3,31): missing return statement: control reaches end of non-void block
    *** Error at (3,31): incompatible return types in blocked expression
    ```

实现时，对每个 Lambda 表达式你需要先想办法得到它内部的所有 `return` 语句，然后可用递归的方式求它们类型的上界。

#### 函数变量

可直接将一个方法名赋值给一个局部变量，此变量就会具有函数类型，可像函数一样调用。

首先你需要把基础框架中的形如 `undeclared variable 'name'` 之类的报错信息，改为 `undeclared symbol 'name'`，因为此时的符号可能不仅仅是一个变量，还可能是一个函数。

然后你需要理解并修改框架中对抽象语法树 `VarSel` 节点的处理，原来只能处理类的成员变量，你需要将其扩展为支持类的成员方法。当将方法名赋值给一个变量时，方法名的访权限与调用该方法时的情形相同，例如 `var f = A.print;` 没有访问权限错误当且仅当在同一地点访问 `A.print(...);` 时没有权限错误。下面是几个错误的例子：

* 错例 3-7：

    ```java
    class A {
        int sf(int x) { return x - 1; }
        int(int) vf;
    }

    class Main {
        int f(int x) { return x + 2; }

        static void main() {
            var a = new A();
            var f1 = f;         // bad
            var f2 = a.sf;
            var f3 = a.vf;      // bad
        }
    }
    ```

    报错：

    ```
    *** Error at (11,18): can not reference a non-static field 'f' from static method 'main'
    *** Error at (13,20): field 'vf' of 'class A' not accessible here
    ```

    `f1` 是因为在 `class Main` 的 `static` 方法里访问了非 `static` 字段；`f3` 是因为在 `class Main` 里无法访问 `class A` 的私有字段；而 `f2` 赋值成功，具有类型 `int => int`。

#### 函数调用

原来只能调用成员方法和静态方法，现在可以调用任意类型为函数类型的表达式（其本质就是个函数）。

在 PA1 中，我们已经将语法规范从原来的

```text
call ::= (expr '.')? id '(' exprList ')'
```

改为了

```text
call ::= expr '(' exprList ')'
```

因此基础框架中对抽象语法树 `Call` 节点的处理需要重写。

首先，如果一个表达式不是函数类型需要报错：

* 错例 3-8：

    ```java
    class Main {
        static void main() {
            var str = "2333";
            str(2333);
        }
    }
    ```

    报错：

    ```
    *** Error at (4,12): string is not a callable type
    ```

如果参数个数或类型不匹配也要报错：

* 错例 3-9：

    ```java
    class Main {
        static void main() {
            var f = fun(int x, int y) => x + y;
            f(1);
            f(1, "2");
            (fun (int x, int y) => x * y)(1, 2, 3);
        }
    }
    ```

    报错：

    ```
    *** Error at (4,10): function 'f' expects 2 argument(s) but 1 given
    *** Error at (5,14): incompatible argument 2: string given, int expected
    *** Error at (6,38): lambda expression expects 2 argument(s) but 3 given
    ```

    注意当参数个数不匹配时，如果被调用者有符号名(如第一个错误)，应该输出 `function 'name' expects x argument(s) but y given`，如果被调用者是个表达式没有符号名(如第三个错误)，只需输出 `lambda expression expects x argument(s) but y given`。

其余诸如方法访问权限的错误，已在上一节“函数变量”中报过，此时无需再报。

### 符号表格式化打印

在 PA2 阶段，我们最终会将你构造出来的作用域和符号表进行格式化打印，并与标准输出比对是否一致。基础框架已实现该功能(如 Java 版在 `src/main/java/decaf/printing/PrettyScope.java`)，对于一个作用域，打印格式都遵循如下流程：

1. 打印作用域名(`GlOBAL`、`CLASS`、`FORMAL`、`LOCAL`)
2. 增加缩进
3. 打印该作用域直接包含的符号位置、名称与类型。
3. 遍历所有子作用域，依次递归打印它们
4. 减少缩进

新增特性的打印格式如下：

#### 抽象类/方法

打印抽象类/方法时，其修饰符 `abstract` 打印为 `ABSTRACT`。如

```java
abstract class Foo {
    abstract void bar();
}
class Main {
    static void main() {}
}
```

对应的符号表打印为：

```
GLOBAL SCOPE:
    (1,10) -> ABSTRACT class Foo
    (4,1) -> class Main
    CLASS SCOPE OF 'Foo':
        (2,19) -> ABSTRACT function bar : () => void
        FORMAL SCOPE OF 'bar':
            (2,19) -> variable @this : class Foo
    CLASS SCOPE OF 'Main':
        (5,17) -> STATIC function main : () => void
        FORMAL SCOPE OF 'main':
            <empty>
            LOCAL SCOPE:
                <empty>
```

#### `var` 局部变量定义

此时变量类型应该已被确定，与其他直接给出类型的变量格式一样，如：

```java
static void main() {
    var i = 0;
    var b = true;
    var c = new Main();
}
```

对应的符号表打印为：

```
...
FORMAL SCOPE OF 'main':
    <empty>
    LOCAL SCOPE:
        (3,13) -> variable i : int
        (4,13) -> variable b : bool
        (5,13) -> variable c : class Main
```

#### Lambda 表达式

与局部作用域 `LocalScope` 一样，符号包括参数和其内部定义的变量，如

```java
static void main() {
    var f = fun(int x) {
        var g = fun(int y) => x + y;
    };
}
```

对应的符号表打印为：

```
...
FORMAL SCOPE OF 'main':
    <empty>
    LOCAL SCOPE:
        (6,13) -> variable f : int => void
        LOCAL SCOPE:
            (6,25) -> variable x : int
            (7,17) -> variable g : int => int
            LOCAL SCOPE:
                (7,29) -> variable y : int
```

## 实验评分和实验报告

我们提供了若干测试程序和标准输出，你的输出需要与标准输出完全一致才行。我们还保留了一些未公开的测试例子，但考虑到报错这件事本身没有标准答案，我们未公开的测例中不会有公开测例不涵盖的报错情况，只要你没有使用非常规的特判，通过了所有公开测例后基本也能通过所有未公开测例。

实验评分分两部分：

- 评测结果：80%。这部分是机器检查，要求你的输出和标准输出**一模一样**。我们会有**未公开**的测例。
- 实验报告（根目录下 `report-PA2.pdf` 文件）：20%。要求用中文简要叙述你的工作内容。

此外，请在报告中回答以下问题(请根据你选择的实验框架回答)：

1. 实验框架中是如何实现根据符号名在作用域中查找该符号的？在符号定义和符号引用时的查找有何不同？

2. 对 AST 的两趟遍历分别做了什么事？分别确定了哪些节点的类型？

2. 在遍历 AST 时，是如何实现对不同类型的 AST 节分发相应的处理函数的？请简要分析。