---
description: 为Swift改进OC API
---

# 混编--NS\_REFINED\_FOR\_SWIFT

> [https://juejin.cn/post/7024572794943832101](https://juejin.cn/post/7024572794943832101)

原理：在OC的方法或属性后面加上`NS_REFINED_FOR_SWIFT`，编译器会隐藏改方法（实际上是在OC的方法名前加上`__`，这样在Xcode中输入方法名时不会有代码提示），在swift中可以重新实现自己期望的方法或属性。



添加了 `NS_REFINED_FOR_SWIFT` 的 Objective-C API 在导入到 Swift 时，具体的 API 重命名规则如下：

* 对于初始化方法，在其第一个参数标签前面加 "\_\_"
* 对于其它方法，在其基名前面加 "\_\_"
* 下标方法将被视为任何其它方法，在方法名前面加 "\_\_"，而不是作为 Swift 下标导入
* 其他声明将在其名称前加上 "\_\_"，例如属性



适用场景：改进OC的api，在swift中提供更好的版本。

* 你想在 Swift 中使用某个 Objective-C API 时，使用不同的方法声明，但要使用类似的底层实现
* 你想在 Swift 中使用某个 Objective-C API 时，采用一些 Swift 的特有类型，比如元组
* 你想在 Swift 中使用某个 Objective-C API 时，重新排列、组合、重命名参数等等，以使该 API 与其它 Swift API 更匹配
* 利用 Swift 支持默认参数值的优势，来减少导入到 Swift 中的一组 Objective-C API 数量
* 解决 Swift 调用 Objective-C 的 API 时可能由于数据类型等不一致导致无法达到预期的问题。例如， Objective-C 方法返回 NSNotFound，在 Swift 中期望返回 nil&#x20;

{% tabs %}
{% tab title="示例1" %}


```objectivec
@interface Color : NSObject
- (void)getRed:(nullable CGFloat *)red
         green:(nullable CGFloat *)green
          blue:(nullable CGFloat *)blue
         alpha:(nullable CGFloat *)alpha NS_REFINED_FOR_SWIFT;
@end
```



```swift
extension Color {
    var rgba: (red: CGFloat, green: CGFloat, blue: CGFloat, alpha: CGFloat) {
        var r: CGFloat = 0.0
        var g: CGFloat = 0.0
        var b: CGFloat = 0.0
        var a: CGFloat = 0.0
        __getRed(&r, green: &g, blue: &b, alpha: &a)
        return (red: r, green: g, blue: b, alpha: a)
    }
}
```
{% endtab %}

{% tab title="示例2" %}
SDWebImage中的sd\_setImageWithURL方法有9个。其中有四个方法后面标记了NS\_REFINED\_FOR\_SWIFT。这样在swift中使用时，只会提示5个方法。

```objectivec
// UIImageView+WebCache.h
- (void)sd_setImageWithURL:(nullable NSURL *)url NS_REFINED_FOR_SWIFT;
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder NS_REFINED_FOR_SWIFT;
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options NS_REFINED_FOR_SWIFT;
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                 completed:(nullable SDExternalCompletionBlock)completedBlock NS_REFINED_FOR_SWIFT;
```

```swift
extension UIImageView {
    open func sd_setImage(with url: URL?, placeholderImage placeholder: UIImage?, options: SDWebImageOptions = [], context: [SDWebImageContextOption : Any]?)
    open func sd_setImage(with url: URL?, completed completedBlock: SDExternalCompletionBlock? = nil)
    open func sd_setImage(with url: URL?, placeholderImage placeholder: UIImage?, options: SDWebImageOptions = [], completed completedBlock: SDExternalCompletionBlock? = nil)
    open func sd_setImage(with url: URL?, placeholderImage placeholder: UIImage?, options: SDWebImageOptions = [], progress progressBlock: SDImageLoaderProgressBlock?, completed completedBlock: SDExternalCompletionBlock? = nil)
    open func sd_setImage(with url: URL?, placeholderImage placeholder: UIImage?, options: SDWebImageOptions = [], context: [SDWebImageContextOption : Any]?, progress progressBlock: SDImageLoaderProgressBlock?, completed completedBlock: SDExternalCompletionBlock? = nil)
}
```
{% endtab %}

{% tab title="示例3" %}


```objectivec
// Declare in Objective-C
@interface MyClass : NSObject
- (NSInteger)indexOfString:(NSString *)aString; 
@end
  
// Generated Swift Interface, 它将返回一个有效的 NSInteger 值或者 NSNotFound
open func index(of aString: String) -> Int
```

```swift
extension MyClass {
    func index(of aString: String) -> Int? { 
        let index = Int(__index(of: aString)) 
        if (index == NSNotFound) {
            return nil    
        }
        return index
    }
}
```
{% endtab %}

{% tab title="示例4" %}


```
@interface MyClass : NSObject
```

```
+ (instancetype)sharedInstance NS_REFINED_FOR_SWIFT;
@end
```

```swift
extension MyClass {
    static var shared: MyClass {
        return __sharedInstance()
    }
}
```
{% endtab %}
{% endtabs %}

