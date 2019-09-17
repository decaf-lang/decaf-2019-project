# 《编译原理》2019年秋季学期 Decaf 课程实验新特性综述

声明：

1. 标注有“参考实现”的部分给出了一个可供参考的实现方案，不是实验要求。大家可自行决定如何实现。
2. 具体细节上的要求有待实验各阶段进一步明确。

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

报错信息均为：'Foo' is not abstract and does not override all abstract methods

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
VTABLE(_V_Foo) { // _V_ 是虚表标签的前缀
    // empty
    "Foo"
}

VTABLE(_V_Bar) {
    _V_Foo
    "Bar"
    _L_Bar_foo; // _L_ 是函数入口标签的前缀
    _L_Bar_bar;
}
```

抽象类的静态方法与普通静态方法无区别，为它们各自生成一段函数即可。

## 特性二：本地类型推导

新增 'var' 关键字，支持编译器进行本地类型推导。所谓“本地”，是指仅推断一个表达式的类型，而不是推断一个函数的类型（“全局”推导）。

### 语法

新增 'var' 关键字，并支持作为一个简单表达式出现函数体内，或出现在 for 循环的初始化语句和更新语句的位置。

```
simple ::= 原来的
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
var const = (int x) => (int y) => x; // 类型特化的 K 组合子
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
type ::= 'int' | 'bool' | 'string' | 'void' | 'class' id | type '[' ']'
       | '(' type ')' | argType '=>' type

argType ::= '(' typeList? ')' | type

typeList ::= type (',' type)*
```

这里新增操作符 '=>'。规定：'=>' 右结合，且优先级最低。
元组类型只允许出现在函数类型的参数位置，不支持单独的元组类型如 `()`，`(int, string)`。
一元组就是普通的类型，即 `(int[])` 就是 `int[]`。

例：`int => int[]` 是 `int => (int[])` 而不是 `(int => int)[]`。

#### 与实际类型的对应

类型字面量（语法元素）与实际类型（语义元素）的对应如下：

```
'int'                   --> IntType
'bool'                  --> BoolType
'string'                --> StringType
'void'                  --> VoidType
'class' className       --> ClassType(className)
elemType '[' ']'        --> ArrayType(elemType)
'(' type ')'            --> type
'(' ')' '=>' type       --> FunType([], type)
'(' t1 ',' t2 ',' ... ')' '=>' type     --> FunType([t1, t2, ...], type)
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
       | '(' varList ')' '=>' (expr | stmtBlock)

call ::= expr '(' exprList ')'
```

注意这里我们要求 Lambda 表达式的每个参数必须标明类型，但是返回的表达式不标注类型。与类型中的 '=>' 相同，它具有右结合性和最低优先级。

#### 类型推导与检查

对于 `(t1 x1, t2 x2, ...) => y`，若推导出 `y` 的类型为 `t`，那么整个 Lambda 表达式的类型为 `FunType([t1, t2, ...], t)`。

若 `y` 是一个 `BlockStmt`，则 `y` 的类型要么是 VoidType（当 BlockStmt 的所有执行路径不返回值时），要么是 BlockStmt 的所有执行路径返回值的最小“上界”。定义：称 t 是类型 t1, t2, ..., tn 的上界，若 t1 <: t, t2 <: t , ..., tn <: t。若该上界不存在，应该报错：Incompatible return types in blocked expression

类型检查规则同原有规则，如对于赋值语句和函数调用，应遵循子类型的相应规则。

#### 捕获变量与闭包生成（参考实现）

针对每种类型的 Lambda 表达式，为了支持它像对象一样可以到处传递，我们可以把它真的实例化为一个对象，并规定 apply 方法为 Lambda 表达式调用时执行的函数。例如，任何 int => int 类型的 Lambda 表达式都应该是如下类的某个子类的实例：

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
        var func = (int y) => {
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

若 Lambda 表达式函数体内对于某个本地变量进行了赋值，那么为了保证该本地变量下次读取时确实是更新了的值，我们必须捕获这个变量的引用。因此，对于 IntType 就不得不特殊处理，所以可暂时规定不能对整数赋值。但是对其他类型赋值，按照上述实现是可行的，如：

```
class Foo {
    // ...
    void foo() {
        var x = 5;          // x : int
        var a = new A();    // a : class A
        var func = (int y) => {
            a = null;
            return x + y;
        }

        func(10);
        a; // --> a == null
    }
}
```

代码执行后 `a` 应为 `null`。解开 Lambda 后：

```
class Lambda$1 extends Lambda$int$int {
    int x;
    class A a;

    int apply(int y) {
        a = null;
        return x + y;
    }
}

class Foo {
    // ...
    void foo() {
        var x = 5;          // x : int
        var a = new A();    // a : class A
        var func = new Lambda$1();
        func.x = x;
        func.a = a;

        func.apply(10); // func.a == a == null
        a; // --> a == null
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
    void foreach(int => void action) { ... }

    void print(int x) { ... }

    void test() {
        this.foreach(print);
    }
}
```

由于 `print` 的类型就是 `int => void`，可以直接传给 `foreach` 作为参数。生成代码时，直接将 `foreach` 封装为相应的闭包对象即可：

```
class IntList {
    void foreach(int => void action) { ... }

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
