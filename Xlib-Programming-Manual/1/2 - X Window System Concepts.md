# X窗口系统概念

X 窗口系统很复杂，但它是建立在一些可以快速理解的前提之上的。本节描述了这些主要的概念。

## 显示屏(Displays)和屏幕(Screens)

关于X的第一个也是最明显的一点是，它是一个用于点阵显示的窗口系统。

> 在点阵显示中，屏幕上的每个点（称为像素，或图片元素）对应于内存中的一个或多个比特。程序仅仅通过向显示内存写入来修改显示。点阵图形也被称为光栅图形，因为大多数点阵显示器使用电视式扫描线技术：整个屏幕由电子束在显示管的表面一次扫描线或光栅不断刷新。比特映射图形（或内存映射图形）这一术语更为通用，因为它也适用于其他面向点的显示器，如液晶屏。

它支持 **彩色以及单色和灰度** 显示。

一个稍微不寻常的特点是，显示器被定义为由键盘、鼠标等指向性设备和 **一个或多个** 屏幕组成的工作站。多个屏幕可以一起工作，鼠标的移动 **允许跨越物理屏幕的界限** 。只要多个屏幕是由一个用户用一个键盘和指向设备控制的，它们就只包括一个显示器。

## 服务器-客户端模型 

接下来要注意的是，==X是一个面向网络的窗口系统== 。一个应用程序不需要在实际支持显示的同一系统上运行。虽然许多应用程序可以在工作站上本地执行，但其他应用程序可以在其他机器上执行，通过网络向特定的显示器发送请求，并从控制显示器的系统接收键盘和指针事件。

控制每个显示器的程序被称为服务器。一开始，服务器这个词的用法可能看起来有点奇怪，当你坐在工作站时，你往往认为服务器是网络上的东西（如文件或打印服务器），而不是控制你自己显示器的本地程序。需要记住的是，你的显示器可以被网络上的其他系统所访问，对于这些系统来说，在你的系统中执行的代码确实充当了真正的显示服务器。

服务器在本地或远程系统上运行的用户程序（称为客户端或应用程序）和本地系统的资源之间充当中介角色。服务器主要执行以下任务：

- 允许多个客户端访问显示器。
- 翻译来自客户端的网络信息。
- 通过发送网络信息将用户输入的信息传递给客户端。
- 进行图形绘制（由服务端完成）。
- 维护复杂的数据结构，包括窗口、光标、字体和 “图形上下文”，作为客户机之间可以共享的资源，并通过资源ID简单地加以引用。服务器维护的资源减少了必须由每个客户端维护的数据量和必须通过网络传输的数据量。

由于X窗口系统使网络对客户透明，只要它们所运行的主机得到控制该显示器的服务器的许可，这些程序就可以连接到网络中的任何显示器。在网络环境中，一个用户在网络中的几个不同的主机上运行程序是很常见的，所有的程序都是从一个屏幕上调用并显示它们的窗口。

在实践中，每个用户都坐在一台服务器前，可以在本地启动应用程序在本地服务器上显示，也可以在远程主机上启动应用程序在本地服务器上显示，当然前提是远程主机有连接到本地服务器的权限。网络中的所有其他用户处于类似的情况，他们可以在自己的系统上或在你的系统上运行应用程序，但在大多数情况下，他们将在自己的服务器上显示。这种对网络的使用被称为分布式处理。分布式处理有助于解决系统负载不平衡的问题。当一台主机过载时，该机的用户可以安排他们的一些程序在其他主机上运行。

这种安排的一个极端是PC服务器或X终端。因为这些单任务系统只能运行X服务器（有时还有一个窗口管理器），坐在其中一个服务器上的用户必须在整个网络的系统上运行所有的客户端，并将其结果显示在PC或X终端的屏幕上。这使得单任务的PC或X终端看起来和工作起来就像多任务工作站上的X。

## 窗口管理器

X编程中的另一个重要概念是，应用程序实际上并不控制诸如窗口出现的位置或它的大小。考虑到多处理器、多客户端对同一工作站显示器的访问，客户端必须不依赖于特定的窗口配置。相反，客户端会给出它希望显示多长时间和在什么地方。屏幕布局或外观以及用户与系统互动的风格由一个单独的程序决定，称为 **窗口管理器** 。

==窗口管理器只是用 `Xlib` 编写的另一个程序，只是它被赋予了控制屏幕上窗口布局的特殊权限。== 窗口管理器通常允许用户移动或调整窗口的大小，启动新的应用程序，并控制屏幕上窗口的堆叠，但只能根据窗口管理器的窗口布局策略。窗口布局策略是一组规则，规定了窗口和图标的可允许的大小和位置。

==与公民不同，窗口管理员有权利但没有责任。== 程序必须准备好与任何类型的窗口管理器合作，或者根本不合作（有相当简单的方法来为这些突发事件准备程序）。像窗口管理器 twm 不会强制执行任何窗口布局策略，但客户仍应假定可能有一个。例如，在一个新窗口显示在屏幕上之前，窗口管理器必须被告知该窗口的理想尺寸。如果窗口管理器不接受所需的窗口尺寸和位置，程序必须准备好接受不同的尺寸或位置，或者能够显示诸如 “太小了！我无法完成必要显示“ 这样的信息。

