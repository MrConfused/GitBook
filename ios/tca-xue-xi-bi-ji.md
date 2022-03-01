---
description: 记录学习TCA及使用过程中的问题
---

# TCA学习笔记

## 第三方博客

1. github仓库：[pointfreeco / swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture)
2. 官方文档：[**ComposableArchitecture** Documentation](https://pointfreeco.github.io/swift-composable-architecture/)
3. 王巍博客：[TCA - SwiftUI 的救星？(一) ](https://onevcat.com/2021/12/tca-1/)| [TCA - SwiftUI 的救星？(二)](https://onevcat.com/2021/12/tca-2/)
4. rxSwift和combine的操作符对应关系：[CombineCommunity / rxswift-to-combine-cheatsheet](https://github.com/CombineCommunity/rxswift-to-combine-cheatsheet)

## 问题

### 在一个Effect中发送多个Action

```swift
Effect.merge(Effect(value:), Effect(value:))
```

### Effect转成Action

```swift
Effect().map { _ in TimerAction.timerTicked }
```

### State值实现双向绑定，当作binding使用

```swift
// 1.为 State 中的需要和 UI 绑定的变量添加 @BindableState
struct MyState: Equatable {
  @BindableState var foo: Bool = false
  @BindableState var bar: String = ""
}

// 2.将 Action 声明为 BindableAction，
//   然后添加一个“特殊”的 case binding(BindingAction<Counter>)
enum MyAction: BindableAction {
  case binding(BindingAction<MyState>)
}

// 3.在 Reducer 中处理这个 .binding，并添加 .binding() 调用。
let myReducer = {
  // ...
case .binding:
  return .none
}
 .binding()
 
// 使用
struct MyView: View {
  let store: Store<MyState, MyAction>
  var body: some View {
    WithViewStore(store) { viewStore in
      Toggle("Toggle!", isOn: viewStore.binding(\.$foo))
      TextField("Text Field!", text: viewStore.binding(\.$bar))
    }
  }
}
```

### 创建一个async Effect

```swift
Effect.task {
  do {
    let members = try await environment.client.getMembers()
    try environment.localDatabaseClient.saveMembers(members)
    return .complete(.success(()))
  } catch {
    return .complete(.failure(error))
  }
}
.receive(on: DispatchQueue.main)
.eraseToEffect()
```

上面用的是task方法是

{% code title="Effect/Concurrency.swift" %}
```swift
static func task(
    priority: TaskPriority? = nil, 
    operation: @escaping @Sendable () async -> Output
) -> Self where Failure == Never
```
{% endcode %}

operation是个非throws的闭包，查看文档发现task还有另外一个方法，operation是个throws的闭包：

{% code title="Effect/Concurrency.swift" %}
```swift
static func task(
    priority: TaskPriority? = nil, 
    operation: @escaping @Sendable () async throws -> Output
) -> Self where Failure == Error
```
{% endcode %}

使用这个方法，我们可以不在闭包中去catch错误：

```swift
Effect<Void, Error>.task {
    let members = try await environment.client.getBoardMembers(on: boardID)
    try environment.localDatabaseClient.saveTrelloMembers(members)
    return ()
}
.map { Result<Void, Failure>.success($0) }
.catch { Just(Result<Void, Failure>.failure($0)) } // 这两步将Effect转成Result
.map { .Complete($0) }
.receive(on: DispatchQueue.main)
.eraseToEffect()
```
