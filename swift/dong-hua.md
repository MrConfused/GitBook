---
description: 实现效果：动画先放大展示，然后摇动，停200ms，摇动，缩小回原样，等2s，循环展示
---

# 动画

### 方案1：使用UIViewPropertyAnimator

```swift
let animator = UIViewPropertyAnimator(duration: 1, curve: .easeInOut)
animator.addAnimations {
    [0.0, -3.0, 3.0, -3.0, 3.0, -3.0, 3.0, 0.0].forEach {
        let angle = CGFloat.pi * CGFloat($0) / CGFloat(180.0)
        let rotation = CGAffineTransform(rotationAngle: angle)
        view.transform = view.transform.concatenating(rotation)
    }
}
animator.addCompletion(animatorCompletion)
animator.startAnimation(afterDelay: delay)
```

* 放大动画正常展示，但是来回摇动的动画不会展示任何效果
*
  * 调试发现，如果只向一个方向旋转，动画是正常播放的，
  * 原因猜测是在通过concatenating连接两个旋转角度相反的动效，会导致动画被抵消
* 并且animator在view被复用时需要移除存在的animator，处理逻辑也很复杂

### 方案2：使用UIView.animateKeyFrame

```swift
func playShakeAnimation(
    view: UIView,
    delay: TimeInterval,
    withScale: CGFloat,
    animatorCompletion: @escaping (Bool) -> Void
) {
    UIView.animateKeyframes(
        withDuration: 1.2,
        delay: delay,
        options: .allowUserInteraction,
        animations: {
            let angle: CGFloat = 3.0 * Double.pi / 180.0
            let scaleTransform = CGAffineTransform(scaleX: withScale, y: withScale)
            UIView.addKeyframe(withRelativeStartTime: 0, relativeDuration: 0, animations: {
                view.transform = .init(rotationAngle: 0).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 0, relativeDuration: 0.1, animations: {
                view.transform = .init(rotationAngle: -angle).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 0.1, relativeDuration: 0.2, animations: {
                view.transform = .init(rotationAngle: angle).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 0.3, relativeDuration: 0.2, animations: {
                view.transform = .init(rotationAngle: -angle).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 0.5, relativeDuration: 0.2, animations: {
                view.transform = .init(rotationAngle: angle).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 0.7, relativeDuration: 0.2, animations: {
                view.transform = .init(rotationAngle: -angle).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 0.9, relativeDuration: 0.2, animations: {
                view.transform = .init(rotationAngle: angle).concatenating(scaleTransform)
            })
            UIView.addKeyframe(withRelativeStartTime: 1.1, relativeDuration: 0.1, animations: {
                view.transform = .init(rotationAngle: 0).concatenating(scaleTransform)
            })
        },
        completion: animatorCompletion
    )
}

func playAnimation(view: UIView, delay: TimeInterval) {
    playScaleAnimation(view: view, scale: 1.1, delay: delay) { [weak self] _ in
        self?.playShakeAnimation(view: view, delay: 0.1, withScale: 1.1) { [weak self] _ in
            self?.playShakeAnimation(view: view, delay: 0.2, withScale: 1.1) { [weak self] _ in
                self?.playScaleAnimation(view: view, scale: 1, delay: 0.2) { [weak self] _ in
                    self?.playAnimation(view: view, delay: 2)
                }
            }
        }
    }
}
```

* 动效正常，但是由于多个动效直接是通过completion闭包来处理的，导致在展示动画的view被collectionView复用时，移除动画的逻辑较为复杂

### 方案3: 使用layerAnimation

```swift
func playAnimation(view: UIView) {
    let scaleAnimation1 = CABasicAnimation(keyPath: "transform.scale")
    scaleAnimation1.duration = 0.3
    scaleAnimation1.toValue = baseIsCompact ? 1.1 : 1.05
    scaleAnimation1.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
    scaleAnimation1.fillMode = CAMediaTimingFillMode.forwards
    let rotateAngle: CGFloat = baseIsCompact ? 3 : 1.5
    let rotateAngles: [CGFloat] = [
        0,
        -CGFloat.pi * rotateAngle / 180.0,
        CGFloat.pi * rotateAngle / 180.0,
        -CGFloat.pi * rotateAngle / 180.0,
        CGFloat.pi * rotateAngle / 180.0,
        -CGFloat.pi * rotateAngle / 180.0,
        CGFloat.pi * rotateAngle / 180.0,
        0
    ]
    let rotateTimingFunctions = [
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut),
        CAMediaTimingFunction(name: .easeInEaseOut)
    ]
    let rotateAnimation1 = CAKeyframeAnimation(keyPath: "transform.rotation")
    rotateAnimation1.duration = 1.2
    rotateAnimation1.beginTime = 0.5
    rotateAnimation1.values = rotateAngles
    rotateAnimation1.timingFunctions = rotateTimingFunctions
    let rotateAnimation2 = CAKeyframeAnimation(keyPath: "transform.rotation")
    rotateAnimation2.duration = 1.2
    rotateAnimation2.beginTime = 2
    rotateAnimation2.values = rotateAngles
    rotateAnimation2.timingFunctions = rotateTimingFunctions
    let scaleAnimation2 = CABasicAnimation(keyPath: "transform.scale")
    scaleAnimation2.beginTime = 3.4
    scaleAnimation2.duration = 0.3
    scaleAnimation2.toValue = 1.0
    scaleAnimation2.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
    scaleAnimation2.fillMode = CAMediaTimingFillMode.forwards
    let group = CAAnimationGroup()
    group.animations = [scaleAnimation1, rotateAnimation1, rotateAnimation2, scaleAnimation2]
    group.duration = 3.7 + 2
    group.isRemovedOnCompletion = false
    group.repeatCount = .infinity
    view.layer.add(group, forKey: nil)
}
```

* 移除动画时只需要调用layer.removeAllAnimations()