如果你很难想象这种情况，可以想象一个不允许窗口重叠的窗口管理器。这就是所谓的平铺式窗口管理器。一些平铺式窗口管理器只允许瞬时窗口（如弹出式菜单）重叠。另一方面，twm 窗口管理器被称为 `read-estate-driven` ，因为键盘输入被自动分配到鼠标指针当前位于的窗口。

X在某种程度上是不寻常的，因为它没有强制要求一个特定类型的窗口管理器。它的开发者试图使X本身尽可能不受窗口管理或用户界面政策的影响。虽然X11发行版包括twm作为一个窗口管理器的样本，但个别制造商期望编写他们自己的窗口管理器和用户界面指南。

从长远来看，X的开发者很可能做出了正确的选择，但因为缺乏明确的用户界面指南，X将进入一段实验期，在这段时间里，市场可能会出现比目前更好的设计。然而，一些行业观察家谴责此举，指出它削弱了X作为一个标准用户平台的吸引力 -- X程序可能可以在多个供应商的系统间移植，但如果用户必须在每个系统上处理不同的用户界面，这种移植性的好处就会失去一半。在市场上出现一个明确的用户界面标准之前，开发者必须小心翼翼地编写他们的程序，使它们的程序能够在不同的窗口管理器和用户界面下运行。

## 事件

就像任何鼠标驱动的窗口系统一样，X客户端必须准备好对许多不同的事件作出反应。事件包括用户输入（按键、鼠标点击或鼠标移动），以及与其他程序的交互（例如，当另一个重叠的窗口被移动、关闭或调整大小时；如果一个窗口被遮挡的部分被暴露出来，客户端必须重新绘制它）。许多不同类型的事件可以在任何时候、以任何顺序发生。它们按照发生的顺序被放在队列中，通常由客户端按照这个顺序进行处理。事件驱动的编程使得让用户告诉程序该做什么变得很自然，而不是反过来。

处理事件的需要是窗口系统下的编程与传统 `Unix` 或 `PC` 编程的主要区别。X程序不使用标准的 C 函数来获取字符，也不轮询输入。相反，有一些接收事件的函数，程序必须根据接收到的事件的类型进行分支并执行适当的响应。但与传统程序不同，X程序必须在任何时候为任何类型的事件做好准备。在传统程序中，程序处于控制状态，在特定时间要求某些类型的输入。在X程序中，用户在大部分时间都处于控制地位。

## 对X的扩展

关于X的最后一件事是，它是可扩展的。代码包括一个定义好的机制，用于整合扩展，这样供应商在增加功能时就不会被迫以不兼容的方式来破坏现有系统。这些扩展就像核心的Xlib例程一样被使用，并在同一层级上执行。有些扩展是MIT X联盟的标准，如支持非矩形窗口的Shape扩展和支持键盘和鼠标以外的输入设备的X Input扩展。

扩展有客户端和服务器端的代码。一个服务器供应商不需要为所有的标准扩展提供支持。因此，在使用一个扩展之前，你必须询问服务器是否支持该扩展。在编写本文时（1994年），只有Shape扩展被广泛支持。

## X窗口系统的软件架构

显示服务器是一个运行在每个支持图形显示器、键盘和鼠标的系统上的程序。

Xlib提供了连接到特定显示服务器、创建窗口、绘制图形、响应事件等的功能。Xlib调用被翻译成协议请求，通过tcp/ip发送至本地服务器或网络上的另一个服务器。在X发布的许多示例应用程序中，有xterm（一个终端模拟器）、xcalc（一个计算器）、xmh（一个邮件处理器）、xclock（一个时钟）和troff预览器。

窗口管理器只是另一个用X库编写的程序，只是按照惯例，它被赋予了控制屏幕上窗口布局的特殊权限。

客户端是一个比应用程序稍微笼统的术语，尽管它们几乎是同义词。除了窗口管理器，所有的客户端都被称为应用程序。当本手册中的声明只适用于窗口管理器或只适用于由窗口管理器管理的应用程序时，将使用适当的术语。在其他情况下，将使用任何一个看起来更自然的术语。

应用程序和窗口管理器可以只用Xlib来编写，也可以用一套更高级别的子程序库来编写，这些子程序库被称为工具包。工具包实现了一套用户界面功能，如菜单或命令按钮（一般称为工具包部件），并允许应用程序使用面向对象的编程技术来操作这些功能。工具包的内在因素使程序员能够创建新的部件。

有几个工具包随X11版本一起发布，其中最引人注目的是Xt工具包，它是由Digital和MIT开发的，还有Interviews工具包，它是由斯坦福大学开发。Xt现在已经正式成为X11标准的一部分。

工具包可以使编程变得更容易，使完成的项目更彻底。工具包有内置的用户可配置性和与窗口管理器交互的内置代码，这将为你省去很多麻烦。建议你在大部分的X编程中使用一个工具包。然而，所有现有的C语言的工具包也需要或允许你使用Xlib代码。而且，不仅如此，它们还在内部使用Xlib；所以了解Xlib将有助于你理解工具包的工作原理。

然而，使用工具包也是有代价的。一个是使用工具包的特定程序的可执行文件比使用Xlib编写的同等程序大得多。另一个问题是，工具包利用高度抽象的概念，并且由于其面向对象的设计，需要严格的编程惯例。这些都需要时间来学习。本手册描述了如何用 `Xlib` 编写程序。
