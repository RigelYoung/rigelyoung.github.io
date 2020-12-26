---
layout:     post
title:      UNVEIL: A Large-Scale,Automated Approach to Detecting Ransomware论文阅读
subtitle:   USENIX2016的勒索病毒检测论文 个人向笔记
date:       2020-12-15
author:     Rigel
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - 勒索病毒 - 论文
---

# UNVEIL: A Large-Scale,Automated Approach to Detecting Ransomware

Ransomware typically operates by locking the desktop of the victim to render the system inaccessible to the user,or by encrypting,overwriting,or deleting the user’s files. 

In this paper, we present a novel **dynamic analysis system** called UNVEIL that is specifically designed to detect ransomware. The key insight of the analysis is that **in order to mount a successful attack, ransomware must tamper with a user’s files or desktop**. UNVEIL automatically **generates an artificial user environment, and detects when ransomware interacts with user data. In parallel, the approach tracks changes to the system’s desktop that indicate ransomware-like behavior**. 



## 1.introduction

Compared to traditional malware, ransomware exhibits behavioral differences. For example, traditional malware typically aims to achieve stealth so it can collect banking credentials or keystrokes without raising suspicion. In contrast, ransomware behavior is in direct opposition to stealth, since the entire point of the attack is to openly notify the user that she is infected.

Today, an important enabler for **behavior-based** malware detection is **dynamic analysis**. These systems execute a captured malware sample **in a controlled environment, and record its behavior (e.g., system calls, API calls, and network traffic)**. Unfortunately, malware detection systems that focus on stealthy malware behavior (e.g., suspicious operating system functionality for keylogging) might **fail to detect ransomware because this class of malicious code engages in activity that appears similar to benign applications that use encryption or compression**.   有可能只是正常使用的加密压缩操作。

In this paper,we present a novel dynamic analysis system that is designed to analyze and detect ransomware attacks and model their behaviors. In our approach, **the system automatically creates an artificial, realistic execution environment and monitors how ransomware interacts with that environment. Closely monitoring process interactions with the filesystem allows the system to precisely characterize cryptographic ransomware behavior**.    即创造一个理想环境，监测进程与filesystem的交互，从而能够识别是否为勒索病毒的加密行为。

**In parallel, the system tracks changes to the computer’s desktop that indicates ransomware-like behavior**. Our automated approach, called UNVEIL, **allows the system to analyze many malware samples at a large scale, and to reliably detect and flag those that exhibit ransomware-like behavior**. In addition, the system is able to provide insights into how the ransomware operates, and how to automatically differentiate between different classes of ransomware.             同时检测桌面是否有勒索病毒类似的修改（如变成勒索界面）。 并且大规模分析勒索病毒样本。



