=================
liblogging-stdlog
=================

------------------------
standard logging library
------------------------

:Author: Rainer Gerhards <rgerhards@adiscon.com>
:Date: 2014-02-21
:Manual section: 3
:Manual group: standard logging library

SYNOPSIS
========

::
   
   #include <liblogging/stdlog.h>

   int stdlog_init(int options);
   void stdlog_deinit();

   stdlog_channel_t stdlog_open(const char *ident,
             const int options, const int facility,
             const char *channelspec);
   int stdlog_log(stdlog_channel_t channel, const int severity,
                  const char *fmt, ...);
   void stdlog_close(stdlog_channel_t channel);

DESCRIPTION
===========

**stdlog_init(options)** is used to initialize the logging system.
It must only be called once during the lifetime of a process. If no
special options are desired, stdlog_init() is optional. If it is not
called, the first call to any of the other calls will initiate it.
This feature is primarily for backward compatibility with how the
legacy **syslog(3)** API worked. It does not play well with multithreaded
applications. With them, call stdlog_init() explicitely from the
startup thread.

The following options can be given:

:STDLOG_SIGSAFE: request signal-safe mode. If and only if this is 
   specified library calls are signal-safe. With the current version
   of the library, there is no functional difference in signal-safe
   mode. Future versions may offer additional features which may not
   all work in signal-safe mode.

**stdlog_deinit(void)** is used to clean up resources including closing
file handles at library exit. No library calls are permitted after it
has been called. It's usage is optional if no cleanup is required (this
will leave resource leaks which will be reported by tools like
valgrind).


