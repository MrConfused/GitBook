---
description: Swift 在编译过程中的中间产物，通过 SIL 可以了解swift 底层的实现细节
---

# SIL(Swift Intermediate Language )

### swift的编译过程：

LLVM编译架构的三段式设计（前端、中间优化器、后端）：

![LLVM三段式](../.gitbook/assets/image.png)

* 前端：**解析源代码**。通过词法分析、语法分析、语义分析，检查代码中是否存在错误，然后构建**抽象语法树**（Abstract Syntax Tree ， **AST**），然后生成**中间代码（** Intermediate Representation，**IR）。**
* 中间优化器：负责各种优化，**缩小包的体积**（剥离符号 **）** 、**改善代码的运行时间（** 消除冗余计算、减少指针跳转次数等）。
* 后端：代码生成器。将IR映射到目标指令集，生成**机器语言**，并且进行机器相关的代码优化。



![Clang编译过程](<../.gitbook/assets/image (3).png>)



![swift编译过程](<../.gitbook/assets/image (5).png>)

与clang相比，swift编译在llvn对前端流程中，在AST和IR中间多了一层SIR。

目的是弥补clang对缺陷，例如：无法执行一些**高级分析，可靠的诊断和优化。**



### 源码生成sil文件的命令：

```bash
// 将m ain.swift 编译成 SIL 代码
swiftc -emit-sil main.swift 

// 将 main.swift 编译成 SIL，并保存到 main.sil 文件中
swiftc -emit-sil main.swift >> main.sil

// 将 main.swift 编译成 SIL的同时， 将命名重整后的符号恢复原样，并保存到 main.sil 文件中
swiftc -emit-sil main.swift | xcrun swift-demangle >> main.sil
```



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

### sil对struct的处理

#### 存储属性

#### 计算属性

#### 实例方法

#### 静态方法

#### Codable

### sil对enum的处理

#### Codable

### sil对class的处理

#### 存储属性

#### 计算属性

#### 实例方法

#### 静态方法

#### Codable

### sil对protocol的处理

#### extension

