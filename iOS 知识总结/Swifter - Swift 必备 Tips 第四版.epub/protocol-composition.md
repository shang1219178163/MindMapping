# protocol 组合

在 Swift 中我们可以使用 `Any` 来表示任意类型 (如果你对此感到模糊或者陌生的话，可以先看看 Apple 的 Swift 官方教程或者本书的[这篇 tip](any-anyobject.md))，充满好奇心的同学可能已经发现，`Any` 这个类型的定义十分奇怪，它是一个 `protocol<>` 的同名类型。

`protocol<>` 这样形式的写法在日常 Swift 使用中其实并不多见，这其实是 Swift 的协议组合的用法。标准的语法形式是下面这样的：

```swift
protocol<ProtocolA, ProtocolB, ProtocolC>
```

尖括号内是具体协议的名称，这里表示将名称为 `ProtocolA`，`ProtocolB` 以及 `ProtocolC` 的协议组合在一起的一个新的匿名协议。实现这个匿名协议就意味着要同时实现这三个协议所定义的内容。所以说，这里的 protocol 组合的写法和下面的新声明的 `ProtocolD` 是相同的：

```swift
protocol ProtocolD: ProtocolA, ProtocolB, ProtocolC {

}
```

那么，在 `Any` 定义的时候的里面什么都不写的 `protocol<>` 是什么意思呢？从语意上来说，这代表一个 "需要实现空协议的协议"，其实就是任意类型的意思了。

除了可以方便地表达空协议这一概念以外，protocol 的组合相比于新创建一个协议的最大区别就在于其匿名性。有时候我们可以借助这个特性写出更清晰的代码。因为 Swift 的类型组织是比较松散的，你的类型可以由不同的 `extension` 来定义实现不同的协议，Swift 也并没有要求它们在同一个文件中。这样，当一个类型实现了很多协议时，在使用这个类型的时候我们很可能在不查询相关代码的情况下很难知道这个类型所实现的协议。

举个理想化的例子，比如我们有下面的三个协议，分别代表了三种动物的叫的方式，而有一种谜之动物，同时实现了这三个协议：

```swift
protocol KittenLike {
    func meow() -> String
}

protocol DogLike {
    func bark() -> String
}

protocol TigerLike {
    func aou() -> String
}

class MysteryAnimal: KittenLike, DogLike, TigerLike {
    func meow() -> String {
        return "meow"
    }

    func bark() -> String {
        return "bark"
    }

    func aou() -> String {
        return "aou"
    }
}
```

现在我们想要检查某种动物作为宠物的时候的叫声的话，我们可能要重新定义一个叫做 `PetLike` 的协议，表明其实现 `KittenLike` 和 `DogLike`；如果稍后我们又想检查某种动物作为猫科动物的叫声的话，我们也许又要去定义一个叫做 `CatLike` 这样的实现 `KittenLike` 和 `TigerLike` 的协议。最后我们大概会写出这样的东西：

```swift
protocol PetLike: KittenLike, DogLike {

}

protocol CatLike: KittenLike, TigerLike {

}

struct SoundChecker {
    static func checkPetTalking(pet: PetLike) {
        //...
    }

    static func checkCatTalking(cat: CatLike) {
        //...
    }
}
```

虽然没有引入定义任何新的内容，但是为了实现这个需求，我们还是添加了两个空 protocol，这可能会让人困惑，代码的使用者 (也包括一段时间后的你自己) 可能会去猜测 `PetLike` 和 `CatLike` 的作用 -- 其实它们除了标注以外并没有其他作用。借助 protocol 组合的特性，我们可以很好的解决这个问题。protocol 组合是可以使用 [typealias](typealias.md) 来命名的，于是可以将上面的新定义 protocol 的部分换为：

```swift
typealias PetLike = protocol<KittenLike, DogLike>
typealias CatLike = protocol<KittenLike, TigerLike>
```

这样既保持了可读性，也没有多定义不必要的新类型。

另外，其实如果这两个临时协议我们就只用一次的话，如果上下文里理解起来不会有困难，我们完全可以直接将它们匿名化，变成下面这样：

```swift
struct SoundChecker {
    static func checkPetTalking(pet: protocol<KittenLike, DogLike>) {
        //...
    }

    static func checkCatTalking(cat: protocol<KittenLike, TigerLike>) {
        //...
    }
}
```

这样的好处是定义和使用的地方更加接近，这在代码复杂的时候读代码时可以少一些跳转，多一些专注。但是因为使用了匿名的协议组合，所以能表达的信息毕竟少了一些。如果要实际使用这种方法的话，还是需要多多斟酌。

虽然这一节已经够长了，不过我还是想多提一句关于实现多个协议时协议内方法冲突的解决方法。因为在 Swift 的世界中没有人限制或者保证过不同协议的方法不能重名，所以这是有可能出现的情况。比如有 `A` 和 `B` 两个协议，定义如下：

```swift
protocol A {
    func bar() -> Int
}

protocol B {
    func bar() -> String
}
```

两个协议中 `bar()` 只有返回值的类型不同。我们如果有一个类型 `Class` 同时实现了 A 和 B，我们要怎么才能避免和解决调用冲突呢？

```swift
class Class: A, B {
    func bar() -> Int {
        return 1
    }

    func bar() -> String {
        return "Hi"
    }
}
```

这样一来，对于 `bar()`，只要在调用前进行类型转换就可以了：

```swift
let instance = Class()
let num = (instance as A).bar()  // 1
let str = (instance as B).bar()  // "Hi"
```
