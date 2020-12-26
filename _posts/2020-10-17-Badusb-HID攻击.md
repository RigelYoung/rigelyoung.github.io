---
layout:     post
title:      Badusb HID攻击
subtitle:   为协会迎新时准备的小技术展示
date:       2020-10-17
author:     Rigel
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Badusb - HID
---

# HID攻击

“BadUSB”是2014年计算机安全领域的热门话题之一，该漏洞由Karsten Nohl和Jakob Lell共同发现，并在2014的BlackHat安全大会上公布。    

### 简介

通过硬件直接插入对方电脑，让对方电脑执行代码，达到干扰、控制主机或者窃取信息等目的。

### 威胁

​	BadUSB的威胁在于：恶意代码存在于U盘的固件中，PC上的杀毒软件无法访问到U盘存放固件的区域，因此也就意味着杀毒软件和U盘格式化都无法应对BadUSB的攻击。

​	这里并不是严格意义上的badusb。严格意义上的badusb是对通用的USB设备（比如U盘）的固件进行重新烧录；下面的是digispark的开发板。

![-1708090750](HID%E6%94%BB%E5%87%BB.assets/-1708090750.jpg)



## HID攻击

​	HID是Human Interface Device的缩写，由其名称可以了解HID设备是直接与人交互的设备，例如键盘、鼠标与游戏杆等。不过HID设备并不一定要有人机接口，只要符合HID类别规范的设备都是HID设备。一般来讲针对HID的攻击主要集中在键盘鼠标上，因为只要控制了用户键盘，基本上就等于控制了用户的电脑。**攻击者会把攻击隐藏在一个正常的鼠标键盘中，当用户将含有攻击向量的鼠标或键盘，插入电脑时，恶意代码会被加载并执行。**

​	现在的USB设备很多，比如音视频设备、摄像头等，因此要求系统提供**最大的兼容性，甚至免驱**；所以在设计USB标准的时候没有要求每个USB设备像网络设备那样占有一个唯一可识别的MAC地址让系统进行验证，而是**允许一个USB设备具有多个输入输出设备的特征**。这样就可以通过**把badusb的HID描述符改成键盘描述符，伪装成一个USB键盘，并通过虚拟键盘输入集成到U盘固件中的指令和代码而进行攻击。**



正常的U盘插到电脑里的大概流程是下面这样的

```
           电脑识别为U盘的固件
固件信息-------------------------->U盘----->读取U盘的内容----->显示示U盘的内容Copy
```

构造badusb的固件让电脑识别为键盘

```
           电脑识别为键盘的固件
固件信息-------------------------->键盘Copy
```

再构造里面的内容

```
键盘----->模拟人敲命令----->根据写好的脚本敲命令
```



## HID简介

https://zhuanlan.zhihu.com/p/37659947

HID 全称为 Human Interface Device 直译为人类接口设备，也被称为人体学输入设备，是指与人类直接交互的计算机设备，而pc端上的”HID”一般指的是USB-HID标准，更多指微软在USB委员会上提议创建的一个人体学输入设备工作组。   不过HID设备并不一定要有人机接口，只要符合HID类别规范的设备都是HID设备。

其实就是设备管理器里的…这个

![img](HID%E6%94%BB%E5%87%BB.assets/v2-f51a69237552c664d455738f51ef7650_720w.jpg)

![img](HID%E6%94%BB%E5%87%BB.assets/v2-f94d9cb60f74c032cb0a606cd2fbecd3_720w.jpg)

**1.1HID标准的概念之前，**设备往往要匹配鼠标和键盘,更换新的设备时候，就需要重载所有协议和设备，极其繁琐

类似于TCP/IP协议族一样，在标准化协议发布之前，设备之间的通讯由开发商单方面的协议主导，不同品牌的设备从而因为协议不同，而无法通信。