**stdlog_open()** is used to open a log channel which can be used in 
consecutive calls to *stdlog_log()*. The string given to *ident* is
used to identify the message source. It's handling is depending on the
output driver. For example, the file: and syslog: drivers prepend it 
to the message, while the journal: driver ignores it (as the journal
automatically identifies messages based on the application who submits
them. In general, you can think of it as being equivalent to the
*ident* specified in the traditional **openlog(3)** call. The value
given in *options* controls handling of the channel. It can be used to
override options set during **stdlog_init()**. Note that for signal-safeness
you need to specify **STDLOG_SIGSAFE**. The *facility* field contains a
facility similar to the tradtional syslog facility. Again, it is 
driver-dependent on how this field is actually used. The *channelspec*
filed is a **channel specification** string, which allows to control
the destination of this channel. To use the default output channel
specification, provide NULL to *channelspec*. Doing so is highly suggested
if there is no pressing need to do otherwise.

**stdlog_close()** is used to close the associated channel. The channel
specifier must not be used after *stdlog_close()* has been called. If done
so, unpredictable behaviour will happen, as the memory it points to has
been free'ed.

**stdlog_log()** is the equivalent to the **syslog(3)** call. It offers a
similar interface, but there are notable differences. The *channel* 
parameter is used to specify the log channel to use to. Use *NULL* to select
the default channel. This is sufficient for most applications. The *severity*
field contains a syslog-like severity. The *fmt* is a restricted set of
printf-like formats. This set has some restrictions in order to provide
a signal-safe implementation. The remaining parameters are values to be
used with the format string. The **stdlog_log()** supports log message sizes
of slighlty less than 4KiB. The exact size depends on the log driver
and parameters specified in *stdlog_open()**. The reason is that the
log drivers may need to add headers and trailers to the message
text, and this is done inside the same 4KiB buffer that is also used for
the actual message text. For example, the "syslog:" driver adds a traditional
syslog header, which among others contains the *ident* string provided
by **stdlog_open()**. If the complete log message does not fit into
the buffer, it is silently truncated.

FACILITIES
==========
The following facilities are supported. Please note that they are mimiced
after the traditional syslog facilities, but liblogging-stdlog uses
different numerical values. This is necessary to provide future enhancements.
Do **not** use the LOG_xxx #defines from syslog.h but the following
STDLOG_xxx defines:

::

   STDLOG_KERN     - kernel messages
   STDLOG_USER	   - random user-level messages
   STDLOG_MAIL	   - mail system
   STDLOG_DAEMON   - system daemons
   STDLOG_AUTH	   - security/authorization messages
   STDLOG_SYSLOG   - messages generated internally by syslogd
   STDLOG_LPR	   - line printer subsystem
   STDLOG_NEWS	   - network news subsystem
   STDLOG_UUCP	   - UUCP subsystem
   STDLOG_CRON     - clock daemon
   STDLOG_AUTHPRIV - security/authorization messages (private)
   STDLOG_FTP      - ftp daemon

   STDLOG_LOCAL0   - reserved for application use
   STDLOG_LOCAL1   - reserved for application use
   STDLOG_LOCAL2   - reserved for application use
   STDLOG_LOCAL3   - reserved for application use
   STDLOG_LOCAL4   - reserved for application use
   STDLOG_LOCAL5   - reserved for application use
   STDLOG_LOCAL6   - reserved for application use
   STDLOG_LOCAL7   - reserved for application use

Regular applications should use facilities in the **STDLOG_LOCALx**
range. Note that non-priviledged applications may not be able to use
all of the system-defined facilites. Note that it is also safe to
refer to applicaton specific facilities via

::

   STDLOG_LOCAL0 + offset

if offest is in the range of 0 to 7.

SEVERITY
========
The following severities are supported:

::

   STDLOG_EMERG	  - system is unusable
   STDLOG_ALERT   - action must be taken immediately
   STDLOG_CRIT    - critical conditions
   STDLOG_ERR     - error conditions
   STDLOG_WARNING - warning conditions
   STDLOG_NOTICE  - normal but significant condition
   STDLOG_INFO    - informational
   STDLOG_DEBUG   - debug-level messages

These reflect the traditional syslog severity mappings. Note that
different output drivers may have different needs and may map
severities into a smaller set.

THREAD- AND SIGNAL-SAFENESS
===========================

The following calls are **not** thread- or signal-safe:

* **stdlog_init()**
* **stdlog_deinit()**
* **stdlog_open()**
* **stdlog_close()**

For the **stdlog_log()** call, it depends:

* if either the library has been initialized with the option *STDLOG_SIGSAFE*
  or the channel has been opened with it, the call is both thread-safe and
  signal-safe.
* if the library has been initialized by **stdlog_init()** or the channel has
  been opened by **stdlog_open()**, the call is thread-safe but **not**
  signal-safe.
* if the library has not been initialized and the default (NULL) channel is
  used, the call is neither thread- nor signal-safe.

For multithreaded applications, it is **highly recommended** to initialize
the library via **stdlog_init()** on the main thread **before** any other
threads are started.

Thread- and signal-safeness, if given, does not require different
channels. It is perfectly fine to use the same channel in multiple threads.
Note however that interrupted system calls will not
be retried. An error will be returned instead. This may happen if a thread
is inside a **stdlog_log()** call while an async signal handler using that
same call is activated. Depending on timing, the first call may or may not
complete successfully. It is the caller's chore to check return status and
do retries if necessary.

CHANNEL SPECIFICATIONS
======================
The channel is described via a single-line string. Currently, the following
channels can be selected:

* "syslog:", which is the traditional syslog output to /dev/log
* "journal:", which emits messages via the native systemd journal API
* "file:<name>", which writes messages in a syslog-like format to
  the file specified as "name"

If no channel specification is given, the default is "syslog:". The
default channel can be set via the **LIBLOGGING_STDLOG_DFLT_LOG_CHANNEL**
environment variable.

Not all output channel drivers are available on all platforms. For example,
the "journal:" driver is not available on BSD. It is highly suggested that
application developers **never** hardcode any channel specifiers inside
their code but rather permit the administrator to configure these. If there
is no pressing need to select different channel drivers, it is suggested
to rely on the default channel spec, which always can be set by the
system administrator.

RETURN VALUE
============

When successful **stdlog_init()** and **stdlog_log()** return zero and 
something else otherwise. **stdlog_open()** returns a channel descriptor
on success and *NULL* otherwise. In case of failure *errno* is set
appropriately.

The **stdlog_deinit()** and **stdlog_close()** calls do not return
any status.


EXAMPLES
========

A typical single-threaded application just needs to know about
the **stdlog_log()** call:

::

    status = stdlog_log(NULL, STDLOG_ERR,
                        "New session %d of user %s",
                        sessid, username);

Being thread- and signal-safe requires a little bit more of setup:

::

    /* on main thread */
    status = stdlog_init(STDLOG_SIGSAFE);

    /* here comes the rest of the code, including worker
     * thread startup.
     */


    /* And do this in threads, signal handlers, etc: */
    status = stdlog_log(NULL, STDLOG_ERR,
                        "New session %d of user %s",
                        sessid, username);

SEE ALSO
========
**syslog(3)**

COPYRIGHT
=========

This page is part of the *liblogging* project, and is available under
the same BSD 2-clause license as the rest of the project.