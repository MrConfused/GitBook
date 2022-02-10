---
description: Swift 在编译过程中的中间产物，通过 SIL 可以了解swift 底层的实现细节
---

# SIL(Swift Intermediate Language )



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
* IR 不合适源码级别分析，对于源代码级别的静态分析不够友好
* clang 的静态分析实现（CFG，Control Flow Graph）不够精确，和 IR 之间有很多重复的逻辑。



## swift编译器

![swift编译过程](<../.gitbook/assets/image (5).png>)

![swift的前端编译过程](<../.gitbook/assets/image (1).png>)

与clang相比，swift的前端编译流程中，在AST和IR中间多了一层SIL，以弥补clang的缺陷，原本在 clang 中独立的静态分析和 IR 生成阶段整合了起来**。**

可以看到Clang在实现的时候，很多阶段交织在一起，耦合严重，而swift加入了sil中间层，解决了这个问题。

### sil

#### 特点

*   Fully represents program semantics&#x20;

    能够完整的表达程序的语义。
*   Designed for both code generation and analysis&#x20;

    被设计用于代码生成和静态分析
*   Sits on the hot path of the compiler pipeline&#x20;

    处于编译的流水线中，而不是独立于之外
*   Bridges the abstraction gap between source and LLVM

    在源代码和 LLVM 之间架起了抽象的桥梁

#### 语法

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

#### 作用

* 看编译器不同级别的优化（比如无用代码裁剪，不同写法对性能影响）
* 了解函数派发方式（`final`/`static`/`dynamic` 等）对函数的影响，KVO
* 了解swift语言的底层实现
* 等等

#### Builtin <a href="#builtin" id="builtin"></a>

Builtin 将 LLVM IR 的类型和方法直接暴露给 Swift 标准库，使用时没有额外的运行时负担。

#### 源码生成sil文件的命令：

```bash
// 将m ain.swift 编译成 SIL 代码
swiftc -emit-sil main.swift 

// 将 main.swift 编译成 SIL，并保存到 main.sil 文件中
swiftc -emit-sil main.swift >> main.sil

// 将 main.swift 编译成 SIL的同时， 将命名重整后的符号恢复原样，并保存到 main.sil 文件中
swiftc -emit-sil main.swift | xcrun swift-demangle >> main.sil
```

## 阅读sil代码

### 空文件生成的sil：

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

### sil对enum的处理

{% code title="Model.swift" %}
```
enum TaskType: Int {
    case login
}
```
{% endcode %}

