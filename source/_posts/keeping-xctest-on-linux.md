---
title: 在Linux上保持XCtest同步
date: 2017-04-09 11:41:01
tags:
---

作者 Ole Begemann

[原文链接](https://oleb.net/blog/2017/03/keeping-xctest-in-sync/)

[Swfit是跨平台的](https://swift.org/about/#platform-support)，但在Apple平台中和在其他系统中运作方式并不一致，主要因为两个原因：

* [Objective-C runtime](https://developer.apple.com/reference/objectivec/objective_c_runtime) 仅适用于Apple平台
* 在非Apple系统中，[Foundation](https://github.com/apple/swift-corelibs-foundation)框架和其他的[核心库](https://swift.org/core-libraries/)的实现方法是不同的。 这就意味着有些Foundation API可能会在macOS／iOS和Linux下产生不同的结果(虽然既定目标是无差异的实现)，甚至是还不能被完整执行。

因此，如果你写一个并非仅仅在Apple上执行的库，那么在macOS／iOS下和在Linux下测试你的代码将是一个很好的主意。

#### 在Linux下发现测试
任何的Unit Testing框架都必须要能发现那些它要运行的测试。在Apple平台，[XCTest框架](https://developer.apple.com/reference/xctest)使用Objective-C runtime枚举了一个测试目标(Target)中所有的测试单元和其中的方法。然而因为Objective-C runtime在Linux下并不可用，同时swift runtime目前还达到同样的功能，因此在Linux下XCTest就[需要开发者提供一个明确的测试列表](https://github.com/apple/swift-corelibs-xctest/blob/master/Documentation/Linux.md#additional-considerations-for-swift-on-linux)去执行。

#### allTests 属性
这项工作(可以方便的通过[Swfit包管理器](https://swift.org/package-manager/)创建)的方法是额外的添加一个名为`allTests`的属性到你[XCTestCase](https://developer.apple.com/reference/xctest/xctestcase)的每个子类中，一个测试方法和名字的列表作为这个属性的返回值。例如，一个只包含了单个测试方法的类可能是这样的：

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
包管理器会创建另一个名为`LinuxMain.swift`的文件，在非Apple平台上这个文件将作为测试的运行者(test runner)。 它包含了一个对`XCTMain(_:)`的调用，在这个调用中你必须列出所有的测试单元：

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

这两步都是很容易被忘记的。甚至更糟糕，当你不可避免的忘记了其中一个，并不会有什么明显的错误发生 - 在Linux上你的测试仍然被通过，如果你不手动对比在macOS上和在Linux上被执行的测试数目， 你甚至可能不会注意到有些测试在Linux上并没有被运行。

这种情况在我这里已经出现过很多次了，所以我决定对此做点什么。

#### 保障测试 防止疏忽
让我们建立一个机制，当我们忘了其中一项时，这个机制能自动的使测试单元失败。我们将添加下面的测试到每个`XCTestCase`类中(以及它们的`allTests`属性中)：

```swift
class BananaTests: XCTestCase {
    static var allTests = [
        ("testLinuxTestSuiteIncludesAllTests",
         testLinuxTestSuiteIncludesAllTests),
        // Your other tests here...
    ]

    func testLinuxTestSuiteIncludesAllTests() {
        #if os(macOS) || os(iOS) || os(tvOS) || os(watchOS)
            let thisClass = type(of: self)
            let linuxCount = thisClass.allTests.count
            let darwinCount = Int(thisClass
                .defaultTestSuite().testCaseCount)
            XCTAssertEqual(linuxCount, darwinCount,
                "\(darwinCount - linuxCount) tests are missing from allTests")
        #endif
    }

    // Your other tests here...
}
```
这个测试对比了`allTests`属性数组中的数目和通过Objective-C runtime发现的测试数目，如果两者有差异，那么这个测试将会失败。这正是我们想要的啊！
(依赖Obj-C runtime意味着这个测试只能在Apple平台上工作，在Linux上它将不能被编译，所以我们需要把它写在`#if os(macOS)...`中)

#### 当你忘记添加一个测试到`allTests`中时，测试失败
为了验证这点，让我们添加一个其他的测试，这次“忘记”了去升级`allTests`:

```swift
import XCTest
import BananaKit

class BananaTests: XCTestCase {
    static var allTests = [
        ("testLinuxTestSuiteIncludesAllTests",
         testLinuxTestSuiteIncludesAllTests),
        ("testYellowBananaIsRipe", testYellowBananaIsRipe),
        // testGreenBananaIsNotRipe is missing!
    ]

    // ...

    func testGreenBananaIsNotRipe() {
        let banana = Banana(color: .green)
        XCTAssertFalse(banana.isRipe)
    }
}
```
现在当你运行这个测试在macOS上时，我们的保障测试程序将会失败：

![因为我们忘了添加一个测试到`allTests`数组中，所以测试失败了](keeping-xctest-on-linux/xcode-xctest-linux-safeguard-1550px.png)

我非常喜欢这个。很显然，只有当你希望`allTests`属性数组中包含每个测试时它才有用。也就是说，你将必须像上面所做的那样在条件编译块中包含进去任何针对Darwin或Linux的测试。我相信对于大部分代码库来说这是一个可以接受的限制。

#### 保障 LinuxMain.swift
另一个问题呢，验证`LinuxMain.swift`是否完整？ 这个更难。`LinuxMain.swift`不是(或者说 不能是)当前测试目标(Target)的一部分，所以你很难轻易的验证在`XCTMain`中有些什么。

![当你试着添加`LinuxMain.swift`到测试目标时将会出现错误](keeping-xctest-on-linux/xcode-adding-LinuxMain-to-test-target-1400px.png)

我能想到的唯一的解决办法可能是添加一个*Run Script*到你的测试目标中，用脚本去解析`LinuxMain.swift`中的代码，然后用某种方法去比较测试目标中数组的个数和测试单元的个数。我没有试过，但听起来很复杂。

#### 总结
即使用新的测试，事情也还远远达不到完美，因为还有两个事情你有可能会忘记。每次你创建一个新的XCTestCase类，你必须：

1. 复制粘贴 `testLinuxTestSuiteIncludesAllTests` 测试到新的类中。
2. 更新 `LinuxMain.swift`。

然而，我觉得这已经比之前好很多了， 因为这个新测试涵盖了绝大多数情况 - 在已有的测试目标中添加了一个测试并忘记了更新`allTests`数组。

我并不期待Swift的映射能力能变的强大到使这一切都不再被需要。