We implemented a prototype of UNVEIL in Windows on top of the popular open source malware analysis framework [Cuckoo Sandbox [13]](https://cuckoosandbox.org/). **Our system is implemented through custom Windows kernel drivers that provide monitoring capabilities for the filesystem**. Furthermore,**we added components that run outside the sandbox to monitor the user interface of the target computer system**.      作者在windows上实现了



In summary, this paper makes the following contributions:

- We present a novel technique to detect ransomware known as **ﬁle lockers** that targets files stored on a victim’s computer. Our technique is based on **monitoring system-wide filesystem accesses in combination with the deployment of automatically generated artificial user environments** for triggering ransomware. 
- We present a novel technique to detect ransomware known as **screen lockers**. **Such ransomware prevents access to the computer system itself**. Our technique is based on detecting locked desktops **using dissimilarity scores of screenshots** taken from the analysis system’s desktop **before, during, and after** executing the malware sample. 
- We performed a large-scale evaluation to show that our approach can effectively detect ransomware. We automatically detected and verified 13,637 ransomware samples from a dataset of 148,223 recent general malware（**从已经分析过的样本库中去检索**）. In addition, we **found one previously unknown ransomware sample that does not belong to any previously reported family**. Our evaluation demonstrates that our technique works well in practice (**achieving a true positive [TP] rate 96.3% at zero false positives**[FPs]),and **is useful in automatically identifying ransomware samples submitted to analysis and detection systems**.



### 文章架构

![image-20201214215143175](UNVEIL%20A%20Large-Scale,Automated%20Approach%20to%20Detecting%20Ransomware.assets/image-20201214215143175.png)



## 2.background



C&C 服务器的全称是 Command and Control Server，翻译过来就是命令和控制服务器



Our detection approach assumes that ransomware samples can and will use all of the techniques that other malware samples may use. In addition, our system assumes that successful ransomware attacks perform one or more of the following activities. 

- Persistent desktop message.       A popular technique is to call dedicated API functions (e.g., CreateDesktop()) to create a new desktop and **make it the default configuration to lock the victim out of the compromised system**.

  CreateDesktop [1] ，[Windows API](https://baike.baidu.com/item/Windows API)函数名，主要作用是创建新的、与调用进程的窗口站相关联的桌面，可用SetThreadDesktop将它分配给调用线程，并可以使用SwitchDesktop切换为当前桌面。

- Indiscriminate encryption and deletion of the user’s private files. 

- **Selective** encryption and deletion of the user’s private files **based on certain attributes (e.g., size, date accessed, extension)**.     to avoid detection      The sample can also inject malicious code into any Windows application to obtain this type of information (e.g., directly reading process memory). 



## 3.UNVEIL Design

 describe our techniques for detecting multiple classes of ransomware attacks.



### 3.1Detecting File Lockers

We first describe why our system creates a unique, artificial user environment in each malware run. We then present the design of the filesystem activity monitor and describe how UNVEIL uses the output of the filesystem monitor to detect ransomware.

#### 3.1.1Generating Artificial User Environments

an unrealistic looking user environment can be a telltale sign that the code is running in a malware analysis system.        **fingerprint就是通过环境中的一些数据去识别code运行环境，从而得知是否是真实用户环境**           如 static features inside analysis systems (e.g., name of a computer，user data)       attacker可以通过侦察攻击来识别环境

​	通常我们用行为签名或代码特征去识别恶意软件，那么恶意软件也可以通过行为签名去识别分析环境，从而躲避检查。       [恶意软件也有“指纹” 或催生新入侵侦测形式](http://www.cniteyes.com/archives/15968)

​	所以文章要求自动创建出不会被发现的人造分析环境。



##### 侦察攻击

https://www.sohu.com/a/200097297_100042196

**在发起攻击之前，黑客会对目标或目标群体进行广泛的侦查。**侦查包括探测网络的漏洞。攻击者将在网络的周边或局域网(LAN)内进行侦查。典型的侦查攻击探测使用了签名匹配技术，通过网络活动日志寻找可能代表恶意行为的重复模式。然而，基于签名的检测通常会产生一串嘈杂的假警报。

##### 签名匹配技术

https://www.secpulse.com/archives/75180.html

**行为签名**可以说是直接决定了Cuckoo究竟可以标记出多少恶意行为甚至是直接标记出样本所属的恶意软件家族

如：我们只提一个规则：creates_null_reg_entry.py。这条规则的目的是用来检出一些首字节为0x00的注册表项（该手段常用于恶意软件隐藏注册表痕迹，详情可以去网上自行搜索）。

​	即签名不只是恶意软件中的静态代码特征，还有行为特征，都可算作签名。

[cuckoo signatures](https://cuckoo.readthedocs.io/en/latest/customization/signatures/)

>With Cuckoo you’re able to create some customized signatures that you can run against the analysis results in order to identify some predefined pattern that might represent a particular malicious behavior or an indicator you’re interested in.
>
>These signatures are very useful to give a context to the analyses: both because they simplify the interpretation of the results as well as for automatically identifying malware samples of interest.
>
>Some examples of what you can use Cuckoo’s signatures for:
>
>- Identify a particular malware family you’re interested in by isolating some unique behaviors (like file names or mutexes).
>- Spot interesting modifications the malware performs on the system, such as installation of device drivers.
>- Identify particular malware categories, such as Banking Trojans or Ransomware by isolating typical actions commonly performed by those.
>- Classify samples into the categories malware/unknown (it is not possible to identify clean samples)
>
>You can find signatures created by us and by other Cuckoo users on our [Community](https://github.com/cuckoosandbox/community) repository.



#### 3.1.2Filesystem Activity Monitor

The filesystem monitor in UNVEIL has direct access to data buffers involved in I/O requests, giving the system full visibility into all filesystem modifications. Each I/O operation contains the process name, timestamp, operation type, filesystem path and the pointers to the data buffers with the corresponding entropy information in read/write requests.  The generation of I/O requests happens at the lowest possible layer to the filesystem. 

For example, there are multiple ways to read, write, or list files in user-/kernel-mode, but **all of these functions are ultimately converted to a sequence of I/O requests**. Whenever a user thread invokes an I/O API, an I/O request is generated and is passed to the filesystem driver. 

![image-20201220141744122](UNVEIL%20A%20Large-Scale,Automated%20Approach%20to%20Detecting%20Ransomware.assets/image-20201220141744122.png)



**UNVEIL’s monitor sets callbacks on all I/O requests to the filesystem** generated on behalf of any user-mode processes.



user mode process interactions with the filesystem are formalized as access patterns. We consider access patterns in terms of I/O traces, where a trace T is a sequence of ti such that 

![image-20201220155338480](UNVEIL%20A%20Large-Scale,Automated%20Approach%20to%20Detecting%20Ransomware.assets/image-20201220155338480.png)

把IO交互抽象成这样的模型，就会发现病毒样本中独特且重复性的pattern。这是因为每个样本都使用自己的IO策略去处理filesystem. 这样，可以通过IO模型去得知该病毒属于哪个家族。



##### I/O Data Buffer Entropy

通过对比计算buffer前后的熵，可以推算出是否为加密类勒索病毒行为。



##### **Constructing Access Patterns**

对于每个进程执行，根据filename和时间戳提取出IO trace，从而检查哪个进程access了哪些文件。 我们检测的标准就是识别write和delete.    只要有encrypt,overwrite或delete类似pattern，就认为是suspicious。



IO pattern可以归结为三类：

![image-20201220161927588](UNVEIL%20A%20Large-Scale,Automated%20Approach%20to%20Detecting%20Ransomware.assets/image-20201220161927588.png)

第二种只是unlink文件，并不删除。所以victim是可以直接恢复文件的。



###### unlink

一个文件是否存在取决于它的inode是否存在,你在目录里看到的是目录项里一条指向该inode的链接,而不是文件的本身.
  当你调用unlink的时候他直接把目录项里的该条链接删除了,但是inode并没有动,该文件还是存在的,这时候你会发现,目录里找不到该文件,但是已经打开这个文件的进程可以正常读写.只有当打开这个inode的所有文件描述符被关闭,指向该inode的链接数为0的情况下,这个文件的inode才会被真正的删除.



### 3.2 Detecting Screen Lockers

对比运行前后屏幕截图，判断是否有大部分的屏幕突然改变。         无恶意的换桌面怎么办？？

​	对比方法就是依靠像素间的关系提取出图片中物体的结构。

另外，还会提取图片中的文字，并识别与勒索相关的关键字。



## 4. Unveil Implementation

 We elaborate on how UNVEIL automatically generates artificial, but realistic user environments **for each analysis run**, how the **system-wide monitoring** was implemented, and **how we deployed** the prototype of our system.



### 4.1 Generating User Environments

对于每个样本执行，每个文件扩展名生成随机数量的对应类型文件。

另外，需要添加一些额外的属性去使勒索病毒识别分析环境更加困难。

#### **valid content**  

首先要保证生成的文件的有效性。比如正确的header和内容。   可以通过标准库(e.g., python-docx, python-pptx, OpenSSL)去生成这些文件。    

​	依据经验，勒索病毒喜欢加密4类文件：documents, keys and licenses, file archives, and media.

- Document extensions include txt, doc(x), ppt(x), tex, xls(x), c, pdf and py. 
- Keys and license extensions include key, pem, crt, and cer. 
- Archive extensionsinclude zip and rar files. 
- media extensions include jp(e)g, mp3, and avi. 

每次生成这些扩展名类型的子集文件。

​	其次，利用google搜索结果来当成sentence list，去填充file content；利用meaningful word list去命名文件，使得attacker难以识别这是自动生成的文件。



#### File Paths

对于目录结构，也要伪造的真实一点。

- 同样利用meaningful word list命名
- 不同类型的文件放到有要求的目录下 (e.g., the system does not create document files in a directory with image files, but rather under My Documents)
- 目录树的depth是不固定的



#### Time attributes

创建文件带有不同的creation, modiﬁcation, and access times，并且对应的目录的创建时间为所有文件的最小值，修改时间为所有文件的最大值。



### 4.2 Filesystem Activity Monitor

对于api hook，通过 System Service Descriptor Table (SSDT)去hook filesystem api或system calls不适合我们的要求。原因如下：

- api hook 可以通过复制一个有目标代码的dll并动态加载它，从而绕过hook  
  - 同样的文件，第二次载入内存时，肯定是需要rebase的。如果你查一查，第二次载入的kernel32.dll的基地址应该可以在它的relocation table里找到。   https://bbs.pediy.com/thread-35752.htm
- windows是有crypto api的，可以用这些api去加密。https://blog.csdn.net/kamaliang/article/details/6608786        单纯hook这些api，attacker完全可以自己实现加密功能，而不用这些api
- 另外，Hooking system calls via the SSDT有其他限制。 it is prevented on 64bit systems due to Kernel Patch Protection (KPP). 
-  Furthermore, most SSDT functions are undocumented and subject to change across different versions of Windows.         所以不通用



#### api hook

hook也是分操作系统层级的，在每个层级都可以hook



所以Unveil使用Windows Filesystem Minifilter Driver framework.

https://blog.csdn.net/forcj/article/details/103148406/



### 4.3 Desktop Lock Monitor

For dissimilarity testing, a python script implements the Structural Similarity Image Metric (SSIM).

The system also employs Tesseract-OCR [38], an open source OCR engine, to extract text from the selected areas of the screenshots. 





## 5. Evaluation

### 5.1 setup

在环境中做了一些防止虚拟机检测的措施：

- changing the IP address range and the MAC addresses of the VMs to prevent the VMs from being fingerprinted by malware authors.
-  permitted controlled access to the Internet via a filtered host-only adapter
- SMTP traffic was redirected to a local honeypot to prevent spam, and network bandwidth was limited to mitigate potential DoS attacks.

每运行完一个sample，就把vm回退到初始状态。



非恶意软件也可能会有加密和压缩，导致产生和勒索病毒相似的IO trace。这里文章指出，非恶意软件因为最终目的是产生一个加密/压缩的文件版本，并不会影响原文件，所以原文件还是完整的。若用户故意自动删除原文件，是有可能产生误报的。   同样，非恶意软件可能会替换文件中的内容，如制作ppt，但是正常软件都是一次修改一个文件，并不会批量修改多个文件。



Unveil是运行在内核层，目标是检测用户层的勒索病毒。这有一个风险：勒索病毒有可能也运行在内核态并破坏Unveil用户监视filesystem的hook。但是这需要勒索病毒通过特权去加载内核代码，或利用内核漏洞。 目前为止，勒索病毒大多都在用户态，因为这已经足够attack了。



#### Dataset

DataSet是[ADO.NET](https://baike.baidu.com/item/ADO.NET)的中心概念。可以把DataSet当成内存中的数据库，DataSet是不依赖于数据库的独立数据集合。

所谓独立，就是说，即使断开[数据链路](https://baike.baidu.com/item/数据链路/7181323)，或者关闭数据库，DataSet依然是可用的，DataSet在内部是用XML来描述数据的，由于XML是一种与平台无关、与语言无关的[数据描述语言](https://baike.baidu.com/item/数据描述语言/6814688)，而且可以描述复杂关系的数据，比如父子关系的数据，所以DataSet实际上可以容纳具有复杂关系的数据，而且不再依赖于数据库链路。

DataSet 是 ADO. NET结构的主要组件，它是**从[数据源](https://baike.baidu.com/item/数据源)中检索到的数据在内存中的缓存**。DataSet 由一组 DataTable 对象组成，可使这些对象与 DataRelation 对象互相关联。还可通过使用 UniqueConstraint 和 ForeignKeyConstraint 对象在 DataSet 中实施[数据完整性](https://baike.baidu.com/item/数据完整性)。 [1] 

正是由于DataSet才使得程序员在编程时可以屏蔽数据库之间的差异，从而获得一致的编程模型。DataSet支持多表、表间关系、数据约束等，和关系数据库的模型基本一致。

