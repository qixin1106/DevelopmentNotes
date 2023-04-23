# Xcode-macOS-Hardened Runtime(强化运行时)选项介绍

强化运行时与系统完整性保护 （SIP） 一起，通过防止某些类别的攻击（如代码注入、动态链接库 （DLL） 劫持和进程内存空间篡改）来保护软件的运行时完整性。若要为 App 启用强化运行时，请在 Xcode 中导航到目标的“签名和功能”信息，然后单击 + 按钮。在出现的窗口中，选择“强化运行时”。

强化运行时不会影响大多数应用的操作，但它不允许某些不太常见的功能，例如实时 （JIT） 编译。如果您的应用依赖于强化运行时限制的功能，请添加授权以禁用单个保护。您可以通过启用 Xcode 中列出的运行时例外或访问权限之一来添加授权。确保仅使用应用功能绝对必要的权利。

您只能将授权添加到可执行文件。共享库、框架和进程内插件继承其主机可执行文件的授权。

这些布尔授权的默认值为 false。当 Xcode 对您的代码进行签名时，仅当值为 true 时，它才会包含授权。如果要手动对代码进行签名，请遵循此约定以确保最大的兼容性。如果值为 false，则不要包含授权。

> 要上传要公证的 macOS 应用程序，您必须启用强化运行时功能。


## Runtime Exceptions

### Allow Execution of JIT-compiled Code Entitlement

一个布尔值，指示应用是否可以使用该标志创建可写内存和可执行内存。MAP_JIT
key：`com.apple.security.cs.allow-jit`

它是允许JIT编译的代码执行。JIT是Just-In-Time的缩写，是一种动态编译技术，可以在程序运行时将字节码编译成本地机器码，从而提高程序的运行速度。这个选项允许JIT编译器在运行时编译代码，从而提高程序的性能。但是这也会增加程序的安全风险，因为它允许动态生成代码并执行它们。

> Just-In-Time技术是一种动态编译技术，可以在程序运行时将字节码编译成本地机器码，从而提高程序的运行速度。JIT技术的应用场景包括但不限于：Java虚拟机、JavaScript引擎、Python解释器等。 

***JIT技术的优点包括但不限于：***

JIT技术可以减少库存，降低成本。
JIT技术可以提高生产效率，减少浪费。
JIT技术可以提高产品质量，减少缺陷率。
JIT技术可以提高企业的灵活性和响应速度。

***JIT技术的缺点包括但不限于：***

* JIT技术需要更多的计算资源，可能会影响程序的性能。
* JIT技术可能会增加程序的启动时间。
* JIT技术可能会增加程序的复杂性，使得调试更加困难。
* JIT技术可能会增加程序的安全风险。


### Allow Unsigned Executable Memory Entitlement

一个布尔值，指示应用是否可以创建可写内存和可执行内存，而不受使用该标志施加的限制。MAP_JIT
key：`com.apple.security.cs.allow-unsigned-executable-memory`

它允许应用程序在运行时使用未签名的可执行内存。这个entitlement会暴露应用程序在内存不安全的代码语言中的常见漏洞，所以你需要仔细考虑你的应用程序是否需要这个例外。

在极少数情况下，应用可能需要重写或修补 C 代码、使用早已弃用的代码（从根本上来说是不安全的）或使用 DVDPlayback 框架。添加允许未签名的可执行内存授权以启用这些用例。否则，应用可能会崩溃或以意外方式运行。NSCreateObjectFileImageFromMemory

### Allow DYLD Environment Variables Entitlement

一个布尔值，指示应用是否可能受到动态链接器环境变量的影响，可用于将代码注入应用进程。
key： `com.apple.security.cs.allow-dyld-environment-variables`

它允许你的应用程序使用动态链接器变量来修改其运行时行为。如果你的应用程序依赖于动态链接器变量来修改其运行时行为，那么你可以将Allow DYLD Environment Variables Entitlement添加到你的应用程序中。这会导致macOS动态链接器(dyld)从以DLYD_开头的环境变量中读取。

### Disable Library Validation Entitlement

一个布尔值，指示应用是否加载任意插件或框架，而无需代码签名。
key：`com.apple.security.cs.disable-library-validation`

Library Validation是macOS的一项安全功能，它可以确保只有经过Apple签名的库才能被加载。如果你禁用了Library Validation，那么你的应用程序就可以加载未经过Apple签名的库，这可能会导致安全问题。


### Disable Executable Memory Protection Entitlement

一个布尔值，指示是否在启动应用和执行期间禁用所有代码签名保护。
key：`com.apple.security.cs.disable-executable-page-protection`

Disable Executable Memory Protection entitlement是一种极端的entitlement，它会从你的应用程序中删除基本的安全保护，使攻击者能够在不被检测到的情况下重写你的应用程序可执行代码。


### Debugging Tool Entitlement

一个布尔值，指示应用是否为调试器，是否可以附加到其他进程或获取任务端口。
key：`com.apple.security.cs.debugger`

它是一种允许应用程序在调试时使用调试工具的技术。当非root用户使用带有调试工具授权的应用程序时，系统会显示一个授权对话框，要求系统管理员的凭据。如果授权成功，则调试器在授权过期之前会收到一个10小时的会话。

