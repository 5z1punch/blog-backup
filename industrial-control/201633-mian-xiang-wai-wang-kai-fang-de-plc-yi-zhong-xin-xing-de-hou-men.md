# 2016-3-3-面向外网开放的PLC - 一种新型的后门

title: 面向外网开放的PLC - 一种新型的后门 date: 2016-03-03 categories: industrial\_control

### tags: \[industrial, program\]

## 日他妈，当时投drops那份带自己的分析的找不到md文件了，只剩这个纯翻译的了，觉得这个看的蛋疼可以去drops镜像上找一篇相似的文章。那个带了一点点分析。

> 摘要：工业控制系统（ICS）是在生产和控制过程中不可或缺的组件。我们的现代化基础设施在很大程度上依赖于他们。不幸的是，从安全角度来看，数以千计的PLC以面向公网的方式被部署。而拥有安全特性的PLC基本上不存在。尽管某些PLC已经具备安全防护功能，但是他们往往被忽视或者禁用，因为安全性往往会降低运营效率。因此，我们常常是可能加载任意代码到面向公网的plc。除了在权限控制上的严重问题，攻击者有可能利用plc作为一个进入生产网络甚至公司内网的网关。在本文中，我们分析和讨论这一威胁载体，并且，我们将证明，这种利用方式是真实可行的。出于演示的目的，我们开发一个运行在plc上的端口扫描器和一个socks代理。这个扫描器和代理使用plc的原生编程语言Statement List（STL）编写。我们的研究洞察到，什么样的动作攻击者可以轻松地执行，而哪些是不容易在plc上实现的。

### 1. 引言

工业控制系统（ICS）是在生产和控制过程中不可或缺的组件。我们的现代化基础设施在很大程度上依赖于他们。智能制造（工业4.0）技术的引进进一步增加了对工业控制系统\[1\]的依赖性。现代基础设施已经在遭受攻击，并且这些设施提供了从简单的xss漏洞，到重大的协议设计缺陷这样广阔的攻击范围。

对工控系统攻击的典型案例是臭名昭著的Stuxnet蠕虫病毒，该病毒针对伊朗铀浓缩设备进行攻击。然而越来越多的攻击者将目标放在一般的生产系统\[6\]。比如最近一个例子，2014年一家德国钢厂的高炉强制关机。攻击者通过钢厂的业务网络获得了生产控制系统的权限\[7\]。这是一个典型的攻击媒介，因为企业网络为员工服务，而员工很容易收到网络钓鱼等等攻击的威胁。

除了针对业务逻辑的网络钓鱼，在更多的情况下，甚至有更简单的方式进入产业控制系统。扫描数据显示，数以千计的ICS（比如plc）可以直接公网访问\[8,9,10\]。尽管在这种公网直接访问的方式下，只有一个生产设备的plc可以被我们访问到，然而该plc可能连接到生产内网，而这个内网中将会有更多的plc。这就是我们所说的"deep"产业网络。本文中，我们研究攻击者如何通过公网plc访问到深层工业网络。

我们采取的方法是将plc变成网关（本文采用西门子系列plc相关技术和特性），这种方法在缺乏适当权限认证手段的plc上是可行的。经验丰富的攻击者拥有某plc的访问权限时，可以往上面上传或者下载代码，只要代码是由MC7字节码组成，这是plc的原生代码形式。我们研究了运行时环境中的plc，并发现可以通过上传mc7代码来实现很多网络服务。特别是，我们实现了

* 一个针对西门子plc的 `SNMP` 扫描器
* 一个功能上完全成熟的，为西门子plc编写的，`SOCKS` 代理

