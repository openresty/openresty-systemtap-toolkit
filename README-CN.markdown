NAME
====

nginx-systemtap-toolkit - 基于 [SystemTap](http://sourceware.org/systemtap/wiki) 为 NGINX 打造的实时分析和诊断工具集

Table of Contents
=================

* [NAME](#name)
* [Status](#status)
* [Prerequisites](#prerequisites)
    * [For old Linux systems](#for-old-linux-systems)
* [Permissions](#permissions)
* [Tools](#tools)
    * [ngx-active-reqs](#ngx-active-reqs)
    * [ngx-req-distr](#ngx-req-distr)
    * [ngx-shm](#ngx-shm)
    * [ngx-cycle-pool](#ngx-cycle-pool)
    * [ngx-leaked-pools](#ngx-leaked-pools)
    * [ngx-backtrace](#ngx-backtrace)
    * [ngx-body-filters](#ngx-body-filters)
    * [ngx-header-filters](#ngx-header-filters)
    * [ngx-pcrejit](#ngx-pcrejit)
    * [ngx-sample-bt](#ngx-sample-bt)
    * [sample-bt](#sample-bt)
    * [ngx-sample-lua-bt](#ngx-sample-lua-bt)
    * [fix-lua-bt](#fix-lua-bt)
    * [ngx-lua-bt](#ngx-lua-bt)
    * [ngx-sample-bt-off-cpu](#ngx-sample-bt-off-cpu)
    * [sample-bt-off-cpu](#sample-bt-off-cpu)
    * [ngx-sample-bt-vfs](#ngx-sample-bt-vfs)
    * [sample-bt-vfs](#sample-bt-vfs)
    * [ngx-accessed-files](#ngx-accessed-files)
    * [accessed-files](#accessed-files)
    * [ngx-pcre-stats](#ngx-pcre-stats)
    * [ngx-accept-queue](#ngx-accept-queue)
    * [tcp-accept-queue](#tcp-accept-queue)
    * [ngx-recv-queue](#ngx-recv-queue)
    * [tcp-recv-queue](#tcp-recv-queue)
    * [ngx-lua-shdict](#ngx-lua-shdict)
    * [ngx-lua-conn-pools](#ngx-lua-conn-pools)
    * [check-debug-info](#check-debug-info)
    * [ngx-phase-handlers](#ngx-phase-handlers)
    * [resolve-inlines](#resolve-inlines)
    * [resolve-src-lines](#resolve-src-lines)
* [Community](#community)
    * [English Mailing List](#english-mailing-list)
    * [Chinese Mailing List](#chinese-mailing-list)
* [Bugs and Patches](#bugs-and-patches)
* [TODO](#todo)
* [Author](#author)
* [Copyright & License](#copyright--license)
* [See Also](#see-also)

状态
======
这些脚本可以用在生产环境。

前提条件
=============

你的 Linux 系统需要 systemtap 2.1+ 和 perl 5.6.1+ 及以上的版本。如果要从源码编译最新版本的 systemtap，你可以参考这个文档：http://openresty.org/#BuildSystemtap

另外，如果你不是从源码编译的 NGINX，你需要保证你的 NGINX 和其他依赖组件的 （DWARF）调试信息已经打开了（或者单独安装了）。

最后，你也需要安装 kernel debug symbols 和 kernel headers。通常只用在你的 Linux 系统中，安装和 `kernel` 包匹配的 `kernel-devel` 和 `kernel-debuginfo` 就可以了。

对于旧的 Linux 系统
---------------------
如果你的 Linux 内核版本低于 3.5，那么你可能需要给内核打上这个补丁（如果你之前没有打的话）：[utrace  patch](http://sourceware.org/systemtap/wiki/utrace)，这样才能让你的 systemtap 安装得到用户空间追踪的支持。但是如果你使用的是 RedHat 系列的 Linux 发行版本（比如RHEL, CentOS 和 Fedora），那么旧的内核也应该已经安装了 utrace 这个补丁。

3.5+ 的主流 Linux 内核，都会有探针 API 的支持，以便对用户空间进行追踪。

[Back to TOC](#table-of-contents)

权限
===========
运行基于 systemtap 的工具集需要特别的用户权限。为了避免用 root 用户运行这些工具集，你可以把自己的用户名（非 root 用户），加入到 `stapusr` 和 `staprun` 用户组中。
但是如果你自己的账户名和 正在运行 NGINX 进程的账户名不同，那么你还是需要运行 "sudo" 或者其他同样效果的命令，让这些工具集带有 root 访问权限的运行起来。

[Back to TOC](#table-of-contents)

工具集
=====

[Back to TOC](#table-of-contents)

ngx-active-reqs
---------------

这个工具会列出来指定的 NGINX worker 或者 master 进程正在处理的所有活跃请求的详细信息。
当指定为 master 进程的 pid 时，它所有的 worker 进程都会被监控。

这里有一个例子：
    # 假设 NGINX worker 的 pid 是 32027

    $ ./ngx-active-reqs -p 32027
    Tracing 32027 (/opt/nginx/sbin/nginx)...

    req "GET /t?", time 0.300 sec, conn reqs 18, fd 8
    req "GET /t?", time 0.276 sec, conn reqs 18, fd 7
    req "GET /t?", time 0.300 sec, conn reqs 18, fd 9
    req "GET /t?", time 0.300 sec, conn reqs 18, fd 10
    req "GET /t?", time 0.300 sec, conn reqs 18, fd 11
    req "GET /t?", time 0.300 sec, conn reqs 18, fd 12
    req "GET /t?", time 0.300 sec, conn reqs 18, fd 13
    req "GET /t?", time 0.300 sec, conn reqs 18, fd 14
    req "GET /t?", time 0.276 sec, conn reqs 18, fd 15
    req "GET /t?", time 0.276 sec, conn reqs 18, fd 16

    found 10 active requests.
    212 microseconds elapsed in the probe handler.

`time` 字段是当前请求从开始到现在的时间（单位是秒）。
`conn reqs` 字段列出来当前（keep-alive）下游连接已经处理的请求数。
`fd` 字段是当前下游连接的文件描述符 ID。
`-m` 选型会让这个工具去分析每一个活跃请求的请求内存池：

    $ ./ngx-active-reqs -p 12141 -m
    Tracing 12141 (/opt/nginx/sbin/nginx)...

    req "GET /t?", time 0.100 sec, conn reqs 11, fd 8
        pool chunk size: 4096
        small blocks (< 4017): 3104 bytes used, 912 bytes unused
        large blocks (>= 4017): 0 blocks, 0 bytes (used)
        total used: 3104 bytes

    req "GET /t?", time 0.100 sec, conn reqs 11, fd 7
        pool chunk size: 4096
        small blocks (< 4017): 3104 bytes used, 912 bytes unused
        large blocks (>= 4017): 0 blocks, 0 bytes (used)
        total used: 3104 bytes

    req "GET /t?", time 0.100 sec, conn reqs 11, fd 9
        pool chunk size: 4096
        small blocks (< 4017): 3104 bytes used, 912 bytes unused
        large blocks (>= 4017): 0 blocks, 0 bytes (used)
        total used: 3104 bytes

    total memory used for all 3 active requests: 9312 bytes
    274 microseconds elapsed in the probe handler.

对于并不十分忙碌的 NGINX 服务器，可以用 `-p` 选项值，方便的指定 NGINX 的 master 进程 pid。
另外一个有用的选项是 `-k`，它会在当前事件循环中没有活跃请求的时候，一直保持探测。

[Back to TOC](#table-of-contents)

ngx-req-distr
-------------

这个工具分析（下游）请求和连接，在指定的 NGINX master 进程的所有 NGINX worker 进程中的分布。

    # NGINX master 进程的 pid 存放在这个 pid 文件中
    #   /opt/nginx/logs/nginx.pid

    $ ./ngx-req-distr -m `cat /opt/nginx/logs/nginx.pid`
    Tracing 4394 4395 4396 4397 4398 4399 4400 4401 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    worker 4394:    0 reqs
    worker 4395:    200 reqs
    worker 4396:    1600 reqs
    worker 4397:    0 reqs
    worker 4398:    2100 reqs
    worker 4399:    4400 reqs
    worker 4400:    0 reqs
    worker 4401:    1701 reqs

    $ ./ngx-req-distr -c -m `cat /opt/nginx/logs/nginx.pid`
    Tracing 4394 4395 4396 4397 4398 4399 4400 4401 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    worker 4394:    0 reqs, 0 conns
    worker 4395:    2100 reqs,      21 conns
    worker 4396:    501 reqs,       6 conns
    worker 4397:    2100 reqs,      21 conns
    worker 4398:    100 reqs,       1 conns
    worker 4399:    2200 reqs,      22 conns
    worker 4400:    800 reqs,       8 conns
    worker 4401:    2200 reqs,      22 conns

[Back to TOC](#table-of-contents)

ngx-shm
-------

这个工具分析在指定的正在运行的 NGINX 进程中，所有的共享内存区域。

    # 你需要确保指定的 worker 仍然在处理请求
    # 否则必须在你的 nginx.conf 里面设置 timer_resoluation

    # 假设 NGINX worker 的 pid 是 15218

    $ cd /path/to/nginx-systemtap-toolkit/

    # 列出内存区域
    $ ./ngx-shm -p 15218
    Tracing 15218 (/opt/nginx/sbin/nginx)...

    shm zone "one"
        owner: ngx_http_limit_req
        total size: 5120 KB

    shm zone "two"
        owner: ngx_http_file_cache
        total size: 7168 KB

    shm zone "three"
        owner: ngx_http_limit_conn
        total size: 3072 KB

    shm zone "dogs"
        owner: ngx_http_lua_shdict
        total size: 100 KB

    Use the -n <zone> option to see more details about each zone.
    34 microseconds elapsed in the probe.

    # show the zone details
    $ ./ngx-shm -p 15218 -n dogs
    Tracing 15218 (/opt/nginx/sbin/nginx)...

    shm zone "dogs"
        owner: ngx_http_lua_shdict
        total size: 100 KB
        free pages: 88 KB (22 pages, 1 blocks)

    22 microseconds elapsed in the probe.

[Back to TOC](#table-of-contents)

ngx-cycle-pool
--------------

这个工具计算在指定的 NGINX （worker）进程内，NGINX 全局 "cycle pool" 的内存占用。

"cycle pool" 主要是提供给配置相关的数据块分配，以及 NGINX 服务器配置生命周期中其他长期使用的数据块
（比如为 ngx_lua 模块，而存储在正则表达式缓存中编译好的 PCRE 数据）。

    # 你需要确保 worker 正在处理请求
    # 或者 nginx.conf 里面设置了 timer_resoluation

    # 假设 NGINX worker pid 是 15004

    $ ./ngx-cycle-pool -p 15004
    Tracing 15004 (/usr/local/nginx/sbin/nginx)...

    pool chunk size: 16384
    small blocks (< 4096): 96416 bytes used, 1408 bytes unused
    large blocks (>= 4096): 6 blocks, 26352 bytes (used)
    total used: 122768 bytes

    12 microseconds elapsed in the probe handler.

"large blocks" 的内存块大小，近似以 Linux 上 glibc 的 `malloc` 初始化实现为基准。
如果你用其他分配器取代了 `malloc`，然后这个工具很有可能内存访问错误而退出，
或者给出一个毫无意义的 "large blocks" 总大小（但即使在这样恶劣的情况下，SystemTap 也不会对正在分析的 NGINX 进程有任何影响）。

[Back to TOC](#table-of-contents)

ngx-leaked-pools
----------------

跟踪 NGINX 内存池的创建和销毁，以及给出泄露池前 10 的调用栈。

调用栈是 16 进制地址的原始格式。
可以使用 `ngx-backtrace` 工具打印出源代码文件名，源码所在行数以及函数名。

    # 假设 NGINX worker pid 是 5043

    $ ./ngx-leaked-pools -p 5043
    Tracing 5043 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    28 pools leaked at backtrace 0x4121aa 0x43c851 0x4300a0 0x42746a 0x42f927 0x4110d8 0x3d35021735 0x40fe29
    17 pools leaked at backtrace 0x4121aa 0x44d7bd 0x44e425 0x44fcc1 0x47996d 0x43908a 0x4342c3 0x4343bd 0x43dfcc 0x44c20e 0x4300a0 0x42746a 0x42f927 0x4110d8 0x3d35021735 0x40fe29
    16 pools leaked at backtrace 0x4121aa 0x44d7bd 0x44e425 0x44fcc1 0x47996d 0x43908a 0x4342c3 0x4343bd 0x43dfcc 0x43f09e 0x43f6e6 0x43fcd5 0x43c9fb 0x4300a0 0x42746a 0x42f927 0x4110d8 0x3d35021735 0x40fe29

    Run the command "./ngx-backtrace -p 5043 <backtrace>" to get details.
    For total 200 pools allocated.

    $ ./ngx-backtrace -p 5043 0x4121aa 0x44d7bd 0x44e425 0x44fcc1 0x47996d 0x43908a 0x4342c3 0x4343bd
    ngx_create_pool
    src/core/ngx_palloc.c:44
    ngx_http_upstream_connect
    src/http/ngx_http_upstream.c:1164
    ngx_http_upstream_init_request
    src/http/ngx_http_upstream.c:645
    ngx_http_upstream_init
    src/http/ngx_http_upstream.c:447
    ngx_http_redis2_handler
    src/ngx_http_redis2_handler.c:108
    ngx_http_core_content_phase
    src/http/ngx_http_core_module.c:1407
    ngx_http_core_run_phases
    src/http/ngx_http_core_module.c:890
    ngx_http_handler
    src/http/ngx_http_core_module.c:872

这个脚本需要 NGINX 实例已经打上最新的 dtrace 补丁。可以从 [nginx-dtrace](https://github.com/openresty/nginx-dtrace) 项目里获得更多细节。

[OpenResty](http://openresty.org/) 1.2.3.3+ 版本的安装包，默认已经包含了正确的 dtrace 补丁。
你只用在 build 的时候，加上 `--with-dtrace-probes` 这个配置选项。

[Back to TOC](#table-of-contents)

ngx-backtrace
-------------

从 `ngx-leaked-pools` 之类的工具生成的 16 进制地址的原始调用栈，转换为人类可读的格式。

    # 假设 NGINX worker 进程 pid 是 5043

    $ ./ngx-backtrace -p 5043 0x4121aa 0x44d7bd 0x44e425 0x44fcc1 0x47996d 0x43908a 0x4342c3 0x4343bd
    ngx_create_pool
    src/core/ngx_palloc.c:44
    ngx_http_upstream_connect
    src/http/ngx_http_upstream.c:1164
    ngx_http_upstream_init_request
    src/http/ngx_http_upstream.c:645
    ngx_http_upstream_init
    src/http/ngx_http_upstream.c:447
    ngx_http_redis2_handler
    src/ngx_http_redis2_handler.c:108
    ngx_http_core_content_phase
    src/http/ngx_http_core_module.c:1407
    ngx_http_core_run_phases
    src/http/ngx_http_core_module.c:890
    ngx_http_handler
    src/http/ngx_http_core_module.c:872

[Back to TOC](#table-of-contents)

ngx-body-filters
----------------

按照实际运行的顺序，打印出所有输出体过滤器。
    # 假设 NGINX worker 进程 pid 是 30132

    $ ./ngx-body-filters -p 30132
    Tracing 30132 (/opt/nginx/sbin/nginx)...

    WARNING: Missing unwind data for module, rerun with 'stap -d ...'
    ngx_http_range_body_filter
    ngx_http_copy_filter
    ngx_output_chain
    ngx_http_lua_capture_body_filter
    ngx_http_image_body_filter
    ngx_http_charset_body_filter
    ngx_http_ssi_body_filter
    ngx_http_postpone_filter
    ngx_http_gzip_body_filter
    ngx_http_chunked_body_filter
    ngx_http_write_filter

    113 microseconds elapsed in the probe handler.

[Back to TOC](#table-of-contents)

ngx-header-filters
------------------

按照实际运行的顺序，打印出所有输出头过滤器。

    $ ./ngx-header-filters -p 30132
    Tracing 30132 (/opt/nginx/sbin/nginx)...

    WARNING: Missing unwind data for module, rerun with 'stap -d ...'
    ngx_http_not_modified_header_filter
    ngx_http_lua_capture_header_filter
    ngx_http_headers_filter
    ngx_http_image_header_filter
    ngx_http_charset_header_filter
    ngx_http_ssi_header_filter
    ngx_http_gzip_header_filter
    ngx_http_range_header_filter
    ngx_http_chunked_header_filter
    ngx_http_header_filter

    137 microseconds elapsed in the probe handler.

[Back to TOC](#table-of-contents)

ngx-pcrejit
-----------

这个脚本跟踪指定 NGINX worker 进程里面 PCRE 编译的正则表达式的执行（比如 `pcre_exec` 的调用），
并且检测它们是否被 JIT 执行。

    # 假设正在处理请求的 NGINX worker 进程是 31360.

    $ ./ngx-pcrejit -p 31360
    Tracing 31360 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    ngx_http_lua_ngx_re_match: 1000 of 2000 are PCRE JIT'd.
    ngx_http_regex_exec: 0 of 1000 are PCRE JIT'd.

当 PCRE 是静态链接到你的 NGINX 时，记得在你的 PCRE 编译时打开调试符号。

所以你应该这样子 build 你的 NGINX 和 PCRE：

    ./configure --with-pcre=/path/to/my/pcre-8.31 \
        --with-pcre-jit \
        --with-pcre-opt=-g \
        --prefix=/opt/nginx
    make -j8
    make install

对于动态链接 PCRE 的情况，你仍然需要为 PCRE 安装调试符号（或者是 debuginfo RPM 包，对于基于 Yum 的系统）。

[Back to TOC](#table-of-contents)

ngx-sample-bt
-------------

这个工具已经被重命名为 [sample-bt](#sample-bt)，因为这个工具并不只针对 NGINX，所以保留 `ngx-` 这个前缀没有什么意义。

[Back to TOC](#table-of-contents)

sample-bt
---------

这个脚本可以对你指定的 *任意* 用户进程（没错，不仅仅是 NGINX！）进行调用栈的采样。调用栈可以是用户空间，可以是内核空间，或者是两者兼得。
它的输出是汇总后的调用栈（按照总数）。

例如，采样一个正在运行的 NGINX worker 进程（pid 是 8736）的用户空间 5 秒钟：

    $ ./sample-bt -p 8736 -t 5 -u > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in user-space only...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_df60590ce8827444bfebaf5ea938b5a_11577'
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 24

结果的输出文件 `a.bt` 可以使用 Brendan Gregg 的 [FlameGraph 工具集](https://github.com/brendangregg/FlameGraph) 来生成火焰图:

    stackcollapse-stap.pl a.bt > a.cbt
    flamegraph.pl a.cbt > a.svg

这里的 `stackcollapse-stap.pl` 和 `flamegraph.pl` 都来自 FlameGraph 工具集。
如果一切顺利，你可以用你的浏览器打开这个 `a.svg` 文件。

这里有一个采样用户空间的火焰图示例（请用一个支持 SVG 渲染的现代浏览器打开这个链接）:

http://agentzh.org/misc/nginx/user-flamegraph.svg

想获得更多火焰图相关的信息，可以看下 Brendan Gregg 的这些博客:

* [Flame Graphs](http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/)
* [Linux Kernel Performance: Flame Graphs](http://dtrace.org/blogs/brendan/2012/03/17/linux-kernel-performance-flame-graphs/)

你也可以指定 `-k` 选项，来采样内核空间的调用栈，比如

    $ ./sample-bt -p 8736 -t 5 -k > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in kernel-space only...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_bf5516bdbf2beba886507025110994e_11738'
    WARNING: Time's up. Quitting now...(it may take a while)

只有指定的 NGINX worker 进程的内核空间代码会被采样。

一个只有内核空间采样的火焰图示例在这里:

http://agentzh.org/misc/nginx/kernel-flamegraph.svg

你也可以指定 `-k` 和 `-u` 选项，来同时采样用户空间和内核空间，比如

    $ ./sample-bt -p 8736 -t 5 -uk > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in both user-space and kernel-space...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_90327f3a19b0e42dffdef38d53a5860_11799'
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 38
    WARNING: There were 73 transport failures.

一个用户和内核空间采样的火焰图示例在这里:

http://agentzh.org/misc/nginx/user-kernel-flamegraph.svg

实际上，这个脚本非常通用，也可以采样 NGINX 之外的其他用户进程。

它对目标进程的开销通常比较小。比如，在 Linux 内核 3.6.10 和 systemtap 2.5 的环境中，使用 `ab -k -c2 -n100000` 来进行测试，NGINX worker 进程处理最简单的 "hello world" 请求，吞吐量（req/sec）只下降了 11%（只有这个工具运行时）。对于非常成熟的生产环境的程序来说，这个影响会更小。比如在一个生产水平的 Lua CDN 应用中，只观察到 6% 的吞吐量下降。

[Back to TOC](#table-of-contents)

ngx-sample-lua-bt
-----------------

*警告* 这个工具只能和解释后的 Lua 代码工作，并且有很多的限制。
对于 LuaJIT 2.1，推荐使用新的 [ngx-lj-lua-stacks](https://github.com/openresty/stapxx#ngx-lj-lua-stacks) 工具来采样解释后 和/或 编译后的 Lua 代码。

和 [sample-bt](#sample-bt) 这个脚本类似, 不过采样的是 Lua 语言级别的调用栈。

当你在 NGINX 里面使用标准的 Lua 5.1 解释器时，需要指定 `--lua51` 选项；如果用的是 LuaJIT 2.0 就指定 `--luajit20`。

除了 NGINX 可执行文件之外，你还需要为 Lua 库打开或者安装调试符号。

同时，在 build 你的 Lua 库的时候，不要忽略帧指针（frame pointers）。

如果使用的是 LuaJIT 2.0, 你需要这样去 build LuaJIT 2.0 库:

    make CCDEBUG=-g

这个脚本生成的 Lua 调用栈，在 Lua 函数定义的地方会使用 Lua 代码源文件名和所在行数。
所以为了获得更有意义的调用信息，你可以调用 `fix-lua-bt` 脚本去处理 `ngx-sample-lua-bt` 的输出。

这里有一个例子，NGINX 使用的是标准 Lua 5.1 解释器：

    # sample at 1K Hz for 5 seconds, assuming the Nginx worker
    #   or master process pid is 9766.
    $ ./ngx-sample-lua-bt -p 9766 --lua51 -t 5 > tmp.bt
    WARNING: Tracing 9766 (/opt/nginx/sbin/nginx) for standard Lua 5.1...
    WARNING: Time's up. Quitting now...(it may take a while)

    $ ./fix-lua-bt tmp.bt > a.bt

如果使用的是 LuaJIT 2.0:

    # sample at 1K Hz for 5 seconds, assuming the Nginx worker
    #   or master process pid is 9768.
    $ ./ngx-sample-lua-bt -p 9768 --luajit20 -t 5 > tmp.bt
    WARNING: Tracing 9766 (/opt/nginx/sbin/nginx) for LuaJIT 2.0...
    WARNING: Time's up. Quitting now...(it may take a while)

    $ ./fix-lua-bt tmp.bt > a.bt

得到的输出文件 `a.bt`，可以用 Brendan Gregg 的 [FlameGraph 工具集](https://github.com/brendangregg/FlameGraph) 来生成火焰图:

    stackcollapse-stap.pl a.bt > a.cbt
    flamegraph.pl a.cbt > a.svg

 `stackcollapse-stap.pl` 和 `flamegraph.pl` 都来自 FlameGraph 工具集。
 如果一切顺利，你可以用浏览器打开 `a.svg` 这个文件。

这里有一个采样用户空间的火焰图示例（请用一个支持 SVG 渲染的现代浏览器打开这个链接）:

http://agentzh.org/misc/flamegraph/lua51-resty-mysql.svg

想获得更多火焰图相关的信息，可以看下 Brendan Gregg 的这些博客:

* [Flame Graphs](http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/)
* [Linux Kernel Performance: Flame Graphs](http://dtrace.org/blogs/brendan/2012/03/17/linux-kernel-performance-flame-graphs/)

如果用 `-t` 选项指定了 NGINX master 进程的 pid，那这个工具会同时自动探测它所有的 worker 进程。
If the pid of the Nginx master proces is specified as the `-t` option value,
then this tool will automatically probe all its worker processes at the same time.

[Back to TOC](#table-of-contents)

fix-lua-bt
----------

让 `ngx-sample-lua-bt` 生成的原始调用栈更有可读性。

`ngx-sample-lua-bt` 生成的原始调用栈看上去是这样子的:

    C:0x7fe9faf52dd0
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:65
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:176
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:418
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:711

这个脚本处理过之后，我们看到的是

    C:0x7fe9faf52dd0
    resty.mysql:_get_byte3
    resty.mysql:_recv_packet
    resty.mysql:_recv_field_packet
    resty.mysql:read_result

这里有一个例子:

    ./fix-lua-bt tmp.bt > a.bt

其中输入文件 `tmp.bt` 是之前 `ngx-sample-lua-bt` 生成的。

参考 `ngx-sample-lua-bt`。

[Back to TOC](#table-of-contents)

ngx-lua-bt
----------

这个工具可以把 NGINX worker 进程中 Lua 的当前调用栈 dump 出来。

这个工具在定位 Lua 热循环引起的 NGINX worker 持续 100% CPU 占用问题的时候非常有效。

如果用的是 LuaJIT 2.0, 请指定 --luajit20 选项, 像这样:

    $ ./ngx-lua-bt -p 7599 --luajit20
    WARNING: Tracing 7599 (/opt/nginx/sbin/nginx) for LuaJIT 2.0...
    C:lj_cf_string_find
    content_by_lua:2
    content_by_lua:1

如果用的是标准 Lua 5.1 解释器, 请指定 --lua51 选项:

    $ ./ngx-lua-bt -p 13611 --lua51
    WARNING: Tracing 13611 (/opt/nginx/sbin/nginx) for standard Lua 5.1...
    C:str_find
    content_by_lua:2
    [tail]
    content_by_lua:1

[Back to TOC](#table-of-contents)

ngx-sample-bt-off-cpu
---------------------

这个工具已经重命名为 [sample-bt-off-cpu](#sample-bt-off-cpu)，因为这个工具并不只针对 NGINX，所以保留 `ngx-` 这个前缀没有什么意义。

[Back to TOC](#table-of-contents)

sample-bt-off-cpu
-----------------

类似 [sample-bt](#sample-bt)，不过是用来分析某个用户进程 off-CPU time（不只是 NGINX，其他的应用也可以分析）。

为什么 off-CPU time 这么重要? 可以从 Brendan Gregg 这篇非常棒的博客 "Off-CPU Performance Analysis" 里面看到细节:

http://dtrace.org/blogs/brendan/2011/07/08/off-cpu-performance-analysis/

这个工具默认是采样用户空间调用栈。 并且输出里面 1 个逻辑上的采样，对应 1 微秒的 off-CPU time。

这里有个例子来演示这个工具的用法:

    # assuming the nginx worker process to be analyzed is 10901.
    $ ./sample-bt-off-cpu -p 10901 -t 5 > a.bt
    WARNING: Tracing 10901 (/opt/nginx/sbin/nginx)...
    WARNING: _stp_read_address failed to access memory location
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 23

这里 `-t 5` 选项让工具采样 5 秒钟。

产生的 `a.bt` 文件， 就像 [sample-bt](#sample-bt) 以及其他脚本生成的文件一样，可以拿来生成火焰图。
这个类型的火焰图叫做 "off-CPU Flame Graphs"，而经典的火焰图本质上是 "on-CPU Flame Graphs"。

下面是一个 "off-CPU 火焰图" 的例子， NGINX worker 进程正在用 lua-resty-mysql 访问 MySQL：

http://agentzh.org/misc/flamegraph/off-cpu-lua-resty-mysql.svg

默认的，off-CPU 时间间隔小于 4 us（微秒，*其实 us 代表的是 μs，代码里面不好敲，改为 us*）的会被丢弃，不显示出来。你可以用 `--min` 来修改这个阈值：

    $ ./sample-bt-off-cpu -p 12345 --min 10 -t 10

这个命令的意思是对 pid 为 12345 的用户进程进行一共 10 秒钟的采样，并且忽略 off-CPU 时间间隔小于 10 微秒的采样。

`-l` 选项可以控制导出不同调用栈的上限。默认情况下，会导出 1024 个不同的最热的调用栈。

`--distr` 选项可以指定为所有比 `--min` 这个阈值大的 off-CPU 时间间隔，打印出一个以2为底数的对数柱状图。比如，

    $ ./sample-bt-off-cpu -p 10901 -t 3 --distr --min=1
    WARNING: Tracing 10901 (/opt/nginx/sbin/nginx)...
    Exiting...Please wait...
    === Off-CPU time distribution (in us) ===
    min/avg/max: 2/79/1739
    value |-------------------------------------------------- count
        0 |                                                     0
        1 |                                                     0
        2 |@                                                   10
        4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@        259
        8 |@@@@@@@                                             44
       16 |@@@@@@@@@@                                          62
       32 |@@@@@@@@@@@@@                                       79
       64 |@@@@@@@                                             43
      128 |@@@@@                                               31
      256 |@@@                                                 22
      512 |@@@                                                 22
     1024 |                                                     4
     2048 |                                                     0
     4096 |                                                     0

通过这个图，我们可以看到大部分采样都落在 `[4us, 8us)` 的 off-CPU 时间间隔范围内。最大的 off-CPU 时间间隔是 1739 微秒，也就是 1.739 毫秒。

你可以指定 `-k` 选项来采样内核空间的调用栈，而不是采样用户空间调用栈。如果你两个都想采样，你可以把 `-k` 和 `-u` 这两个选项都加上。

[Back to TOC](#table-of-contents)

ngx-sample-bt-vfs
-----------------

这个工具已经被重命名为 [sample-bt-vfs](#sample-bt-vfs)，因为这个工具并不只针对 NGINX，所以保留 `ngx-` 这个前缀没有什么意义。

[Back to TOC](#table-of-contents)

sample-bt-vfs
-------------

类似 [sample-bt](#sample-bt)，但是这个工具是在虚拟文件系统（VFS）之上采样用户空间调用栈，以便渲染出文件 I/O 火焰图，这个火焰图可以准确的反映出在任意正在运行的用户进程中，文件 I/O 数据量或者文件 I/O 延迟在不同的用户空间代码路径的分布。

默认的，一个调用栈的采用对应的是一个字节的数据量（读或者写）。默认情况下，`vfs_read` 和 `vfs_write` 都会被追踪。比如，

    $ ./sample-bt-vfs -p 12345 -t 3 > a.bt
    WARNING: Tracing 20636 (/opt/nginx/sbin/nginx)...
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 2

我们可以这样渲染出一个读/写 VFS I/O 火焰图：

    $ stackcollapse-stap.pl a.bt > a.cbt
    $ flamegraph.pl a.cbt > a.svg

这里的工具 `stackcollapse-stap.pl` 和 `flamegraph.pl` 都来自 Brendan Gregg 的 FlameGraph 工具集:

https://github.com/brendangregg/FlameGraph

这里有一个 "文件 I/O 火焰图" 的例子:

http://agentzh.org/misc/flamegraph/vfs-index-page-rw.svg

这个火焰图呈现的是一个 NGINX worker 进程处理请求而加载默认的起始页面（比如 `/index.html`）。我们可以看到在标准访问日志模块里面的文件写操作，以及在标准静态模块里面的文件读操作。
这个火焰图里面所有的采样空间是 1481361，也就是说在 VFS 中一共有 1481361 个字节的实际读写数据。

你可以指定 `-r` 选项来只跟踪文件读操作：

    $ ./sample-bt-vfs -p 12345 -t 3 -r > a.bt

这里有一个 "文件读取操作火焰图" 的例子，是采样一个正在加载默认起始页面的 NGINX 进程：

http://agentzh.org/misc/flamegraph/vfs-index-page-r.svg

我们看到火焰图里面呈现的只有标准的 NGINX "静态" 模块。

类似的，你可以指定 `-w` 选项来只跟踪文件写操作：


    $ ./sample-bt-vfs -p 12345 -t 3 -w > a.bt

这里有一个 NGINX(打开了调试日志) 的 "文件写操作火焰图":

http://agentzh.org/misc/flamegraph/vfs-debug-log.svg

下面是另外一个 "文件写操作火焰图"，NGINX 关闭了调试日志：

http://agentzh.org/misc/flamegraph/vfs-access-log-only.svg

我们可以看到这里只有访问日志的写操作出现在火焰图中。

这里不要混淆了文件 I/O 和磁盘 I/O，因为我们仅仅在虚拟文件系统（VFS）这个（高）级别进行了探测。所以这里系统 page cache 可以节省很多磁盘读操作。

一般来说，我们对花费在 VFS 读写上面的延迟（比如，时间）更感兴趣。你可以指定 `--latency` 选项去跟踪内核调用延迟而不是数据量：

    $ ./sample-bt-vfs -p 12345 -t 3 --latency > a.bt

这个例子里面，1个采样相当于1毫秒的文件 I/O 时间（或者更准确的说，是 `vfs_read` 或者 `vfs_write` 的调用时间）。

这里有一个对应的示例:

http://agentzh.org/misc/flamegraph/vfs-latency-index-page-rw.svg

火焰图里面展示的一共有 1918669 个采样，意味着在 3 秒的取样间隔中，一共有 1,918,669 毫秒（也就是 1.9 秒）花在文件读写上面。

你也可以也 `--latency` 选项一起使用 `-r` 或者 `-w` 来筛选出文件读或者写操作。

这个工具可以用来检测带调试符号的任意用户进程（不仅仅是 NGINX 进程）。

[Back to TOC](#table-of-contents)

ngx-accessed-files
------------------

这个工具已经被重命名为 [accessed-files](#accessed-files)，因为这个工具并不只针对 NGINX，所以保留 `ngx-` 这个前缀没有什么意义。

[Back to TOC](#table-of-contents)

accessed-files
--------------

通过指定 `-p` 选项，找出来任意用户进程（对的，不限于 NGINX！）最常读写的文件名。

`-r` 选项可以指定去分析被读取的文件，比如,

    $ ./accessed-files -p 8823 -r
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    === Top 10 file reads ===
    #1: 10 times, 720 bytes reads in file index.html.
    #2: 5 times, 75 bytes reads in file helloworld.html.
    #3: 2 times, 26 bytes reads in file a.html.

`-w` 选项是指定分析被写入的文件：

    $ ./accessed-files -p 8823 -w
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    === Top 10 file writes ===
    #1: 17 times, 1600 bytes writes in file access.log.

你可以同时指定 `-r` 和 `-w` 两个选项:

    $ ./accessed-files -p 8823 -w -r
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    === Top 10 file reads/writes ===
    #1: 17 times, 1600 bytes reads/writes in file access.log.
    #2: 10 times, 720 bytes reads/writes in file index.html.
    #3: 5 times, 75 bytes reads/writes in file helloworld.html.
    #4: 2 times, 26 bytes reads/writes in file a.html.

默认的，Ctrl-C 会终止对进程的采样。`-t` 选项指定准确的秒数来控制采样周期，比如，

    $ ./accessed-files -p 8823 -r -t 5
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Please wait for 5 seconds.

    === Top 10 file reads ===
    #1: 10 times, 720 bytes reads in file index.html.
    #2: 5 times, 75 bytes reads in file helloworld.html.
    #3: 2 times, 26 bytes reads in file a.html.

默认的，最多打印 10 个不同的文件名。你可以用 `-l` 选项控制这个阈值。比如，

    $ ./accessed-files -p 8823 -r -l 20

[Back to TOC](#table-of-contents)

ngx-pcre-stats
--------------

这个工具会展示一个正在运行的 NGINX worker 进程中，PCRE 正则表达式执行效率的各类统计分析。

这个工具需要 Linux 内核的 uretprobes 支持。

同时你也需要确保 NGINX、PCRE 和 LuaJIT 在编译的时候，都已经开启了调试符号。
比如你在 NGINX 或者 OpenResty 中通过源码编译 PCRE，你在指定 `--with-pcre=PATH` 选项的同时，也需要指定 `--with-pcre-opt=-g` 这个选项。

下面的例子是分析指定 NGINX worker 进程中 PCRE 正则执行的时间分布。需要注意的是给出的时间是微秒（`us`）级别的，也就是 1e-6 秒。这里用了 `--exec-time-dist` 选项。

    $ ./ngx-pcre-stats -p 24528 --exec-time-dist
    Tracing 24528 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    Logarithmic histogram for pcre_exec running time distribution (us):
    value |-------------------------------------------------- count
        1 |                                                       0
        2 |                                                       0
        4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 36200
        8 |@@@@@@@@@@@@@@@@@@@@                               14707
       16 |                                                     105
       32 |@@@                                                 2892
       64 |                                                     114
      128 |                                                       0
      256 |                                                       0

同样的你可以指定 `--data-len-dist` 选项，来分析在独立运行中匹配到的那些字符串数据长度的分布。

    $ ./ngx-pcre-stats -p 24528 --data-len-dist
    Tracing 24528 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    Logarithmic histogram for data length distribution:
    value |-------------------------------------------------- count
        1 |                                                       0
        2 |                                                       0
        4 |@@@                                                 3001
        8 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  48016
       16 |                                                       0
       32 |                                                       0
          ~
     1024 |                                                       0
     2048 |                                                       0
     4096 |@@@                                                 3001
     8192 |                                                       0
    16384 |                                                       0

`--worst-time-top` 选项可以用来分析使用 ngx_lua 模块的 [ngx.re API](http://wiki.nginx.org/HttpLuaModule#ngx.re.match) 匹配到的各个正则的最差执行时间：

    $ ./ngx-pcre-stats -p 24528 --worst-time-top --luajit20
    Tracing 24528 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    Top N regexes with worst running time:
    1. pattern "elloA": 125us (data size: 5120)
    2. pattern ".": 76us (data size: 10)
    3. pattern "a": 64us (data size: 12)
    4. pattern "b": 29us (data size: 12)
    5. pattern "ello": 26us (data size: 5)

Note that the time values given above are just for individual runs and are not accumulated.

And the `--total-time-top` option is similar to `--worst-time-top`, but using accumulated regex execution time.

    $ ./ngx-pcre-stats -p 24528 --total-time-top --luajit20
    Tracing 24528 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    Top N regexes with longest total running time:
    1. pattern ".": 241038us (total data size: 330110)
    2. pattern "elloA": 188107us (total data size: 15365120)
    3. pattern "b": 28016us (total data size: 36012)
    4. pattern "ello": 26241us (total data size: 15005)
    5. pattern "a": 26180us (total data size: 36012)

The -t option can be used to specify the time period
(in seconds) for sampling instead of requiring the user to
hit Ctrl-C to end sampling:

    $ ./ngx-pcre-stats -p 8701 --total-time-top --luajit20 -t 5
    Tracing 8701 (/opt/nginx/sbin/nginx)...
    Please wait for 5 seconds.

    Top N regexes with longest total running time:
    1. pattern ".": 81us (total data size: 110)
    2. pattern "elloA": 62us (total data size: 5120)
    3. pattern "ello": 46us (total data size: 5)
    4. pattern "b": 19us (total data size: 12)
    5. pattern "a": 9us (total data size: 12)

LuaJIT 2.1 is also supported when the `--luajit20` is specified (yeah, I know the option name is confusing).

[Back to TOC](#table-of-contents)

ngx-accept-queue
----------------

This tool has been renamed to [tcp-accept-queue](#tcp-accept-queue) because this tool is not specific to Nginx
in any way and it makes no sense to keep the `ngx-` prefix in its name.

[Back to TOC](#table-of-contents)

tcp-accept-queue
----------------

This tool samples the SYN queue and ACK backlog queue for the sockets listening on the local port specified by the `--port` option
for the time interval when it is running. It can work on any server processes even it is not Nginx.

This is a real-time sampling tool.

SYN queue or ACK backlog queue overflowing often results in connecting timeout errors on the client side.

By default, the tool prints out up to 10 queue overflow events and then quits immediately. For example:

    $ ./tcp-accept-queue --port=80
    WARNING: Tracing SYN & ACK backlog queue overflows on the listening port 80...
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:15 2013 PDT] ACK backlog queue is overflown: 129 > 128

From the output, we can see a lot of ACK backlog queue overflows happening when the tool is running. This means the corresponding SYN packets were dropped in the kernel.

You can specify the `--limit` option to control the maximal number of issues reported:

    $ ./tcp-accept-queue --port=80 --limit=3
    WARNING: Tracing SYN & ACK backlog queue overflows on the listening port 80...
    [Tue May 14 12:29:25 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:25 2013 PDT] ACK backlog queue is overflown: 129 > 128
    [Tue May 14 12:29:25 2013 PDT] ACK backlog queue is overflown: 129 > 128

Or just hit Ctrl-C to end.

You can also specify the `--distr` option to make this tool just print out a histogram for the distribution
of the queue lengths:

    $ ./tcp-accept-queue --port=80 --distr
    WARNING: Tracing SYN & ACK backlog queue length distribution on the listening port 80...
    Hit Ctrl-C to end.
    SYN queue length limit: 512
    Accept queue length limit: 128
    ^C
    === SYN Queue ===
    min/avg/max: 0/2/8
    value |-------------------------------------------------- count
        0 |@@@@@@@@@@@@@@@@@@@@@@@@@@                         106
        1 |@@@@@@@@@@@@@@@                                     60
        2 |@@@@@@@@@@@@@@@@@@@@@                               84
        4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@       176
        8 |@@                                                   9
       16 |                                                     0
       32 |                                                     0

    === Accept Queue ===
    min/avg/max: 0/93/129
    value |-------------------------------------------------- count
        0 |@@@@                                                20
        1 |@@@                                                 16
        2 |                                                     3
        4 |@@                                                  11
        8 |@@@@                                                23
       16 |@@@                                                 16
       32 |@@@@@@                                              33
       64 |@@@@@@@@@@@@                                        63
      128 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 250
      256 |                                                     0
      512 |                                                     0

From the outputs, we can see that for 106 samples (i.e., 106 new connecting request), the SYN queue length remains 0; for samples, the SYN queue is of the length 1; and for 84 samples, the queue size is within the interval [2, 4); and so on. We can see most of the samples have the SYN queue size 0 ~ 8.

You need to hit Ctrl-C to make this tool print out the histgram when the `--distr` option is specified. Alternatively, you can specify the `--time` option to specify the exact number of seconds for real-time sampling:

    $ ./tcp-accept-queue --port=80 --distr --time=3
    WARNING: Tracing SYN & ACK backlog queue length distribution on the listening port 80...
    Sampling for 3 seconds.
    SYN queue length limit: 512
    Accept queue length limit: 128

    === SYN Queue ===
    min/avg/max: 6/7/10
    value |-------------------------------------------------- count
        1 |                                                    0
        2 |                                                    0
        4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@             76
        8 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@          82
       16 |                                                    0
       32 |                                                    0

    === Accept Queue ===
    min/avg/max: 128/128/129
    value |-------------------------------------------------- count
       32 |                                                     0
       64 |                                                     0
      128 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@            158
      256 |                                                     0
      512 |                                                     0

Even though the accept queue is not overflowing, long latency involved in accept queueing can also lead to client connecting timeout. The `--latency` option can be specified to analyze the accept queueing latency for a given listening port:

    $ ./tcp-accept-queue -port=80 --latency
    WARNING: Tracing accept queueing latency on the listening port 80...
    Hit Ctrl-C to end.
    ^C
    === Queueing Latency Distribution (microsends) ===
    min/avg/max: 28/3281400/3619393
      value |-------------------------------------------------- count
          4 |                                                     0
          8 |                                                     0
         16 |                                                     1
         32 |@                                                   12
         64 |                                                     6
        128 |                                                     2
        256 |                                                     0
        512 |                                                     0
            ~
       8192 |                                                     0
      16384 |                                                     0
      32768 |                                                     2
      65536 |                                                     0
     131072 |                                                     0
     262144 |                                                     0
     524288 |                                                     7
    1048576 |@@                                                  28
    2097152 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@     516
    4194304 |                                                     0
    8388608 |                                                     0

The `--time` option can also be specified to control the sampling time in seconds:

    $ ./tcp-accept-queue --port=80 --latency --time=5
    WARNING: Tracing accept queueing latency on the listening port 80...
    Sampling for 5 seconds.

    === Accept Queueing Latency Distribution (microsends) ===
    min/avg/max: 3604825/3618651/3639329
      value |-------------------------------------------------- count
     524288 |                                                    0
    1048576 |                                                    0
    2097152 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@              37
    4194304 |                                                    0
    8388608 |                                                    0

This tool requires a Linux kernel compiled by gcc 4.5+ (preferrably gcc 4.7+) because gcc versions older than 4.5 generated incomplete DWARF debug info for C inlined functions. It is also recommended to enable DWARF format version 3 or above when compiling the kernel (by passing the `-gdwarf-3` or `-gdwarf-4` option to the `gcc` command line).

[Back to TOC](#table-of-contents)

ngx-recv-queue
--------------

This tool has been renamed to [tcp-recv-queue](#tcp-recv-queue) because this tool is not specific to Nginx
in any way and it makes no sense to keep the `ngx-` prefix in its name.

[Back to TOC](#table-of-contents)

tcp-recv-queue
--------------

This tool can analyze the queueing latency involved in the TCP receive queue.

The queueing latency defined here is the delay between the following two events:

1. The first packet enteres the TCP receive queue since the last recvmsg() syscalls (and the like) initiated on the userland.
2. The next recvmsg() syscall (and the like) that consumes the TCP receive queue.

Large receive queueing lantencies often mean the user process is just too busy to consume the incoming requests, probably leading to timeout errors on the client side.

Zero-length data packets in the TCP receive queue (i.e., the FIN packets) are ignored by this tool.

You are only required to specify the destination port number for the receiving packets via the `--dport` option.

Here is an example for analyzing the MySQL server listening on the 3306 port:

    $ ./tcp-recv-queue --dport=3306
    WARNING: Tracing the TCP receive queues for packets to the port 3306...
    Hit Ctrl-C to end.
    ^C
    === Distribution of First-In-First-Out Latency (us) in TCP Receive Queue ===
    min/avg/max: 1/2/42
    value |-------------------------------------------------- count
        0 |                                                       0
        1 |@@@@@@@@@@@@@@                                     20461
        2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  69795
        4 |@@@@                                                6187
        8 |@@                                                  3421
       16 |                                                     178
       32 |                                                       8
       64 |                                                       0
      128 |                                                       0

We can see that most of the latency times fall into the interval `[2us, 4us)`. And the worst latency is 42us.

You can also specify the exact sampling time interval (in seconds) via the `--time` option. For example, to analyze the Nginx server listening on the port 8080 for 5 seconds:

    $ ./tcp-recv-queue --dport=1984 --time=5
    WARNING: Tracing the TCP receive queues for packets to the port 1984...
    Sampling for 5 seconds.

    === Distribution of First-In-First-Out Latency (us) in TCP Receive Queue ===
    min/avg/max: 1/1401/12761
    value |-------------------------------------------------- count
        0 |                                                       0
        1 |                                                       1
        2 |                                                       1
        4 |                                                       5
        8 |                                                     152
       16 |@@                                                  1610
       32 |                                                      35
       64 |@@@                                                 2485
      128 |@@@@@@                                              4056
      256 |@@@@@@@@@@@@                                        7853
      512 |@@@@@@@@@@@@@@@@@@@@@@@@                           15153
     1024 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  31424
     2048 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                   20454
     4096 |@                                                    862
     8192 |                                                      19
    16384 |                                                       0
    32768 |                                                       0

Successfully tested on Linux kernel 3.7 and should work for other versions of kernel as well.

This tool requires a Linux kernel compiled by gcc 4.5+ (preferrably gcc 4.7+) because gcc versions older than 4.5 generated incomplete DWARF debug info for C inlined functions. It is also recommended to enable DWARF format version 3 or above when compiling the kernel (by passing the `-gdwarf-3` or `-gdwarf-4` option to the `gcc` command line).

[Back to TOC](#table-of-contents)

ngx-lua-shdict
--------------

This tool analyzes shared memory dict and tracks dict operations in the specified running nginx process.

You can specify the `-f` option to fetch the data from the shared memory dict name by the specified dict and key.
Specify the `--raw` option when you need dump the raw value of the given key.

Specify the `--lua51` option when you're using the standard Lua 5.1 interpreter in your Nginx build, or `--luajit20` if LuaJIT 2.0 is used instead. Currently only LuaJIT is supported.

Here's a sample command to fetch the data from the shared memory dict:

    # assuming the nginx worker pid is 5050
    $ ./ngx-lua-shdict -p 5050 -f --dict dogs --key Jim --luajit20
    Tracing 5050 (/opt/nginx/sbin/nginx)...

    type: LUA_TBOOLEAN
    value: true
    expires: 1372719243270
    flags: 0xa

    6 microseconds elapsed in the probe handler.

Similarly, you can specify the `-w` option to track dict writes for the given key:

    $./ngx-lua-shdict -p 5050 -w --key Jim --luajit20
    Tracing 5050 (/opt/nginx/sbin/nginx)...

    Hit Ctrl-C to end

    set Jim exptime=4626322717216342016
    replace Jim exptime=4626322717216342016
    ^C

If you don't specify `-f` or `-w`, this tool will fetch the data by default.

[Back to TOC](#table-of-contents)

ngx-lua-conn-pools
----------------

Dumps connections pools status of [ngx_lua](http://wiki.nginx.org/HttpLuaModule), reports the number of both out-of-pool and in-pool connections, calculates connections reused times statistics of in-pool connections, and prints the capacity of each pool.

Specify the `--lua51` option when you're using the standard Lua 5.1 interpreter in your Nginx build, or `--luajit20` if LuaJIT 2.0 is used instead.

Here's a sample command:

    # assuming the nginx worker pid is 19773
    $ ./ngx-lua-conn-pools -p 19773 --luajit20
    Tracing 19773 (/opt/nginx/sbin/nginx)...

    pool "127.0.0.1:11213"
        out-of-pool reused connections: 2
        in-pool connections: 183
            reused times (max/min/avg): 9322/1042/3748
        pool capacity: 1024

    pool "127.0.0.1:11212"
        out-of-pool reused connections: 2
        in-pool connections: 182
            reused times (max/min/avg): 10283/414/3408
        pool capacity: 1024

    pool "127.0.0.1:11211"
        out-of-pool reused connections: 2
        in-pool connections: 183
            reused times (max/min/avg): 7109/651/3867
        pool capacity: 1024

    pool "127.0.0.1:11214"
        out-of-pool reused connections: 2
        in-pool connections: 183
            reused times (max/min/avg): 7051/810/3807
        pool capacity: 1024

    pool "127.0.0.1:11215"
        out-of-pool reused connections: 2
        in-pool connections: 183
            reused times (max/min/avg): 7275/1127/3839
        pool capacity: 1024

    For total 5 connection pool(s) found.
    324 microseconds elapsed in the probe handler.

You can specify the `--distr` option to get the distribution of numbers of resued times:

    $ ./ngx-lua-conn-pools -p 19773 --luajit20 --distr
    Tracing 15001 (/opt/nginx/sbin/nginx) for LuaJIT 2.0...

    pool "127.0.0.1:6379"
        out-of-pool reused connections: 1
        in-pool connections: 19
            reused times (max/avg/min): 2607/2503/2360
            reused times distribution:
    value |-------------------------------------------------- count
      512 |                                                    0
     1024 |                                                    0
     2048 |@@@@@@@@@@@@@@@@@@@                                19
     4096 |                                                    0
     8192 |                                                    0

        pool capacity: 1000

    For total 1 connection pool(s) found.
    218 microseconds elapsed in the probe handler.

[Back to TOC](#table-of-contents)

check-debug-info
----------------

This tool checks which executable files do not contain debug info in any running process that you specify.

Basically, just run it like this:

    ./check-debug-info -p <pid>

The executable file associated with the process and all the .so files already loaded by the process will be checked for dwarf info.

The process is not required to be nginx, but can be any user processes.

Here is a complete example:

    $ ./check-debug-info -p 26482
    File /usr/lib64/ld-2.15.so has no debug info embedded.
    File /usr/lib64/libc-2.15.so has no debug info embedded.
    File /usr/lib64/libdl-2.15.so has no debug info embedded.
    File /usr/lib64/libm-2.15.so has no debug info embedded.
    File /usr/lib64/libpthread-2.15.so has no debug info embedded.
    File /usr/lib64/libresolv-2.15.so has no debug info embedded.
    File /usr/lib64/librt-2.15.so has no debug info embedded.

For now, this tool does not support separate .debug files yet.

[Back to TOC](#table-of-contents)

ngx-phase-handlers
------------------
This tool dumps all the handlers registered by all the nginx modules for every nginx running phase in the order they actually run.

This is very useful in debugging Nginx configuration issues caused by misinterpreting the running order of the Nginx configuration directives.

Here is an example for an Nginx worker process with very few Nginx modules enabled:

    # assuming the nginx worker pid is 4876
    $ ./ngx-phase-handlers -p 4876
    Tracing 4876 (/opt/nginx/sbin/nginx)...
    pre-access phase
        ngx_http_limit_req_handler
        ngx_http_limit_conn_handler

    content phase
        ngx_http_index_handler
        ngx_http_autoindex_handler
        ngx_http_static_handler

    log phase
        ngx_http_log_handler

    22 microseconds elapsed in the probe handler.

Here is another example for an Nginx worker process with quite a few Nginx modules enabled:

    $ ./ngx-phase-handlers -p 24980
    Tracing 24980 (/opt/nginx/sbin/nginx)...
    post-read phase
        ngx_http_realip_handler

    server-rewrite phase
        ngx_coolkit_override_method_handler
        ngx_http_rewrite_handler

    rewrite phase
        ngx_coolkit_override_method_handler
        ngx_http_rewrite_handler

    pre-access phase
        ngx_http_realip_handler
        ngx_http_limit_req_handler
        ngx_http_limit_conn_handler

    access phase
        ngx_http_auth_request_handler
        ngx_http_access_handler

    content phase
        ngx_http_lua_content_handler (request content handler)
        ngx_http_index_handler
        ngx_http_autoindex_handler
        ngx_http_static_handler

    log phase
        ngx_http_log_handler

    44 microseconds elapsed in the probe handler.

[Back to TOC](#table-of-contents)

resolve-inlines
---------------

This tool calls the `addr2line` utility to resolve the inlined function frames generated by those `sample-*` tools like [sample-bt](#sample-bt).

It accepts two command-line arguments, the `.bt` file and the executable file.

For example,

```
resolve-inlines a.bt /path/to/nginx > new-a.bt
```

Right now, inlined functions in PIC (Position-Indenpendent Code) are not yet supported but are technically feasiable. (Patches welcome!)

[Back to TOC](#table-of-contents)

resolve-src-lines
-----------------

Similar to [resolve-inlines](#resolve-inlines) but expand function frames to both their function names and source line positions (i.e., source file names and source line numbers).

[Back to TOC](#table-of-contents)

Community
=========

[Back to TOC](#table-of-contents)

English Mailing List
--------------------

The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

[Back to TOC](#table-of-contents)

Chinese Mailing List
--------------------

The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

[Back to TOC](#table-of-contents)

Bugs and Patches
================

Please submit bug reports, wishlists, or patches by

1. creating a ticket on the [GitHub Issue Tracker](http://github.com/agentzh/nginx-systemtap-toolkit/issues),
1. or posting to the [OpenResty community](http://wiki.nginx.org/HttpLuaModule#Community).

[Back to TOC](#table-of-contents)

TODO
====

[Back to TOC](#table-of-contents)

Author
======

Yichun "agentzh" Zhang (章亦春), CloudFlare Inc.

[Back to TOC](#table-of-contents)

Copyright & License
===================

This module is licenced under the BSD license.

Copyright (C) 2012-2013 by Yichun Zhang (agentzh) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

* Redistributions of source code must retain the above copyright
notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright
notice, this list of conditions and the following disclaimer in the
documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED
TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[Back to TOC](#table-of-contents)

See Also
========
* You can find even more tools in the stap++ project: https://github.com/openresty/stapxx
* SystemTap Wiki Home: http://sourceware.org/systemtap/wiki
* Nginx home: http://nginx.org
* Perl Systemtap Toolkit: https://github.com/agentzh/perl-systemtap-toolkit
[Back to TOC](#table-of-contents)