这种技术的优势是可以帮助开发人员更快地诊断和解决应用程序中的问题，从而提高开发效率。



## Resource Access

### Audio Input Entitlement

一个布尔值，指示应用是否可以使用内置麦克风录制音频并使用核心音频访问音频输入。
key：`com.apple.security.device.audio-input`

开启Audio Input Entitlement只是允许应用程序使用内置麦克风录制音频并使用Core Audio访问音频输入。但是，如果您的应用程序需要访问麦克风，则必须在应用程序中请求用户授权，以便系统显示一个对话框，询问用户是否允许该应用程序访问麦克风。如果用户授权，则应用程序可以使用麦克风。如果用户拒绝，则应用程序无法使用麦克风。因此，Privacy - Microphone Usage Description和Audio Input Entitlement是两个不同的机制，它们都需要在应用程序中使用麦克风时进行处理。

### Camera Entitlement

一个布尔值，指示应用是否可以与内置和外部相机交互，以及捕获动画和静止图像。
key：`com.apple.security.device.camera`

在Xcode中，Camera Entitlement和Privacy - Camera Usage Description都是与相机权限相关的设置。Camera Entitlement是一个应用程序的权限，它允许应用程序访问相机硬件。在macOS 10.14及更高版本中，用户必须明确授权每个应用程序访问相机。

Privacy - Camera Usage Description是应用程序的信息属性列表（Info.plist）文件中的键，它描述了应用程序为什么需要访问设备的相机。这个键对应的值是一个字符串，它会显示在用户授权请求对话框中。例如，如果您的应用程序需要访问相机以拍摄照片，则可以将此键设置为“允许此应用程序使用您的相机拍摄照片”。

### Location Entitlement

一个布尔值，指示应用是否可以从“定位服务”访问位置信息。
key：`com.apple.security.personal-information.location`

Location Entitlement是一个应用程序的权限，它允许应用程序访问设备的位置服务。如果您的应用程序需要定位服务，则需要开启这个权限。

Location Entitlement和Privacy - Location Usage Description是两个不同的权限。Location Entitlement允许应用程序访问设备的位置服务，而Privacy - Location Usage Description则是一个描述应用程序使用位置服务的字符串。如果您的应用程序需要定位服务，则需要开启这两个权限。

### Address Book Entitlement

一个布尔值，指示应用是否可能对用户通讯簿中的联系人具有读写访问权限。
key：`com.apple.security.personal-information.addressbook`

Address Book Entitlement允许应用程序访问用户的联系人数据库。而Privacy - Contacts Usage Description是应用程序在请求用户授权时显示的文本，以便用户了解应用程序将如何使用他们的联系人数据。

因此，这两个概念是不同的，但在应用程序中需要同时使用。如果您的应用程序需要访问用户的联系人数据库，则必须在Info.plist文件中包含NSContactsUsageDescription键，并提供一个字符串值，以解释应用程序如何使用此数据。

### Calendars Entitlement

一个布尔值，指示应用是否对用户的日历具有读写访问权限。
key：`com.apple.security.personal-information.calendars`

Calendars Entitlement是一种权限，它允许应用程序访问用户的日历数据库。而Privacy - Calendars Usage Description是应用程序在请求用户授权时显示的文本，以便用户了解应用程序将如何使用他们的日历数据。

如果您的应用程序需要访问用户的日历数据库，则必须在Info.plist文件中包含NSCalendarsUsageDescription键，并提供一个字符串值，以解释应用程序如何使用此数据.

### Photos Library Entitlement

一个布尔值，指示应用是否对用户的照片图库具有读写访问权限。
key： `com.apple.security.personal-information.photos-library`

与上面类似,略

### Apple Events Entitlement

一个布尔值，指示应用是否可以提示用户授予将 Apple 事件发送到其他应用的权限。
key： `com.apple.security.automation.apple-events`

Apple Events Entitlement和Privacy - AppleEvents Sending Usage Description是两个不同的概念。前者是一种权限，允许应用程序使用Apple事件来控制其他应用程序，而后者是一种描述，用于在应用程序请求访问受保护的用户数据时向用户显示的消息。例如，当您的应用程序请求访问另一个应用程序的数据时，系统会显示一个消息，其中包含您在Privacy - AppleEvents Sending Usage Description中指定的文本。这有助于用户了解您的应用程序如何使用他们的数据。

> Apple Events 是一种进程间通信消息，可以将命令发送到应用程序。例如，当您使用 AppleScript 脚本更改桌面图片、控制其他应用程序的用户界面、更改屏幕保护程序、导航文件系统层次结构等时，就可以使用 System Events 应用程序。
> 
> 在 macOS 应用程序中，开发人员可以使用 Apple Events Entitlements 来授予应用程序特定的功能。例如，一个应用程序需要 HomeKit Entitlement 来访问用户的家庭自动化网络。应用程序将其授权作为键值对嵌入在其二进制可执行文件的代码签名中。



## 结尾

如果您的mac app是一个sandbox程序，那么您可以不开启Hardened Runtime。但是，如果您的应用程序需要访问受保护的资源或执行受限制的操作，则需要启用Hardened Runtime。Hardened Runtime是一种安全机制，可防止某些类型的攻击，例如代码注入、动态链接库（DLL）劫持和进程内存空间篡改。