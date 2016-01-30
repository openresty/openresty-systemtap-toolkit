NAME
====

nginx-systemtap-toolkit - Real-time analyzing and diagnosing tools for Nginx based on [SystemTap](http://sourceware.org/systemtap/wiki)

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

Status
======

These scripts are considered production-ready.

Prerequisites
=============

You need at least systemtap 2.1+ and perl 5.6.1+ on your Linux system. For building latest systemtap from source, please refer to this document: http://openresty.org/#BuildSystemtap

Also, you should ensure the (DWARF) debuginfo for your Nginx (and other dependencies) is already enabled (or installed separately)
if you did not compile your Nginx from source.

Finally, you need to install the kernel debug symbols and kernel headers as well. Usually you just need the `kernel-devel` and `kernel-debuginfo` packages (matching your current `kernel` package) from your Linux distributions, respectively.

For old Linux systems
---------------------

If you are on Linux kernels older than 3.5, then you may have to apply the [utrace patch](http://sourceware.org/systemtap/wiki/utrace) (if not yet) to your kernel to get
user-space tracing support for your systemtap installation. But if you are using Linux distributions in the RedHat family (like RHEL, CentOS, and Fedora), then your old kernel should already has the utrace patch applied.

The mainstream Linux kernel 3.5+ does have support for the uprobes API for userspace tracing.

[Back to TOC](#table-of-contents)

Permissions
===========

Running systemtap-based tools requires special user permissions. To prevent running
these tools with the root account,
you can add your own (non-root) account name to the `stapusr` and `staprun` user groups.
But if the user account running the Nginx process is different from your current
user account, then you will still be required to run "sudo" or other means to run these tools
with root access.

[Back to TOC](#table-of-contents)

Tools
=====

[Back to TOC](#table-of-contents)

ngx-active-reqs
---------------

This tool lists detailed information about all the active requests that
are currently being processed by the specified Nginx worker or master process. When the master process pid is specified, all its worker processes will be monitored.

Here is an example:

    # assuming the nginx worker pid is 32027

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

The `time` field is the elapsed time (in seconds) since the current request started.
The `conn reqs` field lists the requests that have been processed on the current (keep-alive) downstream connection.
The `fd` field is the file descriptor ID for the current downstream connection.

The `-m` option will tell this tool to analyze the request memory pools for each active request:

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

For Nginx servers that are not busy enough, it is handy to specify the Nginx master process pid as the `-p` option value.
Another useful option is `-k`, which will keep probing when there's no active requests found in the current event cycle.

[Back to TOC](#table-of-contents)

ngx-req-distr
-------------

This tool analyzes the (downstream) request and connection distributions
among all the nginx worker processes for the specified nginx master process.

    # here the nginx master pid is stored in the pid file
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

This tool analyzes all the shared memory zones in the specified running nginx process.

    # you should ensure the worker is still handling requests
    # otherwise the timer_resoluation must be set in your nginx.conf

    # assuming the nginx worker pid is 15218

    $ cd /path/to/nginx-systemtap-toolkit/

    # list the zones
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

This tool computes the real-time memory usage of the nginx global "cycle pool"
in the specified nginx (worker) process.

The "cycle pool" is mainly for configuration related data block allocation and other long-lived
data blocks with a lifetime as long as the nginx server configuration (like the compiled PCRE data stored in the regex cache for the ngx_lua module).

    # you should ensure the worker is handling requests
    # or the timer_resoluation is set in your nginx.conf

    # assuming the nginx worker pid is 15004

    $ ./ngx-cycle-pool -p 15004
    Tracing 15004 (/usr/local/nginx/sbin/nginx)...

    pool chunk size: 16384
    small blocks (< 4096): 96416 bytes used, 1408 bytes unused
    large blocks (>= 4096): 6 blocks, 26352 bytes (used)
    total used: 122768 bytes

    12 microseconds elapsed in the probe handler.

The memory block size for the "large blocks" is approximated based on
the intermal implementation of glibc's `malloc` on Linux. If you have replaced the `malloc` with other allocator,
then this tool is very likely to quit with memory access errors
or to give meaningless numbers for the "large blocks" total size
(but even in such bad cases, SystemTap should not affect the nginx process being analyzed at all).

[Back to TOC](#table-of-contents)

ngx-leaked-pools
----------------

Tracks creations and destructions of Nginx memory pools and report the top 10 leaked pools'
backtraces.

The backtraces are in the raw form of hexidecimal addresses.
You can use the `ngx-backtrace` tool to print out the source
code file names, source line numbers, as well as function names.

    # assuming the nginx worker pid is 5043

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

This script requires Nginx instances that have applied the latest dtrace patch. See the [nginx-dtrace](https://github.com/openresty/nginx-dtrace) project for more details.

The bundle [OpenResty](http://openresty.org/) 1.2.3.3+ includes the right dtrace patch by default. And you just need to build it with the `--with-dtrace-probes` configure option.

[Back to TOC](#table-of-contents)

ngx-backtrace
-------------

Prints out a human readable form for the raw backtraces consisting of hexidecimal addresses generated by other tools like `ngx-leaked-pools`.

    # assuming the nginx worker process pid is 5043

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

Print out all the output body filters in the order that they actually run.

    # assuming the nginx worker process pid is 30132

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

Print out all the output header filters in the order that they actually run.

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

This script tracks the PCRE compiled regex execution (i.e., the `pcre_exec` calls)
in the specified Nginx worker process,
and checks whether the compiled regexes being executed is JIT'd or not.

    # assuming the Nginx worker process handling the traffic is 31360.

    $ ./ngx-pcrejit -p 31360
    Tracing 31360 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    ngx_http_lua_ngx_re_match: 1000 of 2000 are PCRE JIT'd.
    ngx_http_regex_exec: 0 of 1000 are PCRE JIT'd.

When statically linking PCRE with your Nginx, it is important to enable
debug symbols in your PCRE compilation.
That is, you should build your Nginx and PCRE like this:

    ./configure --with-pcre=/path/to/my/pcre-8.31 \
        --with-pcre-jit \
        --with-pcre-opt=-g \
        --prefix=/opt/nginx
    make -j8
    make install

For dynamically-linked PCRE, you are still need
to install the debug symbols for your PCRE (or the debuginfo RPM package for Yum-based systems).

[Back to TOC](#table-of-contents)

ngx-sample-bt
-------------

This tool has been renamed to [sample-bt](#sample-bt) because this tool is not specific to Nginx
in any way and it makes no sense to keep the `ngx-` prefix in its name.

[Back to TOC](#table-of-contents)

sample-bt
---------

This script can be used to sample backtraces in either user space or kernel space
or both for *any* user process that you specify (yes, not just Nginx!).
It outputs the aggregated backtraces (by count).

For example, to sample a running Nginx worker process (whose pid is 8736) in user space
only for total 5 seconds:

    $ ./sample-bt -p 8736 -t 5 -u > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in user-space only...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_df60590ce8827444bfebaf5ea938b5a_11577'
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 24

The resulting output file `a.bt` can then be used to generate a Flame Graph by using Brendan Gregg's [FlameGraph tools](https://github.com/brendangregg/FlameGraph):

    stackcollapse-stap.pl a.bt > a.cbt
    flamegraph.pl a.cbt > a.svg

where both the `stackcollapse-stap.pl` and `flamegraph.pl` are from the FlameGraph toolkit.
If everything goes right, you can now use your web browser to open the `a.svg` file.

A sample flame graph for user-space-only sampling can be seen here (please open the link with a modern web browser that supports SVG rendering):

http://agentzh.org/misc/nginx/user-flamegraph.svg

For more information on the Flame Graph thing, please check out Brendan Gregg's blog posts below:

* [Flame Graphs](http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/)
* [Linux Kernel Performance: Flame Graphs](http://dtrace.org/blogs/brendan/2012/03/17/linux-kernel-performance-flame-graphs/)

You can also sample the backtraces in the kernel-space by specifying the `-k` option, as in

    $ ./sample-bt -p 8736 -t 5 -k > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in kernel-space only...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_bf5516bdbf2beba886507025110994e_11738'
    WARNING: Time's up. Quitting now...(it may take a while)

Only the kernel-space code in the context of the specified nginx worker process
will be sampled.

A sample flame graph for kernel-space-only sample can be seen here:

http://agentzh.org/misc/nginx/kernel-flamegraph.svg

You can also sample in both the user space and kernel space by specifying the `-k` and `-u` options at the same time, as in

    $ ./sample-bt -p 8736 -t 5 -uk > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in both user-space and kernel-space...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_90327f3a19b0e42dffdef38d53a5860_11799'
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 38
    WARNING: There were 73 transport failures.

A sample flame graph for kenerl-and-user-space sampling can be seen here:

http://agentzh.org/misc/nginx/user-kernel-flamegraph.svg

In fact, this script is general enough and can be used to sample user processes other than Nginx.

The overhead exposed on the target process is usually small. For example, the throughput (req/sec) limit of an nginx worker process doing simplest "hello world" requests drops by only 11% (only when this tool is running), as measured by `ab -k -c2 -n100000` when using Linux kernel 3.6.10 and systemtap 2.5. The impact on full-fledged production processes is usually smaller than even that, for instance, only 6% drop in the throughput limit is observed in a production-level Lua CDN application.

[Back to TOC](#table-of-contents)

ngx-sample-lua-bt
-----------------

*WARNING* This tool can only work with interpreted Lua code and has various limitations. For
LuaJIT 2.1, it is recommended to use the new [ngx-lj-lua-stacks](https://github.com/openresty/stapxx#ngx-lj-lua-stacks)
tool for sampling both interpreted and/or compiled Lua code.

Similar to the [sample-bt](#sample-bt) script, but samples the Lua language level backtraces.

Specify the `--lua51` option when you're using the standard Lua 5.1 interpreter in your Nginx build, or `--luajit20` if LuaJIT 2.0 is used instead.

You need to enable or install the debug symbols for your Lua library, in addition to your Nginx executable.

Also, you should not omit frame pointers while building your Lua library.

If LuaJIT 2.0 is used, you need to build your LuaJIT 2.0 library like this:

    make CCDEBUG=-g

The Lua backtraces generated by this script use Lua source file name and source line number where the Lua function is defined. So to get more meaningful backtraces, you can call the `fix-lua-bt` script to process the output of this script.

Here is an example for standard Lua 5.1 interpreter embedded Nginx:

    # sample at 1K Hz for 5 seconds, assuming the Nginx worker
    #   or master process pid is 9766.
    $ ./ngx-sample-lua-bt -p 9766 --lua51 -t 5 > tmp.bt
    WARNING: Tracing 9766 (/opt/nginx/sbin/nginx) for standard Lua 5.1...
    WARNING: Time's up. Quitting now...(it may take a while)

    $ ./fix-lua-bt tmp.bt > a.bt

Or if LuaJIT 2.0 is used:

    # sample at 1K Hz for 5 seconds, assuming the Nginx worker
    #   or master process pid is 9768.
    $ ./ngx-sample-lua-bt -p 9768 --luajit20 -t 5 > tmp.bt
    WARNING: Tracing 9766 (/opt/nginx/sbin/nginx) for LuaJIT 2.0...
    WARNING: Time's up. Quitting now...(it may take a while)

    $ ./fix-lua-bt tmp.bt > a.bt

The resulting output file `a.bt` can then be used to generate a Flame Graph by using Brendan Gregg's [FlameGraph tools](https://github.com/brendangregg/FlameGraph):

    stackcollapse-stap.pl a.bt > a.cbt
    flamegraph.pl a.cbt > a.svg

where both the `stackcollapse-stap.pl` and `flamegraph.pl` are from the FlameGraph toolkit.
If everything goes right, you can now use your web browser to open the `a.svg` file.

A sample flame graph for user-space-only sampling can be seen here (please open the link with a modern web browser that supports SVG rendering):

http://agentzh.org/misc/flamegraph/lua51-resty-mysql.svg

For more information on the Flame Graph thing, please check out Brendan Gregg's blog posts below:

* [Flame Graphs](http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/)
* [Linux Kernel Performance: Flame Graphs](http://dtrace.org/blogs/brendan/2012/03/17/linux-kernel-performance-flame-graphs/)

If the pid of the Nginx master proces is specified as the `-t` option value,
then this tool will automatically probe all its worker processes at the same time.

[Back to TOC](#table-of-contents)

fix-lua-bt
----------

Fixes the raw Lua backtraces generated by the `ngx-sample-lua-bt` script and makes it more readable.


The original backtraces generated by `ngx-sample-lua-bt` looks like this:

    C:0x7fe9faf52dd0
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:65
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:176
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:418
    @/home/agentzh/git/lua-resty-mysql/lib/resty/mysql.lua:711

And after being processed by this script, we get

    C:0x7fe9faf52dd0
    resty.mysql:_get_byte3
    resty.mysql:_recv_packet
    resty.mysql:_recv_field_packet
    resty.mysql:read_result

Here's a sample command:

    ./fix-lua-bt tmp.bt > a.bt

where the input file `tmp.bt` is generated by `ngx-sample-lua-bt` earlier.

See also `ngx-sample-lua-bt`.

[Back to TOC](#table-of-contents)

ngx-lua-bt
----------

This tool dumps out the current Lua-land backtrace in the current running Nginx worker process.

This tool is very useful in locating the infinite Lua loop that keeps the Nginx worker
spinning with 100% CPU usage.

If LuaJIT 2.0 is used, specify the --luajit20 option, like this:

    $ ./ngx-lua-bt -p 7599 --luajit20
    WARNING: Tracing 7599 (/opt/nginx/sbin/nginx) for LuaJIT 2.0...
    C:lj_cf_string_find
    content_by_lua:2
    content_by_lua:1

If the standard Lua 5.1 interpreter is used instead, specify the --lua51 option:

    $ ./ngx-lua-bt -p 13611 --lua51
    WARNING: Tracing 13611 (/opt/nginx/sbin/nginx) for standard Lua 5.1...
    C:str_find
    content_by_lua:2
    [tail]
    content_by_lua:1

[Back to TOC](#table-of-contents)

ngx-sample-bt-off-cpu
---------------------

This tool has been renamed to [sample-bt-off-cpu](#sample-bt-off-cpu) because this tool is not specific to Nginx
in any way and it makes no sense to keep the `ngx-` prefix in its name.

[Back to TOC](#table-of-contents)

sample-bt-off-cpu
-----------------

Similar to [sample-bt](#sample-bt) but analyzes the off-CPU time for a particular user process (not only Nginx, but also any other applications).

Why does off-CPU time matter? Check out Brendan Gregg's excellent blog post "Off-CPU Performance Analysis" for details:

http://dtrace.org/blogs/brendan/2011/07/08/off-cpu-performance-analysis/

By default, this tool samples the userspace backtraces. And 1 (logical) sample of backtraces in the output corresponds to 1 microsecond of off-CPU time.

Here is an example to demonstrate this tool's usage:

    # assuming the nginx worker process to be analyzed is 10901.
    $ ./sample-bt-off-cpu -p 10901 -t 5 > a.bt
    WARNING: Tracing 10901 (/opt/nginx/sbin/nginx)...
    WARNING: _stp_read_address failed to access memory location
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 23

where the `-t 5` option makes the tool sample for 5 seconds.

The resulting `a.bt` file can be used to render Flame Graphs just as with [sample-bt](#sample-bt) and its other friends. And this type of flamegraphs can be called "off-CPU Flame Graphs" while the classic flamegraphs are essentially "on-CPU Flame Graphs".

Below is such a "off-CPU flamegraph" for a loaded Nginx worker process accessing MySQL with the lua-resty-mysql library:

http://agentzh.org/misc/flamegraph/off-cpu-lua-resty-mysql.svg

By default, off-CPU time intervals shorter than 4 us (microseconds) are discarded. You can control this threshold via the `--min` option, as in

    $ ./sample-bt-off-cpu -p 12345 --min 10 -t 10

where we ignore off-CPU time intervals shorter than 10 us and sample the user process with the pid 12345 for total 10 seconds.

The `-l` option can be control the upper limit of different backtraces to be outputed. By default, the hottest 1024 different backtraces are dumped.

The `--distr` option can be specified to print out a base-2 logarithmic histogram for all the off-CPU time intervals (larger than the threshold specified by the `--min` option). For example,

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

Here we can see that most of the samples (for total 259 samples) fall in the off-CPU time interval range `[4us, 8us)`. And the largest off-CPU time interval is 1739us, i.e., 1.739ms.

You can specify the `-k` option to sample the kernel space backtraces instead of sampling userland backtraces. If you want to sample both the userland and kernelspace, then you can specify both the `-k` and `-u` options.

[Back to TOC](#table-of-contents)

ngx-sample-bt-vfs
-----------------

This tool has been renamed to [sample-bt-vfs](#sample-bt-vfs) because this tool is not specific to Nginx
in any way and it makes no sense to keep the `ngx-` prefix in its name.

[Back to TOC](#table-of-contents)

sample-bt-vfs
-------------

Similar to [sample-bt](#sample-bt) but samples the userspace backtraces on the Virtual File System (VFS) level for rendering File I/O Flame Graphs, which can show exactly how file I/O data volumn or file I/O latency is distributed among different userspace code paths within any running user process.

By default, 1 sample of backtrace corresponds of 1 byte of data volumn (read or written). And by default, both `vfs_read` and `vfs_write` are tracked. For example,

    $ ./sample-bt-vfs -p 12345 -t 3 > a.bt
    WARNING: Tracing 20636 (/opt/nginx/sbin/nginx)...
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 2

We can then render a flamegraph for read/write VFS I/O like this:

    $ stackcollapse-stap.pl a.bt > a.cbt
    $ flamegraph.pl a.cbt > a.svg

where the tools `stackcollapse-stap.pl` and `flamegraph.pl` are from Brendan Gregg's FlameGraph toolkit:

https://github.com/brendangregg/FlameGraph

One sample "file I/O flamegraph" is here:

http://agentzh.org/misc/flamegraph/vfs-index-page-rw.svg

This graph was rendered when the Nginx worker process is loaded by requests to its default index page (i.e., `/index.html`). We can see both the file writes in the standard access logging module and the file reads in the standard "static" module. The total sample space in this graph, 1481361, means for total 1481361 bytes of data actually read or written on VFS.

You can also track file reading only by specifying the `-r` option:

    $ ./sample-bt-vfs -p 12345 -t 3 -r > a.bt

Here is an example of "file reading flamegraph" by sampling a Nginx loaded by requests accessing its default index page:

http://agentzh.org/misc/flamegraph/vfs-index-page-r.svg

We can see that only the standard nginx "static" module is the only thing shown in the graph.

Similarly, you can specify the `-w` option to track file writing only:


    $ ./sample-bt-vfs -p 12345 -t 3 -w > a.bt

Here is a sample "file writing flamegraph" for Nginx (with debugging logs enabled):

http://agentzh.org/misc/flamegraph/vfs-debug-log.svg

And below is another example for "file writing flamegraphs" for Nginx with debugging logs turned off:

http://agentzh.org/misc/flamegraph/vfs-access-log-only.svg

We can see that only access logging appears in the graph.

Do not confuse file I/O here with disk I/O because we are only probing on the (high) Virtual File System level. So the system page cache can save many disk reads here.

Generally, we are more interested in the latency (i.e, time) spent on the VFS reads and writes. You can specify the `--latency` option to track kernel call latency instead of the data volumn:

    $ ./sample-bt-vfs -p 12345 -t 3 --latency > a.bt

In this case, 1 sample corresponds to 1 microsends of file I/O time (or to be more correct, the `vfs_read` or `vfs_write` calls' time).

Here is an example for this:

http://agentzh.org/misc/flamegraph/vfs-latency-index-page-rw.svg

The total samples shown in the graph, 1918669, indicate for total 1,918,669 microsends (or 1.9 seconds) were spent on both file reading and writing during the 3 seconds sampling interval.

One can also combine either the `-r` or `-w` option with the `--latency` option to filter out file reads or file writes.

This tool can be used to inspect any user process (not only Nginx processes) with debug symbols enabled.

[Back to TOC](#table-of-contents)

ngx-accessed-files
------------------

This tool has been renamed to [accessed-files](#accessed-files) because this tool is not specific to Nginx
in any way and it makes no sense to keep the `ngx-` prefix in its name.

[Back to TOC](#table-of-contents)

accessed-files
--------------

Find out the names of the files most frequently read from or written to in any user process (yes, not only nginx!) specified by the `-p` option.

The `-r` option can be specified to analyze files that are read from. For example,

    $ ./accessed-files -p 8823 -r
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    === Top 10 file reads ===
    #1: 10 times, 720 bytes reads in file index.html.
    #2: 5 times, 75 bytes reads in file helloworld.html.
    #3: 2 times, 26 bytes reads in file a.html.

And the `-w` option can be used to analyze files that are written to instead:

    $ ./accessed-files -p 8823 -w
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    === Top 10 file writes ===
    #1: 17 times, 1600 bytes writes in file access.log.

And you can specify both the `-r` and `-w` options:

    $ ./accessed-files -p 8823 -w -r
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Hit Ctrl-C to end.
    ^C
    === Top 10 file reads/writes ===
    #1: 17 times, 1600 bytes reads/writes in file access.log.
    #2: 10 times, 720 bytes reads/writes in file index.html.
    #3: 5 times, 75 bytes reads/writes in file helloworld.html.
    #4: 2 times, 26 bytes reads/writes in file a.html.

By default, hitting Ctrl-C will end the sampling process. And the `-t` option can be specified to control the sampling period by exact number of seconds, as in

    $ ./accessed-files -p 8823 -r -t 5
    Tracing 8823 (/opt/nginx/sbin/nginx)...
    Please wait for 5 seconds.

    === Top 10 file reads ===
    #1: 10 times, 720 bytes reads in file index.html.
    #2: 5 times, 75 bytes reads in file helloworld.html.
    #3: 2 times, 26 bytes reads in file a.html.

By default, at most 10 different file names are printed out. You can control this upper limit by specifying the `-l` option. For instance,

    $ ./accessed-files -p 8823 -r -l 20

[Back to TOC](#table-of-contents)

ngx-pcre-stats
--------------

This tool can perform various statistical analysis of PCRE regex execution
performance in a running Nginx worker process.

This tool requires uretprobes support in the Linux kernel.

Also, you need to ensure that debug symbols are enabled in your
Nginx build, PCRE build, and LuaJIT build. For example, if you build PCRE from source with your Nginx or OpenResty by specifying the
`--with-pcre=PATH` option, then you should also specify the `--with-pcre-opt=-g` option at the same time.

Below is an example that analyzes the PCRE regex executation time distribution for a given Nginx worker process. Note that, the time is given in microseconds (`us`), i.e., 1e-6 seconds. The `--exec-time-dist` option is used here.

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

Also, you can specify the `--data-len-dist` option to analyze the distribution of the length of those subject string data being matched in individual runs.

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

The `--worst-time-top` option can be specified to analyze the worst execution time of the individual regex matches using the ngx_lua module's [ngx.re API](http://wiki.nginx.org/HttpLuaModule#ngx.re.match):

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

1. creating a ticket on the [GitHub Issue Tracker](http://github.com/openresty/nginx-systemtap-toolkit/issues),
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

Copyright (C) 2012-2016 by Yichun Zhang (agentzh) <agentzh@gmail.com>, CloudFlare Inc.

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
* Perl Systemtap Toolkit: https://github.com/openresty/perl-systemtap-toolkit
[Back to TOC](#table-of-contents)