**1.2而HID标准概念之后，**所有HID定义的设备驱动程序提供可包含任意数量数据类型和格式的自我描述包。计算机上的单个HID驱动程序就可以解析数据和实现数据[I/O](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/I/O%22%20%5Co%20%22I/O)与应用程序功能的动态关联。

**1.3可以说：**HID标准的诞生让接口类型，功能更加丰富，多样化，同时也加快了设备的创新与发展。  USB HID设备的一个好处就是**操作系统自带了HID类的驱动程序**，而用户无需去开发驱动程序，只要使用API系统调用即可完成通信。



**2.1那么造成HID攻击的原因到底是什么**

**HID设备类协议缺陷**

在聊HID设备标识符之前，先说说鼠标和键盘。早期的鼠标键盘，如果网龄久一些的话，都知道，是那种PS/2的接口。

![img](HID%E6%94%BB%E5%87%BB.assets/v2-9cca355a8a224576d41fa1209cc6fac2_720w.jpg)

在今天，大部分的鼠标和键盘都用USB来代替，ps/2的接口已经非常非常少见了。

既然，USB插得有可能是移动储存设备，也有可能是类似键盘鼠标的输入设备，那么计算机是如何分辨的呢？

答案就是 **HID协议中的定义的HID设备标识符**。

计算机不是人，他只认识0和1，Device Class Definition for Human Interface Devices是一个公开的国际标准，用于规定HID设备的类型。当任何一个HID设备在接入电脑时，操作系统就首先会读取其设备标识符。

**你如果具备U盘的标识，那我就给你挂载一个盘符。**

**你如果具备键盘的标识符，我就接受你给我输入的信息。**

原则上HID 是一出厂，就被设定好了的，出厂了就不能再更改了。

老生常谈的一句话“**互联网本身是安全的，自从有了安全的人，互联网就变得不安全了。**”

但是HID规范和协议是公开，通过自定义HID标识，让计算机模拟了键盘的输入。

则达到了我们所说的HID攻击。



## HID协议

https://baike.baidu.com/item/USB-HID/3074554?fr=aladdin

