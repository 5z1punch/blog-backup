---
title: Decrypt php VoiceStar encryption extension
date: '2017-12-12T00:00:00.000Z'
categories: web
tags:
  - php
  - web
---

# Decrypt php VoiceStar encryption extension

## 0x00 引子

昨天头老师扔给我一个php的加密扩展，让我看看能不能解密，我从垃圾桶里拖出我14年版的ida打开简单瞅了一眼，php扩展库的加载流程不是特别熟悉，遂想放弃刚正面用一些杂项手段来搞，看了眼被加密的php文件，文件头部是个标志，google了一发，发现某个解密站点在15年曾经支持过这种加密：  但是目前该站已经无法解密这个方法了。 然而神奇的是，尽管知道了这种方法叫做 AtStar/VoiceStar ，但是用了各种方法都没在网上搜到其他任何关于这种加密的信息。。。。于是，再次老老实实打开ida，正面刚。

## 0x01 定位入口及整体流程

一般来说，php代码加密有三种形式：

* 第一种类似代码混淆，这种方法可以不依赖扩展库，例如phpjiami，一般这种加密需要将解码程序也打包进代码中，也就是我们说的壳，然后壳会被代码混淆，原本的代码会被加密，最终由壳进行解密后执行。这种不依赖扩展库的加密方法有个非常简单的破解方法，因为原代码执行前一定会去调用底层的`zend_compile_string`函数，而且这种加密方法是可以直接运行的，所以我们在运行时把`zend_compile_string` hook住就可以得到源码了。
* 第二种是使用扩展库，如果使用php扩展库那么可以玩儿的加密手段就更多了，比如hook住`zend_compile_*`的一系列函数，在编译流程上动手脚，并且一般我们在线上环境中（比如一个shell上）得到的so库，可能在本地运行不起来，这个时候可能就比较难通过动态调试的方法拿到源码。但是一般来说，只要我们拿到了这个加密库最终输出到zend虚拟机的数据，不管是源码还是opcode，我们一般都能做到代码还原，因为他最终逃不过zend engine\(ze\)。
* 还有一种方法是第二种方法的子集，比如 `Swoole Compile` ，牛逼之处在于他部分脱离了zend虚拟机，对`opcode`做了混淆，这就比较像是vmp，所以破解难度就会变的很大。

但是比较幸运，这次遇到的 voicestar 是第二种方法里比较简单的一种加密方式。

查了下php扩展开发的手册，在载入扩展库时，会首先调用`get_module`来获得模块接口，ida里看下发现返回了一个入口指针，但是我没找到赋值在哪，然后我就把so里的所有function按照起始位置排序了下，然后就看到了`get_module`附近定义的几个函数，特别是 `zm_startup_php_voice` 和 `zm_shutdown_php_voice` 这俩函数从函数名上看上去比较像是入口：

查看`zm_startup_php_voice`：

```c
__int64 zm_startup_php_voice()
{
  *((_DWORD *)&compiler_globals + 135) |= 1u;
  lr = gl((char *)&ll);
  if ( lr )
    php_error_docref0(0LL, 2LL, "No License: %d");
  org_compile_file = (int (__fastcall *)(_QWORD, _QWORD))zend_compile_file;
  zend_compile_file = pcompile_file;
  return 0LL;
}
```

很明显，其关键部分是把 `zend_compile_file` 放到 `org_compile_file` 这个指针中，然后用 `pcompile_file` hook 住 `zend_compile_file`。

`zm_shutdown_php_voice` 主要就是把 `zend_compile_file` 还原。

```c
__int64 zm_shutdown_php_voice()
{
  *((_DWORD *)&compiler_globals + 135) |= 1u;
  zend_compile_file = org_compile_file;
  return 0LL;
}
```

## 0x02 分析 & 解码

`zend_compile_file` 是ze中负责将读入的源代码文件编译为opcode然后执行的函数，那么解密过程应该就是在`pcompile_file`这个hook函数中了。

