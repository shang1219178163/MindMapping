# Swizzle

Swizzle 是 Objective-C 运行时的黑魔法之一。我们可以通过 Swizzle 的手段，在运行时对某些方法的实现进行替换，这是 Objective-C 甚至说 Cocoa 开发中最为华丽，同时也是最为危险的技巧之一。

因为 Objective-C 在方法调用时是通过类的 dispatch table 来用 selector 对实现进行查找的，因此我们在运行时如果能够替换掉某个 selector 对应的实现，那么我们就能在运行时 “重新定义” 这个方法的行为。如果你不太理解的话，可以想象成某个类能响应的方法是存放在一个类似字典的结构中的，键为方法的名字 (也就是 selector)，而值就是方法真正做的事情。执行某个方法时我们告诉 Objective-C 运行时想要执行的方法的名字，然后使用这个名字从这个 “字典” 中取值并执行。通过替换这里的值，我们就可以在不改变原来代码结构的情况下偷天换日了。

一般来说可能不太用得到这样的技术，但是在某些情况下会非常有用，特别是当我们需要触及到一些系统框架的东西的时候。比如我们已经有一个庞大的项目，并使用了很多 `UIButton` 来让用户交互。某一天，产品汪突然说我们需要统计一下整个 app 中用户点击所有按钮的次数。对于完全不懂技术的选手来说，在他们眼中这似乎不应该是什么难事 -- 只要弄个计数器然后在每次点按钮的时候加一就可以了嘛。但是对于每一个以代码为生的人来说，面临的一个严峻的问题是，这要怎么办。

我们当然可以寻遍项目里的所有按钮点击后的事件代码，然后建立一个全局计数器来计数，但是，之后的维护怎么办，寻找的时候发生了遗漏怎么办，新加入的人不知道这茬怎么办？显然这是最糟糕的一条路。另一个方法是创建一个 `UIButton` 的子类，然后重写它的点击事件的方法。这种策略虽然好些，但是我们需要找遍项目中的按钮，并改变它们的继承关系，上面的那些问题也依然存在，而且要是我们已经在项目中使用了其他 `UIButton` 的子类的话，我们就不得不再去为那些子类创建新的子类，费时费力。

这种时候就该轮到 Swizzle 大显身手了。我们在全局范围内将**所有的** `UIButton` 的发送事件的方法换掉，就可以一劳永逸地解决这个问题 -- 没有一段段代码的替换查找，不会遗漏任何按钮，之后开发中也不需要对这个计数的功能特别地注意什么。

在 Swift 中，我们也可以利用 Objective-C 运行时来进行 Swizzle。比如上面的例子，我们就可以使用这样的扩展来完成：

```swift
extension UIButton {
    class func xxx_swizzleSendAction() {
        struct xxx_swizzleToken {
            static var onceToken : dispatch_once_t = 0
        }
        dispatch_once(&xxx_swizzleToken.onceToken) {
            let cls: AnyClass! = UIButton.self

            let originalSelector = #selector(sendAction(_:to:forEvent:))
            let swizzledSelector = #selector(xxx_sendAction(_:to:forEvent:))

            let originalMethod =
                        class_getInstanceMethod(cls, originalSelector)
            let swizzledMethod =
                        class_getInstanceMethod(cls, swizzledSelector)

            method_exchangeImplementations(originalMethod, swizzledMethod)
        }
    }

    public func xxx_sendAction(action: Selector,
                                   to: AnyObject!,
                             forEvent: UIEvent!)
    {
        struct xxx_buttonTapCounter {
            static var count: Int = 0
        }

        xxx_buttonTapCounter.count += 1
        print(xxx_buttonTapCounter.count)
        xxx_sendAction(action, to: to, forEvent: forEvent)
    }
}
```

在 `xxx_swizzleSendAction` 方法 (因为是向一个常用类中添加方法，最好还是加上前缀以防万一) 中，我们先获取将被替换的方法 (`sendAction:to:forEvent:`) 和用来替换它的方法 (`xxx_sendAction:to:forEvent:`) 的 selector，然后通过运行时对这两个方法的具体实现进行了交换。在 `xxx_sendAction:to:forEvent:` 的实现中，我们先将计数器进行加一，然后输出。最后我们看起来是在这个方法中调用了自己，似乎会形成一个死循环。但是因为我们实际上已经交换了 `sendAction:to:forEvent:` 和 `xxx_sendAction:to:forEvent:` 的实现，所以在做这个调用时恰好调用到的是原来的那个方法的实现。同理，在外部使用 `sendAction:to:forEvent:` 的时候 (也就是点击按钮的时候)，实际调用的实现会是我们在这里定义的带有计数器累加的实现。

最后我们需要在 app 启动时调用这个 `xxx_swizzleSendAction` 方法。在 Objective-C 中我们一般在 category 的 `+load` 中完成，但是 Swift 的 `extension` 和 Objective-C 的 category 略有不同，extension 并不是运行时加载的，因此也没有加载时候就会被调用的类似 `load` 的方法。另外，extension 中也不应该做方法重写去覆盖 `load` (其实重写也是无效的)。事实上，Swift 实现的 `load` 并不是在 app 运行开始就被调用的。基于这些理由，我们使用另一个类初始化时会被调用的方法来进行交换：

```swift
extension UIButton {
    override public class func initialize() {
        if self != UIButton.self {
            return
        }
        UIButton.xxx_swizzleSendAction()
    }
}
```

和 `+load` 不同的是，`+initialize` 会在当前类以及它的子类被初始化时调用。在这里我们对当前类的类型进行了判断，来保证安全性。另外，在 `xxx_swizzleSendAction` 中，也使用一个 once_token 来保证交换代码仅会被执行一次。

现在，我们所有的按钮事件都会走我们替换进去的方法了，每点一次实际发送了事件的按钮，你都能在控制台看到当前点击数的输出了。

这种方式的 Swizzle 使用了 Objective-C 的动态派发，对于 `NSObject` 的子类是可以直接使用的，但是对于 Swift 的类，因为默认并没有使用 Objective-C 运行时，因此也没有动态派发的方法列表，所以如果要 Swizzle 的是 Swift 类型的方法的话，我们需要将原方法和替换方法都加上 `dynamic` 标记，以指明它们需要使用动态派发机制。关于这方面的知识，可以参看 [@objc 和 dynamic](objc-dynamic.md) 的内容。

> 我们有另一种方法，甚至可以完全不借助 Objective-C 运行时，而是直接替换 Swift 调用时使用的封装过的类似 “函数指针”，来达到对 Swift 类型进行 “Swizzle” 的目的。但是这个话题和背后的原理超出了本书的范围，如果你对此感兴趣，可以尝试看看 [`SWRoute`](https://github.com/rodionovd/SWRoute) 这个项目以及它背后的[原理](https://github.com/rodionovd/SWRoute/wiki/Function-hooking-in-Swift)。