1、交换的数据存储在称为报表(report)的结构内，设备的[固件](https://baike.baidu.com/item/固件)必须支持HID报表的格式。主机在控制与中断传输中传送与要求报表，来传送与接收数据。报表的格式非常有弹性，可以处理任何类别的数据。

2、每一笔[事务](https://baike.baidu.com/item/事务)可以携带小量或中量的数据。低速设备每一笔事务最大是8个字节，全速设备每一笔事务最大是64个字节，高速设备每一笔事务最大是1024个字节。一个报表可以使用多笔事务。

3、**设备可以在未预期的时间传送信息给[主机](https://baike.baidu.com/item/主机)**，例如键盘的按键或是鼠标的移动。所以**主机会定时轮询设备，来取得最新的数据（这里是主机给USB设备发送信号来要数据。     但是中断的原理也是CPU轮询！！  CPU中有中断暂存器，记录到来的中断信号。CPU每执行完一条指令去检查有无中断信号产生）**。



https://blog.csdn.net/zhoutaopower/article/details/82469665

### USB HID设备类的通信管道

　　 所有的HID设备通过USB的控制管道（默认管道，即端点0）和中断管道（端点1或端点2）与主机进行通信。

　　管道　　　　　　　　要求　　　　　　说明

　　控制（端点0）　　　 必须　　　　　　传输USB描述符、类请求代码以及供查询的消息数据

　　中断输入　　　　　　必须　　　　　　传输从设备到主机的输入数据

　　中断输出　　　　　　可选　　　　　　传输从主机到设备的输出数据

注：USB主机为PC，USB设备如鼠标等。中断端点的描述中，指定了主机轮询的时间 Interval。即，主机会隔一段时间来“要”数据。



控制管道主要用于下面3个方面

- 接收/响应USB主机的控制请求以及相关的类数据
- 在USB主机查询时传输数据（如响应Get_Report请求等）
- 接收USB主机的数据

中断管道主要用于下面两个方面

- USB主机接收USB设备的异步传输数据
- USB主机发送有实时性要求的数据给USB设备

　　从USB主机到USB设备的中断输出数据传输是可选的，当不支持中断输出数据时，USB主机通过控制管道将数据传输给USB设备。



### 与USB HID设备有关的描述符

HID设备的描述符除了5个USB的标准描述符（设备描述符、配置描述符、接口描述符、端点描述符、字符串描述符）外，还包括三个HID设备类特定的描述符：HID描述符、报告描述符、实体描述符。

他们之间的层次关系如图：

![img](HID%E6%94%BB%E5%87%BB.assets/20180906232921460)

　　除了HID的三个特定描述符组成对HID设备的解释外，5个标准描述符中与HID设备有关的部分有：

- 设备描述符中：bDeviceClass, bDeviceSubClass, bDeviceProtocol

  **Subclass Code Description
    0  No Subclass
    1  Boot Interface Subclass
    2 - 255  Reserved**

- 接口描述符中：bInterfaceClass的值必须时0x03, bInterfaceSubClass的值为0或1， 为1表示HID设备是一个启动设备（**Boot Device， 一般对PC机有意义，意思是BIOS启动时能识别您使用的HID设备，只有标准鼠标或者键盘才能称为Boot Device**），为0表示HID设备是操作系统启动后才能识别使用的设备。bInterfaceProtocol的取值含义如下：

　　bInterfaceProtocol的取值（十进制）　　　　含义

　　0　　　　　　　　　　　　　　　　　　　　NONE

　　1　　　　　　　　　　　　　　　　　　　　键盘

　　2　　　　　　　　　　　　　　　　　　　　鼠标

　　3-255　　　　　　　　　　　　　　　　　    保留



下面分别对3个HID设备类特定描述符进行说明：

HID相关描述符类型定义

描述符类型值　　　　　　HID相关描述符类型

0x21　　　　　　　　　  HID描述符

0x22　　　　　　　　　  报表描述符　　

0x23　　　　　　　　　  实体描述符

#### 1.HID描述符

　　HID描述符关联于接口描述符，因而如果一个设备只有一个接口描述符，则无论它有几个端点描述符，HID设备只有一个HID描述符。**HID设备描述符主要描述HID规范的版本号, HID通信所使用的额外描述符， 报表描述符的长度等**。下表为HID描述符的结构。

偏移量　　　　域　　　　　　　　大小　　　　值　　　　描述

0　　　　　　bLength　 　　　　1　　　　　数字　　　 此描述符的长度，以字节为单位

1　　　　　　bDescriptorType   1　　　　　常量　　　描述符种类（此处0X21为HID类）

2　　　　　　bcdHID　　　　　　2　　　　　数字　　　HID规范版本号（BCD码），采用4个16进制的BCD格式编码，如版本1.0，0x0100，版本1.1，0x10110

4　　　　　　bCountryCode　　 1　　　　　 数字　　　硬件目的国家的识别码

5　　　　　　bNumDescriptors　1　　　　　 数字　　　支持的附属描述符数目

6　　　　　　bDescriptorType　  1　　　　　常量　　　HID相关描述符的类型，见下表

7　　　　　　wDescriptorLength  2　　　　数字　　　　报告描述符的总长度

9　　　　　　bDescriptorType　　1　　　　　常量　　　用于识别描述符类型的常量，使用有一个以上描述符的设备

10　　　　　 wDescriptorLength　2　　　　　数字　　　描述符总长度，使用在有一个以上描述符的设备 

#### 2.报告描述符

　　HID设备的报告描述符比较复杂，也比较难理解。报告描述符，其实就是告诉主机，通过中断端点传输的数据，哪些位或者哪些 Bytes ，代表着什么含义。

　　报告描述符的语法不同于USB标准描述符，它是以项目（item）方式排列而成，没有固定长度。HID的报告描述符已经不是简单的描述某个值对饮过的固定意义了，它已经能够组合出很多种情况，而且需要PC上的HID驱动程序提供parser解释器来对描述符的设备情形进行重新解释，进而组合生成本HID硬件设备独特的数据流格式，所以可以把它理解为“报告描述符脚本语言”更为贴切。我们使用“报告描述符”专用脚本语言，让用户来自己定义它们的HID设备都有什么数据，以及这些数据各个位（bit）都有什么意义。

　　有关报告描述符的详细信息可以参考USB HID协议，USB协会提供了一个HID描述符编辑工具称作HID Descriptor Tool，用它可以方便生成我们的报告描述符。

　　一个USB HID设备可以包含多种功能的报告描述符合集，这样可以实现复合设备，如带鼠标功能的USB键盘，这种复合键盘可以通过在报告描述符里包含鼠标和键盘两种报告实现，两个报告用报告ID来区分。

##### 报告（Report）和报告描述符

报告描述等于是告诉了 Host 端，将以什么样的形式进行数据传送，报告描述符赋予了数据的意义，让 HOST 能够在收到裸的数据的时候，根据报告描述符进行分解到正确的数据，并根据 Usage 来进行正确的操作。

比如：上述例子中，以 3 个 Bytes 进行上报：（没有报告ID）

如果鼠标左键按下，则返回 01 00 00（十六进制值）

如果鼠标右键按下，则返回 02 00 00

如果鼠标中键按下，则返回 04 00 00

如果三个键同时按下，则返回07 00 00。

如果鼠标往右移动则，第二字节返回正值，值越大移动速度越快。其它的类推。

#### 3.实体描述符

　　实体描述符被用来描述设备的行为特性。实体描述符是可选的描述符，HID设备可以根据其本体的设备特性选择是否包含实体描述符。



### USB HID类命令（请求）

| 偏移量 | 域            | 大小 | 说明                                                         |
| ------ | ------------- | ---- | ------------------------------------------------------------ |
| 0      | bmRequestType | 1    | HID设备类请求特性如下： 位7： 0＝从USB HOST到USB设备 1＝从USB设备到USB HOST 位6~5： 01＝请求类型为设备类请求 位4~0： 0001＝请求对象为接口（interface）因而，针对HID的设备类请求，仅仅10100001和00100001有效 |
| 1      | bRequest      | 1    | HID类请求（参考表10）                                        |
| 2      | wValue        | 2    | 高字节说明描述符的类型（参考表5），而低字节为非0值时被用来选定实体描述符。 |
| 4      | wIndex        | 2    | 2字节数值，根据不同的bRequest有不同的意义                    |
| 6      | wLength       | 2    | 该请求的数据段长度                                           |

| 数值 | HID类请求描述符 | 注释                                           |
| ---- | --------------- | ---------------------------------------------- |
| 0x01 | GET_REPORT      |                                                |
| 0x02 | GET_IDLE        |                                                |
| 0x03 | GET_PROTOCOL    | 仅仅适应于支持启动功能的HID设备（Boot Device） |
| 0x09 | SET_REPORT      |                                                |
| 0x0A | SET_IDLE        |                                                |
| 0x0B | SET_PROTOCOL    | 仅仅适应于支持启动功能的HID设备（Boot Device） |

　　**USB主机在请求HID设备的配置描述符时，设备需要按照顺序返回下面几种描述符：配置描述符， 接口描述符， HID描述符， 端点描述符。**HID描述符里又包含了其附属的描述符的类型和长度（如报告描述符），然后主机再根据HID描述符的信息请求其相关的描述符。



总的来说，就是根据协议来识别设备和传输数据。



## Digispark代码编写

打开arduino，新建一个项目，会看到如下代码

```c
void setup() {
  // put your setup code here, to run once:
}
void loop() {
  // put your main code here, to run repeatedly:
}
```

光看这个可能理解的不够彻底，可以看看arduino的主函数

```c
#include <Arduino.h>
int main(void)
{
init();
setup();
for (;;) {
loop();
if (serialEventRun) serialEventRun();
}
return 0;
}
```

这就是arduino的大框架，分为setup和loop,解释的也很明显，一个是只执行一次，一个是执行很多次,先执行setup里面代码，再执行loop里面的循环



具体信息都写在头文件中。我们在实现文件中，只是来实现头文件中声明的一些函数。

一上来先引用键盘通讯事件的头

```bash
#include<Keyboard.h> 
```

一头一尾可以看到begin和end

```bash
Keyboard.begin();//键盘事件开始
Keyboard.end(); //键盘事件结束，在此期间的键盘操作才是有效的
```

接下来是一个延时操作，因为有时候由于系统的原因可能出现卡顿，如果操作后没有停顿可能造成执行错误，所以推荐延时使用，延时的基本单位是毫秒，这里的3000毫秒就是3秒

```bash
 delay(3000)；//延时3秒
```

接下来是一个press命令，该命令作用是按下一个建并且持续按住，一直按倒结束命令为止，而key_leaf_gui就是键盘上的左边的start键，也就是win键

```bash
  Keyboard.press(KEY_LEFT_GUI);//按下一个键并持续按住
```



### Keyborad库的分析

![image-20201126154350485](HID%E6%94%BB%E5%87%BB.assets/image-20201126154350485.png)



首先是HID描述符

```c
static const uint8_t _hidReportDescriptor[] PROGMEM = {
  //  Keyboard
    0x05, 0x01,                    // USAGE_PAGE (Generic Desktop)  // 47
    0x09, 0x06,                    // USAGE (Keyboard)
    0xa1, 0x01,                    // COLLECTION (Application)
    0x85, 0x02,                    //   REPORT_ID (2)
    0x05, 0x07,                    //   USAGE_PAGE (Keyboard)
    
  	0x19, 0xe0,                    //   USAGE_MINIMUM (Keyboard LeftControl)
    0x29, 0xe7,                    //   USAGE_MAXIMUM (Keyboard Right GUI)
    0x15, 0x00,                    //   LOGICAL_MINIMUM (0)
    0x25, 0x01,                    //   LOGICAL_MAXIMUM (1)
    0x75, 0x01,                    //   REPORT_SIZE (1)
    
  0x95, 0x08,                    //   REPORT_COUNT (8)
    0x81, 0x02,                    //   INPUT (Data,Var,Abs)
    0x95, 0x01,                    //   REPORT_COUNT (1)
    0x75, 0x08,                    //   REPORT_SIZE (8)
    0x81, 0x03,                    //   INPUT (Cnst,Var,Abs)
    
  0x95, 0x06,                    //   REPORT_COUNT (6)
    0x75, 0x08,                    //   REPORT_SIZE (8)
    0x15, 0x00,                    //   LOGICAL_MINIMUM (0)
    0x25, 0x73,                    //   LOGICAL_MAXIMUM (115)
    0x05, 0x07,                    //   USAGE_PAGE (Keyboard)
    
  0x19, 0x00,                    //   USAGE_MINIMUM (Reserved (no event indicated))
    0x29, 0x73,                    //   USAGE_MAXIMUM (Keyboard Application)
    0x81, 0x00,                    //   INPUT (Data,Ary,Abs)
    0xc0,                          // END_COLLECTION
};

Keyboard_::Keyboard_(void) 
{
	static HIDSubDescriptor node(_hidReportDescriptor, sizeof(_hidReportDescriptor));
	HID().AppendDescriptor(&node);
}
```

其他函数都在库内，看源码就能理解。主要就是，调用了库，那么对应的HID描述符也会写进去，最终烧录。 还有一个更基础的HID库。