```c
int __fastcall pcompile_file(__int64 a1, unsigned int a2)
{
  __int64 v2; // rbx@1
  unsigned int v3; // er13@1
  FILE *v4; // rax@3
  FILE *v5; // rbp@3
  bool v6; // zf@4
  __int64 v7; // rdi@4
  signed __int64 v8; // rcx@4
  char *v9; // rsi@4
  int v10; // er12@9
  int v11; // eax@10
  FILE *v12; // rax@12
  __int64 v13; // rdi@12
  __int64 v14; // rax@12
  int result; // eax@12
  __int64 v16; // rax@14
  const char *v17; // rax@15
  __int64 v18; // [sp+0h] [bp-58h]@1
  __int64 v19; // [sp+8h] [bp-50h]@1
  __int64 v20; // [sp+10h] [bp-48h]@1
  __int64 v21; // [sp+18h] [bp-40h]@1
  char ptr; // [sp+20h] [bp-38h]@4

  v2 = a1;
  v3 = a2;
  v18 = 0LL;
  v19 = 0LL;
  v20 = 0LL;
  v21 = 0LL;
  if ( (unsigned __int8)zend_is_executing() && (LODWORD(v16) = get_active_function_name(), v16) )
  {
    LODWORD(v17) = get_active_function_name();
    strncpy((char *)&v18, v17, 0x1EuLL);
    if ( !(_BYTE)v18 )
      goto LABEL_3;
  }
  else if ( !(_BYTE)v18 )
  {
    goto LABEL_3;
  }
  if ( !strcasecmp((const char *)&v18, "show_source") )
    return 0;
  if ( !strcasecmp((const char *)&v18, "highlight_file") )
    return 0;
LABEL_3:
  v4 = fopen(*(const char **)(a1 + 8), "r");
  v5 = v4;
  if ( !v4 )
    return org_compile_file(v2, v3);
  fread(&ptr, 8uLL, 1uLL, v4);
  v7 = (__int64)"\tATSTAR\t";
  v8 = 8LL;
  v9 = &ptr;
  do
  {
    if ( !v8 )
      break;
    v6 = *v9++ == *(_BYTE *)v7++;
    --v8;
  }
  while ( v6 );
  if ( !v6 )
  {
    fclose(v5);
    return org_compile_file(v2, v3);
  }
  if ( lr )
  {
    php_error_docref0(0LL, 2LL, "No License:");
    result = 0;
  }
  else
  {
    v10 = cle(&ll, v9);
    if ( v10 )
    {
      php_error_docref0(0LL, 2LL, "No License: %d");
      printf("No License:%d\n", (unsigned int)v10, v18);
      result = 0;
    }
    else
    {
      v11 = *(_DWORD *)v2;
      if ( *(_DWORD *)v2 == 2 )
      {
        fclose(*(FILE **)(v2 + 24));
        v11 = *(_DWORD *)v2;
      }
      if ( v11 == 1 )
        close(*(_DWORD *)(v2 + 24));
      v12 = ext_fopen(v5);
      v13 = *(_QWORD *)(v2 + 8);
      *(_QWORD *)(v2 + 24) = v12;
      *(_DWORD *)v2 = 2;
      LODWORD(v14) = expand_filepath(v13, 0LL);
      *(_QWORD *)(v2 + 16) = v14;
      result = org_compile_file(v2, v3);
    }
  }
  return result;
}
```

简单看一下逻辑，首先是判断 `show_source` 和 `highlight_file` 俩函数有没有被禁用，然后判断当前解析的文件有没有"\tATSTAR\t" 这个文件头，如果有，则进入到解密流程，解密流程前面一段都是在判断有没有授权，我们直接跳到83行之后的这个最后的分支上去。 `zend_compile_file` 定义如下：

```c
 extern ZEND_API zend_op_array *(*zend_compile_file)(zend_file_handle *file_handle, int type);
```

