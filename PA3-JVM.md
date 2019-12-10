# PA3-JVM

JVM 字节码生成

**注意：本阶段仅 Scala 版必做。**

## 任务概述

本阶段的任务与 PA3 相同，只不过采用的中间表示是 JVM 字节码。

本阶段的测试方法与 PA3 一致：测试脚本会自动调用 `java` 运行你的编译器生成的 JVM class 文件，检查其输出与标准输出是否完全一致。
与 PA3 所采用的 TAC 相比，JVM 拥有十分丰富的指令集，且具备完善的运行时错误捕获机制。
PA3 中要求的除零错误无需你来实现，JVM 的运行时能自动捕获并抛出异常。
因此，在这一阶段中，你的主要任务是<del>学习 JVM</del>完成新特性的代码生成。

## 实验内容

在正式开始实验之前，请先阅读实验指导书关于 [JVM 字节码简介](https://decaf-lang.gitbook.io/decaf-book/scala-kuang-jia-fen-jie-duan-zhi-dao/pa3jvmjvm-zi-jie-ma-sheng-cheng/jvm) 的部分，了解 JVM 最基本的知识。
如有必要，请查阅 [官方文档](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)。
请注意：本次实验生成的字节码的 major version 是 52（即 Java 8），这已经在框架中实现好了，**请勿修改！**

### 新特性 1：抽象类

支持用 `abstract` 关键字定义抽象类，及修饰其中的抽象成员方法。

提示：在生成抽象类对应的 JVM class 文件时，使用修饰符 `ACC_ABSTRACT` 修饰该类以及其中的抽象成员。

### 新特性 2：局部类型推导

支持编译器进行局部变量的类型推导。
由于类型已经在 PA2 阶段推导完成并标注在语法树上，故变量定义的逻辑与框架中 `LocalVarDef` 完全相同，不需要进行额外处理。

### 新特性 3：First-class Functions

PA3 的文档中给出了一种基于函数“闭包”（由函数指针和捕获变量构成）的机制来实现。
但是，由于 JVM 不支持直接操作内存，这里给出一种面向对象风格的实现方法。

> 此部分仅供参考，你也可以查阅资料选择其他你认为更好的方法来实现。

在 JVM 中，虽然函数不是一等公民，但是对象总是一等公民。因此，一个简单的想法就是把所有的函数（尤其是 lambda 表达式）都看作一个对象。
与其他对象不同的是，这类对象都有一个叫做 `apply` 的成员方法，该方法完成对此函数的调用。
如果一个 lambda 表达式具有捕获变量，那么把这些捕获变量全都作为这个对象的成员就可以了。例如：

```java
class A {
    int field;

    int(int) getf(int local) {
        return fun(int x) {
            return field + local + x;
        };
    }

    int callf() {
        return getf(10)(20);
    }
}
```

可以翻译为如下等价的 Java 代码：

```java
import java.util.function.Function;

public class Lam$1 implements Function<Integer, Integer> {
    A self; // captured: object of class A
    int local; // captured local variable

    @Override
    public Integer apply(Integer x) {
        return self.field + local + x;
    }
}

public class A {
    int field;

    public Function<Integer, Integer> getf(int local) {
        Lam$1 funcObj = new Lam$1();
        funcObj.self = this;
        funcObj.local = local;
        return funcObj;
    }

    public int callf() {
        return getf(10).apply(20);
    }
}
```

可以看出，我们为 `fun(int x) { return field + local + x; }` 这个 lambda 表达式专门生成 (synthesize) 了一个新的类 `Lam$1`，并且把 lambda 表达式的函数体实现在 `apply` 方法里面。
这个类继承了 Java 的“函数式”接口 (Functional Interface) `Function<T, R>`，它表示一个输入 `T` 返回 `R` 的函数对象。
特别地，捕获变量 `self` 和 `local` 都成为该类的成员。
而在 `getf` 方法中，我们要做的事情就是，new 出来这个 `Lam$1` 的实例（对象），初始化其中的捕获变量成员，然后返回该实例 `funcObj`。
最后，在 `callf` 方法中，我们通过调用函数对象的 `apply` 方法来迂回地实现对 lambda 表达式的调用。

上述翻译策略总结如下：

1. 为每个 lambda 表达式生成一个类，继承合适的 Functional Interface，在类中声明其捕获变量作为成员变量，并在 `apply` 方法中真正地实现该 lambda 表达式的函数体；
2. 在每个 lambda 表达式的定义处，new 一个它对应的类（第1步生成的那个）的实例，注意初始化其中的捕获变量；
3. 在每个 lambda 表达式的调用处，调用那个实例（第2步生成的那个）的 `apply` 方法，正常传入用户给的参数。

插入一段题外话：熟悉匿名类的同学会发现，上面这一段 Java 代码可以简写为：

```java
public class A {
    int field;

    public Function<Integer, Integer> getf(int local) {
        return new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer x) {
                return field + local + x;
            }
        };
    }

    public int callf() {
        return getf(10).apply(20);
    }
}
```

但是，这样的实现并不能简化 JVM 字节码生成的过程！
如果你把上述代码交给 Java 编译器 `javac` 处理，你会发现它除了生成 `A.class` 外，还会生成 `A$1.class`。
这个 `A$1.class` 的功能与 `Lam$1` 是一样的。

另外一种需要考虑的情况是，方法名直接当做函数使用。如：

```java
class A {
    void() getf() { return f; }
    void f() { Print("A"); }
}

class B extends A {
    void f() { Print("B") }
}

class Main {
    static void main() {
        class A b = new B();
        b.getf()();
    }
}
```

我们需要为该方法（例子中是 `f`）自动生成一个函数对象，它的 `apply` 方法很简单——真正调用那个方法（`f`）。如下面的 Java 代码所示：

```java
public interface Action$ { public void apply(); } // function of type () => void

public class Action$1 implements Action$ {
    A self;

    @Override
    public void apply() {
        self.f();
    }
}

public class A {
    public Action$ getf() {
        Action$1 funcObj = new Action$1();
        funcObj.self = this;
        return funcObj;
    }

    public void f() {
        System.out.print("A");
    }
}

public class B extends A {
    public void f() {
        System.out.print("B");
    }
}

public class Main {
    public static void main(String[] args) {
        A b = new B();
        b.getf().apply();
    }
}
```

这里的 `Action$1` 类的对象 `funcObj` 就是对 `f` 方法的一层包装，它的 `apply` 方法体就是单纯地调用一下 `f` 而已。

除了上述方法外，JVM 从 7 开始（我们用的8）支持 `INVOKEDYNAMIC` 以更加高效地实现 lambda 调用。
感兴趣的同学可以参考 [这篇文章](https://www.infoq.com/articles/Java-8-Lambdas-A-Peek-Under-the-Hood/) 进行实现。

## 提示

1. 使用 `javap -v class文件名` 反编译字节码，输出可读的 JVM 指令序列，以检查它是否与预期一致。
2. 当 JVM 报出你从未见过的错误时，请尝试用搜索引擎查找错误信息，并理解出错的原因，以便后续的调试。
3. [JVM 文档](https://docs.oracle.com/javase/specs/jvms/se8/html/) 和 [Java Functional Interface 文档](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html) 是你的好朋友。由于 Java Functional Interface 似乎不支持3个参数以上的函数，你可以考虑像 Scala 那样搞一堆 `Function3, Function4, ...` 以适应不同参数个数的函数。
4. 学习 Scala 编译器的实现：你可以将测例翻译成 Scala 代码，然后查看 Scala 编译器生成的字节码，这可以启发你找到科学的实现方案（如果你打算采用 Scala 编译器的那一套的话）。

## 公开测例

本阶段的公开测例与 PA3 相同，即 [S3/](https://github.com/decaf-lang/decaf-2019-TestCases/tree/master/S3) 下的测例。
由于这一阶段不考查对运行时错误的处理，测试脚本已忽略 `test_divisionbyzero1.decaf` 和 `test_divisionbyzero2.decaf` 这两个例子。
隐藏测试集里面的**所有**测例都**不会有**运行时错误。

## 实验评分和实验报告

我们提供了若干测试程序和标准输出，你的输出需要与标准输出完全一致才行。我们会有**未公开**的测例。
但是，这一阶段的全部测例与 PA3 相同（刨开有运行时错误的例子），因此你的实现只要达到 PA3 的要求即可——
对于 PA3 明文规定不考察的那些特性，你**无需**实现。

实验评分分两部分：

- 评测结果：80%。这部分是机器检查，要求你的输出和标准输出**一模一样**。
- 实验报告（根目录下 `report-PA3-JVM.pdf` 文件）：20%。要求用中文或英文简要叙述你的工作内容，**并回答**以下两个问题：

Q1. 请举出**至少两处**本阶段与 PA3 阶段在实现机制上的不同，包括你实现的部分和框架中已经完成的部分。

Q2. 框架中是如何将类似于 `x >= y` 这种比较运算表达式翻译成语义等价的 JVM 指令的？
