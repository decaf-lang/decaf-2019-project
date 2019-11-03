# 《编译原理》2019年秋季学期 Decaf 课程实验新特性综述

声明：

1. 标注有“参考实现”的部分给出了一个可供参考的实现方案，不是实验要求。大家可自行决定如何实现。
2. 具体细节上的要求有待实验各阶段进一步明确。**本文档仅供参考**

## 特性一：抽象类

支持用 abstract 关键字定义抽象类，及修饰其中的抽象成员方法。由于 Decaf 不支持对成员变量和静态方法的重写 (override)，所以不允许出现抽象的成员变量和静态方法。

### 文法

新增关键字 'abstract'。

```
classDef    ::= 'abstract'? 'class' id 'extends' id '{' field* '}'
field       ::= varDef | methodDef
methodDef   ::= 'static'? type id '(' varList ')' stmtBlock
              | 'abstract' type id '(' varList ')' ';'
```

注意：抽象方法不能有函数体；语法上已经限制了成员变量和静态方法不能是抽象的。

### 类型检查

**错误1：**若一个类中含有抽象成员，或者它继承了一个抽象类但没有重写所有的抽象方法，那么该类必须声明为抽象类。

错例1-1：
```
class Foo {
    abstract int foo();

    abstract string bar();
}
```

错例1-2：
```
abstract class Foo {
    abstract int foo();
}

class Baz extends Foo {
    int baz() { return 1; }
}
```

报错信息分别为：'Foo' is not abstract and does not override all abstract methods 和 'Baz' is not abstract and does not override all abstract methods

**错误2：**抽象类不能使用 'new' 进行实例化。

错例2：
```
abstract class Foo {
    abstract int foo();
}

class Main {
    static void main() {
        new Foo();
    }
}
```

报错信息：Cannot instantiate abstract class 'Foo'

**注意：**抽象类中可以定义任何普通的成员（成员变量、成员方法、静态方法），如：

```
abstract class Foo {
    string s;

    int foo() { return 1; }

    static int bar() { return 1; }
}
```

抽象类中的静态方法可以像普通的静态方法一样调用，即：

```
Foo.bar();
```

或者

```
class Baz extends Foo { }
(new Baz()).bar();
```

### 代码生成（参考实现）

由于抽象类仍然是一个类型，所以在运行时仍有可能会进行 instanceof 的判断，故对抽象类生成一个虚表也是有必要的，该虚表至少应该还有两项元信息：即父类的虚表指针（若存在）和该类的名称。

但是，由于抽象类不可实例化，因此抽象类中存在的所有成员方法都是不可能会被直接访问到的，因此抽象类的虚表无需记录任何成员方法的入口地址。这些成员方法的信息只存在于它的非抽象子类中。如：

```
abstract class Foo {
    int foo() { return 1; }
}

class Bar extends Foo {
    int bar() { return 1; }
}
```

对应的虚表为

```
VTABLE<Foo> {
    NULL
    "Foo"
}

VTABLE<Bar> {
    NULL
    "Bar"
    FUNC<Bar.foo>
    FUNC<Bar.bar>
}
```

但是你需要自己保证在一个继承链上，同样的方法名在虚表中的偏移量一致。

抽象类的静态方法与普通静态方法无区别，为它们各自生成一段函数即可。

## 特性二：本地类型推导

新增 'var' 关键字，支持编译器进行本地类型推导。所谓“本地”，是指仅推断一个表达式的类型，而不是推断一个函数的类型（“全局”推导）。

### 语法

新增 'var' 关键字，并支持作为一个简单表达式出现函数体内，或出现在 for 循环的初始化语句和更新语句的位置。

```
simpleStmt ::= 原来的
             | 'var' id '=' expr
```

### 类型推导

针对如下例子：

```
var x = 1;
var s = "123";
```

应该推断出 `x` 的类型是整数，`s` 的类型是字符串。

此外，在实现了特性三的基础上，用 'var' 关键字可以方便的定义 Lambda 表达式，如：

```
var const = fun (int x) => fun (int y) => x; // 类型特化的 K 组合子
```

本地类型推导要求推断出表达式最精确的类型，如：

```
class A {}
class B extends A {}

var o = new B(); // o : class B
```

这里 `o` 的类型应该是 `class B` 而不是 `class A`。有关函数类型的推断请见特性三。

## 特性三：First-class functions

支持函数式语言中常见的 first-class functions，要求支持 lambda 表达式、将方法名直接当做函数使用、正确构造闭包环境等。

### 函数类型

#### 文法

允许用户在程序中标注函数类型：

```
type ::= 原来的
       | type '(' typeList ')'

typeList ::= type (',' type)* | ε
```

这里我们采用类似于 C 语言函数指针和 JVM 类型描述符的语法来标注函数类型，如 `int(class A, string)` 表示该函数返回值类型是 `int`，它接受两个参数，类型分别是 `class A` 和 `string`。

#### 与实际类型的对应

类型字面量（语法元素）与实际类型（语义元素）的对应如下：

```
'int'                   --> IntType
'bool'                  --> BoolType
'string'                --> StringType
'void'                  --> VoidType
'class' className       --> ClassType(className)
elemType '[' ']'        --> ArrayType(elemType)
type '(' t1 ',' t2 ',' ... ')'  --> FunType([t1, t2, ...], type)
```

特别地，VoidType 只能作为函数的返回类型（包括lambda)，不能作为任何变量的类型或者函数参数的类型。

#### 子类型 (Subtyping) 检查

记 <: 是类型上的二元关系，它满足：