其第一个参数是`zend_file_handle`结构，其偏移量24的位置放的是打开的源码文件指针，此分支代码的主要逻辑就是关闭原来打开的代码文件，然后使用`ext_fopen`函数重新打开该php代码文件，然后替换掉 `zend_file_handle`结构体中原来的文件句柄。然后调用 `zend_compile_file`\(hook之前的函数，现在为`org_compile_file`\)。那么很显然，当函数调用到`zend_compile_file`的时候，v2里装的应该就是还原后的代码了，那么还原操作应该是在`ext_fopen`函数内，跟进去看。

```c
FILE *__fastcall ext_fopen(FILE *stream)
{
  int v1; // eax@1
  signed int v2; // ebx@1
  unsigned int v3; // er13@1
  void *v4; // rbp@1
  void *v5; // rcx@2
  __int64 v6; // rax@3
  const void *v7; // r12@4
  FILE *v8; // rbx@4
  __int64 v10; // [sp+0h] [bp-C8h]@1
  __int64 v11; // [sp+30h] [bp-98h]@1
  int v12; // [sp+9Ch] [bp-2Ch]@4

  v1 = fileno(stream);
  __fxstat(1, v1, (struct stat *)&v10);
  v2 = v11 - 8;
  v3 = v11 - 8;
  v4 = malloc((signed int)v11 - 8);
  fread(v4, v2, 1uLL, stream);
  fclose(stream);
  if ( v2 > 0 )
  {
    v5 = v4;
    do
    {
      v6 = (signed int)((((_BYTE)v2 + ((unsigned int)(v2 >> 31) >> 28)) & 0xF) - ((unsigned int)(v2 >> 31) >> 28));
      *(_BYTE *)v5 = ~(*(_BYTE *)v5 ^ (d[v6] + p[2 * v6] + 5));
      v5 = (char *)v5 + 1;
      --v2;
    }
    while ( v2 );
  }
  v7 = zdecode((__int64)v4, v3, (__int64)&v12);
  v8 = tmpfile();
  fwrite(v7, v12, 1uLL, v8);
  free(v4);
  free((void *)v7);
  rewind(v8);
  return v8;
}
```

该部分代码的关键部分应该是在 22-33行，这段代码明显是在做一个对称加密（易得，证略～～2333终于可以说这句话了），虽然没看到密钥，但是我们发现这里面`d`和`p`这俩盒子是放在data段的: 

然后这个对称加\(解\)密做完之后，解密后的内容又进入到了 `zdecode` 这个函数中，跟进去看一眼， `zdecode`其实就是`zcodecom`又封装了一层，直接看`zcodecom`：

```c
void *__fastcall zcodecom(int a1, __int64 a2, int a3, __int64 a4)
{
  int v4; // er13@1
  __int64 v5; // r12@1
  int v6; // er14@3
  void *v7; // r13@3
  int v8; // eax@4
  int v10; // ST08_4@21

  v4 = a3;
  v5 = a4;
  *((_QWORD *)&z + 8) = 0LL;
  *((_QWORD *)&z + 9) = 0LL;
  *((_QWORD *)&z + 10) = 0LL;
  z = 0LL;
  *((_DWORD *)&z + 2) = 0;
  if ( a1 )
    inflateInit_(&z, "1.2.8", 112LL);
  else
    deflateInit_(&z, 1LL, "1.2.8", 112LL);
  z = a2;
  *((_DWORD *)&z + 2) = v4;
  *((_DWORD *)&z + 8) = 100000;
  v6 = 0;
  *((_QWORD *)&z + 3) = &outbuf;
  v7 = malloc(0x186A0uLL);
  while ( 1 )
  {
    if ( a1 )
    {
      v8 = inflate(&z, 0LL);
      if ( v8 == 1 )
      {
LABEL_9:
        if ( 100000 == *((_DWORD *)&z + 8) )
        {
          if ( a1 )
            goto LABEL_11;
LABEL_16:
          deflateEnd(&z);
        }
        else
        {
          v10 = 100000 - *((_DWORD *)&z + 8);
          v7 = realloc(v7, v6 + 100000);
          memcpy((char *)v7 + v6, &outbuf, v10);
          v6 += v10;
          if ( !a1 )
            goto LABEL_16;
LABEL_11:
          inflateEnd((__int64)&z);
        }
        *(_DWORD *)v5 = v6;
        return v7;
      }
    }
    else
    {
      v8 = deflate(&z, 4LL);
      if ( v8 == 1 )
        goto LABEL_9;
    }
    if ( v8 )
      break;
    if ( !*((_DWORD *)&z + 8) )
    {
      v7 = realloc(v7, v6 + 100000);
      memcpy((char *)v7 + v6, &outbuf, 0x186A0uLL);
      *((_QWORD *)&z + 3) = &outbuf;
      *((_DWORD *)&z + 8) = 100000;
      v6 += 100000;
    }
  }
  if ( a1 )
    inflateEnd((__int64)&z);
  else
    deflateEnd(&z);
  *(_DWORD *)v5 = 0;
  return v7;
}
```

