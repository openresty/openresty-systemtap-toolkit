NAME
====

nginx-systemtap-toolkit - Real-time analyzing and diagnosing tools for Nginx based on [SystemTap](http://sourceware.org/systemtap/wiki)

Status
======

These scripts are considered production-ready.

Prerequisites
=============

You need at least systemtap 2.1+ and perl 5.6.1+ on your Linux system.

Also, you should ensure the (DWARF) debuginfo for your Nginx (and other dependencies) is already enabled (or installed separately)
if you did not compile your Nginx from source.

If you are on Linux kernels older than 3.5, then you may have to apply the [utrace patch](http://sourceware.org/systemtap/wiki/utrace) (if not yet) to your kernel to get
user-space tracing support for your systemtap installation. But if you are using Linux distributions in the RedHat family (like RHEL, CentOS, and Fedora), then your old kernel should already has the utrace patch applied.

The mainstream Linux kernel 3.5+ does have support for the uprobes API for userspace tracing.

Permissions
===========

Running systemtap-based tools requires special user permissions. To prevent running
these tools with the root account,
you can add your own (non-root) account name to the `stapusr` and `staprun` user groups.
But if the user account running the Nginx process is different from your current
user account, then you will still be required to run "sudo" or other means to run these tools
with root access.

Tools
=====

ngx-active-reqs
---------------

This tool lists detailed information about all the active requests that
are currently being processed by the specified Nginx worker or master process. When the mater process pid is specified, all its worker processes will be monitored.

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

    $ ./ngx-backtrace -p 5043  -p 5043 0x4121aa 0x44d7bd 0x44e425 0x44fcc1 0x47996d 0x43908a 0x4342c3 0x4343bd
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

This script requires Nginx instances that have applied the latest dtrace patch. See the [dtrace-nginx](https://github.com/agentzh/nginx-dtrace) project for more details.

The upcoming 1.2.3.3 release of the [ngx_openresty](http://openresty.org/) will include the right dtrace patch by default. And you just need to build it with the `--with-dtrace-probes` configure option.

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
debug symbols *and* avoid `-O2` or above
in your PCRE compilation. That is, you should build your Nginx and PCRE like this:

    ./configure --with-pcre=/path/to/my/pcre-8.31 \
        --with-pcre-jit \
        --prefix=/opt/nginx \
        -opt="-g -O1"
    make -j8
    make install

Otherwise SystemTap will get confused.

For dynamically-linked PCRE, you are still recommended to use `-O1` or below. And
you still need
to install the debug symbols for your PCRE (or the debuginfo RPM package for Yum-based systems).

ngx-sample-bt
-------------

This script can be used to sample backtraces in either user space or kernel space
or both for a specific Nginx worker process. It outputs the aggregated backgraces (by count). For example,
to sample a running Nginx worker process (whose pid is 8736) in user space only for total 5 seconds:

    $ ./ngx-sample-bt -p 8736 -t 5 -u > a.bt
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

    $ ./ngx-sample-bt -p 8736 -t 5 -k > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in kernel-space only...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_bf5516bdbf2beba886507025110994e_11738'
    WARNING: Time's up. Quitting now...(it may take a while)

Only the kernel-space code in the context of the specified nginx worker process
will be sampled.

A sample flame graph for kernel-space-only sample can be seen here:

http://agentzh.org/misc/nginx/kernel-flamegraph.svg

You can also sample in both the user space and kernel space by specifying the `-k` and `-u` options at the same time, as in

    $ ./ngx-sample-bt -p 8736 -t 5 -uk > a.bt
    WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in both user-space and kernel-space...
    WARNING: Missing unwind data for module, rerun with 'stap -d stap_90327f3a19b0e42dffdef38d53a5860_11799'
    WARNING: Time's up. Quitting now...(it may take a while)
    WARNING: Number of errors: 0, skipped probes: 38
    WARNING: There were 73 transport failures.

A sample flame graph for kenerl-and-user-space sampling can be seen here:

http://agentzh.org/misc/nginx/user-kernel-flamegraph.svg

In fact, this script is general enough and can be used to sample user processes other than Nginx.

ngx-sample-lua-bt
-----------------

Similar to the `ngx-sample-bt` script, but samples the Lua language level backtraces.

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

ngx-lua-bt
----------

This tool dump out the current Lua-land backtrace in the current running Nginx worker process.

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

ngx-pcre-stats
--------------

This tool can perform various statistical analysis of PCRE regex execution
performance in a running Nginx worker process.

This tool requires uretprobes support in the Linux kernel.

Also, you need to ensure that debug symbols are enabled in your
Nginx build, PCRE build, and LuaJIT build.

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

Community
=========

English Mailing List
--------------------

The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

Chinese Mailing List
--------------------

The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

Bugs and Patches
================

Please submit bug reports, wishlists, or patches by

1. creating a ticket on the [GitHub Issue Tracker](http://github.com/agentzh/nginx-systemtap-toolkit/issues),
1. or posting to the [OpenResty community](http://wiki.nginx.org/HttpLuaModule#Community).

TODO
====

Author
======

Yichun "agentzh" Zhang (章亦春), CloudFlare Inc.

Copyright & License
===================

This module is licenced under the BSD license.

Copyright (C) 2012 by Yichun Zhang (agentzh) <agentzh@gmail.com>, CloudFlare Inc.

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

See Also
========
* SystemTap Wiki Home: http://sourceware.org/systemtap/wiki
* Nginx home: http://nginx.org

