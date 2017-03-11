---
layout: post
title: "fun with lua-nginx-module, openssl, strace, gdb, glibc and Linux VM"
date: 2017-03-07 12:50
comments: true
tags: java, wikipedia, apache lucene, information retrieval
---

**tldr;** A memory trick in [lua-nginx-module](https://github.com/openresty/lua-nginx-module/blob/37e5362088bd659e318aae568b268719bd0d6707/src/ngx_http_lua_module.c#L1294) leads to redundant large memory allocation.

Recently I made one line Nginx configuration change and it resulted in Out of Memory(OOM) Killer killing the Nginx process before it completed loading the new configuration. Here's the line added to the configuration file:
```
lua_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
```

In this post, I'm going to explain how I tracked down the root cause of the issue and document which tools I used and what I learnt during the process chronologically. The post will be a bit of everything.
Before going any further here are the details of the software stack I used:

 - OpenSSL version: `1.0.2j`
 - OS: `Ubuntu Trusty with Linux 3.19.0-80-generic`
 - Nginx: `Openresty bundle 1.11.2`
 - glibc: `Ubuntu EGLIBC 2.19-0ubuntu6.9`

First of all let's start with OOM Killer. It is one of the many functions of Linux kernel that gets triggered when the kernel can not allocate more memory. The job of OOM Killer
is to detect which process is the most damaging(refer to [https://linux-mm.org/OOM_Killer](https://linux-mm.org/OOM_Killer) for more details on how the badness score is calculated) for the system,
and once detected kill the process to free up memory. This means in my case Nginx was asking for more and more memory and eventually kernel failed to allocate more memory and triggered OOM Killer that in its turn killed the Nginx process. Alright, now let's try to see what Nginx has been doing while it was reloading its new configuration. To do so `strace` can be used. It's a great tool to see what a program is doing
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

This was being repeated for many times! There are two interesting lines in the output

```
open("/etc/ssl/certs/ca-certificates.crt", O_RDONLY) = 5
```

which means the operation is related to our config change(the one line mentioned in the beginning) and


```
mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6c927c3000
```

which means the kernel was being asked to allocate 1Mb of memory in the middle of `read`s. Another interesting detail in the `strace` output was that allocated memory never gets `munmap`ed(freed). Notice that `0x7f6dc8266000` gets `munmap`ed after the call to `close`. These facts(and me having almost no prior experience with low level debugging) made me think that Ningx was leaking memory when `lua_ssl_trusted_certificate` is set! Memory leak in Nginx, is not that exciting?! Not so fast. In order to find out what exact component of Nginx was leaking memory I decided to use `gdb`. `gdb` becomes very useful if you compile your program with debug flags enabled. As I mentioned above I was using Nginx Openresty bundle. To compile it with debug flags enabled use the following command:

```
~/openresty-1.11.2.2 $ ./configure -j2 --with-debug --with-openssl=../openssl-1.0.2j/ --with-openssl-opt="-d no-asm -g3 -O0 -fno-omit-frame-pointer -fno-inline-functions"
```

`--with-openssl-opt="-d no-asm -g3 -O0 -fno-omit-frame-pointer -fno-inline-functions"` makes sure that OpenSSL will also be compiled with debug flags enabled. Now that I have debug flags enabled version of Openresty executable I can run it through `gdb` and find out what exact function was triggering the `mmap` call mentioned above. First we need to start `gdb` for debugging Openresty executable:
```
sudo gdb `which openresty`
```
This command will open the `gdb` console that looks like following:

```
GNU gdb (Ubuntu 7.7.1-0ubuntu5~14.04.2) 7.7.1
Copyright (C) 2014 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /usr/local/openresty/bin/openresty...done.
(gdb)
```
As a next step I'll set the command line arguments of my program:
```
(gdb) set args -p `pwd` -c nginx.conf
```
This makes sure that `gdb` when instructed to do so will start Openresty/Nginx using the given command line arguments. The next step is to configure breakpoints to pause the program execution at a given line in a given file, or at a given function call. Because I wanted to find out the suspicious `mmap` caller, after `open`ing the trusted certificate file, I'll first add a break point to pause when
```
open("/etc/ssl/certs/ca-certificates.crt", O_RDONLY) = 5
```
is made. Here's how I did it:
```
break open if strcmp($rdi, "/etc/ssl/certs/ca-certificates.crt") == 0
```
If you have not realized yet, `gdb` is super cool, it even lets you add a custom condition to create more flexible break points. In this case, we are telling `gdb` to pause the execution of the program if `open` function is called and the data set in `rdi` register is "/etc/ssl/certs/ca-certificates.crt". I don't know if there's a better way but I found this just by playing out a bit that `open` function's first argument(path to file) is kept in `rdi` register. Hence the break point condition. Now we can tell `gdb` to execute the program:
```
(gdb) run
```
In the first occurence of `open("/etc/ssl/certs/ca-certificates.crt", O_RDONLY)` call, `gdb` will pause the program execution. Now we can use different `gdb` helper commands to examine the internal state of program at the given time. This is how it looked like when the program execution hit the breakpoint:

```
Breakpoint 1, open64 () at ../sysdeps/unix/syscall-template.S:81
81  ../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb) bt
#0  open64 () at ../sysdeps/unix/syscall-template.S:81
#1  0x00007ffff6a3dec8 in _IO_file_open (is32not64=8, read_write=8, prot=438, posix_mode=<optimized out>, filename=0x7fffffffdb00 "\346\f\362\367\377\177", fp=0x7ffff7f28a10) at fileops.c:228
#2  _IO_new_file_fopen (fp=fp@entry=0x7ffff7f28a10, filename=filename@entry=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", mode=<optimized out>, mode@entry=0x6fb62d "r", is32not64=is32not64@entry=1) at fileops.c:333
#3  0x00007ffff6a323d4 in __fopen_internal (filename=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", mode=0x6fb62d "r", is32=1) at iofopen.c:90
#4  0x00000000005b3fd2 in file_fopen (filename=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", mode=0x6fb62d "r") at bss_file.c:164
#5  0x00000000005b3fff in BIO_new_file (filename=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", mode=0x6fb62d "r") at bss_file.c:172
#6  0x00000000005e8ad3 in X509_load_cert_crl_file (ctx=0x7ffff7f289e0, file=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", type=1) at by_file.c:251
#7  0x00000000005e8626 in by_file_ctrl (ctx=0x7ffff7f289e0, cmd=1, argp=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", argl=1, ret=0x0) at by_file.c:115
#8  0x00000000005e5747 in X509_LOOKUP_ctrl (ctx=0x7ffff7f289e0, cmd=1, argc=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", argl=1, ret=0x0) at x509_lu.c:120
#9  0x00000000005dd5c1 in X509_STORE_load_locations (ctx=0x7ffff7f28750, file=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", path=0x0) at x509_d2.c:94
#10 0x0000000000546e22 in SSL_CTX_load_verify_locations (ctx=0x7ffff7f27fd0, CAfile=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", CApath=0x0) at ssl_lib.c:3231
#11 0x0000000000477d94 in ngx_ssl_trusted_certificate (cf=cf@entry=0x7fffffffe150, ssl=0x7ffff7f27a78, cert=cert@entry=0x7ffff7f22f20, depth=<optimized out>) at src/event/ngx_event_openssl.c:687
#12 0x00000000004f0a1b in ngx_http_lua_set_ssl (llcf=0x7ffff7f22ef8, cf=0x7fffffffe150) at ../ngx_lua-0.10.7/src/ngx_http_lua_module.c:1240
#13 ngx_http_lua_merge_loc_conf (cf=0x7fffffffe150, parent=0x7ffff7f15808, child=0x7ffff7f22ef8) at ../ngx_lua-0.10.7/src/ngx_http_lua_module.c:1158
#14 0x000000000047e2b1 in ngx_http_merge_servers (cmcf=<optimized out>, cmcf=<optimized out>, ctx_index=<optimized out>, module=<optimized out>, cf=<optimized out>) at src/http/ngx_http.c:599
#15 ngx_http_block (cf=0x7fffffffe150, cmd=0x0, conf=0x1b6) at src/http/ngx_http.c:269
#16 0x0000000000460b5b in ngx_conf_handler (last=1, cf=0x7fffffffe150) at src/core/ngx_conf_file.c:427
#17 ngx_conf_parse (cf=cf@entry=0x7fffffffe150, filename=filename@entry=0x7ffff7f0b9e8) at src/core/ngx_conf_file.c:283
#18 0x000000000045e2f1 in ngx_init_cycle (old_cycle=old_cycle@entry=0x7fffffffe300) at src/core/ngx_cycle.c:274
#19 0x000000000044cef4 in main (argc=<optimized out>, argv=<optimized out>) at src/core/nginx.c:276
```
How awesome, `gdb` shows us the complete backtrace of function calls with the passed parameters! To examine the data in registers at this point, you can use `info registers` command. In order to better understand and parse this baktrace I looked into how Nginx(remember Openresty is just Nginx with extra modules) internally works. Everything in Nginx(excep its core) is implemented as a module and those modules register handlers or filters. Nginx's config file consists of three main section, namely main, server and location. Let's say your custom Nginx module introduces a new config directive then you need to also register a handler to do something with the value of that directive. So what happens is Nginx parses the config file and calls the registered config handlers after(there are actually more callback types) each config section is parsed.
Here's how `lua-nginx-module`(the core module of Openresty Nginx bundle) does it:

```
ngx_http_module_t ngx_http_lua_module_ctx = {
#if (NGX_HTTP_LUA_HAVE_MMAP_SBRK)                                            \
    && (NGX_LINUX)                                                           \
    && !(NGX_HTTP_LUA_HAVE_CONSTRUCTOR)
    ngx_http_lua_pre_config,          /*  preconfiguration */
#else
    NULL,                             /*  preconfiguration */
#endif
    ngx_http_lua_init,                /*  postconfiguration */

    ngx_http_lua_create_main_conf,    /*  create main configuration */
    ngx_http_lua_init_main_conf,      /*  init main configuration */

    ngx_http_lua_create_srv_conf,     /*  create server configuration */
    ngx_http_lua_merge_srv_conf,      /*  merge server configuration */

    ngx_http_lua_create_loc_conf,     /*  create location configuration */
    ngx_http_lua_merge_loc_conf       /*  merge location configuration */
};
```

This is how an Nginx module registers its handlers. As you can see from the comments too, `ngx_http_lua_merge_loc_conf` will be called once Nginx parses a location section config and merges it to main section. Now if we return back to the `gdb` output above we can see that at line #13 this function was called. So by definition this function will be called for every location section configured. As you can see [here](https://github.com/openresty/lua-nginx-module/blob/master/src/ngx_http_lua_module.c#L1093) the function is quite straightforward to read and what it mainly does is to inherit relevant config pieces from server section into the location section(every section has its own configuration context) and set default values. As you can see it does also call `ngx_http_lua_set_ssl` function which in its turn calls `ngx_ssl_trusted_certificate` function from Nginx's SSL module if `lua_ssl_trusted_certificate` directive is set. `ngx_ssl_trusted_certificate` is a simple function that sets the verify depth for the SSL context for given configuration section(a location section in this case) and calls another OpenSSL API to load the certificate file(well and some error handling):

```
0649 ngx_int_t
0650 ngx_ssl_trusted_certificate(ngx_conf_t *cf, ngx_ssl_t *ssl, ngx_str_t *cert,
0651     ngx_int_t depth)
0652 {
0653     SSL_CTX_set_verify_depth(ssl->ctx, depth);
0654 
0655     if (cert->len == 0) {
0656         return NGX_OK;
0657     }
0658 
0659     if (ngx_conf_full_name(cf->cycle, cert, 1) != NGX_OK) {
0660         return NGX_ERROR;
0661     }
0662 
0663     if (SSL_CTX_load_verify_locations(ssl->ctx, (char *) cert->data, NULL)
0664         == 0)
0665     {
0666         ngx_ssl_error(NGX_LOG_EMERG, ssl->log, 0,
0667                       "SSL_CTX_load_verify_locations(\"%s\") failed",
0668                       cert->data);
0669         return NGX_ERROR;
0670     }
0671 
0672     /*
0673      * SSL_CTX_load_verify_locations() may leave errors in the error queue
0674      * while returning success
0675      */
0676 
0677     ERR_clear_error();
0678 
0679     return NGX_OK;
0680 }
```
The complete code for Nginx SSL module can be found [here](http://lxr.nginx.org/source/src/event/ngx_event_openssl.c). Alright, now we are halfway through the backtrace and out of the Nginx world already. The next function that was called is `SSL_CTX_load_verify_locations` which is from OpenSSL. At this point the program opened the trusted certificate file and paused. As a next step it will start reading it(refer to the `strace` output above). As my initial purpose was to find out what was initiating the suspicious `mmap` call naturally the next breakpoint I created was:
```
(gdb) b mmap
```
`b` is a shortcut for `break`. Then `(gdb) c` to continue the program execution. And program hits the next breakpoint:

```
Breakpoint 3, mmap64 () at ../sysdeps/unix/syscall-template.S:81
81  ../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb) bt
#0  mmap64 () at ../sysdeps/unix/syscall-template.S:81
#1  0x00007ffff6a44ad2 in sysmalloc (av=0x7ffff6d82760 <main_arena>, nb=48) at malloc.c:2495
#2  _int_malloc (av=0x7ffff6d82760 <main_arena>, bytes=40) at malloc.c:3800
#3  0x00007ffff6a466c0 in __GI___libc_malloc (bytes=40) at malloc.c:2891
#4  0x000000000057d829 in default_malloc_ex (num=40, file=0x6f630f "a_object.c", line=350) at mem.c:79
#5  0x000000000057deb9 in CRYPTO_malloc (num=40, file=0x6f630f "a_object.c", line=350) at mem.c:346
<internal OpenSSL function calls stripped for clarity>
#30 0x000000000065e2f7 in PEM_X509_INFO_read_bio (bp=0x7ffff7f28c50, sk=0x0, cb=0x0, u=0x0) at pem_info.c:248
#31 0x00000000005e8b22 in X509_load_cert_crl_file (ctx=0x7ffff7f289e0, file=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", type=1) at by_file.c:256
#32 0x00000000005e8626 in by_file_ctrl (ctx=0x7ffff7f289e0, cmd=1, argp=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", argl=1, ret=0x0) at by_file.c:115
#33 0x00000000005e5747 in X509_LOOKUP_ctrl (ctx=0x7ffff7f289e0, cmd=1, argc=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", argl=1, ret=0x0) at x509_lu.c:120
#34 0x00000000005dd5c1 in X509_STORE_load_locations (ctx=0x7ffff7f28750, file=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", path=0x0) at x509_d2.c:94
#35 0x0000000000546e22 in SSL_CTX_load_verify_locations (ctx=0x7ffff7f27fd0, CAfile=0x7ffff7f20ce6 "/etc/ssl/certs/ca-certificates.crt", CApath=0x0) at ssl_lib.c:3231
#36 0x0000000000477d94 in ngx_ssl_trusted_certificate (cf=cf@entry=0x7fffffffe150, ssl=0x7ffff7f27a78, cert=cert@entry=0x7ffff7f22f20, depth=<optimized out>) at src/event/ngx_event_openssl.c:687
#37 0x00000000004f0a1b in ngx_http_lua_set_ssl (llcf=0x7ffff7f22ef8, cf=0x7fffffffe150) at ../ngx_lua-0.10.7/src/ngx_http_lua_module.c:1240
#38 ngx_http_lua_merge_loc_conf (cf=0x7fffffffe150, parent=0x7ffff7f15808, child=0x7ffff7f22ef8) at ../ngx_lua-0.10.7/src/ngx_http_lua_module.c:1158
#39 0x000000000047e2b1 in ngx_http_merge_servers (cmcf=<optimized out>, cmcf=<optimized out>, ctx_index=<optimized out>, module=<optimized out>, cf=<optimized out>) at src/http/ngx_http.c:599
<Nginx function calls stripped for clarity>
```
At this point I was completely hyped!!! I "found" a memory leak in OpenSSL! With so much excitement I started to read and try to understand OpenSSL code that's been under development since the [nineties](https://github.com/openssl/openssl/blob/master/crypto/x509/by_file.c#L2). With so much excitement I spent next couple of days and nights to understand those functions and try to find out the memory leak that I was so so so sure that was caused by a bug in of those functions. Seeing a lot of memory leak bug reports for OpenSSL(and specifically related to the functions above) my confidence doubled so that I spent couple of more nights on trying to find the bug! So basically what those function calls do is to first open the trusted certificate file, then allocate a buffer(4096 bytes) and read next 4Kb from the file into the buffer then decrypt the data and convert it to the respective internal representaton of OpenSSL and save it in the certificate store of the given SSL context(which belongs to a location section context). So whenever in the future Nginx needs to do SSL client verification, in the context of that `location` section, it will call `SSL_get_verify_result` function from OpenSSL and pass the saved SSL context from the beginning. Then that function will use the loaded and internalized trusted certificate to verify the client. So the outcome of those nights is that I learnt how all these things fit together but still could not find any bug! I also learnt that that suspicious `mmap` call was initiated by `malloc` which was triggered by `CRYPTO_malloc` from an another OpenSSL function to expand the cert store size so that it can fit the decrypted and internalyzed certificate data. Now that I learnt what exactly was happening I realized that not freeing that allocated memory is expected because OpenSSL keeps it for future potential uses in the lifetime of the process. **But the main issue why Nginx memory consumption grew so fast when `lua_ssl_trusted_certificate` set was still a mystery.** However with all these data I have it's obvious that it was the result of that `mmap` calls per location section. At this point I decided to extract the problem out of Openresty/Nginx context and write a standalone C program that uses the exact same OpenSSL API calls to load the same trusted certificate file repeatedly. Loading it repeatedly would simulate multiple location sections(5000 locations in this case):

```
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <stdint.h>
#include <sys/mman.h>
#include <openssl/ssl.h>
#include <malloc.h>
#include <unistd.h>

void read_cert() {
        const char ca_bundlestr[] = "/etc/ssl/certs/ca-certificates.crt";

        BIO               *outbio = NULL;
        int ret;

        SSL_CTX *ctx;

        outbio  = BIO_new_fp(stdout, BIO_NOCLOSE);

        SSL_library_init();
        SSL_load_error_strings();
        OpenSSL_add_all_algorithms();

        ctx = SSL_CTX_new(SSLv23_method());

        SSL_CTX_set_mode(ctx, SSL_MODE_RELEASE_BUFFERS);
        SSL_CTX_set_mode(ctx, SSL_MODE_NO_AUTO_CHAIN);
        SSL_CTX_set_read_ahead(ctx, 1);

        ret = SSL_CTX_load_verify_locations(ctx, ca_bundlestr, NULL);
        if (ret == 0)
                BIO_printf(outbio, "SSL_CTX_load_verify_locations failed");

        BIO_free_all(outbio);
        SSL_CTX_free(ctx);
}


int main() {
        int i = 0;
        for (i = 0; i < 5000; i++) {
                read_cert();
                //malloc_trim(0);
        }
        malloc_stats();
}
```
If I solve the issue here then I could solve it in the context of Openresty/Nginx too as this was equivalent to the original problem. But guess what, `strace` output for this program was different than what I expected!

```
...
read(3, "fqaEQn6/Ip3Xep1fvj1KcExJW4C+FEaG"..., 4096) = 4096
read(3, "IYWxvemF0Yml6dG9u\nc2FnaSBLZnQuMR"..., 4096) = 4096
read(3, "nVz\naXR2YW55a2lhZG8xHjAcBgkqhkiG"..., 4096) = 4096
read(3, "A\nMIIBCgKCAQEAy0+zAJs9Nt350Ulqax"..., 4096) = 4096
read(3, "MRAwDgYDVQQHEwdDYXJhY2FzMRkwFwYD"..., 4096) = 4096
read(3, "OR1YqI0JDs3G3eicJlcZaLDQP9nL9bFq"..., 4096) = 4096
read(3, "E7zelaTfi5m+rJsziO+1ga8bxiJTyPbH"..., 4096) = 4096
read(3, "Xtdj182d6UajtLF8HVj71lODqV0D1VNk"..., 4096) = 4096
read(3, "AAOCAQ8AMIIBCgKCAQEAt49VcdKA3Xtp"..., 4096) = 4096
brk(0x1cfb000)                          = 0x1cfb000
read(3, "396gwpEWoGQRS0S8Hvbn+mPeZqx2pHGj"..., 4096) = 4096
read(3, "QYwDwYDVR0T\nAQH/BAUwAwEB/zANBgkq"..., 4096) = 4096
read(3, "ETzsemQUHS\nv4ilf0X8rLiltTMMgsT7B"..., 4096) = 4096
read(3, "wVU3RhYXQgZGVyIE5lZGVybGFuZGVuMS"..., 4096) = 4096
read(3, "N/uLicFZ8WJ/X7NfZTD4p7dN\ndloedl4"..., 4096) = 4096
read(3, "fzDtgUx3M2FIk5xt/JxXrAaxrqTi3iSS"..., 4096) = 4096
read(3, "sO+wmETRIjfaAKxojAuuK\nHDp2KntWFh"..., 4096) = 4096
read(3, "8z+uJGaYRo2aWNkkijzb2GShROfyQcsi"..., 4096) = 4096
read(3, "CydAXFJy3SuCvkychVSa1ZC+N\n8f+mQA"..., 4096) = 4096
...
```
Notice the `brk` call does not follow with `mmap` call anymore and also memory was not growing unexpectedly! Alright, I was so tired of this shit at this point and almost gave up. But my curiousity did not let me. I decided to read more on how memory allocation works. So normally when your program tries to allocate more memory it calls `malloc` function(or some modification of it) from `glibc`. `glibc` abstracts out a lot of dirty memory management work from the user space programs and provides an API to work with Virtual Memory. By default when a program calls `malloc` function to allocate more memory in the heap, `malloc` first will try to use `brk` function to move the break up in the memory address space as requested. But `brk` won't work if there's a hole in the heap. What that means is let's say you have 1Gb of heap and its all free. To create a whole in it you can directly call `mmap` function with a specific address(A), and size of memory you want and it will allocate that area in the heap(starting from address A) for you. But because the break is still at the beginning of the heap, when `(s)brk` function is called to allocate B > A bytes of memory it won't be able to complete the request because some part of the memory `brk` tries to allocate has already been allocated(hole). In this situations `malloc` uses `mmap` instead to allocate the requested memory. And because `mmap` calls are expensive to reduce the number of possible `mmap` calls `malloc` allocates 1Mb memory even if the requested memory size is lower than 1Mb. This is documented in the source code at [https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#406](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html#406). If you scroll up you can see that suspcious `mmap` call was allocating 1048576 bytes of memory which is equal to 1Mb - the default value `malloc` uses to call `mmap` when `brk` fails. Exciting!!! Putting the pieces together, the next obvious hypothesis to test is **`brk` call was followed by the `mmap` call in the context of Openresty but not in the standalone C program is because Openresty somewhere before loading the certificate file creates a hole in the memory.** This was not too hard to find by just `grep`ing through the PRs, issues and source code of [`lua-nginx-module`](https://github.com/openresty/lua-nginx-module). So turns out Luajit requires to work in a lower address space to be efficient this is why `lua-nginx-module` guys decided to do following in the very beginning of the program execution:

```
if (sbrk(0) < (void *) 0x40000000LL) {
    mmap(ngx_align_ptr(sbrk(0), getpagesize()), 1, PROT_READ,
         MAP_FIXED|MAP_PRIVATE|MAP_ANON, -1, 0);
}
```
For the complete source code please refer to the [original repository](https://github.com/openresty/lua-nginx-module/blob/37e5362088bd659e318aae568b268719bd0d6707/src/ngx_http_lua_module.c#L1291). Now I have not yet figured out how this let's Luajit to own the lower address space (I would appreciate if someone explains it in the comment) but it is definitely the source of the main issue. To prove that I copied this code to my standalone C program:

```
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <stdint.h>
#include <sys/mman.h>
#include <openssl/ssl.h>
#include <malloc.h>
#include <unistd.h>

#define ngx_align_ptr(p, a) \
        (u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))

ngx_http_lua_limit_data_segment(void) {
        if (sbrk(0) < (void *) 0x40000000LL) {
                mmap(ngx_align_ptr(sbrk(0), getpagesize()), 1, PROT_READ,
                                MAP_FIXED|MAP_PRIVATE|MAP_ANON, -1, 0);
        }
}

void read_cert() {
        const char ca_bundlestr[] = "/etc/ssl/certs/ca-certificates.crt";

        BIO               *outbio = NULL;
        int ret;

        SSL_CTX *ctx;

        outbio  = BIO_new_fp(stdout, BIO_NOCLOSE);

        SSL_library_init();
        SSL_load_error_strings();
        OpenSSL_add_all_algorithms();

        ctx = SSL_CTX_new(SSLv23_method());

        SSL_CTX_set_mode(ctx, SSL_MODE_RELEASE_BUFFERS);
        SSL_CTX_set_mode(ctx, SSL_MODE_NO_AUTO_CHAIN);
        SSL_CTX_set_read_ahead(ctx, 1);

        ret = SSL_CTX_load_verify_locations(ctx, ca_bundlestr, NULL);
        if (ret == 0)
                BIO_printf(outbio, "SSL_CTX_load_verify_locations failed");

        BIO_free_all(outbio);
        SSL_CTX_free(ctx);
}


int main() {
        ngx_http_lua_limit_data_segment();
        int i = 0;
        for (i = 0; i < 5000; i++) {
                read_cert();
                //malloc_trim(0);
        }
        malloc_stats();
        usleep(1000 * 60);
}
```

When I compile and run this program through `strace` I can see the exact same behaviour as I was seeing in the context of Openresty. To be even more sure I edited Openresty source code, commented out `ngx_http_lua_limit_data_segment` and compiled it and confirmed that memory grows does not happen. Eureka!!!


This was a lot of learning for me. As a result of the investigation I also created an [issue](https://github.com/openresty/lua-nginx-module/issues/1005) for lua-nginx-module repository to let them know about this behaviour. This becomes a real issue only if you have a lot of location section defined. For example if you have a big Nginx configuration file with 4k location sections and you add `lua_ssl_trusted_certificate` directive into the main configuration then during the reload/restart/start Nginx's memory consumption will grow up to ~4Gb(4k*1Kb) and will stay like that.
