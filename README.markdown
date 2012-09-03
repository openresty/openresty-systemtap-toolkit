NAME
====

nginx-systemtap-toolkit - Real-time analyzing and diagnosing tools for Nginx based on SystemTap

Status
======

These scripts are still highly experimental and are not yet considered to be production safe.

Prerequisites
=============

You need at least SystemTap 1.8+ and perl 5.6.1+ on your Linux system.

Also, you should install the debuginfo for your Nginx installation.

Permissions
===========

Running systemtap-based tools requires special user permissions. To prevent running
these tools with the root user,
you can add your own (non-root) user name to the `stapusr` and `staprun` user groups.

Tools
=====

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

    # you should ensure the worker is handling requests
    # or the timer_resoluation is set in your nginx.conf

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
    large blocks (>= 4096): 6 blocks, 26352 bytes (used)
    small blocks (< 4096): 96416 bytes used, 1408 bytes unused
    total used: 122768 bytes

    12 microseconds elapsed in the probe handler.

The memory block size for the "large blocks" are approximated based on
the intermal implementation of Gnu libc `malloc` on Linux. If you have replaced the `malloc` with other allocator,
then this tool is very likely to quit with memory access errors
or to give meaningless number for the "large blocks" total size
(but should not affect the nginx process being analyzed).

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

