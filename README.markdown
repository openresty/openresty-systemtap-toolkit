NAME
====

nginx-systemtap-toolkit - Real-time analyzing and diagnosing tools for Nginx based on [SystemTap](http://sourceware.org/systemtap/wiki)

Status
======

These scripts are still highly experimental and are not yet considered to be production safe.

Tested with Nginx 1.2.x on Fedora 17.

Prerequisites
=============

You need at least SystemTap 1.8+ and perl 5.6.1+ on your Linux system.

Also, you should ensure the debuginfo for your Nginx is already installed
if you did not compile your Nginx from source.

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
are currently being processed by the specified Nginx (worker) process.

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

Yichun "agentzh" Zhang (章亦春)

Copyright & License
===================

This module is licenced under the BSD license.

Copyright (C) 2012 by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>.

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