发现这个函数有点复杂，看了半天没看懂在干啥，然后我就把代码复制下来本地编译了一发，但是这段代码调了半晚上都没跑起来，我想了一下，感觉问题应该是出在 `z` 和 `outbuf` 这俩全局变量上，这段代码中对`z`上的偏移频繁操作，`z`应该是某个比较复杂的结构体，并且`inflate`、`inflateEnd`等等这几个函数在编译的时候必须link zlib，`gcc 1.c -o a -lz -g3` ，我猜测 `z` 应该是zlib中某个结构体，我就去查了zlib的手册，看个`Example`（为啥这个zlib的logo看上去这么像某lv文。。。。）： [https://www.zlib.net/zlib\_how.html](https://www.zlib.net/zlib_how.html) 此处的`z`应该是 `z_stream`结构，这个过程应该就是使用zlib解压。我按照zlib手册中的结构和宏试图复原`zcodecom`函数，但是每当执行到这句解压代码时`v8 = inflate(&z, 0LL);` 总会返回一个`0xfffffffb`，查手册发现是`Z_BUF_ERROR`，没辙，c语言太菜搞不定，只能想另外的方法。 此时我已经基本确定这里是一个解压流程，尽管不知道有没有啥另外的操作，索性把数据导出来用python解压下。 此时我已经把从`pcompile_file`到解压流程前的代码都调通了，编译好之后在`zdecode`前下个断点，文件长度存在v3中，此处直接print出来，为0x4f7，查看下`v4`的地址，然后直接从该地址dump binary memory 0x4f7个字节： 

file 瞅一眼:

```text
root@penGun:/tmp# file aaa
aaa: zlib compressed data
```

感觉没啥毛病，上python直接解：

```text
Python 2.7.12 (default, Nov 19 2016, 06:48:10)
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from zlib import *
>>> data = open('aaa','rb').read()
>>> data = decompress(data)
>>> file = open('ddd.php','wb')
>>> file.write(data)
>>> file.close()
```

搞定： 

## 0x03 关于抄ida中c代码

注意导入ida的定义头文件，因为某些类型比较蛋疼，其实手动typedef也是可以的：

```text
typedef unsigned long long __int64;
typedef unsigned long _DWORD;
typedef unsigned long long _QWORD;
typedef unsigned short _WORD;
typedef unsigned char _BYTE;
typedef int bool;
```

直接include plugins目录下的defs.h就好啦。

还要注意某些结构体ida不能识别，所以可能f5出来的代码存在这种形式：

```text
  __int64 v10; // [sp+0h] [bp-C8h]@1
  __int64 v11; // [sp+30h] [bp-98h]@1
  int v12; // [sp+9Ch] [bp-2Ch]@4
```

这几个变量应该本在一个结构体中的，但是这样看下面的代码似乎他们没经过赋值就使用了，此时可以重新写一个结构体，也可以手动布局下堆栈，把变量放在指定的偏移上，其实这两种方法原理完全一样，就是操作起来可能形式不同。

## 0x04 牙疼。。补牙了要。。。

2017.12.12 小鹿