{% code title="Model.sil" %}
```
enum TaskType : Int {
  case login
  init?(rawValue: Int)
  typealias RawValue = Int
  var rawValue: Int { get }
}

// TaskType.init(rawValue:)
sil hidden @Model.TaskType.init(rawValue: Swift.Int) -> Model.TaskType? : $@convention(method) (Int, @thin TaskType.Type) -> Optional<TaskType> {
// %0 "rawValue"                                  // users: %5, %3
// %1 "$metatype"
bb0(%0 : $Int, %1 : $@thin TaskType.Type):
  %2 = alloc_stack $TaskType, var, name "self"    // users: %9, %13, %15
  debug_value %0 : $Int, let, name "rawValue", argno 1 // id: %3
  %4 = integer_literal $Builtin.Int64, 0          // user: %6
  %5 = struct_extract %0 : $Int, #Int._value      // user: %6
  %6 = builtin "cmp_eq_Int64"(%4 : $Builtin.Int64, %5 : $Builtin.Int64) : $Builtin.Int1 // user: %7
  cond_br %6, bb1, bb2                            // id: %7

bb1:                                              // Preds: bb0
  %8 = enum $TaskType, #TaskType.login!enumelt    // users: %10, %12
  %9 = begin_access [modify] [static] %2 : $*TaskType // users: %10, %11
  store %8 to %9 : $*TaskType                     // id: %10
  end_access %9 : $*TaskType                      // id: %11
  %12 = enum $Optional<TaskType>, #Optional.some!enumelt, %8 : $TaskType // user: %14
  dealloc_stack %2 : $*TaskType                   // id: %13
  br bb3(%12 : $Optional<TaskType>)               // id: %14

bb2:                                              // Preds: bb0
  dealloc_stack %2 : $*TaskType                   // id: %15
  %16 = enum $Optional<TaskType>, #Optional.none!enumelt // user: %17
  br bb3(%16 : $Optional<TaskType>)               // id: %17

// %18                                            // user: %19
bb3(%18 : $Optional<TaskType>):                   // Preds: bb1 bb2
  return %18 : $Optional<TaskType>                // id: %19
} // end sil function 'Model.TaskType.init(rawValue: Swift.Int) -> Model.TaskType?'

// Int.init(_builtinIntegerLiteral:)
sil public_external [transparent] @Swift.Int.init(_builtinIntegerLiteral: Builtin.IntLiteral) -> Swift.Int : $@convention(method) (Builtin.IntLiteral, @thin Int.Type) -> Int {
// %0                                             // user: %2
bb0(%0 : $Builtin.IntLiteral, %1 : $@thin Int.Type):
  %2 = builtin "s_to_s_checked_trunc_IntLiteral_Int64"(%0 : $Builtin.IntLiteral) : $(Builtin.Int64, Builtin.Int1) // user: %3
  %3 = tuple_extract %2 : $(Builtin.Int64, Builtin.Int1), 0 // user: %4
  %4 = struct $Int (%3 : $Builtin.Int64)          // user: %5
  return %4 : $Int                                // id: %5
} // end sil function 'Swift.Int.init(_builtinIntegerLiteral: Builtin.IntLiteral) -> Swift.Int'

// ~= infix<A>(_:_:)
sil public_external [transparent] @Swift.~= infix<A where A: Swift.Equatable>(A, A) -> Swift.Bool : $@convention(thin) <T where T : Equatable> (@in_guaranteed T, @in_guaranteed T) -> Bool {
// %0                                             // user: %4
// %1                                             // user: %4
bb0(%0 : $*T, %1 : $*T):
  %2 = metatype $@thick T.Type                    // user: %4
  %3 = witness_method $T, #Equatable."==" : <Self where Self : Equatable> (Self.Type) -> (Self, Self) -> Bool : $@convention(witness_method: Equatable) <τ_0_0 where τ_0_0 : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0, @thick τ_0_0.Type) -> Bool // user: %4
  %4 = apply %3<T>(%0, %1, %2) : $@convention(witness_method: Equatable) <τ_0_0 where τ_0_0 : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0, @thick τ_0_0.Type) -> Bool // user: %5
  return %4 : $Bool                               // id: %5
} // end sil function 'Swift.~= infix<A where A: Swift.Equatable>(A, A) -> Swift.Bool'

// TaskType.rawValue.getter
sil hidden @Model.TaskType.rawValue.getter : Swift.Int : $@convention(method) (TaskType) -> Int {
// %0 "self"                                      // users: %2, %1
bb0(%0 : $TaskType):
  debug_value %0 : $TaskType, let, name "self", argno 1 // id: %1
  switch_enum %0 : $TaskType, case #TaskType.login!enumelt: bb1 // id: %2

bb1:                                              // Preds: bb0
  %3 = integer_literal $Builtin.Int64, 0          // user: %4
  %4 = struct $Int (%3 : $Builtin.Int64)          // user: %5
  return %4 : $Int                                // id: %5
} // end sil function 'Model.TaskType.rawValue.getter : Swift.Int'

// protocol witness for static Equatable.== infix(_:_:) in conformance TaskType
sil private [transparent] [thunk] @protocol witness for static Swift.Equatable.== infix(A, A) -> Swift.Bool in conformance Model.TaskType : Swift.Equatable in Model : $@convention(witness_method: Equatable) (@in_guaranteed TaskType, @in_guaranteed TaskType, @thick TaskType.Type) -> Bool {
// %0                                             // user: %4
// %1                                             // user: %4
bb0(%0 : $*TaskType, %1 : $*TaskType, %2 : $@thick TaskType.Type):
  // function_ref == infix<A>(_:_:)
  %3 = function_ref @Swift.== infix<A where A: Swift.RawRepresentable, A.RawValue: Swift.Equatable>(A, A) -> Swift.Bool : $@convention(thin) <τ_0_0 where τ_0_0 : RawRepresentable, τ_0_0.RawValue : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0) -> Bool // user: %4
  %4 = apply %3<TaskType>(%0, %1) : $@convention(thin) <τ_0_0 where τ_0_0 : RawRepresentable, τ_0_0.RawValue : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0) -> Bool // user: %5
  return %4 : $Bool                               // id: %5
} // end sil function 'protocol witness for static Swift.Equatable.== infix(A, A) -> Swift.Bool in conformance Model.TaskType : Swift.Equatable in Model'

// == infix<A>(_:_:)
sil @Swift.== infix<A where A: Swift.RawRepresentable, A.RawValue: Swift.Equatable>(A, A) -> Swift.Bool : $@convention(thin) <τ_0_0 where τ_0_0 : RawRepresentable, τ_0_0.RawValue : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0) -> Bool

// protocol witness for Hashable.hashValue.getter in conformance TaskType
sil private [transparent] [thunk] @protocol witness for Swift.Hashable.hashValue.getter : Swift.Int in conformance Model.TaskType : Swift.Hashable in Model : $@convention(witness_method: Hashable) (@in_guaranteed TaskType) -> Int {
// %0                                             // user: %2
bb0(%0 : $*TaskType):
  // function_ref RawRepresentable<>.hashValue.getter
  %1 = function_ref @(extension in Swift):Swift.RawRepresentable< where A: Swift.Hashable, A.Swift.RawRepresentable.RawValue: Swift.Hashable>.hashValue.getter : Swift.Int : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (@in_guaranteed τ_0_0) -> Int // user: %2
  %2 = apply %1<TaskType>(%0) : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (@in_guaranteed τ_0_0) -> Int // user: %3
  return %2 : $Int                                // id: %3
} // end sil function 'protocol witness for Swift.Hashable.hashValue.getter : Swift.Int in conformance Model.TaskType : Swift.Hashable in Model'

// RawRepresentable<>.hashValue.getter
sil @(extension in Swift):Swift.RawRepresentable< where A: Swift.Hashable, A.Swift.RawRepresentable.RawValue: Swift.Hashable>.hashValue.getter : Swift.Int : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (@in_guaranteed τ_0_0) -> Int

// protocol witness for Hashable.hash(into:) in conformance TaskType
sil private [transparent] [thunk] @protocol witness for Swift.Hashable.hash(into: inout Swift.Hasher) -> () in conformance Model.TaskType : Swift.Hashable in Model : $@convention(witness_method: Hashable) (@inout Hasher, @in_guaranteed TaskType) -> () {
// %0                                             // user: %3
// %1                                             // user: %3
bb0(%0 : $*Hasher, %1 : $*TaskType):
  // function_ref RawRepresentable<>.hash(into:)
  %2 = function_ref @(extension in Swift):Swift.RawRepresentable< where A: Swift.Hashable, A.Swift.RawRepresentable.RawValue: Swift.Hashable>.hash(into: inout Swift.Hasher) -> () : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (@inout Hasher, @in_guaranteed τ_0_0) -> () // user: %3
  %3 = apply %2<TaskType>(%0, %1) : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (@inout Hasher, @in_guaranteed τ_0_0) -> ()
  %4 = tuple ()                                   // user: %5
  return %4 : $()                                 // id: %5
} // end sil function 'protocol witness for Swift.Hashable.hash(into: inout Swift.Hasher) -> () in conformance Model.TaskType : Swift.Hashable in Model'

// RawRepresentable<>.hash(into:)
sil @(extension in Swift):Swift.RawRepresentable< where A: Swift.Hashable, A.Swift.RawRepresentable.RawValue: Swift.Hashable>.hash(into: inout Swift.Hasher) -> () : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (@inout Hasher, @in_guaranteed τ_0_0) -> ()

// protocol witness for Hashable._rawHashValue(seed:) in conformance TaskType
sil private [transparent] [thunk] @protocol witness for Swift.Hashable._rawHashValue(seed: Swift.Int) -> Swift.Int in conformance Model.TaskType : Swift.Hashable in Model : $@convention(witness_method: Hashable) (Int, @in_guaranteed TaskType) -> Int {
// %0                                             // user: %3
// %1                                             // user: %3
bb0(%0 : $Int, %1 : $*TaskType):
  // function_ref RawRepresentable<>._rawHashValue(seed:)
  %2 = function_ref @(extension in Swift):Swift.RawRepresentable< where A: Swift.Hashable, A.Swift.RawRepresentable.RawValue: Swift.Hashable>._rawHashValue(seed: Swift.Int) -> Swift.Int : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (Int, @in_guaranteed τ_0_0) -> Int // user: %3
  %3 = apply %2<TaskType>(%0, %1) : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (Int, @in_guaranteed τ_0_0) -> Int // user: %4
  return %3 : $Int                                // id: %4
} // end sil function 'protocol witness for Swift.Hashable._rawHashValue(seed: Swift.Int) -> Swift.Int in conformance Model.TaskType : Swift.Hashable in Model'

// RawRepresentable<>._rawHashValue(seed:)
sil @(extension in Swift):Swift.RawRepresentable< where A: Swift.Hashable, A.Swift.RawRepresentable.RawValue: Swift.Hashable>._rawHashValue(seed: Swift.Int) -> Swift.Int : $@convention(method) <τ_0_0 where τ_0_0 : Hashable, τ_0_0 : RawRepresentable, τ_0_0.RawValue : Hashable> (Int, @in_guaranteed τ_0_0) -> Int

// protocol witness for RawRepresentable.init(rawValue:) in conformance TaskType
sil private [transparent] [thunk] @protocol witness for Swift.RawRepresentable.init(rawValue: A.RawValue) -> A? in conformance Model.TaskType : Swift.RawRepresentable in Model : $@convention(witness_method: RawRepresentable) (@in Int, @thick TaskType.Type) -> @out Optional<TaskType> {
// %0                                             // user: %7
// %1                                             // user: %3
bb0(%0 : $*Optional<TaskType>, %1 : $*Int, %2 : $@thick TaskType.Type):
  %3 = load %1 : $*Int                            // user: %6
  %4 = metatype $@thin TaskType.Type              // user: %6
  // function_ref TaskType.init(rawValue:)
  %5 = function_ref @Model.TaskType.init(rawValue: Swift.Int) -> Model.TaskType? : $@convention(method) (Int, @thin TaskType.Type) -> Optional<TaskType> // user: %6
  %6 = apply %5(%3, %4) : $@convention(method) (Int, @thin TaskType.Type) -> Optional<TaskType> // user: %7
  store %6 to %0 : $*Optional<TaskType>           // id: %7
  %8 = tuple ()                                   // user: %9
  return %8 : $()                                 // id: %9
} // end sil function 'protocol witness for Swift.RawRepresentable.init(rawValue: A.RawValue) -> A? in conformance Model.TaskType : Swift.RawRepresentable in Model'

// protocol witness for RawRepresentable.rawValue.getter in conformance TaskType
sil private [transparent] [thunk] @protocol witness for Swift.RawRepresentable.rawValue.getter : A.RawValue in conformance Model.TaskType : Swift.RawRepresentable in Model : $@convention(witness_method: RawRepresentable) (@in_guaranteed TaskType) -> @out Int {
// %0                                             // user: %5
// %1                                             // user: %2
bb0(%0 : $*Int, %1 : $*TaskType):
  %2 = load %1 : $*TaskType                       // user: %4
  // function_ref TaskType.rawValue.getter
  %3 = function_ref @Model.TaskType.rawValue.getter : Swift.Int : $@convention(method) (TaskType) -> Int // user: %4
  %4 = apply %3(%2) : $@convention(method) (TaskType) -> Int // user: %5
  store %4 to %0 : $*Int                          // id: %5
  %6 = tuple ()                                   // user: %7
  return %6 : $()                                 // id: %7
} // end sil function 'protocol witness for Swift.RawRepresentable.rawValue.getter : A.RawValue in conformance Model.TaskType : Swift.RawRepresentable in Model'

// protocol witness for static Equatable.== infix(_:_:) in conformance Int
sil shared_external [transparent] [thunk] @protocol witness for static Swift.Equatable.== infix(A, A) -> Swift.Bool in conformance Swift.Int : Swift.Equatable in Swift : $@convention(witness_method: Equatable) (@in_guaranteed Int, @in_guaranteed Int, @thick Int.Type) -> Bool {
// %0                                             // user: %3
// %1                                             // user: %5
bb0(%0 : $*Int, %1 : $*Int, %2 : $@thick Int.Type):
  %3 = struct_element_addr %0 : $*Int, #Int._value // user: %4
  %4 = load %3 : $*Builtin.Int64                  // user: %7
  %5 = struct_element_addr %1 : $*Int, #Int._value // user: %6
  %6 = load %5 : $*Builtin.Int64                  // user: %7
  %7 = builtin "cmp_eq_Int64"(%4 : $Builtin.Int64, %6 : $Builtin.Int64) : $Builtin.Int1 // user: %8
  %8 = struct $Bool (%7 : $Builtin.Int1)          // user: %9
  return %8 : $Bool                               // id: %9
} // end sil function 'protocol witness for static Swift.Equatable.== infix(A, A) -> Swift.Bool in conformance Swift.Int : Swift.Equatable in Swift'

sil_witness_table hidden TaskType: Equatable module Model {
  method #Equatable."==": <Self where Self : Equatable> (Self.Type) -> (Self, Self) -> Bool : @protocol witness for static Swift.Equatable.== infix(A, A) -> Swift.Bool in conformance Model.TaskType : Swift.Equatable in Model	// protocol witness for static Equatable.== infix(_:_:) in conformance TaskType
}

sil_witness_table hidden TaskType: Hashable module Model {
  base_protocol Equatable: TaskType: Equatable module Model
  method #Hashable.hashValue!getter: <Self where Self : Hashable> (Self) -> () -> Int : @protocol witness for Swift.Hashable.hashValue.getter : Swift.Int in conformance Model.TaskType : Swift.Hashable in Model	// protocol witness for Hashable.hashValue.getter in conformance TaskType
  method #Hashable.hash: <Self where Self : Hashable> (Self) -> (inout Hasher) -> () : @protocol witness for Swift.Hashable.hash(into: inout Swift.Hasher) -> () in conformance Model.TaskType : Swift.Hashable in Model	// protocol witness for Hashable.hash(into:) in conformance TaskType
  method #Hashable._rawHashValue: <Self where Self : Hashable> (Self) -> (Int) -> Int : @protocol witness for Swift.Hashable._rawHashValue(seed: Swift.Int) -> Swift.Int in conformance Model.TaskType : Swift.Hashable in Model	// protocol witness for Hashable._rawHashValue(seed:) in conformance TaskType
}

sil_witness_table hidden TaskType: RawRepresentable module Model {
  associated_type RawValue: Int
  method #RawRepresentable.init!allocator: <Self where Self : RawRepresentable> (Self.Type) -> (Self.RawValue) -> Self? : @protocol witness for Swift.RawRepresentable.init(rawValue: A.RawValue) -> A? in conformance Model.TaskType : Swift.RawRepresentable in Model	// protocol witness for RawRepresentable.init(rawValue:) in conformance TaskType
  method #RawRepresentable.rawValue!getter: <Self where Self : RawRepresentable> (Self) -> () -> Self.RawValue : @protocol witness for Swift.RawRepresentable.rawValue.getter : A.RawValue in conformance Model.TaskType : Swift.RawRepresentable in Model	// protocol witness for RawRepresentable.rawValue.getter in conformance TaskType
}

sil_witness_table public_external Int: Equatable module Swift {
  method #Equatable."==": <Self where Self : Equatable> (Self.Type) -> (Self, Self) -> Bool : @protocol witness for static Swift.Equatable.== infix(A, A) -> Swift.Bool in conformance Swift.Int : Swift.Equatable in Swift	// protocol witness for static Equatable.== infix(_:_:) in conformance Int
}
```
{% endcode %}







