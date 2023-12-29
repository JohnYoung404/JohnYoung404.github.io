---
layout:     post
title:      Introduction to Neo
subtitle:   一个兴趣使然的C/S双端游戏框架
date:       2023-11-20
author:     John Young
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Unity
    - Game Framework
    - Game Server
---

## 前言

哈喽，好久没有更新博客了!
这次是介绍一下我的兴趣项目：一个Unity客户端 + \.net服务器的游戏框架： **Neo** <br/>
Neo暂未开源（主要是考虑到很多第三方的版权问题）
但是我会在这篇文章中介绍详细的技术路线以及demo展示，有问题也可以留言联系我！让我们开始吧~

### 1. 总体架构

<!--- 去掉“--\>”里的反斜杠时mermaid diagram生效：
```mermaid
graph TD
A(Unity App) --\> B(Shared Core)
C(Server Core) --\> B(Shared Core)
D(Console-End Server) --\> C
E(Unity-End Server) --\> C

B --\> Network
B --\> SD(Shared Data)
B --\> BT(Behaviour Tree)
B --\> IE(Inventory Engine)
B --\> Log
B --\> ETC(...)
```
-->

Neo是我这两三年来的一个兴趣项目，我的初衷是在这个项目里集成各种好用的第三方技术和第三方插件，让它变成一个“百宝箱”，能在我遇到问题时拿来即用。随着不停地迭代和进化，Neo的基础功能越来越完善，越来越好用，某些模块甚至好过我参与的商业项目。我想主要原因可能是这是由我一个人开发的，思路不会有团队协作时的混乱感，比较方便做相性更好的技术选型。因此我渐渐想把它往更完善的方向去发展，看看自己到底能走多远。
![](https://johnyoung404.github.io/img/Neo/framework.jpg)

正如上图所示，Neo是C/S双端架构，客户端和服务器都使用C#语言开发。这样设计有一个好处，就是客户端和服务器很容易就可以共用代码。在Neo里，我把共用代码放到一个叫作`Shared Core`的csproj，Unity客户端和`Server Core`都依赖于这个工程生成的dll。在`Shared Core`里有网络框架、背包引擎、C-S共用数据模型、日志、行为树、状态机、Math等模块。<br/>
我将服务端的部分抽出成一个`Server Core`的csproj，然后构建目标也是一个dll，最后ConsoleApp去引用这个dll。这样做的目的是，将来如果有在Unity端运行本地服务器的需求，可以很方便地进行迁移。

### 2.技术细节

#### 2.1 共享程序集：Shared Core

共享的程序集是客户端和服务器都引用到的，因此只能有纯C#逻辑，而不能使用Unity引擎的API。设计理念上，它的子模块需要遵循依赖倒置原则（DIP），子模块的运行不能依赖于外部的实例类，而是依赖于接口，通过事件与调用者通信。它的子模块如下：

* **Log**：客户端或服务器所有日志都通过这个模块进行，会在当前工作目录下根据日期和pid创建日志文件，写入日志会带有时间戳、调用模块、日志等级等信息。对于Unity来说，需要通过回调与Unity.Debug.Log桥接起来；对于服务器控制台程序，需要通过回调与Console.WriteLine桥接起来。这样Unity的控制台和服务端就都能显示日志了。我们的服务端日志大概长这样：
![](https://johnyoung404.github.io/img/Neo/login.png)

* **Util**：有一些基础设施适合放在shared模块，这样就不需要同时维护两份。例如：SingletonBase（单例基类）、ListExtension（列表的洗牌、随机采样等）、StringExtension（用正则表达式进行用户名/密码校验等）、Math（浮点数取整、lerp、定点数等）、Random（掷骰子、圆面随机、球面随机）、PerformanceGuard（用RAII的方式监控关键路径的运行时间）、CalculationEngine（字符串公式计算，Neo使用的是第三方库Jace\.Net）、EventCore（事件系统）

* **Shared Data**：客户端和服务器在数据定义上，也有一些共有部分。例如：客户端和服务端的共用配置表、客户端与服务端协议、客户端与服务端共用的DataModel。关于这一部分，后续的导表工具与网络模块会介绍更多细节。

* **Reflection**：擅用反射可以将你从繁琐重复的各种Register中解放出来。Neo中广泛使用反射进行事件监听。例如，服务端程序集里所有标记了`[REPLCmd]`(Read-Evaluate-Print-Loop)的static方法，都可以通过控制台的用户输入调用；Shared程序集里所有标记了`[UnitTest]`的static方法，都会在收到单元测试指令时运行一次；所有网络协议的处理函数注册，是通过反射进行的，例如登录协议的处理函数，需要标记`[ProtocolHandler(typeof(Req_Login))]`；服务端收到客户端的GM协议时，会查找对应的标记`[GMCmd]`的函数并分析函数签名，决定是否执行。<br/>
下面这个例子展示了反射模块的单元测试：
![](https://johnyoung404.github.io/img/Neo/reflection.jpg)

* **BehaviourTree**：最初的时候，我只把行为树模块放在了客户端，并基于Unity的GraphView写了行为树的图形编辑工具。但是在后来的迭代中，我受到了[《守望先锋》的GDC演讲](https://www.youtube.com/watch?v=odSBJ49rzDo)的启发，觉得行为树做成客户端与服务器共用的话，可能会有更多应用场景。《守望先锋》的gameplay脚本State Script，在我看来和一个带有变量系统的行为树差不多，而State Script是在客户端和服务器各运行一份的，带有预测、回滚等等功能。因此，我心中最理想的行为树框架，也是客户端、服务器共同运行的，同时允许一些表现逻辑只在客户端运行；客户端与服务端的行为树状态可以通过协议同步。这样的话，行为树不仅仅可以做AI逻辑，也可以用来做gameplay逻辑。<br/>
这是Neo中行为树的编辑界面：
![](https://johnyoung404.github.io/img/Neo/bt.jpg)
服务端运行结果（打印 -> 等待3秒 -> 循环5次，每次90%概率打印）：
![](https://johnyoung404.github.io/img/Neo/bt_svr.jpg)

* **InventoryEngine**：对于Shared模块的背包引擎，它提供一个类似于《暗黑破坏神》（diablo-like）的背包系统。它最主要的功能是通过外部数据创建一个背包引擎，判断各种item操作的合法性（特殊要求和通用的位置要求，即两个item位置不能重叠，item不能放到背包之外等等），然后给出结果。出于解耦的需要，item的特殊合法性判断需要通过委托去做（例如：是否可以进背包？、是否可以丢弃？、是否可以堆叠？、是否可以交换位置？、是否可以使用？），同时会将操作结果通过事件发出去（例如`onItemAdd`，`onItemRemove`，`onItemAmountChange`，`onItemSwap`，`onItemDrop`，`onItemUse`）。对于客户端与服务器的外部系统如何使用这个模块，我会在第3部分的实例部分进行更多讲解。

* **其他**：我认为寻路也是适合做在Shared模块的，虽然目前Neo暂时没有。如果做的话，技术路线我会选择基于开源的[Recast](https://github.com/benjamn/recast)。Neo还有一个GridEngine也放在Shared模块，它主要是做基于格子的（例如战棋类游戏）的寻路、范围查找等等。其实只要是不依赖于引擎并且CS双方都可以使用的功能，都是可以考虑进行解耦然后放在Shared模块里的。

#### 2.2 Unity客户端

对于Unity客户端开发，我偏好以package的形式去开发各个模块，这样模块化的处理其实也是一种解耦，大大增加了代码复用的可能性。这一章节，我将主要介绍一下Neo对于客户端基础设施（例如导表、热更新etc）的技术选型。注意，虽然这些第三方的项目都挺优秀，但并不适合即插即用。为了适配Neo的程序集划分以及目录结构，我花了一些工夫去写一些工具脚本，以及调整了目录结构。

* **配置工具**：配置工具我选用的是[luban导表工具](https://github.com/focus-creative-games/luban)，它的功能完善：支持多种编程语言，支持本地化，支持多种导出格式，支持枚举、结构体定义。在Neo中，不仅仅使用luban导出配置表，也使用它导出协议。<br/>
例如Neo中装备的定义如下，其中枚举类型使用了中文别名，导出后会是枚举值：
![](https://johnyoung404.github.io/img/Neo/config.jpg)
再比如Neo中，抽卡协议的定义如下，定义了一个Rpc，以及其arguments和return的结构：
![](https://johnyoung404.github.io/img/Neo/gacha.jpg)

* **热更新**：热更新是一个有趣的话题，我在[这篇blog](https://johnyoung404.github.io/2020/11/17/Unity%E8%B7%A8%E5%B9%B3%E5%8F%B0/)里提到过一些关于Unity的跨平台、热更的知识。基本所有热更框架都是通过内嵌某种语言的虚拟机来解释执行（JIT）从而做到热更的。相信Unity程序员一定接触过各种lua：xlua、tolua、ulua、slua；除了lua，我在网易供职的两年里，还基于网易的in-house引擎NeoX开发过两年的python（苦笑）。<br/>
热更新确实是一个很实际的需求，but at what cost？至少从我好几年的弱类型语言的体验来说，我不觉得lua、python这些语言适合开发所有的游戏逻辑，尤其是异步逻辑以及偏底层的逻辑。使用C#这样强类型语言，效率优势是一方面，表达能力优势是另一方面。例如lua下要写复杂一些的异步逻辑，很可能需要用promise这样的编程范式，如果用回调的话，极易掉入回调地狱(callback hell)，即使是这样，debug起来也是十分痛苦；而在C#中，使用async/await的编程范式能更好的处理异步的情况，也容易处理cancellation或者exception。<br/>
因此我倾向于可以直接使用C#语言的热更框架（例如早先流行的ILRuntime），并且最好没有过多语言层面的限制，可以像原生C#一样去开发。<br/>
最后我选择了[HybridCLR](https://github.com/focus-creative-games/hybridclr)这个框架，花了较多的功夫，踩了相当多的坑之后，整合到了Neo中。不过结果我是非常满意的：`Shared Core`和所有Neo的客户端程序集都是可以热更的，大部分第三方程序集也是可以热更的；Editor模式下正常开发，使用原生dll，不影响debug；PC包则使用打在资源包里的dll，拥有完整的热更功能。

* **资源加载**：这一部分我使用的是[YooAsset](https://github.com/tuyoogame/YooAsset)，在本地用[httpFileServer](https://www.rejetto.com/hfs/)搭建一个文件服务器，就可以测试YooAsset的远程资源包下载。在这里可以介绍一下Neo的游戏启动流程了：Neo有一个引导package，它的代码是不能热更的，它的资源都是built-in打进包的。它的功能就是来做下载的，因此资源量很少，代码也只有资源包下载、热更dll加载等逻辑。<br/>
patch界面长这样：
![](https://johnyoung404.github.io/img/Neo/update.jpg)
下载完最新资源后，会先加载AOT dll的MetaData（HybridCLR用它来生成未出现的泛型类型），再加载普通的热更新dll，然后执行其中的Entry函数。至此之后，所有的逻辑都是可以热更新的了。

* **Debug控制台**：[Quantum Console](https://assetstore.unity.com/packages/tools/utilities/quantum-console-211046)是一个外表非常fancy，功能也非常易用的Unity控制台插件，用来debug非常方便。它也是通过反射扫描dll的方法查找标记了`[QFSW.QC.Command]`的方法，并为它们生成对应的指令，和我们服务端的console有异曲同工之妙。<br/>
这是在登录界面按[`Esc`]键呼出控制台的样子（开发模式下Shared模块的单元测试每次都会跑一遍）：
![](https://johnyoung404.github.io/img/Neo/console.jpg)

* **场景切换**：一个通用的场景切换功能，灵感来自于一个原型制作插件[Feel](https://feel.moremountains.com/)。这个场景切换是一个异步过程：1.用Additive的方式加载过渡场景；2.播放过渡场景的淡入动画与加载circle的动画；3.卸载原场景；4.通过YooAsset的异步加载接口加载目标场景并激活场景；5.播放过渡场景的淡出动画；6.卸载过渡场景。<br/>
从登录场景切换到大厅的过渡效果：
![](https://johnyoung404.github.io/img/Neo/scene_loading.gif)

* **UI框架**：

* **其他模块**：

#### 2.3 .net服务端

* **DB**
* **Bot**
* **Gacha**

#### 2.4 网络通信

* **Protocol**
* **Rpc**
* **DataModel**

### 3.一个实例-背包系统

![](https://johnyoung404.github.io/img/Neo/item_detail.png)