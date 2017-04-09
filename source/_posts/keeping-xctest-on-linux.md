---
title: 在Linux上保持XCtest同步
date: 2017-04-09 11:41:01
tags:
---

作者 Ole Begemann

[原文链接](https://oleb.net/blog/2017/03/keeping-xctest-in-sync/)

Swfit是跨平台的，但在Apple平台和其他系统表现的并不一致，主要因为两个原因：

* Objective-C runtime 仅适用于Apple平台
* 在非Apple系统中，Foundation框架和其他的核心库的实现方法是不同的。 这就意味着有些Foundation API可能会在macOS／iOS和Linux下产生不同的结果，甚至是还不能被完整执行。

因此，如果你写一个并非仅仅在Apple上执行的库，那么在macOS／iOS和Linux两个系统下测试你的代码将是一个很好的主意。

#### 在Linux找到测试
任何的Unit Testing框架都应该能发现那些它要运行的测试。在Apple平台，XCTest框架使用Objective-C runtime枚举了一个测试Target中所有的测试单元和其中的方法。然而因为Objective-C runtime在Linux下并不可用，同时swift runtime目前还达到同样的功能，因此在Linux下XCTest就需要开发者提供一个明确的测试列表去执行。

#### allTests 属性
这项工作(可以方便的通过Swfit Package Manager创建)的方法是额外的添加一个名为`allTests`的属性到你XCTestCase的每个子类中，一个测试方法和名字的列表作为这个属性的返回值。例如，一个只包含了单个测试方法的类可能是这样的：

```swift
// Tests/BananaKitTests/BananaTests.swift
import XCTest
import BananaKi t

class BananaTests: XCTestCase {
	static var allTests = [
        ("testYellowBananaIsRipe", testYellowBananaIsRipe),
    ]

	func testYellowBananaIsRipe() {
        let banana = Banana(color: .yellow)
        XCTAssertTrue(banana.isRipe)
    }
}
```

#### LinuxMain.swift
包管理器会创建另一个名为`LinuxMain.swift`的文件，在非Apple平台上这个文件将作为测试的运行者。 它包含了一个对`XCTMain(_:)`的调用，在这个调用中你必须列出所有的测试单元：

```swift
// Tests/LinuxMain.swift
import XCTest
@testable import BananaKitTests

XCTMain([
    testCase(BananaTests.allTests),
])
```
#### 手动维护是很容易忘记的
这个方法显然并不完美，因为它需要在两个地方去手动维护：

1. 每次你添加一个新的测试，你都必须在类中`allTests`属性中也添加。
2. 每次你创建一个新的测试单元，你都必须也添加在`LinuxMain.swift`的`XCTMain`中。

这两步都是很容易被忘记的。甚至更糟糕，当你不可避免的忘记了其中一个，并不会有什么明显的错误发生 - 在Linux上你的测试仍然被通过，如果你手动不对比在macOS上和在Linux上被执行的测试数目， 你甚至可能不会注意到有些测试在Linux上并没有被运行。

这种情况在我这里已经出现过很多次了，所以我决定对此做点什么。


