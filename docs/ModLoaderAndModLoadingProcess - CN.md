ModLoader 的运行原理及 Mod 的加载流程

# 1. 前提

SugarCube2 是一个全同步的渲染引擎，它以完全同步（没有任何异步操作）的方式将游戏脚本（例如 twee）动态翻译（组装）为 html，并显示出来。

而游戏脚本（twee、JS、css）是以文本的方式内嵌在编译完成的网页 html 中的（tw-storydata 节点）。

为了实现对游戏的修改，我们有几种方法：

1. 在 SugarCube2 将游戏脚本编译为 html 之前修改网页 html 中的 tw-storydata 节点里的游戏脚本，来让 SugarCube2 认为我们的游戏本来就是这样的。
2. 在 SugarCube2 翻译游戏脚本的过程中直接参与编译过程，动态改变编译的输入和编译的输出。
3. 在 SugarCube2 将游戏编译成 html 并显示在网页上之后，直接修改显示的网页内容，让用户觉得游戏本来就是这样的。

对于以上三种方法，对应了三种实现：

1. ModLoader 为了兼容过去的直接修改 html 制作 mod 的方法，采用了直接修改网页 html 中的 tw-storydata 节点里的游戏脚本，使得 mod 安装后内存中的脚本数据格式和直接修改网页后的格式一摸一样。这个功能以[TweeReplacer]()和[ReplacePatch]()两个 addon mod 及对应衍生 addon mod 的形式从 ModLoader 中导出给 mod 作者使用。
2. 通过对 SugarCube2 的 Wikifier 的一些侵入性修改，ModLoader 实现了读取、挂钩并拦截编译引擎的输入输出，提供能动态改变编译的输入和输出的可能。这个功能以[TweePrefixPostfixAddonMod]()这个 addon mod 的形式导出给 mod 作者使用。
3. 由于 SugarCube2 自带渲染结束的 JQuery 事件消息，通过监听 SugarCube2 的 passage 渲染结束消息，我们可以准确得知 SugarCube2 渲染结束的时间点，进而可以通过监听这个消息来实现在 passage 显示后立即修改 html 内容的目的。对此，emicoto 实现了一个名为[Simple Framework（简易框架）](https://github.com/emicoto/DOLMods)的工具来绕过 DoL 本身复杂且混乱的架构，直接在 html 中插入显示内容。

---

为了关注于 ModLoader 的运行原理及 Mod 的加载流程，接下来的介绍着重涉及方法 1 及其相关内容

# 2. ModLoader 的运行原理及 Mod 的加载流程

## 2.1 ModLoader 如何引导

由于 SugarCube2 是一个全同步的渲染引擎，为了实现在 SugarCube2 之前修改网页中的 tw-storydata 节点里的游戏脚本，我们需要抢在启动之前执行 ModLoader 的所有游戏启动前加载工作。

在详细阅读 sugarcube2 的源代码后，我们可以发现，sugarcube2 的启动代码位于 `sugarcube.js#L111` 的一个 `jQuery(() => {})` 闭包函数中在网页加载完成后启动，那么也就意味着，如果能在这个闭包中进行一些修改，插入我们的启动脚本，那么我们就可以让 ModLoader 在 SugarCube2 启动前启动。
而如果审视 ModLoader 的设计需求，我们会发现，ModLoader 需要执行大量的异步操作，其中涉及到从远程加载 mod、从 localStorage/indexDB 中读取 mod 的 zip 文件、从 zip 文件中读取 mod 信息，等等等等。
故，我们需要在 SugarCube2 的启动代码前插入一个可以让我们执行异步代码的时机。在审核 SugarCube2 和 jQuery 的源码后我们会发现，唯一的且最可靠的方法就是添加一个 Promise 并将 SugarCube2 原本的启动代码包装起来，使得 SugarCube2 的启动代码可以等待我们的说偶有异步操作执行完成。

## 2.2 ModLoader 如何初始化及自身的启动

我们在 SugarCube2 启动前执行 Modloader 的 [startInit()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/SC2DataManager.ts#L247) 函数，
并开始初始化 ModLoader。

首先我们保存最原始的未经修改的网页 html 中的 tw-storydata 节点中的所有内容。[initSC2DataInfoCache()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/SC2DataManager.ts#L259)

由于 startInit()是 SC2DataManager 中的成员函数，这样意味着在此同时会初始化 SC2DataManager 中的[所有内部对象和功能性插件](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/SC2DataManager.ts#L25)。
其中包括所有由 ModLoader 实现并开放给 mod 高级作者使用的所有功能。

完成以上初始化过程后就开始最重要的 Mod 加载过程。

## 2.3 Mod 加载执行过程 和 ModLoader 的启动过程

由`startInit()`调用[ModLoader.loadMod()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/ModLoader.ts#L307)，
开始执行 Mod 的加载过程。

Mod 的加载总的来说涉及以下几个步骤：

1. 从某个地方读取 Mod 的 zip 文件。
2. 执行`scriptFileList_inject_early`和`scriptFileList_earlyload`并在此同时执行复杂的加载触发逻辑。
3. 注册 Mod 到 Addon
4. 重建 tw-storydata 节点
5. 执行`scriptFileList_preload`
6. 启动 SugarCube2 正常执行过程

详细地过程如下：

1. 按照 Html 内嵌位置、远程服务器、LocalStorage、IndexDB 的顺序加载 mod。在此同时调用`DependenceChecker.checkFor()`执行依赖检查。
2. 使用 [ModZipReader](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/ModZipReader.ts#L50) 读取 Mod 中的`boot.json`文件，来了解接下来需要为这个 mod 做些什么。
3. 调用[initModInjectEarlyLoadInDomScript()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/ModLoader.ts#L465)将所有`scriptFileList_inject_early`的 js 文件直接注入到 Html 中，Mod 应该将对自身的初始化工作在此处完成，但需注意的是，此处只能执行同步操作（不会等待异步操作完成）。在这个过程中会涉及到检查 Mod 是否可以加载的工作，此工作由“某个 Mod”注册的 `ModLoadControllerCallback.canLoadThisMod` 钩子来完成。例如安全模式就由处在第一个加载的名为 ModLoaderGui 的 Mod 采用这个钩子实现。
4. 触发`AddonPluginHookPoint.afterInjectEarlyLoad`、`ModLoadControllerCallback.afterModLoad`、`AddonPluginHookPoint.afterModLoad` 钩子，告知所有 Mod，当前 Mod 已经加载。如果某些 Mod 需要在非常非常早期执行操作，就可以此钩子处进行，此处的钩子调用会等待返回的异步操作完成，如果存在需要异步初始化的工作可以在此处进行。
5. 调用 [initModEarlyLoadScript()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/ModLoader.ts#L517) 执行所有 `scriptFileList_earlyload` 中的*单行指令*。要特别注意的是，此处使用的是 [JsPreloader.JsRunner()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/JsPreloader.ts#L117)，这个执行器的真实实现是将原始的 js 文件中的代码包装到一个形如 `(async () => {return ${jsCode}\n})()` 的函数中，并等待函数返回的异步调用结束，这个代码由于会在整个文件的第一行开头添加一个`return`指令，按照 JS 的`return`的语义，此处只会执行 js 文件中第一行的代码，或者从第一行开始的闭包函数。
6. 在调用 `initModEarlyLoadScript()` 的过程中，会不断调用 [tryInitWaitingLazyLoadMod()]() 来尝试检查当前是否有 Mod 追加了需要懒加载的 Mod，并加载这些懒加载 Mod。对于加密 mod 的实现就是使用了懒加载 mod 的特性，在此处解密并释放被加载 mod。
7. 要特别注意的是，懒加载的 Mod 由于在此处才会读取到 zip 文件，故懒加载的 mod 的`scriptFileList_inject_early`和`scriptFileList_earlyload`会在[此处](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/ModLoader.ts#L745)同时执行。且在此过程中会不断触发`canLoadThisMod`钩子。
8. 在完成以上的 Mod 本体的 JS 脚本的加载和执行工作后，会触发`AddonPluginHookPoint.afterEarlyLoad`钩子
9. 调用 [registerMod2Addon()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/ModLoader.ts#L384) 来将所有在`boot.json`中声明使用了`addonPlugin` 的 Mod 都注册到对应的 Addon Mod 去。（这些 Addon Mod 必须在此之前调用 `AddonPluginManager.registerAddonPlugin` 将自己注册为一个 Addon Mod）。
10. 此时 Addon Mod 会从 `AddonPluginHookPointExMustImplement.registerMod` 回调函数钩子上收到 mod 的注册回调，此时 Addon Mod 就可以根据自己的设计功能来完成记录或者执行操作等等。
11. 触发 `AddonPluginHookPoint.afterRegisterMod2Addon` 钩子
12. 以上便完成了 Mod 的 Js 功能的加载
13. 触发 `AddonPluginHookPoint.beforePatchModToGame` 钩子
14. 开始合并`styleFileList`、`scriptFileList`、`tweeFileList`的数据到 tw-storydata 节点中，重建 tw-storydata 节点
15. 触发 `AddonPluginHookPoint.afterPatchModToGame` 钩子，Mod 可以在此处修改合并后的游戏脚本数据，`TweeReplacer`、`ReplacePatch` 等 Mod 就是在此处执行替换计算
16. `ModLoader.loadMod()`执行结束，返回 SugarCube2 的代码
17. 再次由 SugarCube2 的代码调用[JsPreloader.startLoad()](https://github.com/Lyoko-Jeremie/sugarcube-2-ModLoader/blob/ac0bb6c59abd93a2a784f2a574f031861bcf269f/src/BeforeSC2/JsPreloader.ts#L51)
18. 执行`scriptFileList_preload`中的文件
19. 触发`AddonPluginHookPoint.afterPreload`
20. 触发`ModLoadControllerCallback.ModLoaderLoadEnd`回调，这是 ModLoader 加载过程中的最后一个回调钩子事件，Mod 可以在此处完成 SugarCube2 启动前最后的收尾工作。如果某个操作没有选择地必须在其他 Mod 最后执行操作，也可选择在此处进行。
21. Mod 加载全部完成，ModLoader 启动完毕，开始启动 SugarCube2 的正常运行流程。接下来 ModLoader 的动作全部由 SugarCube2 触发。

# 3. ModLoader 定制版 SugarCube2 对原版 (DoL 版) 的修改

1. 修改了 SugarCube2 启动点
2. 对 Wikifier ，添加了 `_lastPassageQ`以及对应的数据和操作来跟踪整个脚本的编译过程，目的是跟踪和修改各个编译层级。此处变更涉及到所有接触到编译的各个地方，主要涉及到`macrolib.js`、`parserlib.js`、`wikifier.js`。可以使用`passageObj`为关键字进行查找。
3. 对 img 标签和 svg 标签的拦截，以此实现完全无服务器从内存加载所有图片的目的。
