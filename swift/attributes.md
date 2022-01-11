---
description: swift中的注解
---

# Attributes

> [https://docs.swift.org/swift-book/ReferenceManual/Attributes.html](https://docs.swift.org/swift-book/ReferenceManual/Attributes.html)

swift中有两种属性：一种给声明（declarations）提供额外的信息，另一种给类型（types）提供额外的信息。

使用：

```swift
@attribute_name
@attribute_name(attribute_arguments)
```

{% tabs %}
{% tab title="Declaration Attributes" %}
``

### `1.available` <a href="#available" id="available"></a>

> 表示代码仅在特定的平台和操作系统版本有效。

要求：至少有两个参数；参数之间用`,`分隔；第一个参数必须是平台。

平台参数有：

* `iOS`
* `iOSApplicationExtension`
* `macOS`
* `macOSApplicationExtension`
* `macCatalyst`
* `macCatalystApplicationExtension`
* `watchOS`
* `watchOSApplicationExtension`
* `tvOS`
* `tvOSApplicationExtension`
* `swift`

> 可以用`*`表示所有平台，但是swift不用用`*`代替。

其他参数：

* `unavailable` 参数：指定的平台不可用。指定swift版本的可用性时不能使用该参数。
* `introduced: version_number` 参数：指定的平台引进该声明的版本。
  * 如果指定的参数中除了平台名或者swift版本之外，只有一个`introduced`参数，那么可以使用简写：
  * `@availabel(iOS, introduced: 10.1)` ==> `@available(iOS 10.1, *)`
* `deprecated: version_number` 参数：指定平台弃用该声明的版本（当前仍可使用）。
* `obsoleted: version_number` 参数：指定平台废弃该声明的版本（当前无法使用）。
* `message: content` 参数：使用被弃用或者废弃的声明时，编译器展示的警告信息。
* `renamed: new_name` 参数：使用旧名字时，编译器警告使用新名字。

当需要同时指定 Swift 版本和平台可用性时，需要用一个单独的`@available`来指定swift版本：

```swift
@available(swift 3.0.2)
@available(macOS 10.12, *)
struct MyStruct { }
```



### `2.dynamicCallable` <a href="#dynamiccallable" id="dynamiccallable"></a>

> 用于class、struct、enum或protocol。
>
> 将该类型的实例看作可调用的函数。
>
> 必须实现`dynamicallyCall(withArguments:)`或者`dynamicallyCall(withKeywordArguments:)`。



`dynamicallyCall(withArguments:)` 方法的声明必须至少有一个参数遵循 [`ExpressibleByArrayLiteral`](https://developer.apple.com/documentation/swift/expressiblebyarrayliteral) 协议，而返回值类型可以是任意类型。

```swift
@dynamicCallable
struct TelephoneExchange {
    func dynamicallyCall(withArguments phoneNumber: [Int]) {
        // ...
    }
}

let dial = TelephoneExchange()

// 使用动态方法调用
dial(4, 1, 1)

// 直接调用底层方法
dial.dynamicallyCall(withArguments: [4, 1, 1])
```



`dynamicallyCall(withKeywordArguments:)` 方法声明必须只有一个遵循 [`ExpressibleByDictionaryLiteral`](https://developer.apple.com/documentation/swift/expressiblebydictionaryliteral) 协议的参数，返回值可以任意类型。参数的 [`Key`](https://developer.apple.com/documentation/swift/expressiblebydictionaryliteral/2294108-key) 必须遵循 [`ExpressibleByStringLiteral`](https://developer.apple.com/documentation/swift/expressiblebystringliteral) 协议。

如果实现 `dynamicallyCall(withKeywordArguments:)` 方法，则可以在动态方法调用中包含标签。

```swift
@dynamicCallable
struct Repeater {
    func dynamicallyCall(withKeywordArguments pairs: KeyValuePairs<String, Int>) -> String {
        return pairs
            .map { label, count in
                repeatElement(label, count: count).joined(separator: " ")
            }
            .joined(separator: "\n")
    }
}

let repeatLabels = Repeater()
print(repeatLabels(a: 1, b: 2, c: 3, b: 2, a: 1))
// a
// b b
// c c c
// b b
// a
```



如果你同时实现两种 `dynamicallyCall` 方法，则当在方法调用中包含关键字参数时，会调用 `dynamicallyCall(withKeywordArguments:)` 方法，否则调用 `dynamicallyCall(withArguments:)` 方法。



### `3.dynamicMemberLookup` <a href="#dynamicmemberlookup" id="dynamicmemberlookup"></a>

> 用于class、struct、enum或protocol。
>
> 让其在运行时查找成员。
>
> 必须实现`subscript(dynamicMember:)` 下标。



在使用成员函数时，如果函数列表中找不到对应的实现，则会调用`subscript(dynamicMember:)`方法，将成员信息作为参数传递。



`subscript(dynamicMember:)` 的参数是 [`KeyPath`](https://developer.apple.com/documentation/swift/keypath)，[`WritableKeyPath`](https://developer.apple.com/documentation/swift/writablekeypath) 或 [`ReferenceWritableKeyPath`](https://developer.apple.com/documentation/swift/referencewritablekeypath) 类型。它可以使用遵循 [`ExpressibleByStringLiteral`](https://developer.apple.com/documentation/swift/expressiblebystringliteral) 协议的类型作为参数来接受成员名 -- 通常情况下是 `String`。下标返回值类型可以为任意类型。



根据keyPath来动态地查找成员，可用于创建一个包裹数据的包装类型，该类型支持在编译时期进行类型检查。

```swift
struct Point { var x, y: Int }

@dynamicMemberLookup
struct PassthroughWrapper<Value> {
    var value: Value
    subscript<T>(dynamicMember member: KeyPath<Value, T>) -> T {
        get { return value[keyPath: member] }
    }
}

let point = Point(x: 381, y: 431)
let wrapper = PassthroughWrapper(value: point)
print(wrapper.x)
```



### `4.propertyWrapper` <a href="#propertywrapper" id="propertywrapper"></a>

> 用于class、struct、enum。
>
> 对类似对代码，做同一封装。

* 必须定义一个 `wrappedValue` 实例属性，以供外界存取。

```swift
@propertyWrapper struct UserDefaultsWrapper<T> {
    let key: String
    let defaultValue: T
    var storage: UserDefaults
    
    var wrappedValue: T {
        get {
            storage.value(forKey: key) as? T ?? defaultValue
        }
        set {
            storage.setValue(newValue, forKey: key)
        }
    }
}

//使用
struct SettingsViewModel {
    @UserDefaultsWrapper(key: "mark-as-read", defaultValue: true)
    var autoMarkMessagesAsRead: Bool

    @UserDefaultsWrapper(key: "search-page-size", defaultValue: 20)
    var numberOfSearchResultsPerPage: Int
}
```



* `projectedValue`是它可以用来暴露额外功能的第二个值。
* 使用时在包装对属性前加上`$`来使用。

```swift
@propertyWrapper
struct WrapperWithProjection {
    var wrappedValue: Int
    var projectedValue: String {
        return "I'm \(wrappedValue)"
    }
}

struct SomeStruct {
    @WrapperWithProjection var x = 123
}
let s = SomeStruct()
s.x           // 123
s.$x          // I'm 123
```



### `5.resultBuilder` <a href="#result-builder" id="result-builder"></a>

> 应用于class、struct、enum。
>
> 能按顺序构造一组数据结构。以声明式语法为嵌套数据结构实现DSL。



[https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md#result-building-methods](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md#result-building-methods%EF%BC%89)



结果构造方法：

*   `static func buildExpression(_ expression: Expression) -> Component`

    &#x20; 将输入类型构造成中间结果。利用它来执行预处理，比如，将入参转换为内部类型，或为使用方提供额外的类型推断信息。
* `static func buildBlock(_ components: Component...) -> Component`
* &#x20; 将可变数量的中间结果合并成一个中间结果。必须实现这个方法。
*   `static func buildOptional(_ component: Component?) -> Component`

    &#x20; 将可选的中间结果构造成新的中间结果。用于支持不包含 `else` 闭包的 `if` 表达式。
*   `static func buildEither(first: Component) -> Component`

    &#x20; 构造一个由条件约束的中间结果。同时实现它与 `buildEither(second:)` 来支持 `switch` 与包含 `else` 闭包的 `if` 表达式。&#x20;
*   `static func buildEither(second: Component) -> Component`

    &#x20; 构造一个由条件约束的中间结果。同时实现它与 `buildEither(first:)` 来支持 `switch` 与包含 `else` 闭包的 `if` 表达式。&#x20;
*   `static func buildArray(_ components: [Component]) -> Component`

    &#x20; 将中间结果数组构造成中间结果。用于支持 `for` 循环。
*   `static func buildFinalResult(_ component: Component) -> FinalResult`

    &#x20; 将中间结果构造成最终结果。可以在中间结果与最终结果类型不一致的结果构造器中实现这个方法，或是在最终结果返回前对它做处理。
*   `static func buildLimitedAvailability(_ component: Component) -> Component`

    &#x20; 构建中间结果，用于在编译器控制语句（`if #available`）进行可用性检查之外，传递或擦除类型信息。可以在不同的条件分支上擦除类型信息。



这些静态方法的描述中用到了三种类型。`Expression` 为构造器的输入类型，`Component` 为中间结果类型，`FinalResult` 为构造器最终生成结果的类型。

如果结果构造方法中没有明确 `Expression` 或 `FinalResult`，默认会使用 `Component` 的类型。



结果转换：

* 赋值语句也会像表达式一样被转换，但它的类型会被当作 `()`。重载 `buildExpression(_:)` 方法，接收实参类型为 `()` 来特殊处理赋值。
* 分支语句中对可用性的检查（`if #available`）会转换成 `buildLimitedAvailability(_:)` 调用。这个转换早于 `buildEither(first:)`、`buildEither(second:)` 或 `buildOptional(_:)`。类型信息会因进入的条件分支不同发生改变，使用 `buildLimitedAvailability(_:)` 方法可以将它擦除。
* 分支语句会被转换成一连串 `buildEither(first:)` 与 `buildEither(second:)` 的方法调用。语句的不同条件分支会被映射到一颗二叉树的叶子结点上，语句则变成了从根节点到叶子结点路径的嵌套 `buildEither` 方法调用。
* 分支语句不一定会产生值，就像没有 `else` 闭包的 `if` 语句，会转换成 `buildOptional(_:)` 方法调用。如果 `if` 语句满足条件，它的代码块会被转换，作为实参进行传递；否则，`buildOptional(_:)` 会被调用，并用 `nil` 作为它的实参。
* 代码块或 `do` 语句会转换成 `buildBlock(_:)` 调用。闭包中的每一条语句都会被转换，完成后作为实参传入 `buildBlock(_:)`。
* `for` 循环的转换分为三个部分：一个临时数组，一个 `for` 循环，以及一次 `buildArray(_:)` 方法调用。新的 `for` 循环会遍历整个序列，并把每一个中间结果存入临时数组。临时数组会作为实参传递给 `buildArray(_:)` 调用。
* 如果结果构造器实现了 `buildFinalResult(_:)` 方法，最终结果会转换成对于这个方法的调用。它永远最后执行。



不能在结果构造器的转换中使用 `break`、`continue`、`defer`、`guard`，或是 `return` 语句、`while` 语句、`do-catch` 语句。

### `6.frozen` <a href="#frozen" id="frozen"></a>

> 用于struct或者enum。
>
> 限制对数据结构的修改只能在编译迭代的库中使用，未来版本的库无法修改(添加、删除、重新排序)枚举的case或者结构体的存储属性。（保证了数据结构的ABI兼容性）



启用迭代库模式：命令行使用`-enable-library-evolution` 选项，或者Xcode把`BUILD_LIBRARY_FOR_DISTRIBUTION`设为false。



迭代库模式中，与未冻结的数据结构进行交互的代码被编译时，可以在不重新编译的情况下继续修改。

将数据结构标记伟冻结，会放弃这种灵活，取得性能上的提升。



要求：

* 结构体存储属性的类型以及枚举 case 的关联值必须是 `public` 或使用 `usablefrominline` 特性标记。
* 属性不能有属性观察器。
* 为存储实例属性提供初始值的表达式必须遵循与 `inlinable` 函数相同的限制。



### `7.discardableResult` <a href="#discardableresult" id="discardableresult"></a>

> 用于函数，表示返回值可以不被使用。

### `8.GKInspectable` <a href="#gkinspectable" id="gkinspectable"></a>

> 将自定义 GameplayKit 组件属性公开到 SpriteKit 编辑器 UI。
>
> 应用此特性同时表示应用了 `objc` 特性。

### `9.inlinable` <a href="#inlinable" id="inlinable"></a>

> 用于声明函数、计算属性、下标、遍历构造器、析构器。
>
> 编译器可以在调用处把`inlinable`标记的符号替换为具体的实现。

可以和模块中的public、用`usableFromInline`标记的internal属性交互。

在内联中定义的函数和闭包都是默认内联的。

### `10.usableFromInline` <a href="#usablefrominline" id="usablefrominline"></a>

> 用于声明函数、计算属性、下标、遍历构造器、析构器。
>
> 声明必须具有 `internal` 访问级别修饰符。

* 被标记为 `usableFromInline` 的结构体或类的属性的类型、或者枚举的真实值或关联值，只能是被标记为 public 或者 `usableFromInline` 的类型。
* 与 `public` 访问修饰符相同的是，该特性将声明公开为模块公共接口的一部分。但是编译器不允许在模块外部的代码通过名称引用`usableFromInline`标记的声明，只能通过运行时与声明符号进行交互。

### `11.main` <a href="#main" id="main"></a>

> 用于class、struct、enum。
>
> 表示包含了程序流的顶级入口。

必须实现`func main() { ... }`。

swift最多只能拥有一个[顶级代码](broken-reference)入口。&#x20;



### `12.NSApplicationMain` <a href="#nsapplicationmain" id="nsapplicationmain"></a>

> 用于class。
>
> 表示该类是应用程序委托类。

如果你不想使用这个特性，可以提供一个 `main.swift` 文件，并在代码顶层调用 `NSApplicationMain(_:_:)` 函数。



### `13.UIApplicationMain` <a href="#uiapplicationmain" id="uiapplicationmain"></a>

> 参考NSApplicationMain

### `14.NSCopying` <a href="#nscopying" id="nscopying"></a>

> 用于类的存储变量。
>
> 使得该变量的set方法，使用newValue的`copyWithZone(_:)`的返回值进行行赋值。

### `15.NSManaged` <a href="#nsmanaged" id="nsmanaged"></a>

> 用于修饰 `NSManagedObject` 子类中的实例方法或存储型变量属性。
>
> 表明它们的实现由 `Core Data` 在运行时基于相关实体描述动态提供。



### `16.objc` <a href="#objc" id="objc"></a>

> 可以修饰任何能在OC中使用的声明。

* 编译器隐式地将 `objc` 特性添加到 Objective-C 中定义的任何类的子类。
* 添加了 `objc` 的协议不能继承于没有此特性的协议。
* `objc` 特性应用于枚举时，每一个枚举 case 都会以枚举名称和 case 名称组合的方式暴露在 Objective-C 代码中。
* 添加不必要的 `objc` 特性会增加二进制体积并影响性能。

### `17.objcMembers` <a href="#objcmembers" id="objcmembers"></a>

> 用于class。
>
> 表示将objc用于该类、扩展、子类、子类的扩展。

### `18.nonobjc` <a href="#nonobjc" id="nonobjc"></a>

> 用于方法、属性、下标、构造器。
>
> 表示该声明不能在OC中使用。

可以使用 `nonobjc` 特性解决标有 `objc` 的类中桥接方法的循环问题。

### `19.requires_stored_property_inits` <a href="#requires-stored-property-inits" id="requires-stored-property-inits"></a>

> 用于class。
>
> 要求类中所有存储属性提供默认值作为其定义的一部分。

### `20.testable` <a href="#testable" id="testable"></a>

> 应用于 `import` 声明。
>
> 能访问被导入模块中的任何标有 `internal` 访问级别修饰符的实体，犹如它们被标记了 `public` 访问级别修饰符

### `21.warn-unqualified-access` <a href="#warn-unqualified-access" id="warn-unqualified-access"></a>

> 用于顶级函数、实例方法、类方法、静态方法。
>
> 可以减少在同一作用域里访问的同名函数之间的歧义。
{% endtab %}

{% tab title="Type Attributes" %}
### `1.autoclosure` <a href="#autoclosure" id="autoclosure"></a>

### `2.convention` <a href="#convention" id="convention"></a>

### `3.escaping` <a href="#escaping" id="escaping"></a>
{% endtab %}

{% tab title="Switch Case Attributes" %}
### `1.unknown` <a href="#unknown" id="unknown"></a>
{% endtab %}
{% endtabs %}



``

###
