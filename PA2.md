# PA2: 语义分析

## 任务概述

本阶段的内容是对 PA1-A 中生成的抽象语法树进行语义分析。

## 实验内容

本次实验将给出 decaf 基本框架，其中已经完成了[《Decaf 语言规范》](https://decaf-lang.gitbook.io/workspace/spec) 中所描述语言特征的词法、语法及语义分析。现在，你需要在前一阶段的词法语法分析(同时生成抽象语法树)的基础上，继续针对 Decaf 语言新增的语言特性进行语义分析。

基本框架的实验指导参见：

* [Java](https://decaf-lang.gitbook.io/decaf-book/java-kuang-jia-fen-jie-duan-zhi-dao/pa2-yu-yi-fen-xi)
* [Scala](https://decaf-lang.gitbook.io/decaf-book/scala-kuang-jia-fen-jie-duan-zhi-dao/pa2-yu-yi-fen-xi)
* [Rust](https://decaf-lang.gitbook.io/decaf-book/rust-kuang-jia-fen-jie-duan-zhi-dao/pa2-yu-yi-fen-xi) (这个文档不一定是最新的，建议使用[这个文档](https://mashplant.gitbook.io/decaf-doc/pa2/shi-yan-nei-rong))

简单来说，语义分析需要遍历两趟 AST，第一趟生成符号信息，第二趟进行类型检查。如果在某一趟遍历中发现语义错误，直接终止后续操作，输出编译错误；否则，格式化输出作用域和符号表信息。实验框架中统一定义了一些编译错误的类(如 Java 版在 `src/main/java/decaf/driver/error/`)，报错时选择最接近的错误类，对于新特性你可能需要添加新的错误类。

你首先需要将前一阶段的工作合并到本阶段的框架上，详见下文。然后你需要充分理解基本框架中语义分析的代码结构以及功能，参考框架中对相似语言特征的语义处理过程，根据自己对语言的理解完成新增语言特性的语义分析。

相对于前两次 PA 来说，本次 PA 需要对基本框架做较多的修改，具有一定的难度，建议你**合理安排时间，尽早开始动手**。

### 合并前一阶段的工作

如果你做 PA1-A 时直接在框架整体上开发，你需要先找出最新版框架与上一版框架的差异，合并到你的代码，然后继续迭代开发(如果你是用 `git clone` 的只需 `git pull` 就行了)。

如果你做 PA1-A 时没有直接在框架整体上开发，而使用了只包含 PA1-A 的框架，你可以使用两种方法升级你的代码：

1. 找出 PA2 单独框架相对于 PA1-A 单独框架的差异，将修改应用于你的代码
2. 找出你的代码与 PA1-A 单独框架的差异，将修改应用于 PA2 单独框架

关于“找出两版本代码之间的差异，合并到另一处代码”这件事，有许多工具可以自动化完成。首先需要制作一些补丁文件，我们已经提供了“PA2 单独框架相对于 PA1-A 单独框架的差异”与“最新版框架相对于上一版框架的差异”两种补丁文件，而对于“你的代码与 PA1-A 单独框架的差异”，你可以使用 `git diff` 自行制作补丁。然后可以使用 `git apply` / `patch` 等工具自动完成打补丁操作：

```bash
cd your_project
git apply path/to/your/patches/*.patch
```

```bash
cd your_project
patch -p1 < path/to/your/patches/*.patch
```

如果打补丁过程中遇到无法自动处理的冲突，则需要手动处理。

### 新特性 1：抽象类

支持用 `abstract` 关键字定义抽象类，及修饰其中的抽象成员方法。首先主类 `Main` 不能是抽象的。由于 Decaf 不支持对成员变量和静态方法的重写 (override)，所以不允许出现抽象的成员变量和静态方法。此外，也不允许抽象方法重写基类中的非抽象方法，但可以重写同名且类型相同的抽象方法。

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
  *** Error at (1,1): 'Foo' is not abstract and does not override all abstract methods
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
  *** Error at (5,1): 'Baz' is not abstract and does not override all abstract methods
  ```

> 提示：
>
> 在实现上，可对每个类记录它所有“未被重载的抽象方法”列表。每当处理一个类时，先处理其父类，并将该列表初始化为父类的列表。之后每遇到一个抽象方法，将方法名加入该列表；每重载了一个抽象方法，从列表中删去该方法。如果最后列表非空则该类需要是抽象的。

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
  *** Error at (7,9): cannot instantiate abstract class 'Foo'
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

  对于上述程序，各变量的类型分别推导为：

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

你需要正确识别函数类型的声明，并在之后格式化输出符号表时也格式化输出其类型。

由于助教无法在函数类型的输出格式上达成一致意见，Rust版将和Java/Scala版使用不同的测例(仅仅是在测例答案中函数类型的输出上有区别，其余都是一样的)。下面是Java/Scala版的输出格式的几个例子：

| 声明类型 | 格式化输出结果 | 说明 |
|---------|-------------|-----|
| `void()`              | `() => void`              | 没有参数，返回类型 `void` |
| `int(int)`            | `int => int`              | 接受一个 `int` 类型的参数，返回 `int` |
| `string(int, int)`    | `(int, int) => string`    | 接受两个 `int` 类型的参数，返回 `string` |
| `class Main(int, int)`| `(int, int) => class Main`| 接受两个 `int` 类型的参数，返回 `Main` 的一个实例 |
| `int(int(int))`       | `(int => int) => int`     | 参数是一个 `int` 到 `int` 的函数，返回值是 `int` |
| `int(int)(int)`       | `int => int => int`       | 参数是一个 `int`，返回一个 `int` 到 `int` 的函数（注意这里的返回值也是一个函数） |

在基础框架中已经实现了各类型的格式化输出，你无需修改。

对于Rust版，格式化输出的结果与声明类型的语法完全一致(即上面的表格中第二列的内容与第一列的内容完全一致)。同样的，在基础框架中也已经实现了各类型的格式化输出，你无需修改。

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

当定义一个 Lambda 表达式时，可认为和定义函数类似，打开了一层新的参数作用域 `FormalScope`，存放各参数对应的变量符号，里面再是一层局部作用域 `LocalScope`。作用域中变量的访问规则与普通局部作用域类似：

1. 内层作用域可以访问到外层作用域的所有符号。
2. 在局部作用域中声明的符号不能与任何该声明之前的外层作用域的符号重名。

而不同之处在于：

1. 不能对捕获的外层的**非类**作用域中的符号**直接**赋值，但如果传入的是一个对象或数组的引用，可以通过该引用修改类的成员或数组元素。
2. 如果要将 Lambda 表达式赋值给一个**正在定义的**符号，则 Lambda 内部作用域中的变量既不能与该符号重名，也不能访问到该符号。

例子：

* 错例 3-2：

  ```java
  class Main {
      int v;
      static void main() {}
      void test() {
          var x = new int[10];
          var m = new Main();
          var addx = fun() {
              x[0] = x[0] + 1;        // ok
              int y = 0;
              var addy = fun() {
                  y = y + 1;          // bad
                  m.v = 1;            // ok
                  v = 1;              // ok
              };
              y = -1;                 // ok
              m = null;               // bad
          };
      }
  }
  ```

  报错：

  ```
  *** Error at (11,19): cannot assign value to captured variables in lambda expression
  *** Error at (16,15): cannot assign value to captured variables in lambda expression
  ```

> 为什么要求不能对捕获的外层的非类作用域中的符号直接赋值呢？本质上这是为了减少大家PA3的工作量。非类作用域，也就是局部/参数/(大家可能需要实现的)Lambda作用域，这里的变量都属于局部变量，在运行时它们都存在于栈上。如果需要在Lambda表达式中修改它们，必须要保存对应的内存地址，然而这个Lambda表达式可能在本函数返回之后才被调用，这时这些内存地址就不再有效了。如果一定要实现类似的效果，同时不牺牲安全性为代价的话，可以参考Scala编译器对Lambda表达式中对局部变量赋值的处理，预计工作量不算小，所以我们就直接禁止这样的赋值了。
>
> 顺便说一句，部分助教一直觉得区分局部/参数/(大家可能需要实现的)Lambda作用域这样的设计**非常冗余**，本质上它们都对应于局部变量，完全可以统一用局部作用域的概念表示。也许在未来的基础框架中，将不会有参数作用域的概念，这样上面的叙述也会稍微简洁一点。

* 错例 3-3：

  ```java
  class Main {
      static void main() {
          var f = fun (int x) {
              var g = fun (int y) => x + y;
              var h = fun (int z) {
                  var f1 = f;       // bad
                  var g1 = g;
                  var h1 = h;       // bad
              };
          };
      }
  }
  ```

  报错：

  ```
  *** Error at (6,26): undeclared variable 'f'
  *** Error at (8,26): undeclared variable 'h'
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

> 提示：
>
> 为了后续代码生成的方便，Lambda 表达式也需要有相应的符号。我们建议你对 Lambda 表达式定义一种新的作用域 `LambdaScope` (参考 `LocalScope`)，定义一种新的符号 `LambdaSymbol` (参考 `MethodSymbol`)，以便对 Lambda 表达式具有的属性进行更好的管理。当然我们不会评估你是否真的做了这件事，用自己认为方便的方法实现即可。
>
> 此外对于局部变量符号的查找需要特殊处理。当定义一个变量时，需要检查符号是否与已定义的符号重名，这在第一趟遍历对 `LocalVarDef` 节点处理时进行，此时作用域中只包含当前位置之前的符号。为了正确实现 Lambda 表达式的符号访问规则，可先在作用域中定义符号，再处理初值(可能是一个 Lambda 表达式)，使得该符号对初值表达式可见，这样就能检测出在 Lambda 表达式中定义同名符号的错误了。
>
> 当引用一个变量时，需要检查该符号是否有定义，这在第二趟遍历对 `VarSel` 节点处理时进行，此时作用域中已包含当前作用域的所有符号，查找时除了需要“位置在当前引用位置之前”这个条件外(见 `lookupBefore()` 函数)，如果当前正好位于变量定义语句中，还需要加上“不是当前正在定义的符号”这个条件。如果只是普通的变量定义或 Lambda 表达式没有嵌套，“当前正在定义的符号”只有一个；而如果像上面的例子一样 Lambda 表达式有嵌套，就可能有多个“当前正在定义的符号”，你需要正确维护这些符号，并对 `lookupBefore()` 函数进行一定的修改，以支持嵌套 Lambda 表达式。
>
> Rust版对于上述问题的解决方案也有对应的提示，可以参考Rust版的文档中的[这个部分](https://mashplant.gitbook.io/decaf-doc/pa2/kuang-jia-zhong-bu-fen-shi-xian-de-jie-shi)。

#### Lambda 表达式返回类型

你需要正确推导出 Lambda 表达式的返回类型。

记 $$<:$$ 是类型上的二元关系，它满足：

- 自反性：$$t <: t$$
- 传递性：If $$t_1 <: t_2$$ and $$t_2 <: t_3$$, then $$t_1 <: t_3$$
- 类继承：If $$c_1$$ extends $$c_2$$, then $$ClassType(c_1) <: ClassType(c_2)$$
- 函数：If $$t <: s$$ and $$s_i <: t_i$$ for every $$i$$, then $$FunType([t_1, t_2, \ldots, t_n], t) <: FunType([s_1, s_2, \ldots, s_n], s)$$

对于 `fun (t1 x1, t2 x2, ...) => y`，若推导出 `y` 的类型为 $$t$$，那么整个 Lambda 表达式的类型为 $$FunType([t_1, t_2, \ldots], t)$$。

若 `y` 是一个 `BlockStmt`，如果 `BlockStmt` 的所有执行路径都没有 `return` 语句，则 `y` 的类型是 `void`；否则 `y` 的类型是所有 `return` 语句返回值类型的最小“上界”(无返回值的 `return` 语句可认为返回 `void` 类型)。定义：称 $$t$$ 是类型 $$t_1, t_2, \ldots, t_n$$ 的上界，若 $$t_1 <: t, t_2 <: t, \ldots, t_n <: t$$。若该上界不存在，或返回类型不是 `void` 但存在一条执行路径无返回值，那么需要报错。

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

> 提示：
>
> 实现时，对每个 Lambda 表达式你需要先想办法得到它内部的所有 `return` 语句，然后可用递归的方式求它们类型的上界，下面给出一个参考算法：

> 求类型 $$[t_1, t_2, t_3,\ldots, t_n]$$ 的类型上界：
>
> 1. 选择其中一个非 `null` 的类型 $$t_k$$；
> 2. 如果 $$t_k$$ 是基本类型(`int`, `bool`, `string`, `void`)或数组，检查其他类型是否与 $$t_k$$ 完全等价，如果是返回 $$t_k$$，不是返回“类型不兼容”；
> 3. 如果 $$t_k$$ 是 `ClassType`：
>       1. 令 $$p = t_k$$，检查是否对所有 $$t_i$$ 满足 $$t_i <: p$$，如果是返回 $$p$$，不是继续下面的操作；
>       2. 令 $$p = p$$ 的父类；
>       3. 如果 $$t_k$$ 和其祖先都不是上界，返回“类型不兼容”；
> 4. 如果 $$t_k$$ 是 `FunType`，先检查其他类型是否也都是 `FunType`，且形式与 $$t_k$$ 相同，如果不是直接返回“类型不兼容”，否则：
>       1. 设 $$t_i = FunType([s_{i1}, s_{i2}, \ldots, s_{im}], r_i)$$；
>       2. 求 $$[r_1, r_2, \ldots, r_n]$$ 的类型**上界**，设其为 $$R$$；
>       3. 求 $$[s_{1i}, s_{2i}, \ldots, s_{ni}]$$ 的类型**下界**，设其为 $$T_i$$；
>       4. 返回 $$FunType([T_1, T_2, \ldots, T_m], R)$$。
>
> 类型下界的定义与求法类型上界类似，不过对 `ClassType` 的处理略有不同，请大家自己完成。

#### 函数变量

可直接将一个方法名赋值给一个局部变量，此变量就会具有函数类型，可像函数一样调用。

你需要理解并修改框架中对抽象语法树 `VarSel` 节点的处理，原来只能处理类的成员变量，你需要将其扩展为支持类的成员方法。当将方法名赋值给一个变量时，方法名的访权限与调用该方法时的情形相同，例如 `var f = A.print;` 没有访问权限错误当且仅当在同一地点调用 `A.print(...);` 时没有权限错误。此外，你也不能对一个类已有的成员方法进行赋值。

注意对于数组类型的对象，自带一个名为 `length` 的方法，也能赋值给一个变量。

下面是几个例子：

* 错例 3-7：

  ```java
  class A {
      static int sf(int x) { return x - 1; }
      int(int) vf;
  }

  class Main {
      int f(int x) { return x + 2; }

      static void main() {
          var a = new A();
          var f1 = f;             // bad
          var f2 = a.sf;          // ok
          var f3 = a.vf;          // bad
          a.sf = Main.main;       // bad

          int[] arr;
          var len = arr.length;   // ok
      }
  }
  ```

  报错：

  ```
  *** Error at (11,18): can not reference a non-static field 'f' from static method 'main'
  *** Error at (13,20): field 'vf' of 'class A' not accessible here
  *** Error at (14,14): cannot assign value to class member method 'sf'
  *** Error at (14,14): incompatible operands: int => int = () => void
  ```

  `f1` 是因为在 `class Main` 的 `static` 方法里访问了非 `static` 字段；`f3` 是因为在 `class Main` 里无法访问 `class A` 的私有字段；而 `f2` 赋值成功，具有类型 `int => int`。

  第三个错误是因为尝试给 `class A` 的成员方法 `sf` 赋值；第四个错误是因为 `a.sf` 和 `Main.main` 的类型不匹配。

  而最后的 `len` 赋值成功，类型为 `() => int`，调用它相当于调用 `arr.length()`。

> 提示：
>
> 基础框架中，对成员变量的引用是由 `VarSel` 节点处理的，而对成员方法的调用是由 `Call` 节点处理的。加入新特性后，对 `Call` 节点的语义进行了修改(详见下节)，无论是成员变量还是成员方法都交给 `VarSel` 节点处理。因此你需要参考基础框架对 `Call` 节点的处理，将其中对访问权限的检查迁移到 `VarSel` 节点上，报错的类型基本不变。

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

因此基础框架中对抽象语法树 `Call` 节点的处理需要重写。由于之前已经将 `Call` 原来的功能迁移到了 `VarSel` 节点上，对 `Call` 的处理反而变简单了。

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

1. 打印作用域名(`GLOBAL`、`CLASS`、`FORMAL`、`LOCAL`)
2. 增加缩进
3. 打印该作用域直接包含的符号位置、名称与类型。
4. 遍历所有子作用域，依次递归打印它们
5. 减少缩进

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

与函数定义类似，先是参数作用域 `FormalScope`，然后依次打印各参数，最后再是内部是局部作用域 `LocalScope`。注意每个 Lambda 表达式也是一个符号，有自己的符号名，规定 Lambda 表达式符号名的输出格式为 `lambda@(x,y)`，其中 (x, y) 为 Lambda 表达式在源码中的位置。如：

```java
static void main() {
    var f = fun() {
        var g = fun(int x) => x;
    };
}
```

对应的符号表打印为：

```
...
FORMAL SCOPE OF 'main':
    <empty>
    LOCAL SCOPE:
        (3,9) -> variable f : () => void
        (3,13) -> function lambda@(3,13) : () => void
        FORMAL SCOPE OF 'lambda@(3,13)':
            <empty>
            LOCAL SCOPE:
                (4,13) -> variable g : int => int
                (4,17) -> function lambda@(4,17) : int => int
                FORMAL SCOPE OF 'lambda@(4,17)':
                    (4,25) -> variable @x : int
                    LOCAL SCOPE:
                        <empty>
```

## 实验评分和实验报告

我们提供了若干测试程序和标准输出，你的输出需要与标准输出完全一致才行。我们还保留了一些未公开的测试例子，但考虑到报错这件事本身没有标准答案，我们未公开的测例中不会有公开测例不涵盖的报错情况，只要你没有使用非常规的特判，通过了所有公开测例后基本也能通过所有未公开测例。

实验评分分两部分：

- 评测结果：80%。这部分是机器检查，要求你的输出和标准输出**一模一样**。我们会有**未公开**的测例。
- 实验报告（根目录下 `report-PA2.pdf` 文件）：20%。要求用中文简要叙述你的工作内容。

此外，请在报告中回答以下问题(请根据你选择的实验框架回答)：

1. 实验框架中是如何实现根据符号名在作用域中查找该符号的？在符号定义和符号引用时的查找有何不同？

2. 对 AST 的两趟遍历分别做了什么事？分别确定了哪些节点的类型？

3. 在遍历 AST 时，是如何实现对不同类型的 AST 节点分发相应的处理函数的？请简要分析。
