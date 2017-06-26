:data-transition-duration: 200

.. title:: Systemd

----

whoami
======

.. code-block::

   # getent passwd ssm
   ssm:x:1000:1000:Stig Sandbeck Mathisen,,,:/home/ssm:/bin/zsh

* Lead Infrastructure Engineer @ Sopra Steria
* Debian Developer
* Red Hat Certified Architect

----

What is pid 1?
==============

----

PID 1 alternatives
==================

* System V init
* Daemontools
* Launchd
* Systemd
* SMF

----

init script
===========

.. code-block:: shell

   #!/bin/sh

   set -e

   case $1 in
     start)
       start_service
       ;;
     stop)
       stop_service
       ;;
   esac

----

systemd
=======

* Modern init system
* Service supervisor
* Uses lots of modern features
* Declarative syntax

----

systemd unit
============

.. code-block:: ini

   [Unit]
   Description=System Logging Service
   Requires=syslog.socket
   Documentation=man:rsyslogd(8)
   Documentation=http://www.rsyslog.com/doc/
   
   [Service]
   Type=notify
   ExecStart=/usr/sbin/rsyslogd -n
   StandardOutput=null
   Restart=on-failure
   
   [Install]
   WantedBy=multi-user.target
   Alias=syslog.service

----

Not entirely uncontroversial
============================

----

.. image:: images/bts-727708-done.png
   :height: 313px
   :width: 658px

----

.. image:: images/devuan.org.png
   :height: 271px
   :width: 847px

----

   
Some systemd features
=====================

A few of systemd features that helps you and your fellow sysadmins.

At 3am, I want to sleep. I do not want SMS with “Service X is down”,
and I do not want my systems to wake the on-call personnel, so they
can scratch their heads and call me about “Service X is down, and I
need help fixing it”.

There are a couple of things you can do to avoid this.

----

Automatic restarts
------------------

Sometimes processes die. Particularly at inconvenient times, it
seems. In many cases, the fix is to “restart it, and figure out the
cause later”. You can configure systemd to restart your service. If
the restart is successful, the service is not unavailable, and no SMS
is sent.

----

.. code-block:: ini

   [Service]
   Restart=always

The “Restart=” directive tells systemd to restart the service if the
process terminates. You can set it to “always”, or read the manual
page to see if the other values make sense for you.

Just ensure you follow up on unexpected service restarts. This is
logged in the journal, and you should add this to your monitoring.

----

Improved documentation
----------------------

Not all services are well known, or well documented. The on-call
personnel may not be the one responsible for the architecture or the
day-to-day operations for that server.

You don’t need to edit the original unit file, you can add a drop-in
file in /etc/systemd/system/<yourservice>.d/<something>.conf:

----

# create /etc/systemd/system/mystery.service.d/documentation.conf

.. code-block:: ini

   [Unit]
   Documentation=https://wiki.corp.example.org/SomeClient/CommonFailures \
     https://www.enterpricy.example.org/Documentation/ \
     man:mysteryd(8) \
     file:///opt/mystery/doc/index.html

The content of the “Documentation=” directive is visible when running
“systemctl status servicename”. This helps your on-call person, when
the alarm goes off, to figure out what is wrong, and how to fix
it. Add your own service documentation, and a link to the upstream
documentation.

----

The output will look like this:

::

  root@turbotape:~# systemctl status mystery.service
  ● mystery.service - MYSTERY Scheduler
     Loaded: loaded (/lib/systemd/system/mystery.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/mystery.service.d
             └─documentation.conf
     Active: active (running) since Mon 2016-11-28 06:25:01 CET; 6h ago
       Docs: man:mysteryd(8)
             https://wiki.corp.example.org/SomeClient/CommonFailures
             https://www.enterpricy.example.org/Documentation/
             man:mysteryd(8)
             file:///opt/mystery/doc/index.html
   Main PID: 10015 (mysteryd)
        CPU: 251ms
     CGroup: /system.slice/mystery.service
             ├─10015 /usr/sbin/mysteryd -l
             └─10218 /usr/lib/mystery/notifier/dbus dbus://
  
  Nov 28 06:25:01 turbotape systemd[1]: Started MYSTERY Scheduler.


----

Show connections for a service
------------------------------

Systemd tracks all processes per service by placing them in the same
cgroup. Using “ps”, “awk” and “lsof”, we can print network connections
for a single service, across multiple processes.

----

The oneliner

…ironically enough not on one line

.. code-block:: shell

   ps -e -o pid,cgroup \
     | awk '$2 ~ /dovecot.service/ {print "-p", $1}' \
     | xargs -r lsof -n -i -a

----

What does it do?

The example lists all processes started by “dovecot.service”.

* List all running processes, and print pid and cgroup on each line.

* For each line, check if the “cgroup” matches our regular expression,
  and print the pid. Actually, print a “-p”, and the pid, since this
  is used by lsof.

* Use “xargs” to take the “-p $pid” lines from STDIN, and add them to
  the “lsof” command line.

----

Example output

Here, we see that the “dovecot.service” unit has a number of listening
ports, and one established session.

::
   
  root@mail1:~# ps -e -o pid,cgroup \
  >       | awk '$2 ~ /dovecot.service/ {print "-p", $1}' \
  >       | xargs -r lsof -n -i -a
  COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
  dovecot 17335 root   31u  IPv4 11520166      0t0  TCP *:imap2 (LISTEN)
  dovecot 17335 root   32u  IPv6 11520167      0t0  TCP *:imap2 (LISTEN)
  dovecot 17335 root   33u  IPv4 11520168      0t0  TCP *:imaps (LISTEN)
  dovecot 17335 root   34u  IPv6 11520169      0t0  TCP *:imaps (LISTEN)
  imap-logi 17564 dovenull   18u  IPv6 25385800      0t0  TCP [2001:db8::de:caf:bad]:imaps->[2001:db8::c0:ff:ee]:55043 (ESTABLISHED)
  
