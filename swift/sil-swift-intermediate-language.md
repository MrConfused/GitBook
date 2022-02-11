---
description: Swift 在编译过程中的中间产物，通过 SIL 可以了解swift 底层的实现细节
---

# SIL(Swift Intermediate Language )

由于swift编译器是基于LLVM实现，所以我们先了解一下LLVM。

## LLVM

LLVM是一套用来开发编译器的架构。

LLVM编译架构的三段式设计（前端、中间优化器、后端）：

![LLVM三段式](../.gitbook/assets/image.png)

* 前端：**解析源代码**。通过词法分析、语法分析、语义分析，检查代码中是否存在错误，然后构建**抽象语法树**（Abstract Syntax Tree ， **AST**），然后生成**中间代码（** Intermediate Representation，**IR**，IR的表现形式有三种：text、memory、bitcode**）。**
* 中间优化器：负责各种优化，**缩小包的体积**（剥离符号 **）** 、**改善代码的运行时间（** 消除冗余计算、减少指针跳转次数等）。
* 后端：代码生成器。将IR映射到目标指令集，生成**机器语言**，并且进行机器相关的代码优化。

LLVM使用的是统一的中间代码IR，如果需要支持一种新的语言，只要实现一个新的前端；如果要支持一种新的设备，只要实现一个新的后端。

LLVM把编译器的前端和后端解耦，使得他的可扩展性非常强。

也让LLVM成为了实现编程语言时的通用基础架构：包括GCC家族、Java、.NET、Python、Ruby、Scheme、Haskell、D等。

## Clang

Clang是基于LLVM架构的C/C++/Objective-C编程语言的编译器前端。

![Clang编译过程](<../.gitbook/assets/image (3).png>)

![Clang的编译过程](<../.gitbook/assets/image (2).png>)

可以看到Clang编译过程中有些问题：

* 源代码和 LLVM IR 之间有一个巨大的抽象层，但是这个抽象层在 clang 里面实现的并不完美：
  * 语法分析和语义分析之间很乱
  * AST‘ 转换成 IR 的过程也是相当的曲折
* 由于不同的编程语言生成的统一的IR，导致IR 不合适源码级别分析
* clang 的静态分析实现（CFG，Control Flow Graph）不够精确，和 IR 之间有很多重复的逻辑。



## swift编译器

![swift编译过程](<../.gitbook/assets/image (5).png>)

![swift的前端编译过程](<../.gitbook/assets/image (1).png>)

与clang相比，swift的前端编译流程中，在AST和IR中间多了一层SIL，以弥补clang的缺陷，原本在 clang 中独立的静态分析和 IR 生成阶段整合了起来**。**

可以看到Clang在实现的时候，很多阶段交织在一起，耦合严重，而swift加入了sil中间层，解决了这个问题。

## sil

### 特点

*   Fully represents program semantics&#x20;

    能够完整的表达程序的语义。
*   Designed for both code generation and analysis&#x20;

    被设计用于代码生成和静态分析
*   Sits on the hot path of the compiler pipeline&#x20;

    处于编译的流水线中，而不是独立于之外
*   Bridges the abstraction gap between source and LLVM

    在源代码和 LLVM 之间架起了抽象的桥梁

### 语法

