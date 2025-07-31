# 网页备份

this:  [linux cjktty补丁 作者原文整理](https://zhuanlan.zhihu.com/p/486655122)
other: [linux TTY终端显示中文实现方案现状](https://zhuanlan.zhihu.com/p/486542400)

由于原作者的文章所在的平台已关闭，原文链接已失效，各转载文档均存在图片缺失/格式错误等问题，这里对原文进行整理（修改一些错别字、版式以及内容缺失问题）。

------------------------------------------------------------------------

## 引言

**CJKTTY** 补丁是什么，为什么我写了它

当你不使用 X 的时候，打开电脑，你就在使用虚拟终端。这么多年来它工作的很好，直到它来到了中国。包含中文字符的文件名无法正确显示，中文文档无法阅读。当然可以使用 X , 但是我为什么不能让终端也能显示汉字呢？如果在 X 下我能让屏幕显示汉字，终端下一定也能。为此我开始了 internet 上的搜寻。 我找到了 fbterm
，这是个可以利用 /dev/fb0 实现的终端模拟器，和 XTERM 一样，只不过 XTERM 利用的 X 绘制文字，而 fbterm 直接写入 /dev/fb0。

- **/dev/fb0 是什么？**

> 帧缓冲区设备。帧缓冲区是一块存储区域（内存或者显存或者其他的输出设备的存储空间），内核将其抽象为一个设备。通过访问该设备就能访问帧缓冲区。帧缓冲区的内容既是屏幕映像。由输出设备不停的扫描帧缓冲区生成显示设备的控制信号。\

然而我似乎总是忘记登录后开启
fbterm，对我来说，等看到乱码的时候突然想起没有开启 fbterm
并不是那么愉快的经历。这只是其中一个缺点。fbterm
占用了帧缓冲区设备，导致 w3m 这类使用帧缓存绘制图像的终端 www
浏览器不能正常工作。许多依赖帧缓冲区设备的终端程序都不能被 fbterm
良好的兼容。 于是我继续寻找，找到了 youbest 写的中文补丁。
我有个喜欢使用最新内核的习惯，当内核升级导致 youbest
的补丁再也不能使用的时候，我开始到 youbest 发布补丁的页面留言，希望
youbest 百忙中能修改一下他的补丁。

Youbest
似乎很忙，在新内核的诱惑下，我决定自己在修改。那个时候我才真正的开始看内核的代码。终于了解了虚拟终端的工作原理后，我开始修改内核。由于内核内部结构变动导致
youbest
的补丁无法应用，我几乎是从头开始了开发而不是简单的将无法应用的部分进行修整。唯一得到保留的就是
youbest 补丁中的点阵字库。我将其命名为 CJKTTY , 取能显示 CJK 的 TTY
之意。

**从哪里获得中文显示补丁呢？**

我将补丁托管在了[repo.or.cz/w/linux-2.6/](https://link.zhihu.com/?target=http%3A//repo.or.cz/w/linux-2.6/cjktty.git)，依据你使用的内核版本，签出内核版本 -utf8 分支就可以。

## 原理

## 控制台是如何显示文字呢?

那么，为了能在控制台下显示出汉字到底需要做什么样的修改么？在开始前，我想先介绍一些名字，并介绍一些控制台在硬件上是如何进行文字显示的。
首先我解释一下几个名词，知道的人可以到这里开始阅读。

- **UNICODE**

> 为每一个字符分配全球唯一的一个数字，但是并没有规定这个数字的表示方法。数字的表示方法由
> UTF 规范规定。UTF-16 使用 2 个字节表示一个 UNICODE 数字，但是对于
> \>=216的数字使用 4
> 字节来表达。UTF-8 则对于 \<127 的数字采取单字节表示，大于 127
> 的数字要根据其大小选择 2\~6 个字节进行表示。UNICODE
> 在程序内部则简单的使用 unsigned long 即可表示一个字符。\

- **GLYPH**

> GLYPH
> 指的是字体里的字形。字符总是要在特定的字体下表示的，该表示就是字形。比如一个只包含
> 26 个大小写字母的字体，只包含了 52
> 个字形，如果该字体是先大写后小写排列的，那么数字 0 就表示字形 \'A\' ,
> 数字 1 就表示字形 \'B\'。UNICODE 或者 ASCII 到 GLYPH
> 的映射是由一个称作 CMAP 的映射表做的。如果字体里字符就是按照 UNICODE
> 排列的，则其 CMAP 就是 UNICODE CMAP。同理也有 ASCII CMAP。 VGA
> 自带字体没有提供 CMAP，操作系统假定它的 CMAP 是 ASCII
> CMAP。事实上也是如此。\

- **TTY**

> 内核为终端提供的接口，对应用程序而言就是 TTY 设备。通常是使用 stdin
> stdout 来访问。TTY 提供各种 IOCTL 用来设置终端的模式。TTY
> 也提供了用户控制程序的方法，比如 Ctrl-C 终止当前程序。 TTY
> 可以是显示器 +
> 键盘构成的控制台，也可以是串口（可以通过猫链接到电话线上），可以通过
> pts 模拟。XTERM 即利用 pts 为里面运行的程序提供的模拟的终端 ,
> 对应的设备文件 /dev/pts/\* 由模拟终端程序动态创建。\

- **控制台 (CONSOLE)**

> 控制台特指由显示器 + 键盘构成的终端。其中显示器由显卡控制，而且当前
> VGA 兼容显卡有两种模式，文字模式和图形模式。Linux
> 即可以使用文字模式也可以使用图形模式。
> 控制台对于程序是无法访问的，程序只能通过虚拟终端使用控制台\

- **虚拟终端 (VT)**

> 如果你的电脑只有一个终端，那将是多么乏味。一个需要长时间执行的任务就能导致你什么也做不了，Linux
> 的多任务机制的好处荡然无存。所以，你需要更多的终端。Linux
> 内核使用复用机制，将一个控制台复用为多个终端 (63 个，/dev/tty1 到
> dev/tty63)。 按键 Alt+F1-F12 ( 如果当前在 X 中，需要再按下 Ctrl 键 )
> 能在 12 个终端中进行切换。事实上你拥有 63 个终端，键盘只能切换其中的
> 12 个，其他的终端你可以通过 chvt 命令进行切换。
> 当前拥有显示器和键盘的虚拟终端被称为活动终端或者当前终端。\

- **TTY、控制台和虚拟终端有啥区别和联系？**

> 当你按下 Ctrl-C 的时候，当前执行的程序会被终止。因为 Linux 发送了
> SIGTERM 信号给此终端的前台程序。该信号并不是由 Shell
> 产生，而是内核。不论是在虚拟终端下，还是在 X
> 里的终端模拟器里，这个功能都是一样的。终端的一大功能就是进行任务控制，另一个功能是输入输出。输入输出模式下，还可以选择行编辑模式，回显模式，设置终端速率等等。不管你使用的是何种终端，这些功能都是存在的，因为他们都是一个类型的设备。内核将他们抽象为
> TTY 设备。也就是说，应用程序都是在和 TTY
> 这个抽象层打交道，而不是和具体的设备打交道。 能作为 TTY
> 的设备除了控制台外，还有串口。将两台电脑的串口连接起来，其中一台电脑为串口打开登录程序（执行
> /sbin/agetty ttyS0 38400），另一台就能通过可以进行串口通信的程序 (
> 比如 putty、minicom) 登录对方。 控制台可以作为 TTY
> 设备，但是一台电脑一般只有一个屏幕，也就使用一个控制台，所以 Linux
> 在控制台和 TTY
> 之间加了一层虚拟终端。由虚拟终端将控制台复用，这样就可以使用多个终端而不是只有一个了。多个虚拟终端设备合作使用一个控制台。
> 除了串口和虚拟终端，这些都是在内核实现的 TTY 设备，内核还提供了一个叫
> PTY 的为终端设备，XTERM 之类的程序利用 PTY 提供的功能可以在程序里实现
> TTY 的功能。 那么，虚拟终端就是利用控制台复用出了多个 TTY 。TTY 逻辑由
> TTY
> 子系统完成，复用逻辑由虚拟终端实现，而具体的显示则交给控制台完成。如果说这是一个观察者模型的话，控制台就是观察者，它将虚拟终端的内容呈现到屏幕上。
> 在 Linux 下，控制台分文字模式控制台（vgacon）和图形模式控制台
> (fbcon)。\

- **文字模式控制台 (VGA 文字模式 )**

> 文字模式控制台使用 VGA 兼容显卡的文字模式实现 VGA
> 兼容显卡初始化时默认就处于文字模式，能显示 80x25 个字符。
> 在文字模式下，显卡虽然输出给显示器的是图像，但是显卡提供给内核的却只能显示文字功能。要显示一个字符，内核将要显示的字符的代码和属性写入字符缓冲区相应地址即可。缓冲区如下面所示：\
> **图 1. 缓冲区**

<figure data-size="normal">
<div>
<img
src="https://picx.zhimg.com/v2-28c1c64a196fa887d846db9d1faebe65_1440w.jpg"
class="origin_image zh-lightbox-thumb" data-caption=""
data-size="normal" data-rawwidth="538" data-rawheight="127"
data-original-token="v2-4d02387d15de3f9de4de5ea8051c88fb"
data-original="https://picx.zhimg.com/v2-28c1c64a196fa887d846db9d1faebe65_r.jpg"
width="538" />
</div>
</figure>

\
**表 1. 每个字符的格式**\

  ---------- -------- ----- -------- ------- ---------- ---------------
  属性字节                                   字符字节   
  7          6        5 4   3        2 1 0   7          6 5 4 3 2 1 0
  闪烁       背景色         前景色           GLYPH      
  ---------- -------- ----- -------- ------- ---------- ---------------

> VGA 显卡处于文字模式时，物理地址 0xB8000
> 即文字缓冲区起始地址，大小由其文字模式决定，最大 32KB 。VGA
> 显卡内建字符发生器，不断的扫描字符缓冲区并将文字转换为图形驱动显示器显示。其
> GLYPH 即为字符的 ASCII 码。\

- **字符发生器和字体**

> 字符发生器内建一个或者多个位图字体。使用何种字体取决于设置的文字模式代码。Linux
> 默认使用 80x25 字符 16 色模式 , 每字符 8x16 点阵。
> 字符发生器的字体包含 256 个字形。字符发生器里的文字和字符的对应关系在
> DOS 系统下被称为代码页。显卡自带的被称为 OEM 代码页。
> 字符发生器的字体可以修改。DOS 下通过修改代码页实现。Linux 下可以使用
> setfont。\
> 受限于字符发生器，VGA 的文字模式最多同屏出现 256
> 种不同的字符。对于只有 26
> 个字母的拉丁文字绰绰有余，但是却无法满足拥有上万字符的中文。
> 文字模式控制台由 vgacon （drivers/video/console/vgacon.c）实现。 VGA
> 还有不常使用的单色文字模式，起始地址为 0xB0000
> 。并且字符格式也有所不同，这里不再介绍。\

- **图形模式控制台**

> 图形模式控制台直接操作帧缓冲区显示文字。帧缓冲区是一块存储区域（通常是在显存中），其内容就是显示在屏幕上的图像。显卡直接将其内容转化为显示器控制信号。对于程序而言，**帧缓冲区就是屏幕。**\
> 图像模式控制台使用帧缓冲区作为屏幕输出，要想使用图像模式控制台，必须加载帧缓冲区驱动并有相应的硬件。\
> 不同的显示设备通常使用不同的帧缓冲区驱动，对于 VGA
> 兼容显卡，除了能使用针对显卡写的帧缓冲区驱动 ( 如果有的话
> )，还可以使用通用的 VESA 驱动。VESA 驱动利用 VGA 兼容的显卡的 BIOS 将显卡转入 VGA
> 图形模式并设置期望的缓冲区模式，由显示模式内核参数 vga= 给出。 VGA
> 图形模式下，帧缓冲区的起始地址为 0xA0000 ，最大 64KB
> 大小。像素格式看显示色深。典型的 256 色模式下每像素一个字节。 如果
> BIOS 支持 VBE 扩展，能设置更大的分辨率和色彩深度，帧缓冲区大小也会超过
> 64KB，因而起始地址也不再是 0xA0000，具体细节请参考
> [维基百科](https://en.wikipedia.org/wiki/VESA_BIOS_Extensions)。帧缓存模式下，Linux
> 内核将需要显示的字符首先转换为位图，然后将位图字体写入帧缓存即可变呈现于屏幕。
> 也就是说，图形模式下，由内核来承担字符发生器的工作。
> 理论上来说，这个内核自带的字符发生器将能支持显示多于 255
> 种字符。但是我将在后面告诉读者，因为历史原因，内核实现的字符发生器和
> VGA 的字符发生器有着一样的缺点。 图形控制台由
> fbcon（drivers/video/console/fbcon.c）实现。图形控制台由于需要生成位图字形，需要带上位图字体，编译内核的时候至少需要选择一个字体。位图字体在内核是一个大数组，不同的字体
> ( drivers/video/console/font\_\* ) 存储在各自的数组中。\

## 为何图形模式控制台也不能显示汉字

要解释图形模式控制台为何不能显示汉字，首先我们来了解一下虚拟终端是怎么管理屏幕上的文字显示的。
虚拟终端的实现在 drivers/tty/vt/vt.c 。代表虚拟终端的数据是 struct vc。

::: highlight
``` c
struct vc{ 
     struct vc_data; 
     struct work_struct; 
 }; 
故而 struct vc_data 才是我們要的虛擬終端的定義。我們先來看看 struct vc_data 到底定義了什麼東西吧。
 struct vc_data 的定義在 include/linux/console_struct.h, 定義摘錄如下，爲了不延長篇幅，有省略的部分：
 struct vc_data { 
     struct tty_port port;           /* Upper level data */ 

     unsigned short      vc_num;             /* Console number */ 
     unsigned int    vc_cols;        /* [#] Console size */ 
     unsigned int    vc_rows; 
    省略 ... 
     const struct consw *vc_sw; 
     unsigned short *vc_screenbuf;  /* In-memory character/attribute buffer */
     unsigned int    vc_screenbuf_size; 
    省略 ... 
 }; 
```
:::

vc_screenbuf 存储了虚拟终端要显示在屏幕上的文字。

const struct consw \*vc_sw 指向控制台驱动提供的函数。

虚拟终端利用里面的函数指针调用相应的操作，比如重绘屏幕，绘制一个字符等等。这些操作由
vgacon 和 fbcon 等控制台驱动实现。
当你切换终端的时候，实际上就是把当前终端设置为你要切换过去的终端，并且重新绘制当前终端
vc_data-\>vc_screenbuf 存储的内容。
当你从键盘输入命令或者程序运行过程中要输出内容的时候，虚拟终端首先将输出的字符进行编码转化，转化为对于字符的
GLYPH 代码，并且将 GLYPH
和当前字符属性结合，最后将合成结果写入当前光标所处的位置。内核中实际的算法要复杂的多，还牵涉到中断，但是为了简单快捷的把我们关心的部分的核心表达出来，我使用一下伪代码表示不那么严谨的过程。希望了解全部的读者可以自行查看内核相关的代码，主要代码在
drivers/tty/vt.c 的 do_con_write() 中。

**清单 1. 伪代码**

::: highlight
``` text
vc_write(vc_data * vc, const char * string, int count){ 
            for( ; count ; ){ 
   /* 和当前编码有关，如果是 utf8 就以 utf8 方式解码不是 utf8 就按照 扩展的 ASCII 方式，也就是一个字节就是一个字母。*/
        int glyph = next_char(vc->utf,&string,&count); 
        int c = vc_build_attribute(vc)|(vc_glyph_mask& glyph); 
        // 把当前设置的前景色背景色等属性结合 ,glyph 不能超过描述它的位段 ; 

        vc->vc_screenbuf[vc->vc_pos] = c; // 写入当前位置
         update_pos(); // 更新当前光标位置
     } 
     notify_redraw(vc); // 调用 cosole 这个观察者重会屏幕
 }
```
:::

你也许想知道 vc_screenbuf 指向的缓冲区的格式到底是怎样的，现在也可以回答
前面提到过的为何图像控制台依然不能支持汉字显示的问题了： vc_screenbuf
的格式就是 VGA 文字模式时显卡所使用的文字缓冲区格式。上述伪代码中的
vc_glyph_mask 就是 0xFF, 也就是 glyph 被截断只能 8 比特长度。
打从一开始，fbcon 就是按照 VGA 字符发生器设计的。因为当 vc_screenbuf
的格式和 VGA 字符缓冲区的格式一致的时候，切换终端就可以只需要 memcpy
------快速的拷贝到 VGA
字符缓冲区就能实现"重绘"当前终端。现在看来这种做法局限性非常大，但是单年
PC 性能还不够强大的时候，能做到快速的重绘是非常重要的。
重绘例程中，notify_redraw(vc) 用伪代码表示为

**清单 2. notify_redraw(vc) 伪代码**

::: highlight
``` c
notify_redraw(vc_data * vc){ 
   for(int rows = 0 ; rows < vc->rows ; rows ++){ 
    unsigned short * current_line = & 
        vc->vc_screen_buf [vc_size_row * rows + vc->vc_visible_origin ] ; 
     // 在屏幕的第 row 行第 0 列绘制一行 current_line 指向的内容共 vc->cols 个字符。
     vc->vc_sw->con_puts(vc,current_line,vc->cols,0,rows); 
   } 
 }
```
:::

vc_sw 是个由控制台代码提供的指针，类型为 **struct**consw \* ,
控制台驱动的初始化部分会使用 vt_unbind()
将自己绑定为虚拟终端使用的控制台，传入的信息中就包括 vc_sw。 在 vgacon
中 vc_sw-\>con_puts 事实上就是将要显示的内容简单的拷贝到 VGA
的字符缓冲区。 对于 fbcon 而言则要复杂的多。 fbcon 提供的 con_puts 将
glyph 作为一个下标在 vc-\>vc_font
中找到对应的位图数据，然后拷贝到帧缓冲区。 TTY-\> 虚拟终端 -\> 控制台
-\> 屏幕的路径用一幅图片表达为：

**图 2. 路径**

<figure data-size="normal">
<div>
<img
src="https://pic1.zhimg.com/v2-af5c5ba784ea1a05fe129b876c7e30ee_1440w.jpg"
class="origin_image zh-lightbox-thumb" data-caption=""
data-size="normal" data-rawwidth="540" data-rawheight="358"
data-original-token="v2-aa3b5711c7967b5e99bc137bbb61fff1"
data-original="https://pic1.zhimg.com/v2-af5c5ba784ea1a05fe129b876c7e30ee_r.jpg"
width="540" />
</div>
</figure>

## 实现

## 图形模式控制台的改造

fbcon 将 glyph 作为下标到 vc_font 中获取位图数据，而 glyph 要么是一个
unicode （vc-\>utf=1 的时候，当然是被截断到 8 个比特）要么就是扩展的
ASCII 代码。 由于扩展 ASCII 只有 8 个比特位表示一个字符，所以只能请出
unicode 作为中文的数字表示。要想控制台能支持汉字显示，需要解决 3
个问题：

1.  必须使用 UTF-8 模式 ( 默认 vc-\>utf=1 即可 )
2.  虚拟控制台的 vc_screenbuf 必须修改以为 glyph 提供至少 16bit 的空间。
3.  图形控制台需要 vc_font包含更多的字符，不只是 255
    个，并提供代码绘制双倍宽度的中文字形，字体中的字符按照 UNICODE
    排列，这样 glyph 就是字符的 UNICODE 编码。

### 修改虚拟控制台

一开始，我的打算是 vc_screenbuf 修改为 unsigned long long\* 类型，32bit
给字符属性，分别表示 16bit 终端前景色和背景色。glyph 则拥有 31bit 的空间
, 因为汉字的宽度为双倍的英文字母 ，其中 1 bit 用来表示双字符宽度。比如
\'我\' 会表达为 两个 \'我\'，第二个\'我\'的最高位为
1：绘制任何字形的时候，只绘制字形的左半部分；如果发现最高位为 1
则绘制字体位图中的右半部分。这样同样的绘制代码可以适应英文字母和汉字。
写入 vc_screenbuf 的时候，
如果是双倍宽度的字符，需要同时写入两份，第二份的最高位置 1 就可以。 但是
vc_screenbuf
的格式已经被到处假定为每字符两个字节。如此修改导致牵一发动全身。许多艰涩难懂的代码都依赖
vc_screenbuf 是
**每字符两个字节**的设定，直接修改定义后，光是编译器能直接检测出来的就有百余个地方需要修改，还有更多的逻辑并不能被编译器检测出来。
如此修改的后果就是会出现许多隐晦的错误，非常难于调试。挣扎后，为最终选择了另一条道路
:

### 为汉字重新分配一块 vc_unicode_screenbuf

vc_unicode_screenbuf 紧挨着 vc_screenbuf , 事实上 vc_screenbuf
在分配空间的时候，多分配了一倍的空间，多分配的空间充作
vc_unicode_screenbuf，因此 struct vc_data 里并没有添加
vc_unicode_screenbuf 成员。 vc_unicode_screenbuf 同样为每字符 2
个字节，并不包含字符属性，所以 2 个字节如数用来保存 glyph。vc_screenbuf
格式未变，所以 vgacon 不需要修改，这就减少了大量的工作量。 向
vc_screenbuf 写入字符的时候，同时写入一份到 vc_unicode_screenbuf
。如果是汉字，由于其 glyph 大于 254 , 所以 vc_screenbuf 的那两个字符 (
汉字双倍宽度 ) 实际写入的是 0xff 和 0xfe ( 故而上文提到是 glyph 大于 254
的字符 ,0xfe 被保留它用了 )。0xff 表示该字符的 glyph 要到
vc_unicode_screenbuf 提取，然后绘制左半部分；0xfe 表示该字符的 glyph
要到 vc_unicode_screenbuf 提取，然后绘制右半部分。对于 glyph 大于 254
但是又不是双倍宽度的字符，就不需要 0xfe 作陪了。
比如屏幕上显示的文字是黑底白字的 "牛 B" , vc_screenbuf 的内容就是
"0x00ff, 0x0ffe, 0x0f42 " , vc_unicode_screenbuf 的内容则是 "牛 , 牛 ,b"。
这是因为一个汉字为两倍的英文字母宽度。在屏幕文字缓冲区上也必须占用两个字符的位置。并且必须有一种机制能知道应该绘制左半部分和右半部分，我使用的
就是 0xff 和 0xfe。

### 修改图形控制台绘制代码

要修改的地方只有 3 个。

1.  struct console_font 添加 charcount 成员。将主线内核的字体设置为
    charcount = 255。 主线内核带的字体都是 255 个 glyph
    的，所以没有添加字符个数的必要。不过我们即将要添加的字体会有数万字符。
2.  添加一个新的字体，复盖 UNICODE BMP 基本区域的所有符号。
3.  修改字符绘制代码，添加 vc_unicode_screenbuf 的支持。

字符绘制代码的修改比较繁琐，代码分布在 drivers/video/console/
下的多个文件中。fbcon_putc(s) 由由 vc-\>vc_sw-\>con_putc(s) 调用，
fbcon_putc(s) 转而调用分散于 drivers/video/console/ 的多个 puts
实现。因为终端要支持 console_rotate , decoration , timing ,
故而每种模式下的绘制实现都是不同的。 我拿 drivers/video/console/bitblt.c
最常用的不倾斜、不加装饰等的终端模式为例来讲解绘图部分的修改。
由于中文字体为 16x16 点阵，是对齐的字体，故而其绘制代码为
bit_putcs_aligned() 原先的代码以 glyph 为下标到 vc-\>vc_font-\>data
获得字体数据，然后调用 fb_pad_aligned_buffer 执行块拷贝操作。
我的修改很简单，原来获得字体数据的代码修改后放入 font_bits() 辅助函数。
在 font_bits 里，要判断 glyph 是否为 0xff 或者 0xfe, 如果不是，使用
glyph 为下标获得字体的左半部分后并返回。 如果是，则从
vc_unicode_screenbuf 获得真正的 glyph 数值，然后再依据现有的 glyph 是
0xff 还是 0xfe 去获得字体的右半部分还是左半部分返回。font_bits
获得字体数据后执行 fb_pad_aligned_buffer 块拷贝。 需要修改的地方还有
drivers/video/console/fbcon_ccw.c fbcon_cw.c fbcon_ub.c
。依原理进行修改即可。

## 总结

## 虚拟终端的不足之处

虽然费尽心机添加了中文支持，那只是一个 workaround（权宜之计）,
并不能算真正的支持。要真正的支持必须彻底重写虚拟终端和控制台。而要支持中文，就需要更进一步，全面支持
UNICODE , 包括支持从右向左的书写习惯。 在内核里实现一个全面支持 UNICODE
的控制台并不是一件容易的事情，何况内核的政策也不允许将如此庞大的字库装入内核。于是乎，这里出现了死胡同。KMS
和 Wayland 的出现让这死胡同似乎有了个完美的解。

- **KMS** ：

KMS 是内核模式设置 （Kernel Mode Setting）的缩写。传统上内核使用 VGA
模式，该模式由 BIOS 或者 bootloader 设置。如果启动 Xorg, 则 Xorg
使用自己的驱动将显示模式进行切换。这导致内核并不知道显卡的当前工作状态，虚拟终端切换必须依赖
X 进行。X 锁死会导致整个终端被锁定无法进行切换。待机、休眠等功能必须依靠
X
和内核双方进行深度合作才能实现。让一个用户程序搞垮内核是不可以接受的，故有
KMS , 希望通过把模式设置代码移入内核，减少内核对 X 的依赖。

如果不使用 X , KMS 对于控制台来说就是支持了显示器的本地本分辨率。KMS
优势并不显著。但是 Wayland 的介入使事情发生了变化。 有关于 Wayland
的详细文档请参考 freedesktop.org 上的 [项目首页](https://wayland.freedesktop.org/)。
Wayland 并不只是对桌面带来了福音，同时也为控制台带来了福音，因为 Wayland
可以代替内核自身的虚拟终端和控制台实现。 而这个代替者就是 ***System Compositor***

- **System Compositor** ？

System Compositor 是一个 wayland compositor，只是运行于系统全局范围。

为了懒人我这里稍微讲解一下 wayland compositor 吧。 Wayland 不同于 X , 在
wayland 的世界里，只有 compositor 和 client。Client 利用各种 API
(wayland 给出的示例使用的是 OpenGL ES, 但其实 wayland 并不限制使用的绘图
API 类型 ) 进行窗口绘图，然后将窗口的绘制结果直接提交给 compositor
合成到屏幕上。这样 wayland 本身就不包含绘图 API 而大大简化了 wayland
的设计。Wayland compositor 可以同 X
一样操作显卡向屏幕输出合成后的结果，也可以作为另一个 wayland compositor
的 client。

对于多账户同时登录的实现，固然可以让每一个本地 GUI 会话开启一个 wayland
compositor，但是存在更好的办法就是固定开启一个 system
compositor。而让所有用户会话的 wayland compositor 再作为 system
compositor 的 client. 藉由 system compositor
的合成效果，进行快速用户切换也可以进行一些视觉效果。而且 Xorg
本身也已经支持作为 wayland client 运行，这样可以使用传统的 X
提供桌面，而让 wayland system compositor 实现终端切换。
这还有一个好处，只有 wayland system compositor 是以 root
运行的，而用户会话的 compositor 或 X 就不必以 root 权限运行。 因为
Wayland 非常轻量，所以 system compositor
可以作为系统级服务常驻内存运行。而因为有了 system compositor ,
**内核也不再需要实现虚拟终端了**：只需要实现终端模拟器作为 system
compositor 的 client 。由于是在用户空间实现的，所有可以加入
UNICODE，矢量字体，国际化的书写习惯等等的支持，再也不用受限于内核啦。
Wayland 还是一个非常年轻的项目，Wayland system compositor
目前还只是设想中的概念，需要更多的人关注参与。笔者相信不久的将来 wayland
一定能大有作为。

## 写在最后

本篇简要介绍了 Linux
虚拟终端的工作原理和依赖的硬件细节实现，然后使用了不太优雅的办法让虚拟终端在帧缓存模式下实现汉字的显示。让大家对虚拟终端有了一点点更多的了解，希望本文能对想了解
Linux 的人有所帮助。也再次感谢 IBM DevelopWorks，
它让我有机会把自己的知识共享给更多的人知道。

关于原作者：蔡万钊, 自由职业

Linux 爱好者 , 喜欢使用 Gentoo 发行版，目前是 Gentoo-zh Overlay 维护者。喜欢捣鼓系统，是个爱折腾的人。

