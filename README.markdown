NAME
====

nginx-systemtap-toolkit - Real-time analyzing and diagnosing tools for Nginx based on SystemTap

Prerequisites
=============

You need at least SystemTap 1.8+ and perl 5.8.1+ on your Linux system.

Also, you should install the debuginfo for your Nginx installation.

Tools
=====

ngx-shm
-------

    # find the nginx worker pid
    $ ps aux|grep nginx

    # you should ensure the worker is handling requests
    # or the timer_resoluation is set in your nginx.conf

    # assuming the nginx worker pid is 5382
    $ ngx-shm -p 5382

