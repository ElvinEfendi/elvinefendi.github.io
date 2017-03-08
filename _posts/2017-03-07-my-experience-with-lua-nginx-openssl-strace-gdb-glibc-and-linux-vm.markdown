---
layout: post
title: "fun with lua-nginx-module, openssl, strace, gdb, glibc and Linux VM"
date: 2017-03-07 12:50
comments: true
tags: java, wikipedia, apache lucene, information retrieval
---

Recently I made one line Nginx configuration change and it resulted in Out of Memory(OOM) Killer killing the Nginx process before it completed loading the new configuration:
```
lua_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
```

In this post, I'm going to explain how I tracked down the root cause of the issue and document which tools I used and what I learnt during the process chronologically.
Before going any further here are the details of the software stack I used:

 - Openssl version: `1.0.2j`
 - OS: `Ubuntu Trusty with Linux 3.19.0-80-generic`
 - Nginx: `Openresty bundle 1.11.2`
 - glibc: `Ubuntu EGLIBC 2.19-0ubuntu6.9`

First of all let's start with OOM Killer. It is one of the many functions of Linux kernel that gets triggered when the kernel can not allocate more memory. The job of OOM Killer
is to detect which process is the most damaging(refer to [https://linux-mm.org/OOM_Killer](https://linux-mm.org/OOM_Killer) for more details on how the badness score is calculated) for the system,
and once detected kill the process to free up memory. This means in my case Nginx was asking for more and more memory and eventually kernel failed to allocate more memory and triggered OOM Killer that in its
turn killed the Nginx process. Alright, now let's try to see what Nginx has been doing while it was reloading its new configuration. To do so `strace` can be used. It's a great tool to see what a program is doing
without reading its source code. In this case I executed

```
sudo strace -p `cat /var/run/nginx.pid` -f
```

and then

```
sudo /etc/inid.t/nginx reload
```

`-f` flag tells `strace` to follow the child processes too. You can get a better overview of `strace` at [http://jvns.ca/zines/#strace-zine](http://jvns.ca/zines/#strace-zine).
Following is an interesting piece from the output of `strace` after executing the above commands:

```
[pid 31774] open("/etc/ssl/certs/ca-certificates.crt", O_RDONLY) = 5
[pid 31774] fstat(5, {st_mode=S_IFREG|0644, st_size=274340, ...}) = 0
[pid 31774] mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6dc8266000
[pid 31774] read(5, "-----BEGIN CERTIFICATE-----\nMIIH"..., 4096) = 4096
[pid 31774] read(5, "WIm\nfQwng4/F9tqgaHtPkl7qpHMyEVNE"..., 4096) = 4096
[pid 31774] read(5, "Ktmyuy/uE5jF66CyCU3nuDuP/jVo23Ee"..., 4096) = 4096
...<stripped for clarity>...
[pid 31774] read(5, "MqAw\nhi5odHRwOi8vd3d3Mi5wdWJsaWM"..., 4096) = 4096
[pid 31774] read(5, "dc/BGZFjz+iokYi5Q1K7\ngLFViYsx+tC"..., 4096) = 4096
[pid 31774] brk(0x26d3000)              = 0x26b2000
[pid 31774] mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6c927c3000
[pid 31774] read(5, "/lmci3Zt1/GiSw0r/wty2p5g0I6QNcZ4"..., 4096) = 4096
[pid 31774] read(5, "iv9kuXclVzDAGySj4dzp30d8tbQk\nCAU"..., 4096) = 4096
...<stripped for clarity>...
[pid 31774] read(5, "ye8\nFVdMpEbB4IMeDExNH08GGeL5qPQ6"..., 4096) = 4096
[pid 31774] read(5, "VVNUIEVs\nZWt0cm9uaWsgU2VydGlmaWt"..., 4096) = 4004
[pid 31774] read(5, "", 4096)           = 0
[pid 31774] close(5)                    = 0
[pid 31774] munmap(0x7f6dc8266000, 4096) = 0
```

This was repeating for many times! There are two interesting lines in the output

```
open("/etc/ssl/certs/ca-certificates.crt", O_RDONLY) = 5
```

which means the operation is related to our config change and


```
mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6c927c3000
```

which means the kernel was being asked to allocate 1 megabytes of memory in the middle of `read`s. Another interesting detail in the `strace` output was that allocated memory never gets `munmap`ed(freed). Notice that `0x7f6dc8266000` gets `munmap`ed after the call to `close`. These facts made me think that Ningx was leaking memory when `lua_ssl_trusted_certificate` is set(if you are someone experienced with this stuff you are probably laughing now)! In order to find out what exact component of Nginx was leaking memory I decided to use `gdb`. `gdb` becomes very useful if you compile your program with debug flags enabled. As I mentioned above I was using Nginx Openresty bundle. To compile it with debug flags enabled use the following command:

```
~/openresty-1.11.2.2 $ ./configure -j2 --with-debug --with-openssl=../openssl-1.0.2j/ --with-openssl-opt="-d no-asm -g3 -O0 -fno-omit-frame-pointer -fno-inline-functions"
```

`--with-openssl-opt="-d no-asm -g3 -O0 -fno-omit-frame-pointer -fno-inline-functions"` makes sure that Openssl will also be compiled with debug flags enabled.
talk about gdb, info registers, break point, continue, if strcmp($rdi, "") etc.
then how I decided it was openssl leaking memory and then finally after reading some openssl C code and understanding what it is doing there(decode to interal structures for future use) I finially realized
it is normal that mmap'ed memoery does not get unmapped later on. Then I tried to regenerate the issue with this C program but could not succeed. So this made me think that I might be wrong about 
openssl leaking memory. Then I noticed using gdb that openssl is actually requesting 40 bytes of memory but it is glibc that allocated 1M memory.
then I red on more about brk and mmap and discovered the 
By reading further on `brk` and `mmap` calls I learnt that they are provided by `glibc` and `brk` or `sbrk` is usually what `malloc` function from `glibc` uses to allocate more memory in the heap.
But according to [https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#406](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#406) if `(s)brk` fails to allocate memory, `malloc` will fallback to using
`mmap` with default value of `1024 * 1024` bytes(notice this equals to `1048576` that `mmap` allocates in the above `strace` output). then I learn that brk fails when there's a hole in the heap.
then I check lua-nginx-module issues and PRs and found out this PR(link to memory trick commit) which creates a "hole" that makes all subsequent brk calls to fail and use mmap and allocate 1M heap.