并且他们的实现完全只依靠编译为MC7字节码的`STL`语言代码。我们的扫描器和代理可以部署在plc中，并且不会中断plc中原有程序的运行，这可以使运维很难意识到plc已被感染。为了说明和分析深层工业网络入侵，我们开发了一个概念性证明工具，**PLCinject**（附上github项目地址: [SCADACS/PLCinject](https://github.com/SCADACS/PLCinject)）。根据我们的概念性证明，xxxxxxxx（这段太tmd复杂，我实在没法准确翻译，主要意思就是讲运行在plc上的恶意软件会使其原有代码扩展增加，如果我们**定时**观测原有代码和感染恶意代码后的程序，在统计学上这两者的运行效果有明显的差异，然而其对生产过程的影响微乎其微，除非运营者主动监控从PLC中发出的恶意访问的流量，否则很难在生产过程中发现）。此外，对手可以利用我们的方法，通过工业控制网络来攻击企业的业务网络。这意味着网络管理必须警惕从业务网络正面和背面发起的双向攻击。 本文其余部分组织如下：

* 我们首先在第 **2** 部分讨论我们做的相关工作。
* 在第 **3** 部分，我们给不熟悉工控系统的读者普及技术背景。
* 在第 **4** 部分，我们描述了我们的攻击和入侵方法。
* 第 **5** 部分，我们讨论解决方案。
* 第 **6** 部分，总结全文。

### 2. 相关工作

针对plc的各种攻击已经出现，大多数攻击目标是plc的操作系统。相比之下，我们的着眼于运行在plc上的逻辑程序的威胁。因此，我们不使用任何意想不到的手段。下文中，我们将我们的做法和其他已发布的并广为人知的攻击方法相比较。 Beresfords在美国2011年black hat大会上层发表过关于针对scada攻击的描述。他演示了如何从远程的内存镜像中提取认证凭据。此外，他还展示了如何通过重放攻击开启关停plc（这个其实特别简单，然而2011年确实还是新技术233333）。与我们的攻击方式相反，他不会改变plc的逻辑程序。在2011年，Langner发布了“十四个字节的定时炸弹”\[11\]，其中他介绍了如何注入恶意逻辑代码到plc。他和我们一样，从Stuxnet病毒中借用了相关的代码和技术。他阐明了如何使原有代码失去对设备的控制。相比之下，我们的程序与原有代码并行运行，以达到不干扰原有代码执行的目的。还有一种与Langners类似的攻击手法，由Meixell和Forner发表在2013年美国black hat\[12\]。他们描述了实现plc利用（exploit）的不同方式。其中有一些从逻辑代码中删除安全检查的方法。然而我们的做法有一些不同，因为我们要在添加新功能的同时保留原有的功能。据我所知，第一篇关于plc恶意软件的学术论文是由McLaughlin发表于2011年\[13\]。在这项工作中，他提出了一个动态负载生成的基本方法。他提出了一种基于符号执行的方法，来从plc的逻辑代码中恢复出布尔逻辑。以此为出发点，他视图确定对于plc的不安全状态，并且生成可以触发某个不安全状态的代码。2012年，McLaughlin发表了一篇跟进的文章\[14\]，该文扩展了他原有的方法，他通过模型检查的方式将代码规范到一个预定义的模型。在他的模型中，他可以指定所需的行为，并自动生成攻击代码。McLaughlin在他的工作中着重于操纵一个plc的控制流程。而相比之下，我们将plc作为一个进入内网的网关，并使其原本的功能保留不变。

### 3. 工业控制系统

 上图展现了典型的使用自动化系统的公司结构。工业控制系统由这么几层构成。在顶部是企业资源规划（ERP）系统，其保存着当前可用资源和生产能力的相关数据。制造执行系统（MES）能够管理多个工厂或平台，并且从ERP系统接受任务。在MES下的系统位于工厂内部，监督、控制和数据采集（SCADA）系统控制生产线。他们提供关于目前生产状态的数据，并且他们提供干预手段。存储着有关生产过程的逻辑的设备称为可编程逻辑控制器（PLC）。我们在`3.1`部分更详细地解释他们。人机交互界面（HMI）显示当前的进度，并且允许运营者与生产过程相互作用。

#### 3.1 PLC硬件

一个PLC由一个中央处理单元（CPU），这附加着一些数字量和模拟量的输入和输出。一个存储在集成内存或者一个外挂多媒体卡（MMC）中的PLC程序，定义了如何控制输入和输出。plc有个特性，它能保障工业设备按照定义的执行时间精确地运作。对于通信或者专用应用程序，CPU的功能能够模块化扩展。我们在实验中使用的 **西门子 S7-314C-2 PN/DP** 拥有24位输入，16位输出，5个模拟输入，2个模拟输出和一个MMC插槽。它配备192KB的内部存储器，其中64KB可用于永久存储。此外，该PLC有一个**RS485**和两个**RJ45**接口\[16\]。

#### 3.2 PLC执行环境

西门子PLC运行着实时操作系统，他初始化周期性时间监视。随后操作系统周期性执行四个步骤，如下图：  在第一步中，CPU复制过程镜像的输出值来输出模块的状态。第二步，CPU读取输入模块的状态，并且更新过程映像的输入值。第三步，用户程序在时间片中执行1毫秒的持续时间。每个时间片被分割成三个部分，依次执行：操作系统，用户程序和通信。时间片的个数取决于当前的用户程序。默认情况下，时间应该不长于150毫秒，工程师可以配置不同的值。如果规定的时间用尽，中断例程被调用，在通常情况下CPU返回到周期的开始状态，并重新开始循环时间监视。\[17\]

#### 3.3 软件

西门子为工程师提供他们的完全集成自动化（TIA）门户软件，用于开发PLC。它由两个主要部分组成。`STEP 7` 作为plc的开发环境，`WinCC`用于配置HMI。工程师可以使用梯形图（LAD），功能块图（FBD），结构化控制语言（SCL）和语句表（STL）来为PLC编程。与基于文本的SCL和类似汇编的STL，LAD和FBD语言是图形化的。PLC程序被分成组织块（OB），功能（FC），功能块（FB），数据块（DB），系统功能（SFC），系统功能块（SFB）和系统数据块（SDB）这几个单元。OB，FC和FB包含着实际的代码，而DB存储着数据结构，SDB存储PLC的当前配置。带有前缀M的内存地址被用于内部数据存储寻址。

#### 3.4 PLC程序

一个PLC程序至少由一个组织块（称为OB 1）组成，这就相当于C程序中的main函数。它将由操作系统调用。存在更多的用于特定用途的组织块，比如，OB 100。这个块在PLC启动时被调用一次，并通常用于初始化系统。工程师能够通过使用函数和函数块来封装代码。唯一的差别是一个附加的DB可以作为参数来调用一个FB。SFC和SFB内置于PLC。其代码不能被查看。`STEP 7`软件可基于硬件组态步骤得知哪些SFC和SFB是可用的。下面的例子给出SCL，LAD和STL这三种编程语言的一个概述。每个实例展示了对三个输入和一个输出的相同组态。首先，CPU对输入`0.0`和`0.1`执行一次并操作；之后他对结果和输入`0.2`计算一次逻辑或操作。该结果被写入输出`0.0`，这将在下一个周期为其所连接的导线设置逻辑值。第一个例子使用STL实现上述程序。这将使用四行类汇编代码完成，每一行定义一个指令。

```text
A        %I0.0
A        %I0.1
O        %I0.2
=        %Q0.0
```

下一个例子展示了使用基于文本的SCL语言编写的同一个程序。这个程序能用一行代码实现：

```text
%Q0.0 := (%I0.0 AND %I0.1 ) OR %I0.2 ;
```

LD语言编程需要使用`STEP 7`，输入和输出通过拖放到导线上来定位。新的连接可以通过选择顶部工具栏中的导线工具在预先定义的位置上实现。程序如下图：

以下的说明可以在随plc一同发布的西门子手册中找到\[18\]。CPU有多个用于执行和存储当前状态的寄存器。对于二进制操作，状态字寄存器是非常重要的，所有的二进制操作影响该寄存器。CPU使用多达四个32位累加寄存器用于计算。他们像一个堆栈一样被组织。可以独立地访问和操作顶部寄存器的每个字节。在一个新值被载入累加器1之前，当前的值会被复制到累加器2中。为了让两个数相加，该值必须在执行`+D`操作前成功载入到累加器中。STL程序如下：

```text
L        DW#16#1 // ACCU1=1
L        DW#16#2 // ACCU1=2, ACCU2=1
+D            // ACCU1=ACCU1+ACCU2
```

在程序中被多次使用的代码应该被封装为方法。这些方法在代码的任意地方调用。`CALL`指令用来调用已被定义的方法。必要的参数在函数头部定义，并且必须在每个`CALL`指令下指定实参。

```text
CALL FC1
     VAR1 := 1
     VAR2 := W#16#A
```

如前所述，方法和方法块之间的唯一差别是，方法块可以引用与其相应的数据块。在许多情况下，程序需要分配存储空间给一个特定的方法来读取常量或者保存过程值。将常量直接硬编码入代码这种做法是不常见的，因为该代码在每次需改后都要重新编译。相反地，数据块可以很容易地操纵，甚至远程修改。函数块调用如下：

```text
CALL FB1 , %DB1
    VAR1 := 1
    VAR2 := W#16#A
```

#### 3.5 PLC程序的二进制表示

不管用哪种语言写的代码最终都会编译为`MC 7`。`MC7`的操作码长度是可变的，在很多产品上参数的编码是不同的。前一节的示例程序的二进制表示如下：

```text
00100000          A        %I0.0
00110000          A        %I0.1
01120000          O        %I0.2
41100000          =        %Q0.0
```

#### 3.6 网络协议：

西门子plc使用其自身的`S7Comm`协议来传输块。这是一种基于TCP/IP和基于TCP的ISO传输服务的远程过程调用（RPC）协议。包封装如下图：

协议提供了以下功能：

* 系统状态表请求
* 列出可用的块
* 读写数据
* 块信息请求
* 上传下载块
* 传输块到文件系统
* 启动，关闭，内存初始化
* 调试

这些功能中的任意一个执行都需要一个初始化连接。在一个普通的TCP握手后，基于TCP的ISO传输服务继续进行来协商PDU大小。在`S7Comm`协议中，客户端必须额外地提供他的首选PDU大小，CPU的机架和插槽。cpu将其首选PDU尺寸回应给对方，通讯双方同意使用双方的最小值继续通讯。在初始化后，客户端能够在CPU上调用这些功能。下图展示了下载块并传输到文件系统这个功能的数据包顺序。

该PLC在收到下载请求后控制下载过程。下载块请求的数量取决于块长度和PDU大小。通过下载结束请求表明传输结束。plc在收到会有进一步请求的确认后，会保持等待。最终，被传输的块应通过调用plc控制请求使之持久化。以目标文件系统P作为参数，CPU存储块并执行它。 上传过程类似。工程工作站（EWS）请求上传特定块，并且等待确认。在收到无误的确认后，EWS开始请求块。该响应包含块的数据。EWS重复该过程直到整个块传输完毕。上传结束请求标志着上传完成。被传输的块被结构化，其由头部，数据部分和尾部构成。下表展示了已知字节的结构：

尾部包含用于调用功能的参数信息。并不是所有头部和尾部的字节都被我们所知晓，但是我们已经确定了用来理解其内容的必要区域。

### 4. 攻击描述

搜索引擎**SHODAN**展示了数以千计的可以直接公网访问的工业控制系统\[8,10\]。在第三部分中我们已经展示了，向这些plc上上传或者下载程序是可行的。这使得攻击者有能力修改plc的逻辑代码。此外，plc提供了一个系统库，该库包含了可以建立任意TCP/UDP连接的功能。攻击者可以使用完整的TCP/UDP支持来扫描公网plc背后的本地生产网络。另外，他可以利用这个plc作为网关来到达所有的其他产品或者网络设备。 就像Stuxnet病毒那样，我们将攻击代码附加到已存在于plc的逻辑代码之前。该恶意代码将会在`OB1`的最开始部分执行。这就是为什么plc的功能不会被影响。最简单的方式是下载西门子的`OB1`块，之后添加一条`CALL`指令调用任意一个我们可控制的函数，在我们给出的样例中，该函数叫做`FC 666`。之后，patche后的`OB1`，也就是`FC 666`和其他块，将被上传到PLC。下图展示了代码注入过程：

在下个执行周期时，新上传的包含攻击代码的程序就会被执行，并且不会造成任何服务中断。这个过程使得攻击者能够在plc上运行任意恶意代码。我们随这篇文章发布了一款名叫**PLCinject**的工具，他可以自动化完成这个过程\[22\]。有了这种技术，攻击者可以执行如下图所示的攻击流程：

在第一步中，攻击者注入一个SNMP扫描器，它会与plc上的正常代码一同运行。在完成一个针对本地网络的完整SNMP扫描（第二步）之后，攻击者可以从plc中下载扫描结果（第三步）。攻击者现在拥有了公网plc背后内网的一张缩略图。之后他溢出SNMP扫描器并注入一个socks代理（第四步）。这使得攻击者可以通过充当代理的plc到达本地生产网络的所有plc。在之后的两节中我们将阐明SNMP扫描器和SOCKS代理的实现。我们不会详细解释每个操作和系统调用的细节。对于这些的详细描述我们参考了\[18\]和\[21\]。

#### 4.1 SNMP SCANNER

西门子plc不能作为一个TCP端口扫描器来使用，因为TCP连接函数`TCON`直到该函数成功建立连接之前，都无法被终止。此外，最多只能在西门子**S7-300**并发地运行**8**个TCP连接。因此，该PLC只能在不会发生8个连接同时失败的情况下作为TCP扫描器（如果8个连接同时失败，那么这8个连接函数就无法断开，那么扫描就无法继续进行了）。此限制并不适用于无状态的UDP连接。这就是我们要使用基于UDP的简单网络管理协议（SNMP）的原因。SNMP v1.0 在 RFC 1157\[23\]中被定义，并且它是为监视和控制网络设备而开发的。大量网络设备和大部分西门子 Simatic PLC 有默认支持SNMP。西门子plc在开启SNMP功能的情况下是非常活跃的。通过使用`OID 1.3.6.1.2.1.1.1`读取SNMP系统基本信息（sysDesc）对象，西门子plc将发送其产品类型，产品型号，硬件和固件版本，形如以下SNMP响应：

```text
Siemens, SIMATIC S7, CPU314C-2 PN/DP, 6ES7 314-6EH04-0AB0 , HW: 4, FW: V3.3.10.
```

该系统描述可以用来在漏洞和exp库中匹配已发现的plc。plc的固件不会经常打补丁。主要有两种原因：一方面，plc的固件升级将中断生产过程，这将造成亏损；另一方面，plc的固件补丁能够导致某些产品质量问题，这对客户来说是不能容忍的。这就是为什么找到一个拥有已知漏洞的西门子plc设备的可能性非常之高。该SNMP扫描器可以被分解为以下步骤：

* 获得本地IP和子网
* 计算子网的IP段
* 建立UDP连接
* 发送SNMP请求
* 接受SNMP请求
* 将响应存储到一个数据块中
* 停止扫描并且关闭UDP连接

正如第三章所讲，plc编程与使用C语言在X86系统上正常编程完全不同，每个PLC程序周期性执行，所以他需要在每步之后将程序状态存储到状态变量中。在此我们只解释SNMP扫描步骤1到3。 下图显示第一步的一段代码片段，调用RDSYSST函数。

RDSYSST函数读取内部系统状态表（SSL）来获得plc本地IP，SSL请求通常是用于诊断。14和15行将在RDSYSST功能繁忙的情况下停止该函数。 下图展示了程序如何计算本地网络的第一个ip地址和ip总量。

这是通过将plc的本地ip和子网掩码进行按位`AND`运算来完成的，这将返回本地网络的起始地址（24-30行）。现在SNMP扫描器需要知道子网的ip总量。因此，我们将子网掩码与`0xFFFFFFFF`进行`XOR`运算（35-39行）。结果即为子网的ip总量。 下图展示了如何使用STL语言建立一个UDP连接。首先我们需要调用`TCON`方法，并使用我们`TCON_PAR_SCAN`数据块中的量作为实参。 

在UDP协议的情况下，TCON函数不能建立连接，他只能在TCP协议下完成，因为与UDP相反，他的连接是定向的。 但是只调用一次TCON是不够的，当`#connect`变量在两次调用之间从0上升到1时，连接函数才开始工作。这就是为什么我们在连接函数首次出现之后编写了一个切换功能（10-11行）。这将在一个周期里，TCON首次被调用之后改变`#connect`的值，使之从`False`变为`True`。`TCON`函数在下一个周期被调用时，检测到上升沿信号，之后开始执行。下一步操作是，基于UDP协议的SNMP数据包，并且接收响应。这将通过调用`TUSEND`和`TURCV`函数来完成。之后，SNMP扫描完成，所有数据被存储到可以被攻击者下载的数据块中（step 3）。

#### 4.2 SOCKS5 代理

一旦攻击者已经发现所有的SNMP设备，包括本地plc，下一步就是要连接到他们。这可以通过使用可访问的plc作为进入内网的网关来实现。为了做到这一点，我们选择在plc上实现一个socks5代理。这有两个主要原因，首先，SOCKS协议是轻量级的，非常容易实现。另外所有应用都可以使用此类型的代理，要么应用是原生支持socks协议的，要么可以使用所谓的`proxifier`来为任意程序添加对socks的支持。socks5协议在RFC 1928\[24\]中被定义。通过代理无差错地TCP连接到目标需要以下几步：

* 客户端通过TCP连接到SOCKS服务器，并发送自身支持的身份验证方法列表。
* 服务器使用一个选定的身份验证方法回复。
* 根据所选择的身份验证方法，选定相应的子协议。
* 客户端发送带有目标IP的连接请求。
* 服务器建立连接，并回复。所有后续数据包在客户端和目标之间通过隧道传输。
* 客户端关闭TCP连接。

我们的实现提供了最小化的必要功能，他不支持身份验证，所以我们跳过了第三步。我们也不支持错误处理。并且，只支持连接IPV4地址。一旦客户端连接，我们期望该信息会经过以下几个步骤：

* 客户端提供身份验证方法：可以是任意信息，常用的比如：0x05 0x05 \ \(1 byte\) \ \(n bytes\)
* 服务端选择验证方法：0x05 0x00 （无认证）
* 客户端想要连接目标：0x05 0x01 0x00 0x01 \\(4 bytes\) \\(2 bytes\)
* 服务器验证连接： 0x05 0x00 0x00 0x01 0x00 0x00 0x00 0x00 0x00 0x00
* 客户端和目标现在可以通过与服务器的连接进行通信了。

如前面提到的，plc程序周期性执行。这就是为什么我们使用一个简单的状态机来处理SOCKS协议。因此，我们将每个状态编号，并且使用一个跳转表来执行相应的代码块。如下图：

状态转换是由存储在一个数据块中的状态码的递增来实现的。 每个状态和其动作描述如下：

**绑定监听**: 第一次启动程序需要绑定和监听SOCKS端口1080，这是通过系统调用`TCON`在被动模式下实现的。我们保持在这个状态直到有人连接到这个端口。 **协商**: 我们等待客户端发送任何消息。这通过`TRCV`函数实现，该函数需要`EN_R`参数使之执行。见下图：

**认证**: 第一条消息后，我们发送回复，指明客户端是无认证的。为了这个目的，我们使用`TSEND`系统调用。与`TRCV`相反，这个函数是边沿控制的，这意味着`REQ`参数必须在连续的调用之间从`False`变为`True`，这样来激活发送。如下图所示，我们切换标志并且在`REQ`的上升沿两次调用`TSEND`。

**连接请求**: 然后，我们期望客户端发送一个包含有目标IP和端口号的连接设置信息，该信息将为下一个状态而储存。 **连接**: 我们使用`TCON`建立到目标的连接。 **连接验证**: 当到目标的连接已经建立，我们向客户端发送验证信息。 **代理**: 现在我们只需要在客户端和目标之间打开连接隧道。所有使用`TRCV`从客户端收到的数据被存储到一个缓冲区中，并且`TSEND`函数也从中取出数据发送给客户端。同样的原则也适用于相反的方向，但是我们必须考虑到，发送消息可能需要几个周期，因此，第二个缓冲区被用于确保没有消息被混合或者丢失。`TRCV`的错误标志被作为连接断开的信号。当该信号发生时，我们将发送最后收到的数据，然后跳转到下一状态。 **复位**: 在这种状态下，我们使用`TDISCON`关闭所有连接，并且重置所有标志位到其初始状态。

### 5. 评估

我们分析了以下情况的执行周期时间的差异： \(a\) 一个简单的控制程序作为基准 \(b\) 附加了SOCKS代理的恶意代码版本，在空闲模式下 \(c\) 在负载模式下 空间模式意味着代理已经被加入到控制代码中，但是代理连接没有被建立。 基准程序按字节将输入内存复制到输出内存，这个过程执行了20次，这导致了81920条拷贝指令。为了测量，我们在一个数据块中增加了存储着最后一个执行周期时间的小段代码。西门子plc在`OB1`的一个局部变量`OB1_PREV_CYCLE`中存储最后一个执行周期的时间。我们在每一个场景中测量2046个周期。这三个场景都没有呈现正常的分布。我们使用Kruskal–Wallis 和 Dunn的多重对比测试来进行统计学分析。结果如下图，这三个场景的执行时间不相同。

下表显示了基准程序和运行着代理的程序之间的平均差值，这仅仅只有1.35毫秒。

该代理的最大传输速率大约是40kb/s。如果socks代理单独运行在plc上，速率可以高达730KB/s。所有的网络设备以100Mbit/s的以太网直连plc。最后我们在实验室中测试了所描述的攻击周期。除了常规的通信，我们验证了可以通过使用`tsocks`库的socks隧道实施对DOS漏洞**CVE-2015-2177**的利用。利用代码通过socks隧道成功执行。

### 6. 讨论

我们的攻击有一定的局限性。为了确保plc总是可以应答请求，主程序执行时间需要被监视，当执行时间过长时主程序会被结束。我们上传的snmp扫描器或者代理代码，连同原有程序，不应超过最大总执行时间，150ms。注入一个扫描器或者代理不太可能触发超时，因为代理运行时的附加执行时间为1.35ms，这远远小于150ms。另外，超时可通过在注入程序执行完成后重置时间计数器来避免，这要使用系统调用`RE_TRIGR`\[21\]。针对上文中的攻击，最简单的防护方式是保持plc脱机，或者使用VPN来代替公网访问。如果这是不可能的，应当激活西门子plc的第三级保护级别。这使得plc的读取基于口令，并且有写保护。没有正确密码的攻击者不能修改plc程序。根据我们的研究，这个功能在实际中很少用到。另一个应用保护机制是使用防火墙过滤可疑数据包，比如试图重新编程plc的恶意访问。

### 7. 总结

我们展示了一种新型的攻击载体，它能使外部攻击者利用一个plc作为一个snmp扫描器和访问内网的网关。这使得访问plc背后的控制网络成为可能。（后面还有一丁点，不翻了，就是前面讲的东西）

### 8. 参考文献

* \[1\]S. Heng, “Industry 4.0 upgrading of germany’s industrial capabilities on the horizon,” Deutsche Bank Research, 2014.
* \[2\]NIST, “CVE-2014-2908,” Apr. 2014. \[Online\]. Avail-able:  [https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-2908](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-2908)
* \[3\]——, “CVE-2014-2246,” Mar. 2014. \[Online\]. Avail-able:  [https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-2246](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-2246)
* \[4\]——, “CVE-2012-3037,” May 2012. \[Online\]. Avail-able:  [https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2012-3037](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2012-3037)
* \[5\]D. Beresford, “Exploiting Siemens Simatic S7 PLCs,”Black Hat USA, 2011.
* \[6\]N. Cybersecurity and C. I. C. \(NCCIC\), “Ics-cert monitor,”Sep. 2014.
* \[7\]Bundesamt für Sicherheit in der Informationstechnik, “DieLage der IT-Sicherheit in Deutschland 2014,” 2015.
* \[8\]Industrial Control Systems Cyber Emergency Response Team, “Alert \(ICS-ALERT-12-046-01A\) Increasing Threat to Industrial Control Systems \(Update A\),” Available from ICS-CERT, ICS-ALERT-12-046-01A., Oct. 2012. \[Online\].  Available:  [https://ics-cert.us-cert.gov/alerts/ICS-ALERT-12-046-01A](https://ics-cert.us-cert.gov/alerts/ICS-ALERT-12-046-01A)
* \[9\]J.-O. Malchow and J. Klick,Sicherheit in vernetzten Systemen: 21. DFN-Workshop.  Paulsen, C., 2014, ch.Erreichbarkeit von digitalen Steuergeräten - ein Lagebild, pp. C2–C19.
* \[10\]B. Radvanovsky, “Project shine: 1,000,000 internet-connected scada and ics systems and counting,”Tofino Security, 2013.
* \[11\]R. Langner. \(2011\) A time bomb with fourteen bytes. \[Online\]. Available: [http://www.langner.com/en/2011/07/21/a-time-bomb-with-fourteen-bytes/](http://www.langner.com/en/2011/07/21/a-time-bomb-with-fourteen-bytes/)
* \[12\]B. Meixell and E. Forner, “Out of Control: Demonstrating SCADA Exploitation,”Black Hat USA, 2013.
* \[13\]S. E. McLaughlin, “On dynamic malware payloads aimed at programmable logic controllers.” in HotSec, 2011.
* \[14\]S. McLaughlin and P. McDaniel, “Sabot: specification-based payload generation for programmable logic con-trollers,” in Proceedings of the 2012 ACM conference on Computer and communications security.  ACM, 2012,pp. 439–449.
* \[15\]Wikipedia,  “Automation  Pyramid  \(content  taken\).” \[Online\].   Available:   [https://de.wikipedia.org/wiki/Automatisierungspyramide](https://de.wikipedia.org/wiki/Automatisierungspyramide)
* \[16\]Siemens,  “S7  314C-2PN/DP  Technical  Details.” \[Online\]. Available: [https://support.industry.siemens.com/cs/pd/495261?pdti=td&pnid=13754&lc=de-WW](https://support.industry.siemens.com/cs/pd/495261?pdti=td&pnid=13754&lc=de-WW)
* \[17\]——, “S7-300 CPU 31xC and CPU 31x: Technical specifications.” \[Online\]. Available: [https://cache.industry.siemens.com/dl/files/906/12996906/att\_70325/v1/s7300\_cpu\_31xc\_and\_cpu\_31x\_manual\_en-US\_en-US.pdf](https://cache.industry.siemens.com/dl/files/906/12996906/att_70325/v1/s7300_cpu_31xc_and_cpu_31x_manual_en-US_en-US.pdf)
* \[18\]——. \(2011\)  S7-300 Instruction list S7-300 CPUs and  ET  200  CPUs  .  \[Online\].  Available:  [https://cache.industry.siemens.com/dl/files/679/31977679/att\_81622/v1/s7300\_parameter\_manual\_en-US\_en-US.pdf](https://cache.industry.siemens.com/dl/files/679/31977679/att_81622/v1/s7300_parameter_manual_en-US_en-US.pdf)
* \[19\]SNAP7,  “S7  Protocol.”  \[Online\].  Available:  [http://snap7.sourceforge.net/siemens\_comm.html\#s7\_protocol](http://snap7.sourceforge.net/siemens_comm.html#s7_protocol)
* \[20\]J.   Kühner,   “DotNetSiemensPLCToolBoxLibrary.” \[Online\].  Available:  [https://github.com/jogibear9988/DotNetSiemensPLCToolBoxLibrary](https://github.com/jogibear9988/DotNetSiemensPLCToolBoxLibrary)
* \[21\]Siemens.  \(2006\)  System  Software  for  S7-300/400 System and Standard Functions Volume 1/2. \[Online\].Available: [https://cache.industry.siemens.com/dl/files/574/1214574/att\_44504/v1/SFC\_e.pdf](https://cache.industry.siemens.com/dl/files/574/1214574/att_44504/v1/SFC_e.pdf)
* \[22\]D. Marzin, S. Lau, and J. Klick, “PLCinject Tool.” \[On-line\]. Available: [https://github.com/SCADACS/PLCinject](https://github.com/SCADACS/PLCinject)
* \[23\]J. Case, M. Fedor, M. Schoffstall, and J. Davin, “Simple Network Management Protocol \(SNMP\),” RFC 1157 \(Historic\), Internet Engineering Task Force, May 1990. \[Online\]. Available: [http://www.ietf.org/rfc/rfc1157.txt](http://www.ietf.org/rfc/rfc1157.txt)
* \[24\]M. Leech, M. Ganis, Y. Lee, R. Kuris, D. Koblas, and L. Jones, “SOCKS Protocol Version 5,” RFC 1928 \(Proposed Standard\), Internet Engineering Task Force, Mar. 1996. \[Online\]. Available: [http://www.ietf.org/rfc/](http://www.ietf.org/rfc/)

  rfc1928.txt