- 自反性：t <: t
- 传递性：If t1 <: t2 and t2 <: t3, then t1 <: t3
- 类继承：If c1 extends c2, then ClassType(c1) <: ClassType(c2)
- 函数：If t <: s and si <: ti for every i, then FunType([t1, t2, ..., tn], t) <: FunType([s1, s2, ..., sn], s)

### Lambda 表达式

#### 文法

新增 Lambda 表达式并扩展 call 的文法：

```
expr ::= 原来的
       | 'fun' '(' varList ')' ('=>' expr | stmtBlock)

call ::= expr '(' exprList ')'
```

新增关键字 'fun' 和操作符 '=>'。操作符 '=>' 具有最低优先级。

注意这里我们要求 Lambda 表达式的每个参数必须标明类型，但是返回的表达式不标注类型。

#### 类型推导与检查

对于 `fun (t1 x1, t2 x2, ...) => y`，若推导出 `y` 的类型为 `t`，那么整个 Lambda 表达式的类型为 `FunType([t1, t2, ...], t)`。

若 `y` 是一个 `BlockStmt`，则 `y` 的类型要么是 VoidType（当 BlockStmt 的所有执行路径不返回值时），要么是 BlockStmt 的所有执行路径返回值的最小“上界”。定义：称 t 是类型 t1, t2, ..., tn 的上界，若 t1 <: t, t2 <: t , ..., tn <: t。若该上界不存在，应该报错：Incompatible return types in blocked expression

类型检查规则同原有规则，如对于赋值语句和函数调用，应遵循子类型的相应规则。

#### 捕获变量与闭包生成（参考实现）

针对每种类型的 Lambda 表达式，为了支持它像对象一样可以到处传递，我们可以把它真的实例化为一个对象，并规定 apply 方法为 Lambda 表达式调用时执行的函数。例如，任何 `int(int)` 类型的 Lambda 表达式都应该是如下类的某个子类的实例：

```
abstract class Lambda$int$int {
    abstract int apply(int arg1);
}
```

这样在调用该 Lambda 表达式 `e` 时，就相当于调用其 `apply` 方法：`e(x) === e.apply(x)`

对于 Lambda 表达式中引用的除参数外的其他变量，需要捕获其在当前作用域的值（IntType）或者引用（其他类型）。如：

```
class Foo {
    // ...
    void foo() {
        var x = 5;          // x : int
        var a = new A();    // a : class A
        var func = fun (int y) {
            a.doSomething();
            this.anotherMethod();
            return x + y;
        }
    }
}
```

`func` 需要捕获当前作用域中 `x` 的值， `a` 的引用以及 `this` 的引用。并将捕获的变量包装在闭包对应的类中：

```
class Lambda$1 extends Lambda$int$int {
    int x;
    class A a;
    class Foo $this; // 避免命名冲突

    int apply(int y) {
        a.doSomething();
        $this.anotherMethod();
        return x + y;
    }
}
```

那么 `func` 对应的对象为：

```
var func = new Lambda$1();
func.x = x; // 拷贝作用域中存在的那个 `x` 的值
func.a = a; // 拷贝作用域中存在的那个 `a` 的引用
```

由于这段代码是自动生成的 (synthetic)，我们可以突破成员变量 protected 的束缚。

若 Lambda 表达式函数体内对于某个本地变量进行了赋值，那么为了保证该本地变量下次读取时确实是更新了的值，我们必须捕获这个变量的引用。简单起见，我们规定不能对一个变量直接赋值(无论是传值还是引用)，但如果传入 Lambda 表达式的是一个对象或数组的引用，则可以通过该引用修改类的成员或数组元素。

```
class Foo {
    // ...
    int v;
    void foo() {
        var x = 5;          // x : int
        var a = new Foo();  // a : class Foo
        var func = fun (int y) {
            a.v = y;
            return x + y;
        }

        func(10);
        a.v; // --> a.v == 10
    }
}
```

代码执行后 `a.v` 应为 `10`。解开 Lambda 后：

```
class Lambda$1 extends Lambda$int$int {
    int x;
    class Foo a;

    int apply(int y) {
        a.v = y;
        return x + y;
    }
}

class Foo {
    // ...
    int v;
    void foo() {
        var x = 5;          // x : int
        var a = new Foo();  // a : class A
        var func = new Lambda$1();
        func.x = x;
        func.a = a;

        func.apply(10); // func.a.v == a.v == 10
        a.v; // --> a.v == 10
    }
}
```

提示：

1. 为了避免为每一种类型的闭包都生成一个抽象类，可以考虑设计类似于 java.lang.Object 这样的超类，这样只用为不同参数个数的闭包生成单独的类。（通过类型擦除实现泛型）
2. 除了用类来包装一个函数闭包外，采用函数指针的实现也是可行的

### 将方法名直接当做函数使用

需要返回函数或者接受函数作为参数时，可直接使用类型匹配的函数名，来替代实际的 Lambda 表达式，如：

```
class IntList {
    void foreach(void(int) action) { ... }

    void print(int x) { ... }

    void test() {
        this.foreach(print);
    }
}
```

由于 `print` 的类型就是 `void(int)`，可以直接传给 `foreach` 作为参数。生成代码时，直接将 `foreach` 封装为相应的闭包对象即可：

```
class IntList {
    void foreach(void(int) action) { ... }

    void print(int x) { ... }

    void test() {
        var print$closure = new IntList$print$closure();
        print$closure.$this = this;
        this.foreach(print$closure);
    }
}

class IntList$print$closure extends Lambda$int$void {
    class IntList $this;

    void apply(int x) {
        ... // copy from print
    }
}
```
