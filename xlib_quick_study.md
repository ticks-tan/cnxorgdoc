# 快速了解Xlib

## 前言

很高兴你能发现这篇文章，这篇教程将会带你走进 X 窗口环境（本教程为 [Xlib](https://x.org/wiki/) ）下的图形编程，这里所说的当然不是那种高级而底层的图形编程，仅仅是利用现有的 Xlib 库完成一些简单有趣的小程序。教程本身作用不大，因为实际上 X 程序员都会选择使用更顶层抽象的库，比如 **QT**  、**GTK**  等类似库，如果你想学习并写出功能复杂且美观的桌面软件，这篇教程或许并不适合你，更推荐你学习并使用前面提到的任何一种库，当然，了解一下 **Xlib** 库也不是一件坏事 `^_^` 。

在正式开始学习之前，我会介绍 X窗口环境和 Xlib 的一些概念东西，希望可以给只需要了解 Xlib 的人一点帮助。

## X窗口环境的客户端和服务端模型

X 窗口系统开发的一个主要目标是灵活性。作为较低级别的图形库，需要提供 **绘制窗口** 、**处理用户输入** 、**用不同颜色绘制** 等工具，为此，Xorg 决定将系统分为两部分，一个是客户端，负责告诉服务端需要做什么，一个是服务端，负责在屏幕上进行实际绘制，并读取用户输入数据，发送给客户端处理。

是的，这种行为与我们平时的认知习惯完全相反，通常是服务端控制客户端做哪些事，客户端完成事情，必要时发送数据给服务端处理，X环境完全相反！

客户端与服务端使用 **X消息协议** 通信，该协议最初采用 TCP/IP 协议进行，允许客户端运行在与服务器连接的同一网络下任何机器上。不过大多情况下，客户端和服务端都运行在同一一台机器，所以后来对X服务器进行了优化，使得在同一机器下的服务端与客户端通信性能更高。

## GUI 异步编程模型

为保证性能和流畅，GUI程序通常使用 **异步编程模型** ，也可以称为 **“事件驱动模型”** 。这意味着程序大多数都是空闲的（这里空闲指CPU使用），等待服务器发送的事件/消息，然后根据事件类型和内容作出相应处理。事件有很多种，比如告诉客户端用户在 点(x, y) 处点击了按钮、客户端窗口需要重新绘制等。为了让程序对用户操作及时作出反馈，就需要在很短时间内处理事件（通常小于200毫秒）。

为此，程序在处理事件时会 **“速战速决”** ，处理太慢在用户看来就是卡顿，点击没反应，如果在处理事件时去同步连接数据库、甚至复制一个很大的文件，那用户体验可想而至。不过还好，我们有 **异步模型** ，通过将这些耗时操作放到 **线程池** 或者进程中执行，可以有效提高处理速度！

因此，GUI程序处理流程通常如下：

```c
/// 以下为伪代码！
init_app();           // 对应用程序进行必要初始化
connect_xserver();    // 连接X服务器
xinit();              // 执行与X环境相关初始化
while not quit {
    get_xserver_event(); // 从X服务器获取下一个事件
    process_event();     // 处理收到的事件，速战速决！
}
disconnect_xserver();  // 关闭与X服务器的连接
clean_app();           // 应用程序相关资源释放
```

## Xlib 库的基础概念

上面提到的都是关于 X系统的一些东西，你可以认为我在扯淡，下面介绍的才是 Xlib 库的一些概念。由于 **Xlib** 库是用 **C语言** 编写，所以我希望你有一些 C语言的知识，这样在看一些例子时不会感到疑惑。

上面提到X环境下应用程序与X服务器沟通通过 **X消息协议** ，一次消息传输包含很多内容，应用程序需要按指定格式像X服务器传输消息才会得到回应。为了消除应用程序实际实现X协议层的需要，“Xlib库” 应运而生，这个库使应用程序可以使用底层API访问X服务器。由于协议是标准且公开的，使用任何实现了Xlib协议的客户端都可以与X服务器沟通，这在今天看来微不足道，但在使用字符模式终端时代却是一个重大突破。旧时代的产物，由于各种历史原因，在今天会表现出各种缺点，这很正常。

### X显示器(X Display)

X显示器是使用 Xlib 库的一个主要概念，在 C语言中，它用一个 **[结构体](https://cn.bing.com/search?q=C%E8%AF%AD%E8%A8%80%E7%BB%93%E6%9E%84%E4%BD%93)** 表示：

```c
struct Display {
    ... // 省略字段
}
```

结构体中隐藏了两个队列，一个是服务器发送给客户端的消息队列，一个是客户端发送给服务器的消息队列。当我们打开一个与X服务器连接时，库会返回一个 **指向这个结构体数据的指针（Display*）** ，之后，我们可以将这个指针传递给任何需要给X服务器发送消息或从X服务器接收消息的 Xlib 函数。

### 图形上下文(Graphics Context)

当我们执行各种绘图操作时，有很多需要控制的选项（比如字的背景颜色，前景色，线粗细等），不可能将这些选项作为一个个参数传递，所有 Xlib 使用了一个 **图形上下文结构体(GC)** ，可以在这个结构体内设置绘图参数，然后传递结构体指针给相应函数，不仅可读性更高，也能复用结构体，避免多次使用每次都初始化。

### 对象句柄(Object Handle)

当X服务器为我们创建各种对象（窗口、按钮等）时，相关函数会返回一个 **对象句柄** ，对象句柄是相关实际对象的一个映射，我们可以通过这个句柄去调用 Xlib 相关函数来操作这些对象，就像学校里每个学生都有一个学号，老师可以通过学号找到对应学生，从而实现按照学号点名！

### Xlib结构体的内存分配

Xlib 接口中使用了各种结构体类型，其中一些由用户直接分配，其他由 Xlib 特殊函数分配。由库完成这些结构体的初始化对开发人员是很方便的，因为这些结构体往往包含很多变量，如果让程序员去完成这些初始化会变得非常繁琐，库提供默认值，不仅让初学者很容易使用，也不干扰经验丰富的程序员对这些值进行调节。

内存分配一点要对应内存释放，如果只分配不释放，那和欠钱不还有何区别？在使用 Xlib 是，内存释放分为两类，一种是我们自己使用 `malloc` 分配的内存，使用 `free` 函数释放，另一种由我们调用 Xlib 特定函数由 Xlib 分配，调用 `XFree` 函数释放对应内存。只有合理分配和释放，才能尽量避免内存泄漏。

### Xlib事件

一个 `XEvent` 类型的结构体用来传递从X服务器收到的事件，Xlib 支持大量事件类型。`XEvent` 结构体包含接收到的事件类型和与事件相关的数据（比如屏幕产生事件的位置、与事件相关的鼠标按钮、与“重绘”相关的屏幕区域等）。由于不同事件类型包含的数据不同，读取事件相关数据的方式取决于事件类型，因此一个 `XEvent` 结构体包含了所有可能的事件类型，它在 **C语言** 表现为一个 [联合体](https://cn.bing.com/search?q=C%E8%AF%AD%E8%A8%80%E8%81%94%E5%90%88%E4%BD%93) 。因此，我可以有一个 **XExpose** 事件、一个 **Xbutton** 事件、一个 **XMotion** 事件 · · · · · ·

## 编译基于Xlib的程序（C语言写的）

Xlib 是一个C语言写的库，在安装了 Xlib 的计算机上面会有对应库，只需要在编译时进行链接即可，就像这样：

编译app.c并链接Xlib库：`cc app.c -o app -lX11`

如果提示没有找到 X11 库，可以手动添加 “-L” 参数，指定库目录位置，这里假设在 **/usr/X11/lib** 下：`cc app.c -o app -L /usr/X11/lib -lX11`

## 打开和关闭与X服务器的连接

要与X服务器交流，首先需要打开一个与X服务器的连接。Xlib 提供函数 `XOpenDisplay` 函数获取一个X服务器连接，不过该函数需要传递一个运行X服务器的主机地址和显示器编号的参数。X 窗口系统支持多个显示器连接到同一台机器上，不过通常我们只有一个本机显示器，编号为 “0” ，如果我们要连接本地显示器（即我们程序运行的机器上的显示器），可以指定显示器地址和编号为：**":0"** ，要连接到地址为 “Tony”的机器第一个显示器可以使用：**"Tony:0"**  ，新建 **0-1.c** ，并写入以下内容：

```c
#include <X11/Xlib.h>  // 包含Xlib库的头文件，必须！
#include <stdio.h>     // 输出信息

int main() 
{
    // Xlib 库中X显示器用 Display 结构体表示
    Display* display = NULL;
    // 打开本地第一个显示器
    display = XOpenDisplay(":0");
    // 判断是否连接失败
    if (NULL == display) {
        // 输出错误
        fprintf(stderr, "连接到显示器[%s]失败！\n", ":0");
        // 这里不用调用关闭连接函数，因为没有连接成功
        return 1;
    }
    fprintf(stdout, "连接显示器[%s]成功！\n", ":0");
    // 关闭连接，会自动是否服务器资源
    XCloseDisplay(display);
    return 0;
}
```

注意：
1. X应用程序通常会检查环境变量 **`DISPLAY`** 是否被定义，如果被定义使用环境变量内容为 **`XOpenDisplay()`** 函数的参数。
2. **`XCloseDisplay()`** 函数用来关闭与X服务器的连接，这会导致所有由程序创建的窗口被服务器自动关闭，并且客户端在服务器申请的资源会被释放，不过这不会导致我们的程序终止！

执行 **`cc 0-1.c -o 0-1 -lX11 && ./0-1`** ，不出意外会输出以下信息：

```
[22:41]ticks: Xlib $ cc 0-1.c -o 0-1 -lX11
[22:42]ticks: Xlib $ ./0-1
连接显示器[:0]成功！
[22:42]ticks: Xlib $ 
```

### 查看显示器信息

当我们打开了与X服务器的连接，就可以检查它的一些基本信息，比如它有那些屏幕、屏幕的尺寸（宽度和高度），支持哪些颜色等，这些信息都存储在 **`Display`** 结构体内，不过我们不能直接访问，需要通过 Xlib 的一些特定函数访问！新建一个 **0-2.c** 文件，并写入以下内容：

```c
#include <X11/X.h>
#include <X11/Xlib.h>
#include <stdio.h>

int main()
{
        int screen_num;         // 默认屏幕ID
        int screen_width = 0, screen_height = 0;        // 屏幕宽高
        Window root_windows;    // 根窗口ID，每个屏幕都有一个覆盖整个屏幕的窗口
        // Xlib只支持黑色和白色，使用大整数表示不同颜色
        unsigned long white_pixel;      // 屏幕使用白色的颜色值
        unsigned long black_pixel;      // 屏幕使用黑色的颜色值

        Display* display = XOpenDisplay(":0");
        if (NULL == display) {
                fprintf(stderr, "连接X服务器失败！\n");
                return 1;
        }
        // 查看屏幕默认ID
        screen_num = DefaultScreen(display);
        printf("默认屏幕ID：%d\n", screen_num);
        // 获取屏幕宽高
        screen_width = DisplayWidth(display, screen_num);
        screen_height = DisplayHeight(display, screen_num);
        printf("屏幕宽高为：(%d:%d)\n", screen_width, screen_height);
        // 查看根窗口ID
        root_windows = RootWindow(display, screen_num);
        // 查看白色和黑色颜色
        white_pixel = WhitePixel(display, screen_num);
        black_pixel = BlackPixel(display, screen_num);
        printf("白色和黑色颜色值：(%lu, %lu)\n", white_pixel, black_pixel);

        // 关闭X服务器连接
        XCloseDisplay(display);
        return 0;
}
```

其中 `DefaultScreen` 、`WhitePixel` 、`BlackPixel` 等都是 Xlib 的宏，可简化使用。

保存上述文件，运行：**`cc 0-2.c -o 0-2 -lX11 && ./0-2`**  ，在我的机器上，它的输出为：

```
默认屏幕ID：0
屏幕宽高为：(1920:1080)
白色和黑色颜色值：(16777215, 0)
```

## 创建一个基本窗口

上面的例子无一例外，都没有出现我们期待的窗口。只是在标准输出输出了几行信息，这一点也不酷，我们要学的是如何用 Xlib 创建窗口！不用着急，那是因为我们没有 **“告诉X服务器我要创建窗口”** ，Xlib 库提供了多个 Api 让我们创建窗口，其中一个是 **`XCreateSimpleWindow()`** ，它需要提供很多个参数，虽然比较麻烦，不过这时必须的。函数定义和参数说明如下：
```c
// 创建一个窗口并返回窗口句柄
Windows XCreateSimpleWindow(Display* display, Windows parent, 
                           int x, int y, 
                           unsigned int width, unsigned height,
                           unsigned int border_width,unsigned long border, 
                           unsigned long background);
```
- **Display* display** ：指向显示器的指针
- **Windows parent** ：创建窗口的父窗口（现在知道为啥每个屏幕都有一个根窗口了吧）
- **int x, int y** ：分别表示窗口的 x 和 y 坐标
- **unsigned int width, unsigned int height** ：分别表示窗口宽高（像素）
- **unsigned int border_width** ：窗口边框宽度
- **unsigned long border** ：窗口边框颜色
- **unsigned long background** ：窗口背景颜色

接着编辑 **0-2.c** ，让我们新建一个宽高分别为屏幕宽高 1/3 ，并显示在屏幕中心的窗口，输入以下内容：

```c
... // 省略部分内容

    // 创建一个新窗口
    int win_width = screen_width / 3;
    int win_height = screen_height / 3;
    Window win = XCreateSimpleWindow(display, root_windows, 
                    win_width, win_height,
                    win_width, win_height, 
                    2, black_pixel, white_pixel);
                        
```

先别着急保存运行，因为创建了窗口不代表它会被画在屏幕上。默认情况下，新创建的窗口不会显示在屏幕上（默认是不可见的），我们需要将它 **“映射”到屏幕上** ，使用 **`XMapWindows()`** 可以映射，需要提供两个参数，一个为 **`Display*`** ，一个为 **`Windows`** ，分别表示要映射到的目标屏幕和需要映射的窗口。

映射完窗口还需要我们使用 **`XFlush()`** 将信息刷新到X服务器，以便X服务器进行真正的绘制。继续编辑文件，添加以下信息：

```c
...
    // 窗口映射
    XMapWindow(display, win);
    // 刷新信息到服务器
    XFlush(display);
...
    return 0;
}
```

万事具备，现在可以执行：**`cc 0-2.c -o 0-2 -lX11`** 编译并运行了。但是当你运行后就会发现，没有窗口显示，控制台仅仅输出了几行信息！

上面我们讲过，**`XCloseWindow()`** 会关闭与服务器的连接，同时服务器会清楚该程序创建的所有窗口，再仔细看看程序就会发现程序在刷新信息后马上旧断开连接了，服务器刚绘制完甚至还没开始绘制就我们就断开连接了，所以在我们看来并没有窗口被创建的现象。

要解决这个还不简单，我们过几秒再关闭连接不就行了！前面也讲过，X系统绘制工作不是有客户端负责，所以我们直接让程序休眠几秒，目前我们学到的内容还不足以处理事件，所以不进行事件处理，稍后会讲到如何处理事件。现在在刚才代码刷新信息和关闭连接之间插入 **`sleep(10);`**  代码让我们的程序休眠10秒，记得添加头文件 **`#include <unistd.h>`** 。

再次编译并运行，不出意外你就会看到一个白底黑边的窗口了。这是我们创建的第一个窗口，值得庆祝一下！

## 在窗口中绘图

可以使用 Xlib 提供的图形功能在窗口中绘制像素点、直线、圆、矩形等，不过在绘图前，我们需要定义各种通用绘图参数（比如使用什么颜色、线宽多少等），这可以通过上面提到的 **图形上下文(GC)** 完成。

- **分配一个图形上下文(GC)** 

如前所述，图形上下文包含了很多图形绘制属性，如果让我们一个个初始化就显得很繁琐了，不过好在 Xlib 可以帮我们完成初始化，使用函数 **`XCreateGC()`** 可以分配一个新的 GC 。 为了方便，以后代码示例只会列出部分关键代码！不过相关变量会通过注释说明，避免阅读困难。下面的例子用于创建一个新的 **图形上下文GC** ：

```c
// 用于设置GC属性值
XGCValues values = CapButt | JoinBevel;
// 设置GC属性值的掩码
unsigned long value_mask = GCCapStyle | GCJoinStyle;

// 创建一个新的GC，指定值
// display: 这里表示要显示的屏幕
// win: 这里表示要绘图的窗口
GC gc = XCreateGC(display, win, value_mask, &values);
// 判断是否创建失败
if (gc < 0) {
    fprintf(stderr, "Create new GC error!\n");
}
```

值得注意的是上面的 values 和 value_mask，GC中有很多属性，Xlib 给每一个值赋予的初始值，如果我们想要设置其中的某些值，就需要使用 **值掩码** ，不同属性通过 **[C语言中操作符“或”](https://cn.bing.com/search?q=C%E8%AF%AD%E8%A8%80%E6%93%8D%E4%BD%9C%E7%AC%A6%E6%88%96)** 设置。对应值也通过 **“或”** 连接，比如上面代码中相当于设置了 属性(GCJoinStyle) = 值(JoinBevel) 和 属性(GCCapStyle) = 值(CapButt) ，其中 `GCJoinStyle` 、`JoinBevel` 、`GCCapStyle` 、`CapButt` 都是 Xlib 中用宏定义的整数值！相关参数具体细节将在后面章节深入讲解。

大多数时候绘图属性不会改变，因此当我们定义好绘图上下文后便可以在绘图函数中使用它了。Xlib 库同时也提供了函数用于设置GC相关属性，比如下面代码用于设置背景和前景颜色、填充样式：

```c
/*
display 为显示屏幕
gc 为刚才创建的图形上下文
screen_num 为默认屏幕ID
*/

// 改变GC属性的前景色为白色
XSetForeground(display, gc, WhitePixel(display, screen_num));
// 改变GC属性背景色为黑色
XsetBackground(display, gc, BlackPixel(display, screen_num));
// 设置线条填充，为实心
XSetFillStyle(display, gc, FillSoild);
// 设置线条的属性，参数分别为 屏幕，GC，线宽，线条端点样式，线条连接样式
XSetLineAttributes(display, gc, 2, LineSoild, CapRound, JoinRound);

```

这里我们只介绍部分属性，其他属性可以参考具体头文件，后面章节也会给出部分属性介绍。

- **绘制基本图形（点、线、矩形、圆等）**

创建好GC后，我们便可以使用该GC在窗口上绘制，Xlib 提供了一组函数用于绘制基本图形，比如绘制 **点、线、矩形、圆等** 。下面的代码片段介绍了如何使用这些函数：

```c
/*
display：先前初始化的显示屏幕
win：先前初始化好的窗口句柄
gc：先前创建的图形上下文
*/

// 在 (5, 5) 处画一个像素点
XDrawPoint(display, win, gc, 5, 5);

// 画一条线，两个端点分别为 (20, 20) 和 (40, 100)
XDrawLine(display, win, gc, 20, 20, 40, 100);

// 画一段弧线，中心点为 (x, y)，如果是一个椭圆，指定宽度为 w ，高度为 h
// 开始绘制角度为 angle1，结束角度为 angle2
// 开始角度 0 的方向为 “时钟上的3点方向” ，逆时针为正方向
// 角度单位为一度的64分之1,所以 360 * 64 表示 360度
int x = 30, y = 40;
int h = 15, w = 45;
int angle1 = 0, angle2 = 2.109;
XDrawArc(display, win, gc, x-(w/2), y-(h/2), w, h, angle1, angle2);

// 如果要画圆，指定宽度和高度相同即可，比如在 (50, 100) 处画一个直径为20像素的圆
XDrawArc(display, win, gc, 50-(20/2), 100-(20/2), 20, 20, 0, 360*64);

// 在点 (120, 150) 处画一个宽度:高度为 50:60 的矩形，点位于矩形左上角！
XDrawRectangle(display, win, gc, 120, 150, 50, 60);

// 在点 (60, 150) 处画一个 50x60 大小矩形，并填充颜色
XFillRectangle(display, win, gc, 50, 150, 50, 60);
```

## X事件

在 Xlib 程序中，一切都是 **事件驱动** 的。屏幕上的绘制事件有时会作为对事件的响应，如果程序窗口中的隐藏部分被显示出来（比如窗口被提升到其他窗口上方），X服务器将发送一个 **"expose"** 事件，让程序知道它应该重新绘制该部分窗口。其他的像用户输入（按键，鼠标移动等）也会作为一组事件接收。

### **使用事件掩码注册事件类型**

**默认情况下，X服务器不会向程序发送事件** ，程序在创建一个窗口或多个后，应该告诉X服务器你需要接收哪些事件。你可以注册很多事件，包括各种鼠标事件、按键事件、公开事件等。Xlib 这样做的目的也是为来优化性能，不断向一个（通常一个显示器会运行多个GUI程序）程序发送它不会关注和处理的事件，不仅浪费服务器资源，有时还会导致真正需要接收事件的程序出现延迟！

在 Xlib 中，我们可以使用 **`XSelectInput()`** 函数注册事件，该函数接收三个参数 - **Display结构、窗口句柄(ID)、你想关注事件的事件掩码** 。窗口ID参数允许我们为不同窗口注册不同事件，下面代码片段用于为窗口ID(win)注册一个  **"expose"** 事件：

```c
XSelectInput(display, win, ExposureMask);
```

**`ExposureMask`** 是 **`X.h`** 中宏定义的一个常量，我们可以同时设置多个掩码，方法类似与上面对 **GC** 的属性设置 - 使用 **“或”** 。

```c
// 为窗口win 同时注册 expose事件和鼠标按钮按下事件
XSelectInput(display, win, ExposureMask | ButtonPressMask);
```

> 程序要接收并处理事件必须先进行注册，不然就会出现处理事件代码也写了，但是调试的时候总是无法处理一些事件的情况，并且你可能会忘记检查这里的注册事件代码。

### **在事件循环中接收事件**

注册了感兴趣的事件后，就可以在程序循环中接收并处理事件了，编写循环的方式很多，其中一种可以像这样：

```c
...
...
// 接收的事件
XEvent revent;

// 事件循环
while (1) {
    // 获取下一个事件，放入 revent
    XNectEvent(display, &revent);
    // 根据事件类型，处理事件
    switch (revent.type) {
        /* 处理事件 */
        ...
        ...
            break;
        default: 
            break;
    }
}
...
...
```

函数 **`XNectEvent()`** 作用是从X服务器获取下一个事件，如果（暂时）没有可以获取的事件，函数会阻塞（不会占用CPU），直到X服务器给程序发送事件并成功接收，接收到的事件会被存入第二个参数 **`XEvent`** 结构体中。

这是一个死循环，如果我们想要退出程序，只需要注册一个特殊的退出事件，然后处理事件并退出即可，这个事件将在稍后讲解。

### **Expose事件**

**"expose"** 事件是程序最可能接收到的事件之一，它会在以下情况之一下发送给客户端：

1. 一个覆盖我们窗口的窗口被移开，露出我们窗口的一部分或者全部。
2. 我们的窗口被提升到其他窗口前。
3. 我们的窗口首次映射。
4. 我们的窗口图标被取消。

为了节省内存，当窗口被遮挡时X服务器不会保存窗口内存，其内容会丢失！所以X服务器需要通过传递事件来告诉程序内容需要重新绘制。

当我们获取到一个 **"expose"** 事件时，我们应该从 **`XEvent`** 结构体的 **`xexpose`** 成员中获取数据，它包含了一些属性：

- **int count** ：在X服务器事件队列中等待的其他 expose 事件，如果我们获得连续几个 expose 事件，为了避免多次绘制窗口，会选择在最后一个 expose 事件是绘制，也就是 count 为 0时。
- **Window window** ：发生 expose 事件的窗口句柄（如果我们程序注册了多个窗口的 expose 事件）。
- **int x, y** ：从窗口左上角开始计算，需要重绘的窗口区域坐标（单位像素）。
- **int width, height** ：需要重绘的窗口区域宽高（单位像素）。

### **获取用户输入**

用户输入主要来源于键盘和鼠标，有很多事件用来通知我们用户输入，比如键盘一个按键被按下、键盘上一个按键被释放、鼠标在窗口上移动，鼠标进入或离开我们的窗口等等。

#### **鼠标按下和释放事件**

鼠标按下和释放事件有以下两个事件掩码和事件类型：

事件掩码：
1. **ButtonPressMask** ：在窗口中按下任何按钮时通知我。
2. **ButtonReleaseMask** ：在窗口中按下后抬起任何按钮时通知我。
事件类型：
1. **ButtonPress** ：窗口中有任何按钮按下。
2. **ButtonRelease** ：窗口中有任何按钮抬起。

这些事件类型通过 **`XEvent`** 结构体的 **`xbutton`** 成员访问，包含有一些属性：

- **Window window** ：发生 鼠标点击/释放 事件的窗口句柄（如果我们程序注册了多个窗口的 鼠标点击/释放 事件）。
- **int x, y** ：点击时鼠标位置 - 从窗口左上角开始的 (x, y) 坐标（单位像素）。
- **int button** ：被点击的鼠标按钮编号，1、2、3分别为鼠标左键、右键、中键。
- **Time time** ：事件发生的时间（按毫秒计），可以用来检测短时间内的双击事件。

下面的例子用来演示当我们鼠标左键点击时，在鼠标位置画一个点，右键点击时，擦除鼠标位置的点：

```c
// 这里假设gc1为画点的GC（前景为黑色，背景为白色）
// gc2为擦除的GC，前景为背景色，都为白色
...
XEvent revent;
...
case ButtonPress:
    int x = revent.xbutton.x;
    int y = revent.xbutton.y;
    Window win = revent.xbutton.window;

    switch (revent.xbutton.button) {
        case Button1:
            // 画点
            XDrawPoint(display, win, gc1, x, y);
            break;
        case Button2:
            // 擦除点
            XDrawPoint(display, win, gc2, x, y);
            break;
        default:
            break;
    }
break;
...
```

#### **鼠标移动事件** 

与鼠标按下和抬起事件类似，我们也可以得到各种鼠标移动事件，不过这些事件分为两类，一类是没有鼠标按键按下时的移动事件，一种是在鼠标按键按下同时移动鼠标（拖动）事件。涉及到的事件掩码有：

- **PointerMotionMask** ：当鼠标在窗口中移动且没有按下任何按键时通知我。
- **ButtonMotionMask** ：当鼠标在窗口中移动且有一个或多个按键被按下时通知我。
- **Button1MotionMask** ：与 ButtonMotionMask 相同，不过只有在 Button1 按下时通知。
- **Button2MotionMask** ：与 ButtonMotionMask 相同，不过只有在 Button2 按下时通知。

在事件循环中需要检查的事件：

- **MotionNotify** ：鼠标在窗口中移动时的事件。

鼠标移动事件同样通过 **`XEvent`** 结构体的 **`xbutton`** 字段访问事件数据，包含一些属性：

- **Window window** ：发生鼠标移动事件的窗口句柄（如果我们程序注册了多个窗口的鼠标移动事件）。
- **int x, y** ：点击时鼠标位置 - 从窗口左上角开始的 (x, y) 坐标（单位像素）。
- **unsigned int state** ：鼠标事件掩码位，需要通过 **`&`** 判断，主要有以下位：

1. Button1Mask
2. Button2Mask
3. Button3Mask
4. Button4Mask
5. Button5Mask
6. ShiftMask
7. LockMask
8. ControlMask
9. Mod1Mask
10. Mod2Mask
11. Mod3Mask
12. Mod4Mask
13. Mod5Mask

- **Time time** ：事件发生的时间（按毫秒计），可以用来检测短时间内的双击事件。

#### **鼠标进入和离开窗口事件**

某些应用程序可能会对鼠标进入和离开窗口事件感兴趣，可以通过注册以下事件让X服务器为我们监听，必要时通知我们：

事件掩码：

- **EnterWindowMask** ：鼠标进入窗口时通知我。
- **LeaveWindowMask** ：鼠标离开窗口时通知我。

事件循环需要检测的事件类型：

- **EnterNotify** ：鼠标进入窗口事件。
- **LeaveNotify** ：鼠标离开窗口事件。

鼠标进入/离开事件通过 **`XEvent`** 结构体的 **`xcrossing`** 字段访问，包含了一些属性：

- **Window window** ：发生 鼠标进入/离开 事件的窗口句柄（如果我们程序注册了多个窗口的 鼠标进入/离开 事件）。
- **Window subwindow** ：子窗口ID，在进入事件中为从"ID"窗口进入的或无，在离开事件为移入"ID"窗口或无。
- **int x, y** ：点击时鼠标位置 - 从窗口左上角开始的 (x, y) 坐标（单位像素）。
- **int mode** ：被点击的鼠标按键编号，可以为 Button1、Button2、Button3 。
- **Time time** ：事件发生的时间（按毫秒计），可以用来检测短时间内的双击事件。
- **unsigned long status** ：鼠标移动事件掩码位，需要通过 **`&`** 判断，主要有以下位：

1. Button1Mask
2. Button2Mask
3. Button3Mask
4. Button4Mask
5. Button5Mask
6. ShiftMask
7. LockMask
8. ControlMask
9. Mod1Mask
10. Mod2Mask
11. Mod3Mask
12. Mod4Mask
13. Mod5Mask

- **bool focus** ：如果窗口有键盘焦点，设置为 true ，否则为 false 。

#### **键盘焦点事件**

一个屏幕上可能同时出现多个窗口，但是 **键盘只能同时连接一个窗口**（想象一下输入一个按键有多个窗口同时响应的情况！），键盘焦点可以帮助X服务器知道当前哪个窗口连接了键盘。Xlib 库提供了一些函数来将键盘焦点设置到某个特定窗口，不过用户通常使用窗口管理器来设置（表现出来就是鼠标点击某个窗口），一旦我们窗口有了窗口焦点，每一个按键按下或者释放都会收到X服务器发送给我们的对应事件。

#### **键盘按下和释放事件**

如果我们的窗口获取了窗口焦点，就可以接收键盘事件，不过获取之前需要使用 `XSelectInput()` 设置以下事件掩码：

- **KeyPressMask** ：当我的窗口获取了焦点，并且用户按下按键时通知我。
- **KeyReleaseMask** ：当我的窗口获取了焦点，并且用户抬起按键时通知我。

对应事件类型如下：

- **KeyPress** ：键盘按键按下事件。
- **KetRelease** ：键盘抬起事件。

这些事件数据可以通过 **`XEvent`** 结构体的 **`xkey`** 成员获取，成员包含了以下属性：

- **Window window** ：触发事件的窗口句柄。
- **unsigned int keycode** ：用户按下或者抬起的按键编号，这些是 Xlib 内部定义的编号，每个编号对应有一个按键。
- **int x, y** ：事件发生时，鼠标坐标（从窗口左上角开始计算）。
- **Time time** ：事件发生的时间（按毫秒计），可以用来检测短时间内的双击事件。
- **unsigned int state** ：鼠标事件掩码位，需要通过 **`&`** 判断，主要有以下位：

1. Button1Mask
2. Button2Mask
3. Button3Mask
4. Button4Mask
5. Button5Mask
6. ShiftMask
7. LockMask
8. ControlMask
9. Mod1Mask
10. Mod2Mask
11. Mod3Mask
12. Mod4Mask
13. Mod5Mask

> 可以看出，事件在X程序中占据绝大部分比重，因为X程序是事件驱动的，注册事件 -> 接收事件 -> 处理事件 -> 接收事件. . . 别忘了最开始的注册事件！


## 处理文字和字体

除了在窗口中绘制图形，我们还需要绘制文字。文字绘制需要额外加载一种字体，字体加载也是通过 **`GC`** ，加载好字体和要绘制的字符后就可以在窗口中绘制文字了。

### **字体结构体**

为了支持灵活的文字，Xlib 中定义了一个结构体 -- **`XFontStruct`** ，这个结构体包含了一些字体信息，用来传递给几个处理文字和绘制文字的函数。

### **加载字体**

Xlib 中可以使用 **`XLoadQueryFont()`** 函数加载一个定义有给定名称的字体数据，如果加载成功会返回一个 `XFontStruct` 指针，如果加载失败会返回 `空指针NULL` 。每个字体可以有两个名字，一个是长名称，包含了字体的全部属性（字体家族、名称、大小、粗细等），一个为简短名称，为X服务器配置。比如下面的代码片段尝试加载 **`"*-helvetica-*-12-*"`** 字体（\*与Shell中通配符类似）：

```c
XFontStruct* font_info;

const char* font_name = "*-helvetica-*-12-*";
font_info = XLoadQueryFont(display, font_name);
if (NULL == font_info) {
    fprintf(stderr, "加载字体[%s]失败！\n", font_name);
}
```

### 将字体分配给图形上下文

可以使用  **`XSetFont()`** 讲字体分配给一个初始化好的 GC 对象：

```c
XSetFont(display, gc, font_info->fid);
```

**`fid`** 是 **`XFontStruct`** 结构体的一个字段，用于识别各种加载好的字体。

### 在窗口中绘制文字

当我们将字体加载进初始化好的GC后，就可以通过 Xlib 的 **`XDrawString()`** 函数在窗口中绘制文字了，这个函数会在指定位置绘制指定的文本，需要注意该位置是文字的左下角，字体和大小已经分配给GC了。

下面的代码片段展示如何使用：

```c
int x = 0, y = 0;
const char* hello_str = "Hello World";
// 在窗口左上角绘制文本，Hello World
XDrawString(display, win, gc, x, y, hello_str, strlen(hello_str));

// 通过文字信息获取文本像素宽度
int string_width = XTextWidth(font_info, hello_str, strlen(hello_str));
// 通过文字信息获取文本像素高度
int font_height = font_info->ascent + font_info->descent;
// 设置 坐标(x, y)
x = (win_width - string_width) / 2;
y = (win_height - font_height) / 2;
// 在窗口中间绘制文本 Hello World
XDrawString(display, win, gc, x, y, hello_str, strlen(hello_str));
```

一个字体信息包含了两个字段：**`ascent`** 和 **`descent`** ，用于指定字体高度。基本上，文字的字符是相对一条假想的水平线绘制，一个字符一部分绘制在这条线上方，一部分绘制在这条线下方，字符最高绘制位置相对于水平线像素高度为 **`ascent`** ，字符最低绘制位置相对于水平线像素高度为 **`descent`** ，两个值相加即为字符最大绘制高度。

---

#### 后续内容待更新

你可以点击 [这里](./README.md) 快速返回首页