{% code title="Model.swift" %}
```
```
{% endcode %}

{% code title="Model.sil" %}
```
```
{% endcode %}

#### Codable

### sil对struct的处理

#### 存储属性

#### 计算属性

#### 实例方法

#### 静态方法

#### Codable



### sil对class的处理

#### 存储属性

#### 计算属性

#### 实例方法

#### 静态方法

#### Codable

### sil对protocol的处理

{% code title="Model.swift" %}
```
protocol PointTask {
    var points: Int { get }
    func setPoints(newValue: Int)
    static func log()
}

extension PointTask {
    var points: Int { 99 }

    func setPoints(newValue: Int) {}
    static func log() {}
}
```
{% endcode %}

{% code title="Model.sil" %}
```
protocol PointTask {
  var points: Int { get }
  func setPoints(newValue: Int)
  static func log()
}

extension PointTask {
  var points: Int { get }
  func setPoints(newValue: Int)
  static func log()
}

// PointTask.points.getter
sil hidden @(extension in Model):Model.PointTask.points.getter : Swift.Int : $@convention(method) <Self where Self : PointTask> (@in_guaranteed Self) -> Int {
// %0 "self"                                      // user: %1
bb0(%0 : $*Self):
  debug_value_addr %0 : $*Self, let, name "self", argno 1 // id: %1
  %2 = integer_literal $Builtin.Int64, 99         // user: %3
  %3 = struct $Int (%2 : $Builtin.Int64)          // user: %4
  return %3 : $Int                                // id: %4
} // end sil function '(extension in Model):Model.PointTask.points.getter : Swift.Int'

// Int.init(_builtinIntegerLiteral:)
sil public_external [transparent] @Swift.Int.init(_builtinIntegerLiteral: Builtin.IntLiteral) -> Swift.Int : $@convention(method) (Builtin.IntLiteral, @thin Int.Type) -> Int {
// %0                                             // user: %2
bb0(%0 : $Builtin.IntLiteral, %1 : $@thin Int.Type):
  %2 = builtin "s_to_s_checked_trunc_IntLiteral_Int64"(%0 : $Builtin.IntLiteral) : $(Builtin.Int64, Builtin.Int1) // user: %3
  %3 = tuple_extract %2 : $(Builtin.Int64, Builtin.Int1), 0 // user: %4
  %4 = struct $Int (%3 : $Builtin.Int64)          // user: %5
  return %4 : $Int                                // id: %5
} // end sil function 'Swift.Int.init(_builtinIntegerLiteral: Builtin.IntLiteral) -> Swift.Int'

// PointTask.setPoints(newValue:)
sil hidden @(extension in Model):Model.PointTask.setPoints(newValue: Swift.Int) -> () : $@convention(method) <Self where Self : PointTask> (Int, @in_guaranteed Self) -> () {
// %0 "newValue"                                  // user: %2
// %1 "self"                                      // user: %3
bb0(%0 : $Int, %1 : $*Self):
  debug_value %0 : $Int, let, name "newValue", argno 1 // id: %2
  debug_value_addr %1 : $*Self, let, name "self", argno 2 // id: %3
  %4 = tuple ()                                   // user: %5
  return %4 : $()                                 // id: %5
} // end sil function '(extension in Model):Model.PointTask.setPoints(newValue: Swift.Int) -> ()'

// static PointTask.log()
sil hidden @static (extension in Model):Model.PointTask.log() -> () : $@convention(method) <Self where Self : PointTask> (@thick Self.Type) -> () {
// %0 "self"                                      // user: %1
bb0(%0 : $@thick Self.Type):
  debug_value %0 : $@thick Self.Type, let, name "self", argno 1 // id: %1
  %2 = tuple ()                                   // user: %3
  return %2 : $()                                 // id: %3
} // end sil function 'static (extension in Model):Model.PointTask.log() -> ()'
```
{% endcode %}

#### extension

