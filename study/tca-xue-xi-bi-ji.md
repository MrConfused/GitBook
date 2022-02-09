---
description: 记录学习TCA及使用过程中的问题
---

# TCA学习笔记

## 第三方博客

1. [pointfreeco / swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture)
2. [**ComposableArchitecture** Documentation](https://pointfreeco.github.io/swift-composable-architecture/)
3. [TCA - SwiftUI 的救星？(一) ](https://onevcat.com/2021/12/tca-1/)| [TCA - SwiftUI 的救星？(二)](https://onevcat.com/2021/12/tca-2/)

## 问题

* 在一个Effect中发送多个Action

```swift
Effect.merge(Effect(value:), Effect(value:))
```

* Effect转成Action

```swift
Effect().map { _ in TimerAction.timerTicked }
```

* State值实现双向绑定，当作binding使用

```swift
// 定义
struct MyState: Equatable {
  @BindableState var foo: Bool = false
  @BindableState var bar: String = ""
}
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

* 创建一个async Effect

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
