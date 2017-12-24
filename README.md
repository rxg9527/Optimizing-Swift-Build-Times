## 优化 Swift 编译时间
优化 Swift 编译时间的一些建议。

Swift 不停在改进 ❤️。然而目前，对于中大型项目而言，漫长的编译时间仍是一个巨大问题。这个仓库的目的就是搜集相关 Tips，帮你减少项目的编译时间。

👷🏻 维护者：[Arek Holko](https://twitter.com/arekholko). 少了什么? 欢迎提 Issues 或者 PR!

（翻译：[Yuen](https://weibo.com/rxg9527)，如果对翻译质量存疑，欢迎提出问题建议）

## Table of Contents

* [1、函数和表达式的类型检测（Type checking of functions and expressions）](#1%E5%87%BD%E6%95%B0%E5%92%8C%E8%A1%A8%E8%BE%BE%E5%BC%8F%E7%9A%84%E7%B1%BB%E5%9E%8B%E6%A3%80%E6%B5%8Btype-checking-of-functions-and-expressions)
* [2、编译缓慢的那些文件（Slowly compiling files）](#2%E7%BC%96%E8%AF%91%E7%BC%93%E6%85%A2%E7%9A%84%E9%82%A3%E4%BA%9B%E6%96%87%E4%BB%B6slowly-compiling-files)
* [3、Debug 时只编译 active 架构（Build active architecture only）](#3debug-%E6%97%B6%E5%8F%AA%E7%BC%96%E8%AF%91-active-%E6%9E%B6%E6%9E%84build-active-architecture-only)
* [4、生成 dSYM（dSYM generation）](#4%E7%94%9F%E6%88%90-dsymdsym-generation)
* [5、Module 优化（Whole Module Optimization）](#5module-%E4%BC%98%E5%8C%96whole-module-optimization)
* [6、CocoaPods 的 Module 优化（Whole Module Optimization for CocoaPods）](#6cocoapods-%E7%9A%84-module-%E4%BC%98%E5%8C%96whole-module-optimization-for-cocoapods)
* [7、第三方依赖（Third\-party dependencies）](#7%E7%AC%AC%E4%B8%89%E6%96%B9%E4%BE%9D%E8%B5%96third-party-dependencies)
* [8、模块化（Modularization）](#8%E6%A8%A1%E5%9D%97%E5%8C%96modularization)
* [9、XIBs](#9xibs)
* [10、Xcode Schemes](#10xcode-schemes)
  * [App](#app)
  * [App \- 单元测试工作流（App \- Unit Test Flow）](#app---%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E5%B7%A5%E4%BD%9C%E6%B5%81app---unit-test-flow)
  * [App \- 所有测试工作流（App \- All Tests Flow）](#app---%E6%89%80%E6%9C%89%E6%B5%8B%E8%AF%95%E5%B7%A5%E4%BD%9C%E6%B5%81app---all-tests-flow)
* [11、使用全新的 Xcode 编译系统（Use the new Xcode build system）](#11%E4%BD%BF%E7%94%A8%E5%85%A8%E6%96%B0%E7%9A%84-xcode-%E7%BC%96%E8%AF%91%E7%B3%BB%E7%BB%9Fuse-the-new-xcode-build-system)
* [12、启用 Concurrent Swift Build Tasks(Enable Concurrent Swift Build Tasks)](#12%E5%90%AF%E7%94%A8-concurrent-swift-build-tasksenable-concurrent-swift-build-tasks)
* [13、在 Xcode 中显示编译时间（Showing build times in Xcode）](#13%E5%9C%A8-xcode-%E4%B8%AD%E6%98%BE%E7%A4%BA%E7%BC%96%E8%AF%91%E6%97%B6%E9%97%B4showing-build-times-in-xcode)


## 1、函数和表达式的类型检测（Type checking of functions and expressions）

在 **Build Settings** --> **Other Swift Flags** 中加入 

* `-Xfrontend -warn-long-function-bodies=100` (`100` 意味着 100 毫秒, 这个数字具体设置多少要依电脑配置和项目大小而定，建议多调整几遍找到大小相对合适的数字，ps: 1 秒= 1000 毫秒)
* `-Xfrontend -warn-long-expression-type-checking=100`

编译后会看到如下图所示的场景：
![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-15140539609818.jpg)

📖 Sources:

* [Guarding Against Long Compiles](http://khanlou.com/2016/12/guarding-against-long-compiles/)
* [Measuring Swift compile times in Xcode 9 · Jesse Squires](https://www.jessesquires.com/blog/measuring-compile-times-xcode9/)
* [Improving Swift compile times — Swift by Sundell](https://www.swiftbysundell.com/posts/improving-swift-compile-times)
* [Swift build time optimizations — Part 2](https://medium.com/swift-programming/swift-build-time-optimizations-part-2-37b0a7514cbe)

## 2、编译缓慢的那些文件（Slowly compiling files）
上一段针对的是函数级别和表达式级别，现在我们关注整个文件的编译时间 
因为这里我们必须用 CLI 命令行界面来编译项目，所以务必将参数设置正确了：

```bash
xcodebuild -destination 'platform=iOS Simulator,name=iPhone 8' \
  -sdk iphonesimulator -project YourProject.xcodeproj \
  -scheme YourScheme -configuration Debug \
  clean build \
  OTHER_SWIFT_FLAGS="-driver-time-compilation \
    -Xfrontend -debug-time-function-bodies \
    -Xfrontend -debug-time-compilation" | \
tee profile.log
```

上述代码针对的是 `.xcodeproj` 的项目，如果项目是 `.xcworkspace` 的话，把上面第二代码中的 
`-project YourProject.xcodeproj` 替换成为 `-workspace YourProject.xcworkspace`。

然后从之前生成的 `profile.log` 文件中提取出编译所需时间。

```shell
awk '/Driver Compilation Time/,/Total$/ { print }' profile.log | \
  grep compile | \
  cut -c 55- | \
  sed -e 's/^ *//;s/ (.*%)  compile / /;s/ [^ ]*Bridging-Header.h$//' | \
  sed -e "s|$(pwd)/||" | \
  sort -ReactNative | \
  tee slowest.log
```

成功执行之后，就会得到一个 `slowest.log` 文件，内容格式如下：

```shell
2.7288 (  0.3%)  {compile: Account.o <= Account.swift }
2.7221 (  0.3%)  {compile: MessageTag.o <= MessageTag.swift }
2.7089 (  0.3%)  {compile: EdgeShadowLayer.o <= EdgeShadowLayer.swift }
2.4605 (  0.3%)  {compile: SlideInPresentationAnimator.o <= SlideInPresentationAnimator.swift }
```

📖 Sources:

* [Diving into Swift compiler performance](https://koke.me/2017/03/24/diving-into-swift-compiler-performance/)


## 3、Debug 时只编译 active 架构（Build active architecture only）
默认就是这个设置，不过安全起见，可以去在 **Build Settings** --> **Build active architecture only** 确认一下
![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-15140549939966.jpg)

📖 Sources:

* [What is Build Active Architecture Only](http://samwize.com/2015/01/14/what-is-build-active-architecture-only/)

## 4、生成 dSYM（dSYM generation）
新项目的默认设置是，Debug 配置编译时不生成 dSYM 文件。但有时候为了在开发时进行 Crash 日志解析，会去修改这个参数。生成 dSYM 会消耗大量时间，可以去确认一下。

📖 Sources:

* [Speeding up Development Build Times With Conditional dSYM Generation](http://holko.pl/2016/10/18/dsym-debug/)

## 5、Module 优化（Whole Module Optimization）
另一个公认的方法是

* 修改 Debug 配置 `Build Settings`  --> `Optimization Level` 为 `Fast, Whole Module Optimization` 
* **只在 Debug 配置**添加 `-Onone` flag 到 `Build Settings`  -->  `Other Swift Flags` 
![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-wmo_9@2x.png)

这告诉了编译器哪些东西？（译者注：这里感觉自己的翻译不到位，放原文吧）

> It runs one compiler job with all source files in a module instead of one job per source file
> 
> Less parallelism but also less duplicated work
> 
> It's a bug that it's faster; we need to do less duplicated work. Improving this is a goal going forward

📖 Sources:

* [Developear - Speeding Up Compile Times of Swift Projects](http://developear.com/blog/2016/12/30/Speed-Swift-Compilation.html)
* [Slava Pestov on Twitter: “@iamkevb It runs one compiler job with all source files in a module instead of one job per source file”](https://twitter.com/slava_pestov/status/911747257103302656)


## 6、CocoaPods 的 Module 优化（Whole Module Optimization for CocoaPods）
如果使用 CocoaPods 的话，上一步 WMO 的配置也要考虑放到使用的 CocoaPods 项目中。
要实现 WMO 配置，需要在项目的 `Podfile` 文件中添加如下的 `post_install` hook：

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      if config.name == 'Debug'
        config.build_settings['OTHER_SWIFT_FLAGS'] = ['$(inherited)', '-Onone']
        config.build_settings['SWIFT_OPTIMIZATION_LEVEL'] = '-Owholemodule'
      end
    end
  end
end
```

## 7、第三方依赖（Third-party dependencies）
在项目中有两种途径可以引入第三方依赖：

1. 源码形式，在你每次 clean 项目编译内容的时候都会都会重新编译，例如：CocoaPods，git submodules, 复制粘贴的 code,  应用 target 依赖的子项目（`subprojects`）形式
2. 提前编译好的 framework/library形式，例如：Carthage, 第三方不透露源码、提供的静态库

因此在你每次 clean build 之后，CocoaPods 大多情况下会重新编译，导致大量的编译时间花销。

Carthage 虽然难用一些，但当你很在意编译时间的话却不失为一个好选择。只有当你在更新依赖列表（添加新 framework，更新 framework 的新版本）的时候，你才会去编译外部依赖。Carthage 也许需要你花 5-15 分钟的时间去完成，但相比使用 CocoaPods 做依赖，单从长远的编译时间花销来看，Carthage 或许会节省不少时间。

📖 Sources:

* time spent waiting for Xcode to finish builds 😅


## 8、模块化（Modularization）
Swift 的增量编译并不完美。有时一些增量编译中，也许仅仅是修改了一个字符串，就会导致整个项目重新编译。这是一个亟待解决的问题。

为了避免这个问题，你可以考虑将 app 拆分为一个个模块。在 iOS 里，有2中方案：动态库和静态库（Xcode9 Beta4 版本开始支持 Swift 静态库）。

假设你的 app 依赖一个叫做 `DatabaseKit` 的内部 framework。模块化的方法能够保证在你对 app 项目做了一些修改时，`DatabaseKit` 不会因为这个增量编译的行为而重新编译。

📖 Sources:

* [Technical Note TN2435 – Embedding Frameworks In An App](https://developer.apple.com/library/content/technotes/tn2435/_index.html)
* [uFeatures](https://github.com/microfeatures/guidelines)


## 9、XIBs
代码 和 XIBs/Storyboards 的取舍一直是个热门话题。不过我们这里只是片面的谈一谈这个问题。这个情况很有趣：当你在 IB 中做了一些改动时，只有这个 IB 文件被编译了（成为 NIB 格式）;与之成鲜明反差的是，有时候当你在 UIView 的一个子类中修改了一行代码，Swift 编译器可能会重新编译项目中的很大一部分代码。

📖 Sources:

* [(…) in a large project incremental build is much faster if only a .xib was changed (vs. only a line of Swift UI code)](https://twitter.com/MichalCiuba/status/925326831074643968)


## 10、Xcode Schemes
假设我们有一个常见设置的项目，它有3个 target：

* `App`
* `AppTests`
* `AppUITests`

只在一个 scheme 上工作没有问题，但是我们还可以优化。下方的配置是我们一直在使用的，包了3个 scheme：

### App

⌘+B 只会编译这个 app。只跑单元测试不跑 UI 测试。 对于 short iterations 很有用, 比如在 UI 代码上, 因为只有需要用到的代码被编译了。

![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-app-scheme@2x.png)

### App - 单元测试工作流（App - Unit Test Flow）

编译内容会包含这个 app 和单元测试 target。运行的话，只会跑单元测试。在涉及到单元测试的代码时很有用，因为只要一编译完项目，就能立即发现在测试里的编译错误了。甚至不需要去运行他们。

当你的 UI 测试需要话费很久的时候，这个Scheme很有用。

![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-app-unit-test-flow@2x.png)

### App - 所有测试工作流（App - All Tests Flow）

编译应用和所有 target。跑所有测试用例。当涉及到那些和 UI 关系紧密包含 UI 测试的代码时作用较大。

![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-app-all-tests-flow@2x.png)

📖 Sources:

* [All About Schemes](http://pilky.me/17/)


## 11、使用全新的 Xcode 编译系统（Use the new Xcode build system）

在 Xcode 9 苹果 [介绍了一种新的 Xcode 编译系统](https://developer.apple.com/library/content/releasenotes/DeveloperTools/RN-Xcode/Chapters/Introduction.html#//apple_ref/doc/uid/TP40001051-CH1-SW878). 这还只是“预览”版本，默认并没有开启。这个新系统比默认的原有的编译系统快的多。
如果想要使用它，到 Xcode 的 `File` 菜单进入`Workspace 或 Project Settings` 的页面，就可以切换到新的编译系统了。
![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-15140594214374.jpg)
📖 Sources:

* [Faster Swift Builds with the New Xcode Build System](https://github.com/quellish/XcodeNewBuildSystem)


## 12、启用 Concurrent Swift Build Tasks(Enable Concurrent Swift Build Tasks)

Xcode 9.2 对于增加 Swift 项目的 `concurrent build tasks` 有一个实验性质的支持。对于一些项目，这个特性可能会大大改善编译时间。另外要注意，当启用这个选项的时候，Xcode 可能会增加大量内存开销。

如果要启用这一特性，退出 Xcode，然后在 Terminal 窗口中输入以下命令：

```
$defaults write com.apple.dt.Xcode BuildSystemScheduleInherentlyParallelCommandsExclusively -bool NO
```

测试以下这个改动对你的项目编译产生了什么影响。对于很多项目来说，这不会导致什么变化；但是对另外一些，这个改动非常重要。如果要弃用这个特性的话，在 Terminal 窗口中输入以下命令，然后重启 Xcode：

```
$defaults delete com.apple.dt.Xcode BuildSystemScheduleInherentlyParallelCommandsExclusively
```

（译者注：为什么是设为 NO 而不是 YES，开发者在 [推特上](https://twitter.com/rballard/status/938216933303885824) 说是当时写错了😂。值得注意的是，根据[这里](https://twitter.com/GalvinLi/status/938252981123801088)所说，开启这个新特性的操作，只会影响原有的默认编译系统 `standard build system`；新的 build system 已经启用这一特性。


📖 Sources:

* [Even faster Swift build times with Xcode 9.2](https://github.com/quellish/XcodeConcurrentSwiftBuilds)

## 13、在 Xcode 中显示编译时间（Showing build times in Xcode）

最后，为了能够准确知道你的编译时间是否改善了，你应该在 Xcode 的 GUI 界面中显示时间。在命令行中运行：

```
$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration -bool YES
```

成功之后，在 `⌘+B` 编译一个项目后你会看到：
![](http://o7y6vb76p.bkt.clouddn.com/2017-12-24-time@2x.png)

建议每次都在一个相对公平的环境下比较编译时间，比如：

1. 退出 Xcode
2. 清理 Derived Data (`$ rm -rf ~/Library/Developer/Xcode/DerivedData`)
3. 打开 Xcode 中的项目
4. 在 Xcode 打开或者完成索引阶段（indexing phase）后，立刻开始编译。因为使用 Xcode 9 编译也会执行索引，所以 `The first approach` 看起来更具代表性。（The first approach seems to be more representative because starting with Xcode 9 building also performs indexing）

另外，你也可以利用命令行统计编译时间:

```
$ time xcodebuild other params
```

📖 Sources:

* [How to enable build timing in Xcode? - Stack Overflow](https://stackoverflow.com/a/2801156/1990236)

