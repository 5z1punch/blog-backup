---
title: PHP中的内存破坏漏洞利用（CVE-2014-8142和CVE-2015-0231）（连载之第三篇）
date: '2016-04-30T00:00:00.000Z'
categories: bin
tags:
  - cve
  - php
  - bin
---

# PHP中的内存破坏漏洞利用（CVE-2014-8142和CVE-2015-0231）（连载之第三篇）

我上周研究这个漏洞的时候发现drops上有原作的前两篇博客，第三篇博客不知道是没通过小编的审查还是太监了，我在这儿翻译下，并且，于此同时投稿的还有一篇针对此内存破坏漏洞的详细利用过程。

## 正文开始：

本文将说明如何通过该漏洞创建一个poc。文章最后的视频展示了自动化远程利用的过程（就是跑了下exp装逼），以及一些在真实环境中利用该漏洞的tips和技巧。

我写了一个尽可能简单的漏洞应用，从代码上来看就是将所有发送到该页面的数据，经过base64解码后，反序列化，之后再将反序列化得到的对象的序列化。

我们需要一种方法来执行任意php代码，当然我可以尝试注入shellcode，但这样的利用不是很有创意，并且也是不稳定的（在较新的php版本中也是不能实现的，除非shellcode非常小）。根据第一部分所说，执行代码，需要调用 `php_execute_script` 和 `zend_eval_string` ，但是，如果要实现远程攻击，我们还要找到 `executor_globals` 和 `JMP_BUF`。

总之，我们要找到（不分先后顺序）：

* executor\_globals
* zend\_eval\_string
* JMP\_BUF
* A way to write arbitrary data to the stack

值得庆幸的是，其中的一些，我们可以很快找到，因为他们在二进制文件中，接下来我们直接dump出php二进制文件的符号表。

好的，我们现在从符号表文件中导出我们需要的地址，并且在gdb中验证他们是正确的。

找 zend\_eval\_string 的地址：

在gdb显示，这个地址是正确的：

找到 executor\_global 的地址：

gdb显示该地址是正确的：

ok，现在我们需要找到 JMP\_BUF ，阅读 `_zend_executor_globals` 对象的源代码，我们发现一个有趣的信息，有一个叫做 bailout 的 JMP\_BUF 指针。我们在gdb中查看这个指针，看看是否有用。

打印出 zend\_executor\_globals 对象：

查看指针 bailout ：

好的，现在我们有地址了，这些地址应该怎么使用呢。在php中 jmp\_buf 被用作c语言层级的一种“try {} - catch{} ”。稍后做更多解释。 我们现在唯一缺一件事了：将恶意代码注入栈中。Stefan在2010年Syscan上发表的一种方法将在下文中解释。 我们应该如何写堆栈，并且保证在接下来程序的运行中不会被覆盖？ [RFC1867](https://www.ietf.org/rfc/rfc1867.txt) 中说明了一种方法，他允许带有`multipart/form-data`的post请求设置堆栈缓冲区。并且由于多种原因，这部分内存不会被php完全重写。现在我们来尝试一下覆盖堆栈： 

好的，现在我们可以任意写堆栈，接下来该写入怎样的数据呢？ 经过一番研究后，我们应该将堆栈布局重写如下：

我们已经在elf文件中找到了`zend_eval_string`，返回指针和`zend_bailout`会被垃圾回收，但我们不关心他们。因为我们可以写堆栈，我们设置两次 pointer\_to\_eval\_string。现在我们的数据如下：

* POP; RET - ?????
* XCHG EAX, ESP; RET - ????
* Zend\_Eval\_String - 0x082da150
* Zend\_Bailout - 0x00000000
* Pointer\_To\_Eval\_String - 0xbfffda04
* Ret Pointer - 0x00000000
* Pointer\_To\_Eval\_String - 0xbfffda04

现在我们需要一些rop gadget，我自己更喜欢ROPGadget。我们需要找到 `XCHG EAX, ESP; RET (0x94 0xc3)`, 和 `POP EBP; RET (0x5d 0xc3)`。

现在我们有这俩地址了（为什么地址这么奇怪，因为这事偏移地址）。我们可以完成堆栈布局：

* POP; RET - 0x000e8e68
* XCHG EAX, ESP; RET - 0x000057b7
* Zend\_Eval\_String - 0x082da150
* Zend\_Bailout - 0x00000000
* Pointer\_To\_Eval\_String - 0xbfffda04
* Ret Pointer - 0x00000000
* Pointer\_To\_Eval\_String - 0xbfffda04

现在来测试下：

额，并没有按照我们预想的来进行。事实上，代码执行流确实尝试跳到我们的gadget \(c394\)上执行。不幸的事，这里需要一个特殊的对象来保证这些gadget正确执行而不crash掉php。你需要的是`SPLObjectStorage`，我们要重新编码我们的攻击代码。一旦我们做到了这一点，再次执行利用，运行如下图：

对于这种方法，就到此为止了，因为他只可以攻击老版本的php，对于新版本的攻击请继续阅读。 其实并没有太大的变化，回想一下刚才我们找到的`php_execute_script`和`jmp_buf`的地址。我们需要他们完成对新版本的利用。

`JMP_BUF` 是被`setjmp` & `longjmp`所调用的。并在遇到一个“不可恢复的”错误时，保存当前运行环境。jmp\_buf的结构，取决于你的架构。对于32位系统，这是6个整数无符号数组。如果是64位，则是8个int。 对于这是如何实现的，需要阅读研究源码，确定每个寄存器在`jmp_buf`中的位置。这里是一个jmp\_buf布局的例子。当然，让我们看看其在php中的布局。 

对于我的机器，寄存器的顺序是：EBX，ESP，EBP，ESI，EDI，EIP。这个顺序的得来，并不是一个简单的搜索能找到的。看上去edi & eip寄存器被做了混淆，是通过`Glibc`做的混淆， glibc有一个叫PTR\_MANGLE宏。在视频中，我们将讨论如何破解JMPBUF。

一旦我们把它破解了，我们需要一种方法来覆盖和释放内存。值得庆幸的是，同一个对象（SPLObjectStorage）使我们能够远程释放内存。将所有剩下的将数据写入到堆栈。就像在第二部分中，我们滥用PHP的内存缓存。我们释放一些内存，写一个小的7字节的字符串来填充它，而当PHP覆盖我们的数据的一部分，我们再做一次。这第二次重写允许我们编写堆栈（我没有测试超过2048的值）。我们希望写入的数据与我们之前在rop链中使用的类似。当然，我们需要对其进行PTR\_MANGLE加密。下面是一些输出样例： 

视频在此： [需要翻墙](https://youtu.be/EidBm7-zgSM)

一些关于视频的注意事项：

* x86 Instruction Chart - [http://sparksandflames.com/files/x86InstructionChart.html](http://sparksandflames.com/files/x86InstructionChart.html)
* Elf Header lowest 3 bits are 000
* lf layout - [http://geezer.osdevbrasil.net/osd/exec/elf.txt](http://geezer.osdevbrasil.net/osd/exec/elf.txt)
* PMAP is your friend when trying to find the "Magic"
* A look at PTR\_MANGLE [http://hmarco.org/bugs/CVE-2013-4788.html](http://hmarco.org/bugs/CVE-2013-4788.html)