[官方文档](https://github.com/apple/swift/blob/main/docs/SIL.rst)

* `%0`、`%1` 等：寄存器，相当于块中的局部变量
* `bb0`、`bb1` 等：代码块，由指令组成，结束时从函数返回或者跳转其他代码块
* `$String`：String 类型
* `$*String`：String 类型的值地址
* `sil_global`：全局变量
* `apply`：调用函数，并传入参数
* `function_ref`：直接函数调用
* `class_method`：通过函数表来查找实现调用
* `sil_vtable`：类的函数表
* `thin`：静态的
* `thick`：动态的，运行时的
* `cond_br`：类似于三目运算符，判断寄存器上值进行代码块跳转

#### 空文件生成的sil：

{% code title="Model.swift" %}
```
```
{% endcode %}

{% code title="Model.sil" %}
```
sil_stage canonical

import Builtin
import Swift
import SwiftShims

// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
  %2 = integer_literal $Builtin.Int32, 0          // user: %3
  %3 = struct $Int32 (%2 : $Builtin.Int32)        // user: %4
  return %3 : $Int32                              // id: %4
} // end sil function 'main'



// Mappings from '#fileID' to '#filePath':
//   'Model/Model.swift' => 'Model.swift'
```
{% endcode %}

* sil文件对第一行表示sil阶段是canonical。sil有两个阶段：raw和canonical
  * raw是第一个阶段，对AST进行基础的担保优化和语法诊断，生成原始的sil
  * canonical是第二阶段，进行特定优化，生成正式的sil
  * 之后的编译会由IRGen把sil生成IR，然后经过编译器后端，生成Mach-O文件
* 接下来是头部声明。sil文件中没有隐式import，如果使用swift或者[Buildin](http://ankit.im/swift/2016/01/12/swift-mysterious-builtin-module/)标准组件的话必须明确的引入。
* 接着是sil的main函数（程序入口）。
  * `@main` 表示方法的名字是main，SIL中所有标识符均以@符号开头。
  * `@convention(c)` 表示使用C函数的方式进行调用
  * `bb0`=basic block zero，表示一个代码块，SIL中没有分支语句，只有入口和出口；
    * `%0`：传入basic block的第一个参数是int32类型，%符号表示一个虚拟寄存器，后续进行降级操作时，才会把虚拟寄存器转换成对应体系结构的真实寄存器。
    * `%1`：第二个参数。
    * `%2`：创建一个类型为Builtin.Int32的值，赋值给%2
      * `// user: %3`：表示赋值语句的结果，会被 id 是 3 的表达式用到。
      * 赋值语句都会注释使用者是谁。
    * %3：创建一个类型为Int32的值，赋值给%3
    * return：返回%3，类型位Int32
      * `// id: %4`：表示该语句对id为4
      * 非赋值语句都会标注 id 是什么。

### 作用

* 看编译器不同级别的优化（比如无用代码裁剪，不同写法对性能影响）
* 了解函数派发方式（`final`/`static`/`dynamic` 等）对函数的影响，KVO
* 了解swift语言的底层实现
* 等等

### Builtin <a href="#builtin" id="builtin"></a>

Builtin 将 LLVM IR 的类型和方法直接暴露给 Swift 标准库，使用时没有额外的运行时负担。

### 源码生成sil文件的命令：

```bash
// 将 main.swift 编译成 SIL 代码
swiftc -emit-sil main.swift 

// 将 main.swift 编译成 SIL，并保存到 main.sil 文件中
swiftc -emit-sil main.swift >> main.sil

// 将 main.swift 编译成 SIL的同时， 将命名重整后的符号恢复原样，并保存到 main.sil 文件中
swiftc -emit-sil main.swift | xcrun swift-demangle > main.sil
```



## 通过SIL看swift中的函数派发方式



### 看一个示例



{% code title="Model.swift" %}
```
protocol Drawing {
    func render()
}

extension Drawing {
    func circle() { print("protocol") }
    func render() { circle() }
}

class SVG: Drawing {
    func circle() { print("class") }
}

SVG().render() // protocol
```
{% endcode %}

{% code title="Model.sil" %}
```
// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
  %2 = metatype $@thick SVG.Type                  // user: %4
  // function_ref SVG.__allocating_init()
  %3 = function_ref @Model.SVG.__allocating_init() -> Model.SVG : $@convention(method) (@thick SVG.Type) -> @owned SVG // user: %4
  %4 = apply %3(%2) : $@convention(method) (@thick SVG.Type) -> @owned SVG // user: %6
  %5 = alloc_stack $SVG                           // users: %6, %10, %9, %8
  store %4 to %5 : $*SVG                          // id: %6
  // function_ref Drawing.render()
  %7 = function_ref @(extension in Model):Model.Drawing.render() -> () : $@convention(method) <τ_0_0 where τ_0_0 : Drawing> (@in_guaranteed τ_0_0) -> () // user: %8
  %8 = apply %7<SVG>(%5) : $@convention(method) <τ_0_0 where τ_0_0 : Drawing> (@in_guaranteed τ_0_0) -> ()
  destroy_addr %5 : $*SVG                         // id: %9
  dealloc_stack %5 : $*SVG                        // id: %10
  %11 = integer_literal $Builtin.Int32, 0         // user: %12
  %12 = struct $Int32 (%11 : $Builtin.Int32)      // user: %13
  return %12 : $Int32                             // id: %13
} // end sil function 'main'

...

// Drawing.render()
sil hidden @(extension in Model):Model.Drawing.render() -> () : $@convention(method) <Self where Self : Drawing> (@in_guaranteed Self) -> () {
// %0 "self"                                      // users: %3, %1
bb0(%0 : $*Self):
  debug_value_addr %0 : $*Self, let, name "self", argno 1 // id: %1
  // function_ref Drawing.circle()
  %2 = function_ref @(extension in Model):Model.Drawing.circle() -> () : $@convention(method) <τ_0_0 where τ_0_0 : Drawing> (@in_guaranteed τ_0_0) -> () // user: %3
  %3 = apply %2<Self>(%0) : $@convention(method) <τ_0_0 where τ_0_0 : Drawing> (@in_guaranteed τ_0_0) -> ()
  %4 = tuple ()                                   // user: %5
  return %4 : $()                                 // id: %5
} // end sil function '(extension in Model):Model.Drawing.render() -> ()'
```
{% endcode %}

可以看到11行SVG初始化之后，调用的是Drawing.render()，并且28行Drawing.render()内部也是直接调用Drawing.circle()，SVG的circle()并没有覆盖协议扩展中的circle()。

原因是extension中声明的函数是静态派发，编译的时候就已经确定了调用地址，类无法重写实现。

### swift的派发机制

swift中有三种函数派发方式：直接派发（静态派发）、函数表派发（动态派发），由于swift用的是OC的运行时机制，所以还有消息派发方式（动态派发）。

#### 直接派发

CPU直接拿到函数地址，进行调用。

* 优点：
  * 使用的指令集最少，效率最高。
  * 编译器优化也会对函数进行内联，省去函数调用的开销，提升执行速度。
* 缺点：没有动态性，不支持继承。

#### 函数表派发

函数表中存储了函数的指针，调用函数时通过函数表找到函数指针进行调用。

* 优点：
  * 查表的实现简单、性能可预知
  * 保证了动态性的同时也兼顾了执行效率
* 缺点：
  * 比静态派发多了两次读（读函数表、读函数指针）和一次跳转（函数跳转）
  * 编译器对有副作用的函数无法优化

#### 消息派发

就是OC中的消息传递，是最动态的派发方式。

* 优点：
  * 动态性最高
  * Method Swizzling
  * isa Swizzling
* 缺点：
  * 效率最低（但是缓存机制会让之后发送相同的消息时，执行速度会很快）

### 影响swift派发机制的因素

#### 1. 数据类型及函数生命的位置

| 类型         | 初始声明                                 | 扩展                                   |
| ---------- | ------------------------------------ | ------------------------------------ |
| 值类型        | <mark style="color:red;">静态派发</mark> | <mark style="color:red;">静态派发</mark> |
| 协议         | 函数表派发                                | <mark style="color:red;">静态派发</mark> |
| 类          | 函数表派发                                | <mark style="color:red;">静态派发</mark> |
| NSObject子类 | 函数表派发                                | <mark style="color:red;">静态派发</mark> |



{% code title="Model.swift" %}
```
protocol MyProtocol {
    func protocolFunc()
}
extension MyProtocol {
    func protocolFuncInExtension() {}
}

class MyClass: MyProtocol {
    func classFunc() {}
    func protocolFunc() {}
}
extension MyClass {
    func classFuncInExtension() {}
}

struct MyStruct {
    func structFunc() {}
}
extension MyStruct {
    func structFuncInExtension() {}
}

class MyNSObjectSubClass: NSObject {
    func nsObjectSubClass() {}
}
extension MyNSObjectSubClass {
    func nsObjectSubClassInExtension() {}
}

MyClass().classFunc()
MyClass().classFuncInExtension()
MyStruct().structFunc()
MyStruct().structFuncInExtension()
MyClass().protocolFunc()
MyClass().protocolFuncInExtension()
MyNSObjectSubClass().nsObjectSubClass()
MyNSObjectSubClass().nsObjectSubClassInExtension()
```
{% endcode %}

{% code title="Model.sil" %}
```
// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
...
  %5 = class_method %4 : $MyClass, #MyClass.classFunc : (MyClass) -> () -> (), $@convention(method) (@guaranteed MyClass) -> () // user: %6
...
  // function_ref MyClass.classFuncInExtension()
  %11 = function_ref @Model.MyClass.classFuncInExtension() -> () : $@convention(method) (@guaranteed MyClass) -> () // user: %12
...
  // function_ref MyStruct.structFunc()
  %17 = function_ref @Model.MyStruct.structFunc() -> () : $@convention(method) (MyStruct) -> () // user: %18
...
  // function_ref MyStruct.structFuncInExtension()
  %22 = function_ref @Model.MyStruct.structFuncInExtension() -> () : $@convention(method) (MyStruct) -> () // user: %23
...
  %27 = class_method %26 : $MyClass, #MyClass.protocolFunc : (MyClass) -> () -> (), $@convention(method) (@guaranteed MyClass) -> () // user: %28
...
  // function_ref MyProtocol.protocolFuncInExtension()
  %35 = function_ref @(extension in Model):Model.MyProtocol.protocolFuncInExtension() -> () : $@convention(method) <τ_0_0 where τ_0_0 : MyProtocol> (@in_guaranteed τ_0_0) -> () // user: %36
...
  %42 = class_method %41 : $MyNSObjectSubClass, #MyNSObjectSubClass.nsObjectSubClass : (MyNSObjectSubClass) -> () -> (), $@convention(method) (@guaranteed MyNSObjectSubClass) -> () // user: %43
...
  // function_ref MyNSObjectSubClass.nsObjectSubClassInExtension()
  %48 = function_ref @Model.MyNSObjectSubClass.nsObjectSubClassInExtension() -> () : $@convention(method) (@guaranteed MyNSObjectSubClass) -> () // user: %49
...
} // end sil function 'main'

sil_vtable MyClass {
  #MyClass.classFunc: (MyClass) -> () -> () : @Model.MyClass.classFunc() -> ()	// MyClass.classFunc()
  #MyClass.protocolFunc: (MyClass) -> () -> () : @Model.MyClass.protocolFunc() -> ()	// MyClass.protocolFunc()
  #MyClass.init!allocator: (MyClass.Type) -> () -> MyClass : @Model.MyClass.__allocating_init() -> Model.MyClass	// MyClass.__allocating_init()
  #MyClass.deinit!deallocator: @Model.MyClass.__deallocating_deinit	// MyClass.__deallocating_deinit
}

sil_vtable MyNSObjectSubClass {
  #MyNSObjectSubClass.nsObjectSubClass: (MyNSObjectSubClass) -> () -> () : @Model.MyNSObjectSubClass.nsObjectSubClass() -> ()	// MyNSObjectSubClass.nsObjectSubClass()
  #MyNSObjectSubClass.deinit!deallocator: @Model.MyNSObjectSubClass.__deallocating_deinit	// MyNSObjectSubClass.__deallocating_deinit
}

sil_witness_table hidden MyClass: MyProtocol module Model {
  method #MyProtocol.protocolFunc: <Self where Self : MyProtocol> (Self) -> () -> () : @protocol witness for Model.MyProtocol.protocolFunc() -> () in conformance Model.MyClass : Model.MyProtocol in Model	// protocol witness for MyProtocol.protocolFunc() in conformance MyClass
}
```
{% endcode %}

可以看到：

* 在调用时只有MyClass.classFunc()、MyClass.protocolFunc()和MyNSObjectSubClass.nsObjectSubClass()是用class\_method调用，且在vtable中，是函数表派发
* 其他都是用function\_ref调用，是直接派发



#### 2. 指定派发方式

* **final**
* **dynamic**
* **@objc**
* **@Objc + dynamic**
* **@inline**

{% code title="Model.swift" %}
```
```
{% endcode %}

{% code title="Model.sil" %}
```
```
{% endcode %}



#### 3. 编译器优化



{% code title="Model.swift" %}
```
```
{% endcode %}

{% code title="Model.sil" %}
```
```
{% endcode %}

### swift函数派发方式总结



| **NSObject**   | @nonobjc 或者 final 修饰的方法                                                | 声明作用域中方法    | 扩展方法及被 dynamic 修饰的方法           |
| -------------- | ---------------------------------------------------------------------- | ----------- | ------------------------------ |
| **Class**      | 不被 @objc 修饰的扩展方法及被 final 修饰的方法                                         | 声明作用域中方法    | dynamic 修饰的方法或者被 @objc 修饰的扩展方法 |
| **Protocol**   | 扩展方法                                                                   | 声明作用域中方法    | @objc 修饰的方法或者被 objc 修饰的协议中所有方法 |
| **Value Type** | 所有方法                                                                   | 无           | 无                              |
| 其他             | 全局方法，staic 修饰的方法；使用 final 声明的类里面的所有方法；使用 private 声明的方法和属性会隐式 final 声明； | <p><br></p> | <p><br></p>                    |





我们知道静态派发要比动态派发效率高。

那么怎么把动态派发消解为静态派发呢？

* struct默认使用静态派发，class默认动态派发。
* 对于class：
  * final会约束类无法被继承，动态派发会转为静态派发
  * private会约束方法或属性无法被外部访问，动态派发会转为静态派发

> swift比OC快的一个关键就是可以消解动态派发。



### V-Table & Witness-Table

{% code title="Model.swift" %}
```
```
{% endcode %}

{% code title="Model.sil" %}
```
```
{% endcode %}



