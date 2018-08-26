---
title: Haproxy Configuration
tags: []
date: 2015-07-27 15:52:00
---

----------------------
                                haproxy
                          configuration manual
                         ----------------------
                              version 1.6
                             willy tarreau
                              2015/07/22

this document covers the configuration language as implemented in the version
specified above. it does not provide any hint, example or advice. for such
documentation, please refer to the reference manual or the architecture manual.
the summary below is meant to help you search sections by name and navigate
through the document.

note to documentation contributors :
    this document is formatted with 80 columns per line, with even number of
    spaces for indentation and without tabs. please follow these rules strictly
    so that it remains easily printable everywhere. if a line needs to be
    printed verbatim and does not fit, please end each line with a backslash
    ('\') and continue on next line, indented by two characters. it is also
    sometimes useful to prefix all output lines (logs, console outs) with 3
    closing angle brackets ('>>>') in order to help get the difference between
    inputs and outputs when it can become ambiguous. if you add sections,
    please update the summary below for easier searching.

summary
-------

1\.    quick reminder about http
1.1\.      the http transaction model
1.2\.      http request
1.2.1\.        the request line
1.2.2\.        the request headers
1.3\.      http response
1.3.1\.        the response line
1.3.2\.        the response headers

2\.    configuring haproxy
2.1\.      configuration file format
2.2\.      quoting and escaping
2.3\.      environment variables
2.4\.      time format
2.5\.      examples

3\.    global parameters
3.1\.      process management and security
3.2\.      performance tuning
3.3\.      debugging
3.4\.      userlists
3.5\.      peers

4\.    proxies
4.1\.      proxy keywords matrix
4.2\.      alphabetically sorted keywords reference

5\.    bind and server options
5.1\.      bind options
5.2\.      server and default-server options
5.3\.      server dns resolution
5.3.1\.      global overview
5.3.2\.      the resolvers section

6\.    http header manipulation

7\.    using acls and fetching samples
7.1\.      acl basics
7.1.1\.      matching booleans
7.1.2\.      matching integers
7.1.3\.      matching strings
7.1.4\.      matching regular expressions (regexes)
7.1.5\.      matching arbitrary data blocks
7.1.6\.      matching ipv4 and ipv6 addresses
7.2\.      using acls to form conditions
7.3\.      fetching samples
7.3.1\.        converters
7.3.2\.        fetching samples from internal states
7.3.3\.        fetching samples at layer 4
7.3.4\.        fetching samples at layer 5
7.3.5\.        fetching samples from buffer contents (layer 6)
7.3.6\.        fetching http samples (layer 7)
7.4\.      pre-defined acls

8\.    logging
8.1\.      log levels
8.2\.      log formats
8.2.1\.        default log format
8.2.2\.        tcp log format
8.2.3\.        http log format
8.2.4\.        custom log format
8.2.5\.        error log format
8.3\.      advanced logging options
8.3.1\.        disabling logging of external tests
8.3.2\.        logging before waiting for the session to terminate
8.3.3\.        raising log level upon errors
8.3.4\.        disabling logging of successful connections
8.4\.      timing events
8.5\.      session state at disconnection
8.6\.      non-printable characters
8.7\.      capturing http cookies
8.8\.      capturing http headers
8.9\.      examples of logs

9\.    statistics and monitoring
9.1\.      csv format
9.2\.      unix socket commands

1\. quick reminder about http
----------------------------

when haproxy is running in http mode, both the request and the response are
fully analyzed and indexed, thus it becomes possible to build matching criteria
on almost anything found in the contents.

however, it is important to understand how http requests and responses are
formed, and how haproxy decomposes them. it will then become easier to write
correct rules and to debug existing configurations.

1.1\. the http transaction model
-------------------------------

the http protocol is transaction-driven. this means that each request will lead
to one and only one response. traditionally, a tcp connection is established
from the client to the server, a request is sent by the client on the
connection, the server responds and the connection is closed. a new request
will involve a new connection :

  [con1] [req1] ... [resp1] [clo1] [con2] [req2] ... [resp2] [clo2] ...

in this mode, called the "http close" mode, there are as many connection
establishments as there are http transactions. since the connection is closed
by the server after the response, the client does not need to know the content
length.

due to the transactional nature of the protocol, it was possible to improve it
to avoid closing a connection between two subsequent transactions. in this mode
however, it is mandatory that the server indicates the content length for each
response so that the client does not wait indefinitely. for this, a special
header is used: "content-length". this mode is called the "keep-alive" mode :

  [con] [req1] ... [resp1] [req2] ... [resp2] [clo] ...

its advantages are a reduced latency between transactions, and less processing
power required on the server side. it is generally better than the close mode,
but not always because the clients often limit their concurrent connections to
a smaller value.

a last improvement in the communications is the pipelining mode. it still uses
keep-alive, but the client does not wait for the first response to send the
second request. this is useful for fetching large number of images composing a
page :

  [con] [req1] [req2] ... [resp1] [resp2] [clo] ...

this can obviously have a tremendous benefit on performance because the network
latency is eliminated between subsequent requests. many http agents do not
correctly support pipelining since there is no way to associate a response with
the corresponding request in http. for this reason, it is mandatory for the
server to reply in the exact same order as the requests were received.

by default haproxy operates in keep-alive mode with regards to persistent
connections: for each connection it processes each request and response, and
leaves the connection idle on both sides between the end of a response and the
start of a new request.

haproxy supports 5 connection modes :
  - keep alive    : all requests and responses are processed (default)
  - tunnel        : only the first request and response are processed,
                    everything else is forwarded with no analysis.
  - passive close : tunnel with "connection: close" added in both directions.
  - server close  : the server-facing connection is closed after the response.
  - forced close  : the connection is actively closed after end of response.

1.2\. http request
-----------------

first, let's consider this http request :

  line     contents
  number
     1     get /serv/login.php?lang=en&profile=2 http/1.1
     2     host: www.mydomain.com
     3     user-agent: my small browser
     4     accept: image/jpeg, image/gif
     5     accept: image/png

1.2.1\. the request line
-----------------------

line 1 is the "request line". it is always composed of 3 fields :

  - a method      : get
  - a uri         : /serv/login.php?lang=en&profile=2
  - a version tag : http/1.1

all of them are delimited by what the standard calls lws (linear white spaces),
which are commonly spaces, but can also be tabs or line feeds/carriage returns
followed by spaces/tabs. the method itself cannot contain any colon (':') and
is limited to alphabetic letters. all those various combinations make it
desirable that haproxy performs the splitting itself rather than leaving it to
the user to write a complex or inaccurate regular expression.

the uri itself can have several forms :

  - a "relative uri" :

      /serv/login.php?lang=en&profile=2

    it is a complete url without the host part. this is generally what is
    received by servers, reverse proxies and transparent proxies.

  - an "absolute uri", also called a "url" :

      http://192.168.0.12:8080/serv/login.php?lang=en&profile=2

    it is composed of a "scheme" (the protocol name followed by '://'), a host
    name or address, optionally a colon (':') followed by a port number, then
    a relative uri beginning at the first slash ('/') after the address part.
    this is generally what proxies receive, but a server supporting http/1.1
    must accept this form too.

  - a star ('*') : this form is only accepted in association with the options
    method and is not relayable. it is used to inquiry a next hop's
    capabilities.

  - an address:port combination : 192.168.0.12:80
    this is used with the connect method, which is used to establish tcp
    tunnels through http proxies, generally for https, but sometimes for
    other protocols too.

in a relative uri, two sub-parts are identified. the part before the question
mark is called the "path". it is typically the relative path to static objects
on the server. the part after the question mark is called the "query string".
it is mostly used with get requests sent to dynamic scripts and is very
specific to the language, framework or application in use.

1.2.2\. the request headers
--------------------------

the headers start at the second line. they are composed of a name at the
beginning of the line, immediately followed by a colon (':'). traditionally,
an lws is added after the colon but that's not required. then come the values.
multiple identical headers may be folded into one single line, delimiting the
values with commas, provided that their order is respected. this is commonly
encountered in the "cookie:" field. a header may span over multiple lines if
the subsequent lines begin with an lws. in the example in 1.2, lines 4 and 5
define a total of 3 values for the "accept:" header.

contrary to a common mis-conception, header names are not case-sensitive, and
their values are not either if they refer to other header names (such as the
"connection:" header).

the end of the headers is indicated by the first empty line. people often say
that it's a double line feed, which is not exact, even if a double line feed
is one valid form of empty line.

fortunately, haproxy takes care of all these complex combinations when indexing
headers, checking values and counting them, so there is no reason to worry
about the way they could be written, but it is important not to accuse an
application of being buggy if it does unusual, valid things.

important note:
   as suggested by rfc2616, haproxy normalizes headers by replacing line breaks
   in the middle of headers by lws in order to join multi-line headers. this
   is necessary for proper analysis and helps less capable http parsers to work
   correctly and not to be fooled by such complex constructs.

1.3\. http response
------------------

an http response looks very much like an http request. both are called http
messages. let's consider this http response :

  line     contents
  number
     1     http/1.1 200 ok
     2     content-length: 350
     3     content-type: text/html

as a special case, http supports so called "informational responses" as status
codes 1xx. these messages are special in that they don't convey any part of the
response, they're just used as sort of a signaling message to ask a client to
continue to post its request for instance. in the case of a status 100 response
the requested information will be carried by the next non-100 response message
following the informational one. this implies that multiple responses may be
sent to a single request, and that this only works when keep-alive is enabled
(1xx messages are http/1.1 only). haproxy handles these messages and is able to
correctly forward and skip them, and only process the next non-100 response. as
such, these messages are neither logged nor transformed, unless explicitly
state otherwise. status 101 messages indicate that the protocol is changing
over the same connection and that haproxy must switch to tunnel mode, just as
if a connect had occurred. then the upgrade header would contain additional
information about the type of protocol the connection is switching to.

1.3.1\. the response line
------------------------

line 1 is the "response line". it is always composed of 3 fields :

  - a version tag : http/1.1
  - a status code : 200
  - a reason      : ok

the status code is always 3-digit. the first digit indicates a general status :
 - 1xx = informational message to be skipped (eg: 100, 101)
 - 2xx = ok, content is following   (eg: 200, 206)
 - 3xx = ok, no content following   (eg: 302, 304)
 - 4xx = error caused by the client (eg: 401, 403, 404)
 - 5xx = error caused by the server (eg: 500, 502, 503)

please refer to rfc2616 for the detailed meaning of all such codes. the
"reason" field is just a hint, but is not parsed by clients. anything can be
found there, but it's a common practice to respect the well-established
messages. it can be composed of one or multiple words, such as "ok", "found",
or "authentication required".

haproxy may emit the following status codes by itself :

  code  when / reason
   200  access to stats page, and when replying to monitoring requests
   301  when performing a redirection, depending on the configured code
   302  when performing a redirection, depending on the configured code
   303  when performing a redirection, depending on the configured code
   307  when performing a redirection, depending on the configured code
   308  when performing a redirection, depending on the configured code
   400  for an invalid or too large request
   401  when an authentication is required to perform the action (when
        accessing the stats page)
   403  when a request is forbidden by a "block" acl or "reqdeny" filter
   408  when the request timeout strikes before the request is complete
   500  when haproxy encounters an unrecoverable internal error, such as a
        memory allocation failure, which should never happen
   502  when the server returns an empty, invalid or incomplete response, or
        when an "rspdeny" filter blocks the response.
   503  when no server was available to handle the request, or in response to
        monitoring requests which match the "monitor fail" condition
   504  when the response timeout strikes before the server responds

the error 4xx and 5xx codes above may be customized (see "errorloc" in section
4.2).

1.3.2\. the response headers
---------------------------

response headers work exactly like request headers, and as such, haproxy uses
the same parsing function for both. please refer to paragraph 1.2.2 for more
details.

2\. configuring haproxy
----------------------

2.1\. configuration file format
------------------------------

haproxy's configuration process involves 3 major sources of parameters :

  - the arguments from the command-line, which always take precedence
  - the "global" section, which sets process-wide parameters
  - the proxies sections which can take form of "defaults", "listen",
    "frontend" and "backend".

the configuration file syntax consists in lines beginning with a keyword
referenced in this manual, optionally followed by one or several parameters
delimited by spaces.

2.2\. quoting and escaping
-------------------------

haproxy's configuration introduces a quoting and escaping system similar to
many programming languages. the configuration file supports 3 types: escaping
with a backslash, weak quoting with double quotes, and strong quoting with
single quotes.

if spaces have to be entered in strings, then they must be escaped by preceding
them by a backslash ('\') or by quoting them. backslashes also have to be
escaped by doubling or strong quoting them.

escaping is achieved by preceding a special character by a backslash ('\'):

  \    to mark a space and differentiate it from a delimiter
  \#   to mark a hash and differentiate it from a comment
  \\   to use a backslash
  \'   to use a single quote and differentiate it from strong quoting
  \"   to use a double quote and differentiate it from weak quoting

weak quoting is achieved by using double quotes (""). weak quoting prevents
the interpretation of:

       space as a parameter separator
  '    single quote as a strong quoting delimiter
  #    hash as a comment start

weak quoting permits the interpretation of variables, if you want to use a non
-interpreted dollar within a double quoted string, you should escape it with a
backslash ("\$"), it does not work outside weak quoting.

interpretation of escaping and special characters are not prevented by weak
quoting.

strong quoting is achieved by using single quotes (''). inside single quotes,
nothing is interpreted, it's the efficient way to quote regexes.

quoted and escaped strings are replaced in memory by their interpreted
equivalent, it allows you to perform concatenation.

  example:
      # those are equivalents:
      log-format %{+q}o\ %t\ %s\ %{-q}r
      log-format "%{+q}o %t %s %{-q}r"
      log-format '%{+q}o %t %s %{-q}r'
      log-format "%{+q}o %t"' %s %{-q}r'
      log-format "%{+q}o %t"' %s'\ %{-q}r

      # those are equivalents:
      reqrep "^([^\ :]*)\ /static/(.*)"     \1\ /\2
      reqrep "^([^ :]*)\ /static/(.*)"     '\1 /\2'
      reqrep "^([^ :]*)\ /static/(.*)"     "\1 /\2"
      reqrep "^([^ :]*)\ /static/(.*)"     "\1\ /\2"

2.3\. environment variables
--------------------------

haproxy's configuration supports environment variables. those variables are
interpreted only within double quotes. variables are expanded during the
configuration parsing. variable names must be preceded by a dollar ("$") and
optionally enclosed with braces ("{}") similarly to what is done in bourne
shell. variable names can contain alphanumerical characters or the character
underscore ("_") but should not start with a digit.

  example:

      bind "fd@${fd_app1}"

      log "${local_syslog}:514" local0 notice   # send to local server

      user "$haproxy_user"

2.4\. time format
----------------

some parameters involve values representing time, such as timeouts. these
values are generally expressed in milliseconds (unless explicitly stated
otherwise) but may be expressed in any other unit by suffixing the unit to the
numeric value. it is important to consider this because it will not be repeated
for every keyword. supported units are :

  - us : microseconds. 1 microsecond = 1/1000000 second
  - ms : milliseconds. 1 millisecond = 1/1000 second. this is the default.
  - s  : seconds. 1s = 1000ms
  - m  : minutes. 1m = 60s = 60000ms
  - h  : hours.   1h = 60m = 3600s = 3600000ms
  - d  : days.    1d = 24h = 1440m = 86400s = 86400000ms

2.4\. examples
-------------

    # simple configuration for an http proxy listening on port 80 on all
    # interfaces and forwarding requests to a single backend "servers" with a
    # single server "server1" listening on 127.0.0.1:8000
    global
        daemon
        maxconn 256

    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms

    frontend http-in
        bind *:80
        default_backend servers

    backend servers
        server server1 127.0.0.1:8000 maxconn 32

    # the same configuration defined with a single listen block. shorter but
    # less expressive, especially in http mode.
    global
        daemon
        maxconn 256

    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms

    listen http-in
        bind *:80
        server server1 127.0.0.1:8000 maxconn 32

assuming haproxy is in $path, test these configurations in a shell with:

    $ sudo haproxy -f configuration.conf -c

3\. global parameters
--------------------

parameters in the "global" section are process-wide and often os-specific. they
are generally set once for all and do not need being changed once correct. some
of them have command-line equivalents.

the following keywords are supported in the "global" section :

 * process management and security
   - ca-base
   - chroot
   - crt-base
   - daemon
   - external-check
   - gid
   - group
   - log
   - log-send-hostname
   - nbproc
   - pidfile
   - uid
   - ulimit-n
   - user
   - stats
   - ssl-server-verify
   - node
   - description
   - unix-bind
   - 51degrees-data-file
   - 51degrees-property-name-list
   - 51degrees-property-separator
   - 51degrees-cache-size

 * performance tuning
   - max-spread-checks
   - maxconn
   - maxconnrate
   - maxcomprate
   - maxcompcpuusage
   - maxpipes
   - maxsessrate
   - maxsslconn
   - maxsslrate
   - noepoll
   - nokqueue
   - nopoll
   - nosplice
   - nogetaddrinfo
   - spread-checks
   - tune.bufsize
   - tune.chksize
   - tune.comp.maxlevel
   - tune.http.cookielen
   - tune.http.maxhdr
   - tune.idletimer
   - tune.lua.forced-yield
   - tune.lua.maxmem
   - tune.lua.session-timeout
   - tune.lua.task-timeout
   - tune.maxaccept
   - tune.maxpollevents
   - tune.maxrewrite
   - tune.pattern.cache-size
   - tune.pipesize
   - tune.rcvbuf.client
   - tune.rcvbuf.server
   - tune.sndbuf.client
   - tune.sndbuf.server
   - tune.ssl.cachesize
   - tune.ssl.lifetime
   - tune.ssl.force-private-cache
   - tune.ssl.maxrecord
   - tune.ssl.default-dh-param
   - tune.ssl.ssl-ctx-cache-size
   - tune.vars.global-max-size
   - tune.vars.reqres-max-size
   - tune.vars.sess-max-size
   - tune.vars.txn-max-size
   - tune.zlib.memlevel
   - tune.zlib.windowsize

 * debugging
   - debug
   - quiet

3.1\. process management and security
------------------------------------

ca-base <dir>
  assigns a default directory to fetch ssl ca certificates and crls from when a
  relative path is used with "ca-file" or "crl-file" directives. absolute
  locations specified in "ca-file" and "crl-file" prevail and ignore "ca-base".

chroot <jail dir>
  changes current directory to <jail dir> and performs a chroot() there before
  dropping privileges. this increases the security level in case an unknown
  vulnerability would be exploited, since it would make it very hard for the
  attacker to exploit the system. this only works when the process is started
  with superuser privileges. it is important to ensure that <jail_dir> is both
  empty and unwritable to anyone.

cpu-map <"all"|"odd"|"even"|process_num> <cpu-set>...
  on linux 2.6 and above, it is possible to bind a process to a specific cpu
  set. this means that the process will never run on other cpus. the "cpu-map"
  directive specifies cpu sets for process sets. the first argument is the
  process number to bind. this process must have a number between 1 and 32 or
  64, depending on the machine's word size, and any process ids above nbproc
  are ignored. it is possible to specify all processes at once using "all",
  only odd numbers using "odd" or even numbers using "even", just like with the
  "bind-process" directive. the second and forthcoming arguments are cpu sets.
  each cpu set is either a unique number between 0 and 31 or 63 or a range with
  two such numbers delimited by a dash ('-'). multiple cpu numbers or ranges
  may be specified, and the processes will be allowed to bind to all of them.
  obviously, multiple "cpu-map" directives may be specified. each "cpu-map"
  directive will replace the previous ones when they overlap.

crt-base <dir>
  assigns a default directory to fetch ssl certificates from when a relative
  path is used with "crtfile" directives. absolute locations specified after
  "crtfile" prevail and ignore "crt-base".

daemon
  makes the process fork into background. this is the recommended mode of
  operation. it is equivalent to the command line "-d" argument. it can be
  disabled by the command line "-db" argument.

deviceatlas-json-file <path>
  sets the path of the deviceatlas json data file to be loaded by the api.
  the path must be a valid json data file and accessible by haproxy process.

deviceatlas-log-level <value>
  sets the level of informations returned by the api. this directive is
  optional and set to 0 by default if not set.

deviceatlas-separator <char>
  sets the character separator for the api properties results. this directive
  is optional and set to | by default if not set.

external-check
  allows the use of an external agent to perform health checks.
  this is disabled by default as a security precaution.
  see "option external-check".

gid <number>
  changes the process' group id to <number>. it is recommended that the group
  id is dedicated to haproxy or to a small set of similar daemons. haproxy must
  be started with a user belonging to this group, or with superuser privileges.
  note that if haproxy is started from a user having supplementary groups, it
  will only be able to drop these groups if started with superuser privileges.
  see also "group" and "uid".

group <group name>
  similar to "gid" but uses the gid of group name <group name> from /etc/group.
  see also "gid" and "user".

log <address> [len <length>] <facility> [max level [min level]]
  adds a global syslog server. up to two global servers can be defined. they
  will receive logs for startups and exits, as well as all logs from proxies
  configured with "log global".

  <address> can be one of:

        - an ipv4 address optionally followed by a colon and a udp port. if
          no port is specified, 514 is used by default (the standard syslog
          port).

        - an ipv6 address followed by a colon and optionally a udp port. if
          no port is specified, 514 is used by default (the standard syslog
          port).

        - a filesystem path to a unix domain socket, keeping in mind
          considerations for chroot (be sure the path is accessible inside
          the chroot) and uid/gid (be sure the path is appropriately
          writeable).

        you may want to reference some environment variables in the address
        parameter, see section 2.3 about environment variables.

  <length> is an optional maximum line length. log lines larger than this value
           will be truncated before being sent. the reason is that syslog
           servers act differently on log line length. all servers support the
           default value of 1024, but some servers simply drop larger lines
           while others do log them. if a server supports long lines, it may
           make sense to set this value here in order to avoid truncating long
           lines. similarly, if a server drops long lines, it is preferable to
           truncate them before sending them. accepted values are 80 to 65535
           inclusive. the default value of 1024 is generally fine for all
           standard usages. some specific cases of long captures or
           json-formated logs may require larger values.

  <facility> must be one of the 24 standard syslog facilities :

          kern   user   mail   daemon auth   syslog lpr    news
          uucp   cron   auth2  ftp    ntp    audit  alert  cron2
          local0 local1 local2 local3 local4 local5 local6 local7

  an optional level can be specified to filter outgoing messages. by default,
  all messages are sent. if a maximum level is specified, only messages with a
  severity at least as important as this level will be sent. an optional minimum
  level can be specified. if it is set, logs emitted with a more severe level
  than this one will be capped to this level. this is used to avoid sending
  "emerg" messages on all terminals on some default syslog configurations.
  eight levels are known :

          emerg  alert  crit   err    warning notice info  debug

log-send-hostname [<string>]
  sets the hostname field in the syslog header. if optional "string" parameter
  is set the header is set to the string contents, otherwise uses the hostname
  of the system. generally used if one is not relaying logs through an
  intermediate syslog server or for simply customizing the hostname printed in
  the logs.

log-tag <string>
  sets the tag field in the syslog header to this string. it defaults to the
  program name as launched from the command line, which usually is "haproxy".
  sometimes it can be useful to differentiate between multiple processes
  running on the same host. see also the per-proxy "log-tag" directive.

lua-load <file>
  this global directive loads and executes a lua file. this directive can be
  used multiple times.

nbproc <number>
  creates <number> processes when going daemon. this requires the "daemon"
  mode. by default, only one process is created, which is the recommended mode
  of operation. for systems limited to small sets of file descriptors per
  process, it may be needed to fork multiple daemons. using multiple processes
  is harder to debug and is really discouraged. see also "daemon".

pidfile <pidfile>
  writes pids of all daemons into file <pidfile>. this option is equivalent to
  the "-p" command line argument. the file must be accessible to the user
  starting the process. see also "daemon".

stats bind-process [ all | odd | even | <number 1-64>[-<number 1-64>] ] ...
  limits the stats socket to a certain set of processes numbers. by default the
  stats socket is bound to all processes, causing a warning to be emitted when
  nbproc is greater than 1 because there is no way to select the target process
  when connecting. however, by using this setting, it becomes possible to pin
  the stats socket to a specific set of processes, typically the first one. the
  warning will automatically be disabled when this setting is used, whatever
  the number of processes used. the maximum process id depends on the machine's
  word size (32 or 64). a better option consists in using the "process" setting
  of the "stats socket" line to force the process on each line.

ssl-default-bind-ciphers <ciphers>
  this setting is only available when support for openssl was built in. it sets
  the default string describing the list of cipher algorithms ("cipher suite")
  that are negotiated during the ssl/tls handshake for all "bind" lines which
  do not explicitly define theirs. the format of the string is defined in
  "man 1 ciphers" from openssl man pages, and can be for instance a string such
  as "aes:all:!anull:!enull:+rc4:@strength" (without quotes). please check the
  "bind" keyword for more information.

ssl-default-bind-options [<option>]...
  this setting is only available when support for openssl was built in. it sets
  default ssl-options to force on all "bind" lines. please check the "bind"
  keyword to see available options.

  example:
        global
           ssl-default-bind-options no-sslv3 no-tls-tickets

ssl-default-server-ciphers <ciphers>
  this setting is only available when support for openssl was built in. it
  sets the default string describing the list of cipher algorithms that are
  negotiated during the ssl/tls handshake with the server, for all "server"
  lines which do not explicitly define theirs. the format of the string is
  defined in "man 1 ciphers". please check the "server" keyword for more
  information.

ssl-default-server-options [<option>]...
  this setting is only available when support for openssl was built in. it sets
  default ssl-options to force on all "server" lines. please check the "server"
  keyword to see available options.

ssl-dh-param-file <file>
  this setting is only available when support for openssl was built in. it sets
  the default dh parameters that are used during the ssl/tls handshake when
  ephemeral diffie-hellman (dhe) key exchange is used, for all "bind" lines
  which do not explicitely define theirs. it will be overridden by custom dh
  parameters found in a bind certificate file if any. if custom dh parameters
  are not specified either by using ssl-dh-param-file or by setting them directly
  in the certificate file, pre-generated dh parameters of the size specified
  by tune.ssl.default-dh-param will be used. custom parameters are known to be
  more secure and therefore their use is recommended.
  custom dh parameters may be generated by using the openssl command
  "openssl dhparam <size>", where size should be at least 2048, as 1024-bit dh
  parameters should not be considered secure anymore.

ssl-server-verify [none|required]
  the default behavior for ssl verify on servers side. if specified to 'none',
  servers certificates are not verified. the default is 'required' except if
  forced using cmdline option '-dv'.

stats socket [<address:port>|<path>] [param*]
  binds a unix socket to <path> or a tcpv4/v6 address to <address:port>.
  connections to this socket will return various statistics outputs and even
  allow some commands to be issued to change some runtime settings. please
  consult section 9.2 "unix socket commands" for more details.

  all parameters supported by "bind" lines are supported, for instance to
  restrict access to some users or their access rights. please consult
  section 5.1 for more information.

stats timeout <timeout, in milliseconds>
  the default timeout on the stats socket is set to 10 seconds. it is possible
  to change this value with "stats timeout". the value must be passed in
  milliseconds, or be suffixed by a time unit among { us, ms, s, m, h, d }.

stats maxconn <connections>
  by default, the stats socket is limited to 10 concurrent connections. it is
  possible to change this value with "stats maxconn".

uid <number>
  changes the process' user id to <number>. it is recommended that the user id
  is dedicated to haproxy or to a small set of similar daemons. haproxy must
  be started with superuser privileges in order to be able to switch to another
  one. see also "gid" and "user".

ulimit-n <number>
  sets the maximum number of per-process file-descriptors to <number>. by
  default, it is automatically computed, so it is recommended not to use this
  option.

unix-bind [ prefix <prefix> ] [ mode <mode> ] [ user <user> ] [ uid <uid> ]
          [ group <group> ] [ gid <gid> ]

  fixes common settings to unix listening sockets declared in "bind" statements.
  this is mainly used to simplify declaration of those unix sockets and reduce
  the risk of errors, since those settings are most commonly required but are
  also process-specific. the <prefix> setting can be used to force all socket
  path to be relative to that directory. this might be needed to access another
  component's chroot. note that those paths are resolved before haproxy chroots
  itself, so they are absolute. the <mode>, <user>, <uid>, <group> and <gid>
  all have the same meaning as their homonyms used by the "bind" statement. if
  both are specified, the "bind" statement has priority, meaning that the
  "unix-bind" settings may be seen as process-wide default settings.

user <user name>
  similar to "uid" but uses the uid of user name <user name> from /etc/passwd.
  see also "uid" and "group".

node <name>
  only letters, digits, hyphen and underscore are allowed, like in dns names.

  this statement is useful in ha configurations where two or more processes or
  servers share the same ip address. by setting a different node-name on all
  nodes, it becomes easy to immediately spot what server is handling the
  traffic.

description <text>
  add a text that describes the instance.

  please note that it is required to escape certain characters (# for example)
  and this text is inserted into a html page so you should avoid using
  "<" and ">" characters.

51degrees-data-file <file path>
  the path of the 51degrees data file to provide device detection services. the
  file should be unzipped and accessible by haproxy with relevavnt permissions.

  please note that this option is only available when haproxy has been
  compiled with use_51degrees.

51degrees-property-name-list [<string>]
  a list of 51degrees property names to be load from the dataset. a full list
  of names is available on the 51degrees website:
  https://51degrees.com/resources/property-dictionary

  please note that this option is only available when haproxy has been
  compiled with use_51degrees.

51degrees-property-separator <char>
  a char that will be appended to every property value in a response header
  containing 51degrees results. if not set that will be set as ','.

  please note that this option is only available when haproxy has been
  compiled with use_51degrees.

51degrees-cache-size <number>
  sets the size of the 51degrees converter cache to <number> entries. this
  is an lru cache which reminds previous device detections and their results.
  by default, this cache is disabled.

  please note that this option is only available when haproxy has been
  compiled with use_51degrees.

3.2\. performance tuning
-----------------------

max-spread-checks <delay in milliseconds>
  by default, haproxy tries to spread the start of health checks across the
  smallest health check interval of all the servers in a farm. the principle is
  to avoid hammering services running on the same server. but when using large
  check intervals (10 seconds or more), the last servers in the farm take some
  time before starting to be tested, which can be a problem. this parameter is
  used to enforce an upper bound on delay between the first and the last check,
  even if the servers' check intervals are larger. when servers run with
  shorter intervals, their intervals will be respected though.

maxconn <number>
  sets the maximum per-process number of concurrent connections to <number>. it
  is equivalent to the command-line argument "-n". proxies will stop accepting
  connections when this limit is reached. the "ulimit-n" parameter is
  automatically adjusted according to this value. see also "ulimit-n". note:
  the "select" poller cannot reliably use more than 1024 file descriptors on
  some platforms. if your platform only supports select and reports "select
  failed" on startup, you need to reduce maxconn until it works (slightly
  below 500 in general). if this value is not set, it will default to the value
  set in default_maxconn at build time (reported in haproxy -vv) if no memory
  limit is enforced, or will be computed based on the memory limit, the buffer
  size, memory allocated to compression, ssl cache size, and use or not of ssl
  and the associated maxsslconn (which can also be automatic).

maxconnrate <number>
  sets the maximum per-process number of connections per second to <number>.
  proxies will stop accepting connections when this limit is reached. it can be
  used to limit the global capacity regardless of each frontend capacity. it is
  important to note that this can only be used as a service protection measure,
  as there will not necessarily be a fair share between frontends when the
  limit is reached, so it's a good idea to also limit each frontend to some
  value close to its expected share. also, lowering tune.maxaccept can improve
  fairness.

maxcomprate <number>
  sets the maximum per-process input compression rate to <number> kilobytes
  per second.  for each session, if the maximum is reached, the compression
  level will be decreased during the session. if the maximum is reached at the
  beginning of a session, the session will not compress at all. if the maximum
  is not reached, the compression level will be increased up to
  tune.comp.maxlevel.  a value of zero means there is no limit, this is the
  default value.

maxcompcpuusage <number>
  sets the maximum cpu usage haproxy can reach before stopping the compression
  for new requests or decreasing the compression level of current requests.
  it works like 'maxcomprate' but measures cpu usage instead of incoming data
  bandwidth. the value is expressed in percent of the cpu used by haproxy. in
  case of multiple processes (nbproc > 1), each process manages its individual
  usage. a value of 100 disable the limit. the default value is 100\. setting
  a lower value will prevent the compression work from slowing the whole
  process down and from introducing high latencies.

maxpipes <number>
  sets the maximum per-process number of pipes to <number>. currently, pipes
  are only used by kernel-based tcp splicing. since a pipe contains two file
  descriptors, the "ulimit-n" value will be increased accordingly. the default
  value is maxconn/4, which seems to be more than enough for most heavy usages.
  the splice code dynamically allocates and releases pipes, and can fall back
  to standard copy, so setting this value too low may only impact performance.

maxsessrate <number>
  sets the maximum per-process number of sessions per second to <number>.
  proxies will stop accepting connections when this limit is reached. it can be
  used to limit the global capacity regardless of each frontend capacity. it is
  important to note that this can only be used as a service protection measure,
  as there will not necessarily be a fair share between frontends when the
  limit is reached, so it's a good idea to also limit each frontend to some
  value close to its expected share. also, lowering tune.maxaccept can improve
  fairness.

maxsslconn <number>
  sets the maximum per-process number of concurrent ssl connections to
  <number>. by default there is no ssl-specific limit, which means that the
  global maxconn setting will apply to all connections. setting this limit
  avoids having openssl use too much memory and crash when malloc returns null
  (since it unfortunately does not reliably check for such conditions). note
  that the limit applies both to incoming and outgoing connections, so one
  connection which is deciphered then ciphered accounts for 2 ssl connections.
  if this value is not set, but a memory limit is enforced, this value will be
  automatically computed based on the memory limit, maxconn,  the buffer size,
  memory allocated to compression, ssl cache size, and use of ssl in either
  frontends, backends or both. if neither maxconn nor maxsslconn are specified
  when there is a memory limit, haproxy will automatically adjust these values
  so that 100% of the connections can be made over ssl with no risk, and will
  consider the sides where it is enabled (frontend, backend, both).

maxsslrate <number>
  sets the maximum per-process number of ssl sessions per second to <number>.
  ssl listeners will stop accepting connections when this limit is reached. it
  can be used to limit the global ssl cpu usage regardless of each frontend
  capacity. it is important to note that this can only be used as a service
  protection measure, as there will not necessarily be a fair share between
  frontends when the limit is reached, so it's a good idea to also limit each
  frontend to some value close to its expected share. it is also important to
  note that the sessions are accounted before they enter the ssl stack and not
  after, which also protects the stack against bad handshakes. also, lowering
  tune.maxaccept can improve fairness.

maxzlibmem <number>
  sets the maximum amount of ram in megabytes per process usable by the zlib.
  when the maximum amount is reached, future sessions will not compress as long
  as ram is unavailable. when sets to 0, there is no limit.
  the default value is 0\. the value is available in bytes on the unix socket
  with "show info" on the line "maxzlibmemusage", the memory used by zlib is
  "zlibmemusage" in bytes.

noepoll
  disables the use of the "epoll" event polling system on linux. it is
  equivalent to the command-line argument "-de". the next polling system
  used will generally be "poll". see also "nopoll".

nokqueue
  disables the use of the "kqueue" event polling system on bsd. it is
  equivalent to the command-line argument "-dk". the next polling system
  used will generally be "poll". see also "nopoll".

nopoll
  disables the use of the "poll" event polling system. it is equivalent to the
  command-line argument "-dp". the next polling system used will be "select".
  it should never be needed to disable "poll" since it's available on all
  platforms supported by haproxy. see also "nokqueue" and "noepoll".

nosplice
  disables the use of kernel tcp splicing between sockets on linux. it is
  equivalent to the command line argument "-ds".  data will then be copied
  using conventional and more portable recv/send calls. kernel tcp splicing is
  limited to some very recent instances of kernel 2.6\. most versions between
  2.6.25 and 2.6.28 are buggy and will forward corrupted data, so they must not
  be used. this option makes it easier to globally disable kernel splicing in
  case of doubt. see also "option splice-auto", "option splice-request" and
  "option splice-response".

nogetaddrinfo
  disables the use of getaddrinfo(3) for name resolving. it is equivalent to
  the command line argument "-dg". deprecated gethostbyname(3) will be used.

spread-checks <0..50, in percent>
  sometimes it is desirable to avoid sending agent and health checks to
  servers at exact intervals, for instance when many logical servers are
  located on the same physical server. with the help of this parameter, it
  becomes possible to add some randomness in the check interval between 0
  and +/- 50%. a value between 2 and 5 seems to show good results. the
  default value remains at 0.

tune.buffers.limit <number>
  sets a hard limit on the number of buffers which may be allocated per process.
  the default value is zero which means unlimited. the minimum non-zero value
  will always be greater than "tune.buffers.reserve" and should ideally always
  be about twice as large. forcing this value can be particularly useful to
  limit the amount of memory a process may take, while retaining a sane
  behaviour. when this limit is reached, sessions which need a buffer wait for
  another one to be released by another session. since buffers are dynamically
  allocated and released, the waiting time is very short and not perceptible
  provided that limits remain reasonable. in fact sometimes reducing the limit
  may even increase performance by increasing the cpu cache's efficiency. tests
  have shown good results on average http traffic with a limit to 1/10 of the
  expected global maxconn setting, which also significantly reduces memory
  usage. the memory savings come from the fact that a number of connections
  will not allocate 2*tune.bufsize. it is best not to touch this value unless
  advised to do so by an haproxy core developer.

tune.buffers.reserve <number>
  sets the number of buffers which are pre-allocated and reserved for use only
  during memory shortage conditions resulting in failed memory allocations. the
  minimum value is 2 and is also the default. there is no reason a user would
  want to change this value, it's mostly aimed at haproxy core developers.

tune.bufsize <number>
  sets the buffer size to this size (in bytes). lower values allow more
  sessions to coexist in the same amount of ram, and higher values allow some
  applications with very large cookies to work. the default value is 16384 and
  can be changed at build time. it is strongly recommended not to change this
  from the default value, as very low values will break some services such as
  statistics, and values larger than default size will increase memory usage,
  possibly causing the system to run out of memory. at least the global maxconn
  parameter should be decreased by the same factor as this one is increased.
  if http request is larger than (tune.bufsize - tune.maxrewrite), haproxy will
  return http 400 (bad request) error. similarly if an http response is larger
  than this size, haproxy will return http 502 (bad gateway).

tune.chksize <number>
  sets the check buffer size to this size (in bytes). higher values may help
  find string or regex patterns in very large pages, though doing so may imply
  more memory and cpu usage. the default value is 16384 and can be changed at
  build time. it is not recommended to change this value, but to use better
  checks whenever possible.

tune.comp.maxlevel <number>
  sets the maximum compression level. the compression level affects cpu
  usage during compression. this value affects cpu usage during compression.
  each session using compression initializes the compression algorithm with
  this value. the default value is 1.

tune.http.cookielen <number>
  sets the maximum length of captured cookies. this is the maximum value that
  the "capture cookie xxx len yyy" will be allowed to take, and any upper value
  will automatically be truncated to this one. it is important not to set too
  high a value because all cookie captures still allocate this size whatever
  their configured value (they share a same pool). this value is per request
  per response, so the memory allocated is twice this value per connection.
  when not specified, the limit is set to 63 characters. it is recommended not
  to change this value.

tune.http.maxhdr <number>
  sets the maximum number of headers in a request. when a request comes with a
  number of headers greater than this value (including the first line), it is
  rejected with a "400 bad request" status code. similarly, too large responses
  are blocked with "502 bad gateway". the default value is 101, which is enough
  for all usages, considering that the widely deployed apache server uses the
  same limit. it can be useful to push this limit further to temporarily allow
  a buggy application to work by the time it gets fixed. keep in mind that each
  new header consumes 32bits of memory for each session, so don't push this
  limit too high.

tune.idletimer <timeout>
  sets the duration after which haproxy will consider that an empty buffer is
  probably associated with an idle stream. this is used to optimally adjust
  some packet sizes while forwarding large and small data alternatively. the
  decision to use splice() or to send large buffers in ssl is modulated by this
  parameter. the value is in milliseconds between 0 and 65535\. a value of zero
  means that haproxy will not try to detect idle streams. the default is 1000,
  which seems to correctly detect end user pauses (eg: read a page before
  clicking). there should be not reason for changing this value. please check
  tune.ssl.maxrecord below.

tune.lua.forced-yield <number>
  this directive forces the lua engine to execute a yield each <number> of
  instructions executed. this permits interruptng a long script and allows the
  haproxy scheduler to process other tasks like accepting connections or
  forwarding traffic. the default value is 10000 instructions. if haproxy often
  executes some lua code but more reactivity is required, this value can be
  lowered. if the lua code is quite long and its result is absolutely required
  to process the data, the <number> can be increased.

tune.lua.maxmem
  sets the maximum amount of ram in megabytes per process usable by lua. by
  default it is zero which means unlimited. it is important to set a limit to
  ensure that a bug in a script will not result in the system running out of
  memory.

tune.lua.session-timeout <timeout>
  this is the execution timeout for the lua sessions. this is useful for
  preventing infinite loops or spending too much time in lua. this timeout has a
  priority over other timeouts. for example, if this timeout is set to 4s and
  you run a 5s sleep, the code will be interrupted with an error after waiting
  4s.

tune.lua.task-timeout <timeout>
  purpose is the same as "tune.lua.session-timeout", but this timeout is
  dedicated to the tasks. by default, this timeout isn't set because a task may
  remain alive during of the lifetime of haproxy. for example, a task used to
  check servers.

tune.maxaccept <number>
  sets the maximum number of consecutive connections a process may accept in a
  row before switching to other work. in single process mode, higher numbers
  give better performance at high connection rates. however in multi-process
  modes, keeping a bit of fairness between processes generally is better to
  increase performance. this value applies individually to each listener, so
  that the number of processes a listener is bound to is taken into account.
  this value defaults to 64\. in multi-process mode, it is divided by twice
  the number of processes the listener is bound to. setting this value to -1
  completely disables the limitation. it should normally not be needed to tweak
  this value.

tune.maxpollevents <number>
  sets the maximum amount of events that can be processed at once in a call to
  the polling system. the default value is adapted to the operating system. it
  has been noticed that reducing it below 200 tends to slightly decrease
  latency at the expense of network bandwidth, and increasing it above 200
  tends to trade latency for slightly increased bandwidth.

tune.maxrewrite <number>
  sets the reserved buffer space to this size in bytes. the reserved space is
  used for header rewriting or appending. the first reads on sockets will never
  fill more than bufsize-maxrewrite. historically it has defaulted to half of
  bufsize, though that does not make much sense since there are rarely large
  numbers of headers to add. setting it too high prevents processing of large
  requests or responses. setting it too low prevents addition of new headers
  to already large requests or to post requests. it is generally wise to set it
  to about 1024\. it is automatically readjusted to half of bufsize if it is
  larger than that. this means you don't have to worry about it when changing
  bufsize.

tune.pattern.cache-size <number>
  sets the size of the pattern lookup cache to <number> entries. this is an lru
  cache which reminds previous lookups and their results. it is used by acls
  and maps on slow pattern lookups, namely the ones using the "sub", "reg",
  "dir", "dom", "end", "bin" match methods as well as the case-insensitive
  strings. it applies to pattern expressions which means that it will be able
  to memorize the result of a lookup among all the patterns specified on a
  configuration line (including all those loaded from files). it automatically
  invalidates entries which are updated using http actions or on the cli. the
  default cache size is set to 10000 entries, which limits its footprint to
  about 5 mb on 32-bit systems and 8 mb on 64-bit systems. there is a very low
  risk of collision in this cache, which is in the order of the size of the
  cache divided by 2^64\. typically, at 10000 requests per second with the
  default cache size of 10000 entries, there's 1% chance that a brute force
  attack could cause a single collision after 60 years, or 0.1% after 6 years.
  this is considered much lower than the risk of a memory corruption caused by
  aging components. if this is not acceptable, the cache can be disabled by
  setting this parameter to 0.

tune.pipesize <number>
  sets the kernel pipe buffer size to this size (in bytes). by default, pipes
  are the default size for the system. but sometimes when using tcp splicing,
  it can improve performance to increase pipe sizes, especially if it is
  suspected that pipes are not filled and that many calls to splice() are
  performed. this has an impact on the kernel's memory footprint, so this must
  not be changed if impacts are not understood.

tune.rcvbuf.client <number>
tune.rcvbuf.server <number>
  forces the kernel socket receive buffer size on the client or the server side
  to the specified value in bytes. this value applies to all tcp/http frontends
  and backends. it should normally never be set, and the default size (0) lets
  the kernel autotune this value depending on the amount of available memory.
  however it can sometimes help to set it to very low values (eg: 4096) in
  order to save kernel memory by preventing it from buffering too large amounts
  of received data. lower values will significantly increase cpu usage though.

tune.sndbuf.client <number>
tune.sndbuf.server <number>
  forces the kernel socket send buffer size on the client or the server side to
  the specified value in bytes. this value applies to all tcp/http frontends
  and backends. it should normally never be set, and the default size (0) lets
  the kernel autotune this value depending on the amount of available memory.
  however it can sometimes help to set it to very low values (eg: 4096) in
  order to save kernel memory by preventing it from buffering too large amounts
  of received data. lower values will significantly increase cpu usage though.
  another use case is to prevent write timeouts with extremely slow clients due
  to the kernel waiting for a large part of the buffer to be read before
  notifying haproxy again.

tune.ssl.cachesize <number>
  sets the size of the global ssl session cache, in a number of blocks. a block
  is large enough to contain an encoded session without peer certificate.
  an encoded session with peer certificate is stored in multiple blocks
  depending on the size of the peer certificate. a block uses approximately
  200 bytes of memory. the default value may be forced at build time, otherwise
  defaults to 20000\.  when the cache is full, the most idle entries are purged
  and reassigned. higher values reduce the occurrence of such a purge, hence
  the number of cpu-intensive ssl handshakes by ensuring that all users keep
  their session as long as possible. all entries are pre-allocated upon startup
  and are shared between all processes if "nbproc" is greater than 1\. setting
  this value to 0 disables the ssl session cache.

tune.ssl.force-private-cache
  this boolean disables ssl session cache sharing between all processes. it
  should normally not be used since it will force many renegotiations due to
  clients hitting a random process. but it may be required on some operating
  systems where none of the ssl cache synchronization method may be used. in
  this case, adding a first layer of hash-based load balancing before the ssl
  layer might limit the impact of the lack of session sharing.

tune.ssl.lifetime <timeout>
  sets how long a cached ssl session may remain valid. this time is expressed
  in seconds and defaults to 300 (5 min). it is important to understand that it
  does not guarantee that sessions will last that long, because if the cache is
  full, the longest idle sessions will be purged despite their configured
  lifetime. the real usefulness of this setting is to prevent sessions from
  being used for too long.

tune.ssl.maxrecord <number>
  sets the maximum amount of bytes passed to ssl_write() at a time. default
  value 0 means there is no limit. over ssl/tls, the client can decipher the
  data only once it has received a full record. with large records, it means
  that clients might have to download up to 16kb of data before starting to
  process them. limiting the value can improve page load times on browsers
  located over high latency or low bandwidth networks. it is suggested to find
  optimal values which fit into 1 or 2 tcp segments (generally 1448 bytes over
  ethernet with tcp timestamps enabled, or 1460 when timestamps are disabled),
  keeping in mind that ssl/tls add some overhead. typical values of 1419 and
  2859 gave good results during tests. use "strace -e trace=write" to find the
  best value. haproxy will automatically switch to this setting after an idle
  stream has been detected (see tune.idletimer above).

tune.ssl.default-dh-param <number>
  sets the maximum size of the diffie-hellman parameters used for generating
  the ephemeral/temporary diffie-hellman key in case of dhe key exchange. the
  final size will try to match the size of the server's rsa (or dsa) key (e.g,
  a 2048 bits temporary dh key for a 2048 bits rsa key), but will not exceed
  this maximum value. default value if 1024\. only 1024 or higher values are
  allowed. higher values will increase the cpu load, and values greater than
  1024 bits are not supported by java 7 and earlier clients. this value is not
  used if static diffie-hellman parameters are supplied either directly
  in the certificate file or by using the ssl-dh-param-file parameter.

tune.ssl.ssl-ctx-cache-size <number>
  sets the size of the cache used to store generated certificates to <number>
  entries. this is a lru cache. because generating a ssl certificate
  dynamically is expensive, they are cached. the default cache size is set to
  1000 entries.

tune.vars.global-max-size <size>
tune.vars.reqres-max-size <size>
tune.vars.sess-max-size <size>
tune.vars.txn-max-size <size>
  these four tunes helps to manage the allowed amount of memory used by the
  variables system. "global" limits the memory for all the systems. "sess" limit
  the memory by session, "txn" limits the memory by transaction and "reqres"
  limits the memory for each request or response processing. during the
  accounting, "sess" embbed "txn" and "txn" embed "reqres".

  by example, we considers that "tune.vars.sess-max-size" is fixed to 100,
  "tune.vars.txn-max-size" is fixed to 100, "tune.vars.reqres-max-size" is
  also fixed to 100\. if we create a variable "txn.var" that contains 100 bytes,
  we cannot create any more variable in the other contexts.

tune.zlib.memlevel <number>
  sets the memlevel parameter in zlib initialization for each session. it
  defines how much memory should be allocated for the internal compression
  state. a value of 1 uses minimum memory but is slow and reduces compression
  ratio, a value of 9 uses maximum memory for optimal speed.  can be a value
  between 1 and 9\. the default value is 8.

tune.zlib.windowsize <number>
  sets the window size (the size of the history buffer) as a parameter of the
  zlib initialization for each session. larger values of this parameter result
  in better compression at the expense of memory usage.  can be a value between
  8 and 15\.  the default value is 15.

3.3\. debugging
--------------

debug
  enables debug mode which dumps to stdout all exchanges, and disables forking
  into background. it is the equivalent of the command-line argument "-d". it
  should never be used in a production configuration since it may prevent full
  system startup.

quiet
  do not display any message during startup. it is equivalent to the command-
  line argument "-q".

3.4\. userlists
--------------
it is possible to control access to frontend/backend/listen sections or to
http stats by allowing only authenticated and authorized users. to do this,
it is required to create at least one userlist and to define users.

userlist <listname>
  creates new userlist with name <listname>. many independent userlists can be
  used to store authentication & authorization data for independent customers.

group <groupname> [users <user>,<user>,(...)]
  adds group <groupname> to the current userlist. it is also possible to
  attach users to this group by using a comma separated list of names
  proceeded by "users" keyword.

user <username> [password|insecure-password <password>]
                [groups <group>,<group>,(...)]
  adds user <username> to the current userlist. both secure (encrypted) and
  insecure (unencrypted) passwords can be used. encrypted passwords are
  evaluated using the crypt(3) function so depending of the system's
  capabilities, different algorithms are supported. for example modern glibc
  based linux system supports md5, sha-256, sha-512 and of course classic,
  des-based method of encrypting passwords.

  example:
        userlist l1
          group g1 users tiger,scott
          group g2 users xdb,scott

          user tiger password $6$k6y3o.ep$jlkbx9za9667qe4(...)xhswrv6j.c0/d7cv91
          user scott insecure-password elgato
          user xdb insecure-password hello

        userlist l2
          group g1
          group g2

          user tiger password $6$k6y3o.ep$jlkbx(...)xhswrv6j.c0/d7cv91 groups g1
          user scott insecure-password elgato groups g1,g2
          user xdb insecure-password hello groups g2

  please note that both lists are functionally identical.

3.5\. peers
----------
it is possible to propagate entries of any data-types in stick-tables between
several haproxy instances over tcp connections in a multi-master fashion. each
instance pushes its local updates and insertions to remote peers. the pushed
values overwrite remote ones without aggregation. interrupted exchanges are
automatically detected and recovered from the last known point.
in addition, during a soft restart, the old process connects to the new one
using such a tcp connection to push all its entries before the new process
tries to connect to other peers. that ensures very fast replication during a
reload, it typically takes a fraction of a second even for large tables.
note that server ids are used to identify servers remotely, so it is important
that configurations look similar or at least that the same ids are forced on
each server on all participants.

peers <peersect>
  creates a new peer list with name <peersect>. it is an independent section,
  which is referenced by one or more stick-tables.

disabled
  disables a peers section. it disables both listening and any synchronization
  related to this section. this is provided to disable synchronization of stick
  tables without having to comment out all "peers" references.

enable
  this re-enables a disabled peers section which was previously disabled.

peer <peername> <ip>:<port>
  defines a peer inside a peers section.
  if <peername> is set to the local peer name (by default hostname, or forced
  using "-l" command line option), haproxy will listen for incoming remote peer
  connection on <ip>:<port>. otherwise, <ip>:<port> defines where to connect to
  to join the remote peer, and <peername> is used at the protocol level to
  identify and validate the remote peer on the server side.

  during a soft restart, local peer <ip>:<port> is used by the old instance to
  connect the new one and initiate a complete replication (teaching process).

  it is strongly recommended to have the exact same peers declaration on all
  peers and to only rely on the "-l" command line argument to change the local
  peer name. this makes it easier to maintain coherent configuration files
  across all peers.

  you may want to reference some environment variables in the address
  parameter, see section 2.3 about environment variables.

  example:
    peers mypeers
        peer haproxy1 192.168.0.1:1024
        peer haproxy2 192.168.0.2:1024
        peer haproxy3 10.2.0.1:1024

    backend mybackend
        mode tcp
        balance roundrobin
        stick-table type ip size 20k peers mypeers
        stick on src

        server srv1 192.168.0.30:80
        server srv2 192.168.0.31:80

3.6\. mailers
------------
it is possible to send email alerts when the state of servers changes.
if configured email alerts are sent to each mailer that is configured
in a mailers section. email is sent to mailers using smtp.

mailer <mailersect>
  creates a new mailer list with the name <mailersect>. it is an
  independent section which is referenced by one or more proxies.

mailer <mailername> <ip>:<port>
  defines a mailer inside a mailers section.

  example:
    mailers mymailers
        mailer smtp1 192.168.0.1:587
        mailer smtp2 192.168.0.2:587

    backend mybackend
        mode tcp
        balance roundrobin

        email-alert mailers mymailers
        email-alert from test1@horms.org
        email-alert to test2@horms.org

        server srv1 192.168.0.30:80
        server srv2 192.168.0.31:80

4\. proxies
----------

proxy configuration can be located in a set of sections :
 - defaults [<name>]
 - frontend <name>
 - backend  <name>
 - listen   <name>

a "defaults" section sets default parameters for all other sections following
its declaration. those default parameters are reset by the next "defaults"
section. see below for the list of parameters which can be set in a "defaults"
section. the name is optional but its use is encouraged for better readability.

a "frontend" section describes a set of listening sockets accepting client
connections.

a "backend" section describes a set of servers to which the proxy will connect
to forward incoming connections.

a "listen" section defines a complete proxy with its frontend and backend
parts combined in one section. it is generally useful for tcp-only traffic.

all proxy names must be formed from upper and lower case letters, digits,
'-' (dash), '_' (underscore) , '.' (dot) and ':' (colon). acl names are
case-sensitive, which means that "www" and "www" are two different proxies.

historically, all proxy names could overlap, it just caused troubles in the
logs. since the introduction of content switching, it is mandatory that two
proxies with overlapping capabilities (frontend/backend) have different names.
however, it is still permitted that a frontend and a backend share the same
name, as this configuration seems to be commonly encountered.

right now, two major proxy modes are supported : "tcp", also known as layer 4,
and "http", also known as layer 7\. in layer 4 mode, haproxy simply forwards
bidirectional traffic between two sides. in layer 7 mode, haproxy analyzes the
protocol, and can interact with it by allowing, blocking, switching, adding,
modifying, or removing arbitrary contents in requests or responses, based on
arbitrary criteria.

in http mode, the processing applied to requests and responses flowing over
a connection depends in the combination of the frontend's http options and
the backend's. haproxy supports 5 connection modes :

  - kal : keep alive ("option http-keep-alive") which is the default mode : all
    requests and responses are processed, and connections remain open but idle
    between responses and new requests.

  - tun: tunnel ("option http-tunnel") : this was the default mode for versions
    1.0 to 1.5-dev21 : only the first request and response are processed, and
    everything else is forwarded with no analysis at all. this mode should not
    be used as it creates lots of trouble with logging and http processing.

  - pcl: passive close ("option httpclose") : exactly the same as tunnel mode,
    but with "connection: close" appended in both directions to try to make
    both ends close after the first request/response exchange.

  - scl: server close ("option http-server-close") : the server-facing
    connection is closed after the end of the response is received, but the
    client-facing connection remains open.

  - fcl: forced close ("option forceclose") : the connection is actively closed
    after the end of the response.

the effective mode that will be applied to a connection passing through a
frontend and a backend can be determined by both proxy modes according to the
following matrix, but in short, the modes are symmetric, keep-alive is the
weakest option and force close is the strongest.

                          backend mode

                | kal | tun | pcl | scl | fcl
            ----+-----+-----+-----+-----+----
            kal | kal | tun | pcl | scl | fcl
            ----+-----+-----+-----+-----+----
            tun | tun | tun | pcl | scl | fcl
 frontend   ----+-----+-----+-----+-----+----
   mode     pcl | pcl | pcl | pcl | fcl | fcl
            ----+-----+-----+-----+-----+----
            scl | scl | scl | fcl | scl | fcl
            ----+-----+-----+-----+-----+----
            fcl | fcl | fcl | fcl | fcl | fcl

4.1\. proxy keywords matrix
--------------------------

the following list of keywords is supported. most of them may only be used in a
limited set of section types. some of them are marked as "deprecated" because
they are inherited from an old syntax which may be confusing or functionally
limited, and there are new recommended keywords to replace them. keywords
marked with "(*)" can be optionally inverted using the "no" prefix, eg. "no
option contstats". this makes sense when the option has been enabled by default
and must be disabled for a specific instance. such options may also be prefixed
with "default" in order to restore default settings regardless of what has been
specified in a previous "defaults" section.

 keyword                              defaults   frontend   listen    backend
------------------------------------+----------+----------+---------+---------
acl                                       -          x         x         x
appsession                                -          -         x         x
backlog                                   x          x         x         -
balance                                   x          -         x         x
bind                                      -          x         x         -
bind-process                              x          x         x         x
block                                     -          x         x         x
capture cookie                            -          x         x         -
capture request header                    -          x         x         -
capture response header                   -          x         x         -
clitimeout                  (deprecated)  x          x         x         -
compression                               x          x         x         x
contimeout                  (deprecated)  x          -         x         x
cookie                                    x          -         x         x
declare capture                           -          x         x         -
default-server                            x          -         x         x
default_backend                           x          x         x         -
description                               -          x         x         x
disabled                                  x          x         x         x
dispatch                                  -          -         x         x
email-alert from                          x          x         x         x
email-alert level                         x          x         x         x
email-alert mailers                       x          x         x         x
email-alert myhostname                    x          x         x         x
email-alert to                            x          x         x         x
enabled                                   x          x         x         x
errorfile                                 x          x         x         x
errorloc                                  x          x         x         x
errorloc302                               x          x         x         x
-- keyword -------------------------- defaults - frontend - listen -- backend -
errorloc303                               x          x         x         x
force-persist                             -          x         x         x
fullconn                                  x          -         x         x
grace                                     x          x         x         x
hash-type                                 x          -         x         x
http-check disable-on-404                 x          -         x         x
http-check expect                         -          -         x         x
http-check send-state                     x          -         x         x
http-request                              -          x         x         x
http-response                             -          x         x         x
http-send-name-header                     -          -         x         x
id                                        -          x         x         x
ignore-persist                            -          x         x         x
log                                  (*)  x          x         x         x
log-format                                x          x         x         -
log-tag                                   x          x         x         x
max-keep-alive-queue                      x          -         x         x
maxconn                                   x          x         x         -
mode                                      x          x         x         x
monitor fail                              -          x         x         -
monitor-net                               x          x         x         -
monitor-uri                               x          x         x         -
option abortonclose                  (*)  x          -         x         x
option accept-invalid-http-request   (*)  x          x         x         -
option accept-invalid-http-response  (*)  x          -         x         x
option allbackups                    (*)  x          -         x         x
option checkcache                    (*)  x          -         x         x
option clitcpka                      (*)  x          x         x         -
option contstats                     (*)  x          x         x         -
option dontlog-normal                (*)  x          x         x         -
option dontlognull                   (*)  x          x         x         -
option forceclose                    (*)  x          x         x         x
-- keyword -------------------------- defaults - frontend - listen -- backend -
option forwardfor                         x          x         x         x
option http-buffer-request           (*)  x          x         x         x
option http-ignore-probes            (*)  x          x         x         -
option http-keep-alive               (*)  x          x         x         x
option http-no-delay                 (*)  x          x         x         x
option http-pretend-keepalive        (*)  x          x         x         x
option http-server-close             (*)  x          x         x         x
option http-tunnel                   (*)  x          x         x         x
option http-use-proxy-header         (*)  x          x         x         -
option httpchk                            x          -         x         x
option httpclose                     (*)  x          x         x         x
option httplog                            x          x         x         x
option http_proxy                    (*)  x          x         x         x
option independent-streams           (*)  x          x         x         x
option ldap-check                         x          -         x         x
option external-check                     x          -         x         x
option log-health-checks             (*)  x          -         x         x
option log-separate-errors           (*)  x          x         x         -
option logasap                       (*)  x          x         x         -
option mysql-check                        x          -         x         x
option pgsql-check                        x          -         x         x
option nolinger                      (*)  x          x         x         x
option originalto                         x          x         x         x
option persist                       (*)  x          -         x         x
option redispatch                    (*)  x          -         x         x
option redis-check                        x          -         x         x
option smtpchk                            x          -         x         x
option socket-stats                  (*)  x          x         x         -
option splice-auto                   (*)  x          x         x         x
option splice-request                (*)  x          x         x         x
option splice-response               (*)  x          x         x         x
option srvtcpka                      (*)  x          -         x         x
option ssl-hello-chk                      x          -         x         x
-- keyword -------------------------- defaults - frontend - listen -- backend -
option tcp-check                          x          -         x         x
option tcp-smart-accept              (*)  x          x         x         -
option tcp-smart-connect             (*)  x          -         x         x
option tcpka                              x          x         x         x
option tcplog                             x          x         x         x
option transparent                   (*)  x          -         x         x
external-check command                    x          -         x         x
external-check path                       x          -         x         x
persist rdp-cookie                        x          -         x         x
rate-limit sessions                       x          x         x         -
redirect                                  -          x         x         x
redisp                      (deprecated)  x          -         x         x
redispatch                  (deprecated)  x          -         x         x
reqadd                                    -          x         x         x
reqallow                                  -          x         x         x
reqdel                                    -          x         x         x
reqdeny                                   -          x         x         x
reqiallow                                 -          x         x         x
reqidel                                   -          x         x         x
reqideny                                  -          x         x         x
reqipass                                  -          x         x         x
reqirep                                   -          x         x         x
reqitarpit                                -          x         x         x
reqpass                                   -          x         x         x
reqrep                                    -          x         x         x
-- keyword -------------------------- defaults - frontend - listen -- backend -
reqtarpit                                 -          x         x         x
retries                                   x          -         x         x
rspadd                                    -          x         x         x
rspdel                                    -          x         x         x
rspdeny                                   -          x         x         x
rspidel                                   -          x         x         x
rspideny                                  -          x         x         x
rspirep                                   -          x         x         x
rsprep                                    -          x         x         x
server                                    -          -         x         x
source                                    x          -         x         x
srvtimeout                  (deprecated)  x          -         x         x
stats admin                               -          -         x         x
stats auth                                x          -         x         x
stats enable                              x          -         x         x
stats hide-version                        x          -         x         x
stats http-request                        -          -         x         x
stats realm                               x          -         x         x
stats refresh                             x          -         x         x
stats scope                               x          -         x         x
stats show-desc                           x          -         x         x
stats show-legends                        x          -         x         x
stats show-node                           x          -         x         x
stats uri                                 x          -         x         x
-- keyword -------------------------- defaults - frontend - listen -- backend -
stick match                               -          -         x         x
stick on                                  -          -         x         x
stick store-request                       -          -         x         x
stick store-response                      -          -         x         x
stick-table                               -          -         x         x
tcp-check connect                         -          -         x         x
tcp-check expect                          -          -         x         x
tcp-check send                            -          -         x         x
tcp-check send-binary                     -          -         x         x
tcp-request connection                    -          x         x         -
tcp-request content                       -          x         x         x
tcp-request inspect-delay                 -          x         x         x
tcp-response content                      -          -         x         x
tcp-response inspect-delay                -          -         x         x
timeout check                             x          -         x         x
timeout client                            x          x         x         -
timeout client-fin                        x          x         x         -
timeout clitimeout          (deprecated)  x          x         x         -
timeout connect                           x          -         x         x
timeout contimeout          (deprecated)  x          -         x         x
timeout http-keep-alive                   x          x         x         x
timeout http-request                      x          x         x         x
timeout queue                             x          -         x         x
timeout server                            x          -         x         x
timeout server-fin                        x          -         x         x
timeout srvtimeout          (deprecated)  x          -         x         x
timeout tarpit                            x          x         x         x
timeout tunnel                            x          -         x         x
transparent                 (deprecated)  x          -         x         x
unique-id-format                          x          x         x         -
unique-id-header                          x          x         x         -
use_backend                               -          x         x         -
use-server                                -          -         x         x
------------------------------------+----------+----------+---------+---------
 keyword                              defaults   frontend   listen    backend

4.2\. alphabetically sorted keywords reference
---------------------------------------------

this section provides a description of each keyword and its usage.

acl <aclname> <criterion> [flags] [operator] <value> ...
  declare or complete an access list.
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  example:
        acl invalid_src  src          0.0.0.0/7 224.0.0.0/3
        acl invalid_src  src_port     0:1023
        acl local_dst    hdr(host) -i localhost

  see section 7 about acl usage.

appsession <cookie> len <length> timeout <holdtime>
           [request-learn] [prefix] [mode <path-parameters|query-string>]
  define session stickiness on an existing application cookie.
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes
  arguments :
    <cookie>   this is the name of the cookie used by the application and which
               haproxy will have to learn for each new session.

    <length>   this is the max number of characters that will be memorized and
               checked in each cookie value.

    <holdtime> this is the time after which the cookie will be removed from
               memory if unused. if no unit is specified, this time is in
               milliseconds.

    request-learn
               if this option is specified, then haproxy will be able to learn
               the cookie found in the request in case the server does not
               specify any in response. this is typically what happens with
               phpsessid cookies, or when haproxy's session expires before
               the application's session and the correct server is selected.
               it is recommended to specify this option to improve reliability.

    prefix     when this option is specified, haproxy will match on the cookie
               prefix (or url parameter prefix). the appsession value is the
               data following this prefix.

               example :
               appsession aspsessionid len 64 timeout 3h prefix

               this will match the cookie aspsessionidxxxx=xxxxx,
               the appsession value will be xxxx=xxxxx.

    mode       this option allows to change the url parser mode.
               2 modes are currently supported :
               - path-parameters :
                 the parser looks for the appsession in the path parameters
                 part (each parameter is separated by a semi-colon), which is
                 convenient for jsessionid for example.
                 this is the default mode if the option is not set.
               - query-string :
                 in this mode, the parser will look for the appsession in the
                 query string.

  when an application cookie is defined in a backend, haproxy will check when
  the server sets such a cookie, and will store its value in a table, and
  associate it with the server's identifier. up to <length> characters from
  the value will be retained. on each connection, haproxy will look for this
  cookie both in the "cookie:" headers, and as a url parameter (depending on
  the mode used). if a known value is found, the client will be directed to the
  server associated with this value. otherwise, the load balancing algorithm is
  applied. cookies are automatically removed from memory when they have been
  unused for a duration longer than <holdtime>.

  the definition of an application cookie is limited to one per backend.

  note : consider not using this feature in multi-process mode (nbproc > 1)
         unless you know what you do : memory is not shared between the
         processes, which can result in random behaviours.

  example :
        appsession jsessionid len 52 timeout 3h

  see also : "cookie", "capture cookie", "balance", "stick", "stick-table",
             "ignore-persist", "nbproc" and "bind-process".

backlog <conns>
  give hints to the system about the approximate listen backlog desired size
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <conns>   is the number of pending connections. depending on the operating
              system, it may represent the number of already acknowledged
              connections, of non-acknowledged ones, or both.

  in order to protect against syn flood attacks, one solution is to increase
  the system's syn backlog size. depending on the system, sometimes it is just
  tunable via a system parameter, sometimes it is not adjustable at all, and
  sometimes the system relies on hints given by the application at the time of
  the listen() syscall. by default, haproxy passes the frontend's maxconn value
  to the listen() syscall. on systems which can make use of this value, it can
  sometimes be useful to be able to specify a different value, hence this
  backlog parameter.

  on linux 2.4, the parameter is ignored by the system. on linux 2.6, it is
  used as a hint and the system accepts up to the smallest greater power of
  two, and never more than some limits (usually 32768).

  see also : "maxconn" and the target operating system's tuning guide.

balance <algorithm> [ <arguments> ]
balance url_param <param> [check_post]
  define the load balancing algorithm to be used in a backend.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <algorithm> is the algorithm used to select a server when doing load
                balancing. this only applies when no persistence information
                is available, or when a connection is redispatched to another
                server. <algorithm> may be one of the following :

      roundrobin  each server is used in turns, according to their weights.
                  this is the smoothest and fairest algorithm when the server's
                  processing time remains equally distributed. this algorithm
                  is dynamic, which means that server weights may be adjusted
                  on the fly for slow starts for instance. it is limited by
                  design to 4095 active servers per backend. note that in some
                  large farms, when a server becomes up after having been down
                  for a very short time, it may sometimes take a few hundreds
                  requests for it to be re-integrated into the farm and start
                  receiving traffic. this is normal, though very rare. it is
                  indicated here in case you would have the chance to observe
                  it, so that you don't worry.

      static-rr   each server is used in turns, according to their weights.
                  this algorithm is as similar to roundrobin except that it is
                  static, which means that changing a server's weight on the
                  fly will have no effect. on the other hand, it has no design
                  limitation on the number of servers, and when a server goes
                  up, it is always immediately reintroduced into the farm, once
                  the full map is recomputed. it also uses slightly less cpu to
                  run (around -1%).

      leastconn   the server with the lowest number of connections receives the
                  connection. round-robin is performed within groups of servers
                  of the same load to ensure that all servers will be used. use
                  of this algorithm is recommended where very long sessions are
                  expected, such as ldap, sql, tse, etc... but is not very well
                  suited for protocols using short sessions such as http. this
                  algorithm is dynamic, which means that server weights may be
                  adjusted on the fly for slow starts for instance.

      first       the first server with available connection slots receives the
                  connection. the servers are chosen from the lowest numeric
                  identifier to the highest (see server parameter "id"), which
                  defaults to the server's position in the farm. once a server
                  reaches its maxconn value, the next server is used. it does
                  not make sense to use this algorithm without setting maxconn.
                  the purpose of this algorithm is to always use the smallest
                  number of servers so that extra servers can be powered off
                  during non-intensive hours. this algorithm ignores the server
                  weight, and brings more benefit to long session such as rdp
                  or imap than http, though it can be useful there too. in
                  order to use this algorithm efficiently, it is recommended
                  that a cloud controller regularly checks server usage to turn
                  them off when unused, and regularly checks backend queue to
                  turn new servers on when the queue inflates. alternatively,
                  using "http-check send-state" may inform servers on the load.

      source      the source ip address is hashed and divided by the total
                  weight of the running servers to designate which server will
                  receive the request. this ensures that the same client ip
                  address will always reach the same server as long as no
                  server goes down or up. if the hash result changes due to the
                  number of running servers changing, many clients will be
                  directed to a different server. this algorithm is generally
                  used in tcp mode where no cookie may be inserted. it may also
                  be used on the internet to provide a best-effort stickiness
                  to clients which refuse session cookies. this algorithm is
                  static by default, which means that changing a server's
                  weight on the fly will have no effect, but this can be
                  changed using "hash-type".

      uri         this algorithm hashes either the left part of the uri (before
                  the question mark) or the whole uri (if the "whole" parameter
                  is present) and divides the hash value by the total weight of
                  the running servers. the result designates which server will
                  receive the request. this ensures that the same uri will
                  always be directed to the same server as long as no server
                  goes up or down. this is used with proxy caches and
                  anti-virus proxies in order to maximize the cache hit rate.
                  note that this algorithm may only be used in an http backend.
                  this algorithm is static by default, which means that
                  changing a server's weight on the fly will have no effect,
                  but this can be changed using "hash-type".

                  this algorithm supports two optional parameters "len" and
                  "depth", both followed by a positive integer number. these
                  options may be helpful when it is needed to balance servers
                  based on the beginning of the uri only. the "len" parameter
                  indicates that the algorithm should only consider that many
                  characters at the beginning of the uri to compute the hash.
                  note that having "len" set to 1 rarely makes sense since most
                  uris start with a leading "/".

                  the "depth" parameter indicates the maximum directory depth
                  to be used to compute the hash. one level is counted for each
                  slash in the request. if both parameters are specified, the
                  evaluation stops when either is reached.

      url_param   the url parameter specified in argument will be looked up in
                  the query string of each http get request.

                  if the modifier "check_post" is used, then an http post
                  request entity will be searched for the parameter argument,
                  when it is not found in a query string after a question mark
                  ('?') in the url. the message body will only start to be
                  analyzed once either the advertised amount of data has been
                  received or the request buffer is full. in the unlikely event
                  that chunked encoding is used, only the first chunk is
                  scanned. parameter values separated by a chunk boundary, may
                  be randomly balanced if at all. this keyword used to support
                  an optional <max_wait> parameter which is now ignored.

                  if the parameter is found followed by an equal sign ('=') and
                  a value, then the value is hashed and divided by the total
                  weight of the running servers. the result designates which
                  server will receive the request.

                  this is used to track user identifiers in requests and ensure
                  that a same user id will always be sent to the same server as
                  long as no server goes up or down. if no value is found or if
                  the parameter is not found, then a round robin algorithm is
                  applied. note that this algorithm may only be used in an http
                  backend. this algorithm is static by default, which means
                  that changing a server's weight on the fly will have no
                  effect, but this can be changed using "hash-type".

      hdr(<name>) the http header <name> will be looked up in each http
                  request. just as with the equivalent acl 'hdr()' function,
                  the header name in parenthesis is not case sensitive. if the
                  header is absent or if it does not contain any value, the
                  roundrobin algorithm is applied instead.

                  an optional 'use_domain_only' parameter is available, for
                  reducing the hash algorithm to the main domain part with some
                  specific headers such as 'host'. for instance, in the host
                  value "haproxy.1wt.eu", only "1wt" will be considered.

                  this algorithm is static by default, which means that
                  changing a server's weight on the fly will have no effect,
                  but this can be changed using "hash-type".

      rdp-cookie
      rdp-cookie(<name>)
                  the rdp cookie <name> (or "mstshash" if omitted) will be
                  looked up and hashed for each incoming tcp request. just as
                  with the equivalent acl 'req_rdp_cookie()' function, the name
                  is not case-sensitive. this mechanism is useful as a degraded
                  persistence mode, as it makes it possible to always send the
                  same user (or the same session id) to the same server. if the
                  cookie is not found, the normal roundrobin algorithm is
                  used instead.

                  note that for this to work, the frontend must ensure that an
                  rdp cookie is already present in the request buffer. for this
                  you must use 'tcp-request content accept' rule combined with
                  a 'req_rdp_cookie_cnt' acl.

                  this algorithm is static by default, which means that
                  changing a server's weight on the fly will have no effect,
                  but this can be changed using "hash-type".

                  see also the rdp_cookie pattern fetch function.

    <arguments> is an optional list of arguments which may be needed by some
                algorithms. right now, only "url_param" and "uri" support an
                optional argument.

  the load balancing algorithm of a backend is set to roundrobin when no other
  algorithm, mode nor option have been set. the algorithm may only be set once
  for each backend.

  examples :
        balance roundrobin
        balance url_param userid
        balance url_param session_id check_post 64
        balance hdr(user-agent)
        balance hdr(host)
        balance hdr(host) use_domain_only

  note: the following caveats and limitations on using the "check_post"
  extension with "url_param" must be considered :

    - all post requests are eligible for consideration, because there is no way
      to determine if the parameters will be found in the body or entity which
      may contain binary data. therefore another method may be required to
      restrict consideration of post requests that have no url parameters in
      the body. (see acl reqideny http_end)

    - using a <max_wait> value larger than the request buffer size does not
      make sense and is useless. the buffer size is set at build time, and
      defaults to 16 kb.

    - content-encoding is not supported, the parameter search will probably
      fail; and load balancing will fall back to round robin.

    - expect: 100-continue is not supported, load balancing will fall back to
      round robin.

    - transfer-encoding (rfc2616 3.6.1) is only supported in the first chunk.
      if the entire parameter value is not present in the first chunk, the
      selection of server is undefined (actually, defined by how little
      actually appeared in the first chunk).

    - this feature does not support generation of a 100, 411 or 501 response.

    - in some cases, requesting "check_post" may attempt to scan the entire
      contents of a message body. scanning normally terminates when linear
      white space or control characters are found, indicating the end of what
      might be a url parameter list. this is probably not a concern with sgml
      type message bodies.

  see also : "dispatch", "cookie", "appsession", "transparent", "hash-type" and
             "http_proxy".

bind [<address>]:<port_range> [, ...] [param*]
bind /<path> [, ...] [param*]
  define one or several listening addresses and/or ports in a frontend.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   no
  arguments :
    <address>     is optional and can be a host name, an ipv4 address, an ipv6
                  address, or '*'. it designates the address the frontend will
                  listen on. if unset, all ipv4 addresses of the system will be
                  listened on. the same will apply for '*' or the system's
                  special address "0.0.0.0". the ipv6 equivalent is '::'.
                  optionally, an address family prefix may be used before the
                  address to force the family regardless of the address format,
                  which can be useful to specify a path to a unix socket with
                  no slash ('/'). currently supported prefixes are :
                    - 'ipv4@'  -> address is always ipv4
                    - 'ipv6@'  -> address is always ipv6
                    - 'unix@'  -> address is a path to a local unix socket
                    - 'abns@'  -> address is in abstract namespace (linux only).
                      note: since abstract sockets are not "rebindable", they
                            do not cope well with multi-process mode during
                            soft-restart, so it is better to avoid them if
                            nbproc is greater than 1\. the effect is that if the
                            new process fails to start, only one of the old ones
                            will be able to rebind to the socket.
                    - 'fd@<n>' -> use file descriptor <n> inherited from the
                      parent. the fd must be bound and may or may not already
                      be listening.
                  you may want to reference some environment variables in the
                  address parameter, see section 2.3 about environment
                  variables.

    <port_range>  is either a unique tcp port, or a port range for which the
                  proxy will accept connections for the ip address specified
                  above. the port is mandatory for tcp listeners. note that in
                  the case of an ipv6 address, the port is always the number
                  after the last colon (':'). a range can either be :
                   - a numerical port (ex: '80')
                   - a dash-delimited ports range explicitly stating the lower
                     and upper bounds (ex: '2000-2100') which are included in
                     the range.

                  particular care must be taken against port ranges, because
                  every <address:port> couple consumes one socket (= a file
                  descriptor), so it's easy to consume lots of descriptors
                  with a simple range, and to run out of sockets. also, each
                  <address:port> couple must be used only once among all
                  instances running on a same system. please note that binding
                  to ports lower than 1024 generally require particular
                  privileges to start the program, which are independent of
                  the 'uid' parameter.

    <path>        is a unix socket path beginning with a slash ('/'). this is
                  alternative to the tcp listening port. haproxy will then
                  receive unix connections on the socket located at this place.
                  the path must begin with a slash and by default is absolute.
                  it can be relative to the prefix defined by "unix-bind" in
                  the global section. note that the total length of the prefix
                  followed by the socket path cannot exceed some system limits
                  for unix sockets, which commonly are set to 107 characters.

    <param*>      is a list of parameters common to all sockets declared on the
                  same line. these numerous parameters depend on os and build
                  options and have a complete section dedicated to them. please
                  refer to section 5 to for more details.

  it is possible to specify a list of address:port combinations delimited by
  commas. the frontend will then listen on all of these addresses. there is no
  fixed limit to the number of addresses and ports which can be listened on in
  a frontend, as well as there is no limit to the number of "bind" statements
  in a frontend.

  example :
        listen http_proxy
            bind :80,:443
            bind 10.0.0.1:10080,10.0.0.1:10443
            bind /var/run/ssl-frontend.sock user root mode 600 accept-proxy

        listen http_https_proxy
            bind :80
            bind :443 ssl crt /etc/haproxy/site.pem

        listen http_https_proxy_explicit
            bind ipv6@:80
            bind ipv4@public_ssl:443 ssl crt /etc/haproxy/site.pem
            bind unix@ssl-frontend.sock user root mode 600 accept-proxy

        listen external_bind_app1
            bind "fd@${fd_app1}"

  see also : "source", "option forwardfor", "unix-bind" and the proxy protocol
             documentation, and section 5 about bind options.

bind-process [ all | odd | even | <number 1-64>[-<number 1-64>] ] ...
  limit visibility of an instance to a certain set of processes numbers.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    all           all process will see this instance. this is the default. it
                  may be used to override a default value.

    odd           this instance will be enabled on processes 1,3,5,...63\. this
                  option may be combined with other numbers.

    even          this instance will be enabled on processes 2,4,6,...64\. this
                  option may be combined with other numbers. do not use it
                  with less than 2 processes otherwise some instances might be
                  missing from all processes.

    number        the instance will be enabled on this process number or range,
                  whose values must all be between 1 and 32 or 64 depending on
                  the machine's word size. if a proxy is bound to process
                  numbers greater than the configured global.nbproc, it will
                  either be forced to process #1 if a single process was
                  specified, or to all processes otherwise.

  this keyword limits binding of certain instances to certain processes. this
  is useful in order not to have too many processes listening to the same
  ports. for instance, on a dual-core machine, it might make sense to set
  'nbproc 2' in the global section, then distributes the listeners among 'odd'
  and 'even' instances.

  at the moment, it is not possible to reference more than 32 or 64 processes
  using this keyword, but this should be more than enough for most setups.
  please note that 'all' really means all processes regardless of the machine's
  word size, and is not limited to the first 32 or 64.

  each "bind" line may further be limited to a subset of the proxy's processes,
  please consult the "process" bind keyword in section 5.1.

  when a frontend has no explicit "bind-process" line, it tries to bind to all
  the processes referenced by its "bind" lines. that means that frontends can
  easily adapt to their listeners' processes.

  if some backends are referenced by frontends bound to other processes, the
  backend automatically inherits the frontend's processes.

  example :
        listen app_ip1
            bind 10.0.0.1:80
            bind-process odd

        listen app_ip2
            bind 10.0.0.2:80
            bind-process even

        listen management
            bind 10.0.0.3:80
            bind-process 1 2 3 4

        listen management
            bind 10.0.0.4:80
            bind-process 1-4

  see also : "nbproc" in global section, and "process" in section 5.1.

block { if | unless } <condition>
  block a layer 7 request if/unless a condition is matched
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes

  the http request will be blocked very early in the layer 7 processing
  if/unless <condition> is matched. a 403 error will be returned if the request
  is blocked. the condition has to reference acls (see section 7). this is
  typically used to deny access to certain sensitive resources if some
  conditions are met or not met. there is no fixed limit to the number of
  "block" statements per instance.

  example:
        acl invalid_src  src          0.0.0.0/7 224.0.0.0/3
        acl invalid_src  src_port     0:1023
        acl local_dst    hdr(host) -i localhost
        block if invalid_src || local_dst

  see section 7 about acl usage.

capture cookie <name> len <length>
  capture and log a cookie in the request and in the response.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   no
  arguments :
    <name>    is the beginning of the name of the cookie to capture. in order
              to match the exact name, simply suffix the name with an equal
              sign ('='). the full name will appear in the logs, which is
              useful with application servers which adjust both the cookie name
              and value (eg: aspsessionxxxxx).

    <length>  is the maximum number of characters to report in the logs, which
              include the cookie name, the equal sign and the value, all in the
              standard "name=value" form. the string will be truncated on the
              right if it exceeds <length>.

  only the first cookie is captured. both the "cookie" request headers and the
  "set-cookie" response headers are monitored. this is particularly useful to
  check for application bugs causing session crossing or stealing between
  users, because generally the user's cookies can only change on a login page.

  when the cookie was not presented by the client, the associated log column
  will report "-". when a request does not cause a cookie to be assigned by the
  server, a "-" is reported in the response column.

  the capture is performed in the frontend only because it is necessary that
  the log format does not change for a given frontend depending on the
  backends. this may change in the future. note that there can be only one
  "capture cookie" statement in a frontend. the maximum capture length is set
  by the global "tune.http.cookielen" setting and defaults to 63 characters. it
  is not possible to specify a capture in a "defaults" section.

  example:
        capture cookie aspsession len 32

  see also : "capture request header", "capture response header" as well as
            section 8 about logging.

capture request header <name> len <length>
  capture and log the last occurrence of the specified request header.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   no
  arguments :
    <name>    is the name of the header to capture. the header names are not
              case-sensitive, but it is a common practice to write them as they
              appear in the requests, with the first letter of each word in
              upper case. the header name will not appear in the logs, only the
              value is reported, but the position in the logs is respected.

    <length>  is the maximum number of characters to extract from the value and
              report in the logs. the string will be truncated on the right if
              it exceeds <length>.

  the complete value of the last occurrence of the header is captured. the
  value will be added to the logs between braces ('{}'). if multiple headers
  are captured, they will be delimited by a vertical bar ('|') and will appear
  in the same order they were declared in the configuration. non-existent
  headers will be logged just as an empty string. common uses for request
  header captures include the "host" field in virtual hosting environments, the
  "content-length" when uploads are supported, "user-agent" to quickly
  differentiate between real users and robots, and "x-forwarded-for" in proxied
  environments to find where the request came from.

  note that when capturing headers such as "user-agent", some spaces may be
  logged, making the log analysis more difficult. thus be careful about what
  you log if you know your log parser is not smart enough to rely on the
  braces.

  there is no limit to the number of captured request headers nor to their
  length, though it is wise to keep them low to limit memory usage per session.
  in order to keep log format consistent for a same frontend, header captures
  can only be declared in a frontend. it is not possible to specify a capture
  in a "defaults" section.

  example:
        capture request header host len 15
        capture request header x-forwarded-for len 15
        capture request header referrer len 15

  see also : "capture cookie", "capture response header" as well as section 8
             about logging.

capture response header <name> len <length>
  capture and log the last occurrence of the specified response header.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   no
  arguments :
    <name>    is the name of the header to capture. the header names are not
              case-sensitive, but it is a common practice to write them as they
              appear in the response, with the first letter of each word in
              upper case. the header name will not appear in the logs, only the
              value is reported, but the position in the logs is respected.

    <length>  is the maximum number of characters to extract from the value and
              report in the logs. the string will be truncated on the right if
              it exceeds <length>.

  the complete value of the last occurrence of the header is captured. the
  result will be added to the logs between braces ('{}') after the captured
  request headers. if multiple headers are captured, they will be delimited by
  a vertical bar ('|') and will appear in the same order they were declared in
  the configuration. non-existent headers will be logged just as an empty
  string. common uses for response header captures include the "content-length"
  header which indicates how many bytes are expected to be returned, the
  "location" header to track redirections.

  there is no limit to the number of captured response headers nor to their
  length, though it is wise to keep them low to limit memory usage per session.
  in order to keep log format consistent for a same frontend, header captures
  can only be declared in a frontend. it is not possible to specify a capture
  in a "defaults" section.

  example:
        capture response header content-length len 9
        capture response header location len 15

  see also : "capture cookie", "capture request header" as well as section 8
             about logging.

clitimeout <timeout> (deprecated)
  set the maximum inactivity time on the client side.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <timeout> is the timeout value is specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the inactivity timeout applies when the client is expected to acknowledge or
  send data. in http mode, this timeout is particularly important to consider
  during the first phase, when the client sends the request, and during the
  response while it is reading data sent by the server. the value is specified
  in milliseconds by default, but can be in any other unit if the number is
  suffixed by the unit, as specified at the top of this document. in tcp mode
  (and to a lesser extent, in http mode), it is highly recommended that the
  client timeout remains equal to the server timeout in order to avoid complex
  situations to debug. it is a good practice to cover one or several tcp packet
  losses by specifying timeouts that are slightly above multiples of 3 seconds
  (eg: 4 or 5 seconds).

  this parameter is specific to frontends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it. an unspecified timeout results in an infinite timeout, which
  is not recommended. such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of expired sessions in
  the system if the system's timeouts are not configured either.

  this parameter is provided for compatibility but is currently deprecated.
  please use "timeout client" instead.

  see also : "timeout client", "timeout http-request", "timeout server", and
             "srvtimeout".

compression algo <algorithm> ...
compression type <mime type> ...
compression offload
  enable http compression.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    algo     is followed by the list of supported compression algorithms.
    type     is followed by the list of mime types that will be compressed.
    offload  makes haproxy work as a compression offloader only (see notes).

  the currently supported algorithms are :
    identity     this is mostly for debugging, and it was useful for developing
                 the compression feature. identity does not apply any change on
                 data.

    gzip         applies gzip compression. this setting is only available when
                 support for zlib was built in.

    deflate      same as "gzip", but with deflate algorithm and zlib format.
                 note that this algorithm has ambiguous support on many
                 browsers and no support at all from recent ones. it is
                 strongly recommended not to use it for anything else than
                 experimentation. this setting is only available when support
                 for zlib was built in.

    raw-deflate  same as "deflate" without the zlib wrapper, and used as an
                 alternative when the browser wants "deflate". all major
                 browsers understand it and despite violating the standards,
                 it is known to work better than "deflate", at least on msie
                 and some versions of safari. do not use it in conjunction
                 with "deflate", use either one or the other since both react
                 to the same accept-encoding token. this setting is only
                 available when support for zlib was built in.

  compression will be activated depending on the accept-encoding request
  header. with identity, it does not take care of that header.
  if backend servers support http compression, these directives
  will be no-op: haproxy will see the compressed response and will not
  compress again. if backend servers do not support http compression and
  there is accept-encoding header in request, haproxy will compress the
  matching response.

  the "offload" setting makes haproxy remove the accept-encoding header to
  prevent backend servers from compressing responses. it is strongly
  recommended not to do this because this means that all the compression work
  will be done on the single point where haproxy is located. however in some
  deployment scenarios, haproxy may be installed in front of a buggy gateway
  with broken http compression implementation which can't be turned off.
  in that case haproxy can be used to prevent that gateway from emitting
  invalid payloads. in this case, simply removing the header in the
  configuration does not work because it applies before the header is parsed,
  so that prevents haproxy from compressing. the "offload" setting should
  then be used for such scenarios. note: for now, the "offload" setting is
  ignored when set in a defaults section.

  compression is disabled when:
    * the request does not advertise a supported compression algorithm in the
      "accept-encoding" header
    * the response message is not http/1.1
    * http status code is not 200
    * response header "transfer-encoding" contains "chunked" (temporary
      workaround)
    * response contain neither a "content-length" header nor a
      "transfer-encoding" whose last value is "chunked"
    * response contains a "content-type" header whose first value starts with
      "multipart"
    * the response contains the "no-transform" value in the "cache-control"
      header
    * user-agent matches "mozilla/4" unless it is msie 6 with xp sp2, or msie 7
      and later
    * the response contains a "content-encoding" header, indicating that the
      response is already compressed (see compression offload)

  note: the compression does not rewrite etag headers, and does not emit the
        warning header.

  examples :
        compression algo gzip
        compression type text/html text/plain

contimeout <timeout> (deprecated)
  set the maximum time to wait for a connection attempt to a server to succeed.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value is specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  if the server is located on the same lan as haproxy, the connection should be
  immediate (less than a few milliseconds). anyway, it is a good practice to
  cover one or several tcp packet losses by specifying timeouts that are
  slightly above multiples of 3 seconds (eg: 4 or 5 seconds). by default, the
  connect timeout also presets the queue timeout to the same value if this one
  has not been specified. historically, the contimeout was also used to set the
  tarpit timeout in a listen section, which is not possible in a pure frontend.

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it. an unspecified timeout results in an infinite timeout, which
  is not recommended. such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of failed sessions in
  the system if the system's timeouts are not configured either.

  this parameter is provided for backwards compatibility but is currently
  deprecated. please use "timeout connect", "timeout queue" or "timeout tarpit"
  instead.

  see also : "timeout connect", "timeout queue", "timeout tarpit",
             "timeout server", "contimeout".

cookie <name> [ rewrite | insert | prefix ] [ indirect ] [ nocache ]
              [ postonly ] [ preserve ] [ httponly ] [ secure ]
              [ domain <domain> ]* [ maxidle <idle> ] [ maxlife <life> ]
  enable cookie-based persistence in a backend.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <name>    is the name of the cookie which will be monitored, modified or
              inserted in order to bring persistence. this cookie is sent to
              the client via a "set-cookie" header in the response, and is
              brought back by the client in a "cookie" header in all requests.
              special care should be taken to choose a name which does not
              conflict with any likely application cookie. also, if the same
              backends are subject to be used by the same clients (eg:
              http/https), care should be taken to use different cookie names
              between all backends if persistence between them is not desired.

    rewrite   this keyword indicates that the cookie will be provided by the
              server and that haproxy will have to modify its value to set the
              server's identifier in it. this mode is handy when the management
              of complex combinations of "set-cookie" and "cache-control"
              headers is left to the application. the application can then
              decide whether or not it is appropriate to emit a persistence
              cookie. since all responses should be monitored, this mode only
              works in http close mode. unless the application behaviour is
              very complex and/or broken, it is advised not to start with this
              mode for new deployments. this keyword is incompatible with
              "insert" and "prefix".

    insert    this keyword indicates that the persistence cookie will have to
              be inserted by haproxy in server responses if the client did not

              already have a cookie that would have permitted it to access this
              server. when used without the "preserve" option, if the server
              emits a cookie with the same name, it will be remove before
              processing.  for this reason, this mode can be used to upgrade
              existing configurations running in the "rewrite" mode. the cookie
              will only be a session cookie and will not be stored on the
              client's disk. by default, unless the "indirect" option is added,
              the server will see the cookies emitted by the client. due to
              caching effects, it is generally wise to add the "nocache" or
              "postonly" keywords (see below). the "insert" keyword is not
              compatible with "rewrite" and "prefix".

    prefix    this keyword indicates that instead of relying on a dedicated
              cookie for the persistence, an existing one will be completed.
              this may be needed in some specific environments where the client
              does not support more than one single cookie and the application
              already needs it. in this case, whenever the server sets a cookie
              named <name>, it will be prefixed with the server's identifier
              and a delimiter. the prefix will be removed from all client
              requests so that the server still finds the cookie it emitted.
              since all requests and responses are subject to being modified,
              this mode requires the http close mode. the "prefix" keyword is
              not compatible with "rewrite" and "insert". note: it is highly
              recommended not to use "indirect" with "prefix", otherwise server
              cookie updates would not be sent to clients.

    indirect  when this option is specified, no cookie will be emitted to a
              client which already has a valid one for the server which has
              processed the request. if the server sets such a cookie itself,
              it will be removed, unless the "preserve" option is also set. in
              "insert" mode, this will additionally remove cookies from the
              requests transmitted to the server, making the persistence
              mechanism totally transparent from an application point of view.
              note: it is highly recommended not to use "indirect" with
              "prefix", otherwise server cookie updates would not be sent to
              clients.

    nocache   this option is recommended in conjunction with the insert mode
              when there is a cache between the client and haproxy, as it
              ensures that a cacheable response will be tagged non-cacheable if
              a cookie needs to be inserted. this is important because if all
              persistence cookies are added on a cacheable home page for
              instance, then all customers will then fetch the page from an
              outer cache and will all share the same persistence cookie,
              leading to one server receiving much more traffic than others.
              see also the "insert" and "postonly" options.

    postonly  this option ensures that cookie insertion will only be performed
              on responses to post requests. it is an alternative to the
              "nocache" option, because post responses are not cacheable, so
              this ensures that the persistence cookie will never get cached.
              since most sites do not need any sort of persistence before the
              first post which generally is a login request, this is a very
              efficient method to optimize caching without risking to find a
              persistence cookie in the cache.
              see also the "insert" and "nocache" options.

    preserve  this option may only be used with "insert" and/or "indirect". it
              allows the server to emit the persistence cookie itself. in this
              case, if a cookie is found in the response, haproxy will leave it
              untouched. this is useful in order to end persistence after a
              logout request for instance. for this, the server just has to
              emit a cookie with an invalid value (eg: empty) or with a date in
              the past. by combining this mechanism with the "disable-on-404"
              check option, it is possible to perform a completely graceful
              shutdown because users will definitely leave the server after
              they logout.

    httponly  this option tells haproxy to add an "httponly" cookie attribute
              when a cookie is inserted. this attribute is used so that a
              user agent doesn't share the cookie with non-http components.
              please check rfc6265 for more information on this attribute.

    secure    this option tells haproxy to add a "secure" cookie attribute when
              a cookie is inserted. this attribute is used so that a user agent
              never emits this cookie over non-secure channels, which means
              that a cookie learned with this flag will be presented only over
              ssl/tls connections. please check rfc6265 for more information on
              this attribute.

    domain    this option allows to specify the domain at which a cookie is
              inserted. it requires exactly one parameter: a valid domain
              name. if the domain begins with a dot, the browser is allowed to
              use it for any host ending with that name. it is also possible to
              specify several domain names by invoking this option multiple
              times. some browsers might have small limits on the number of
              domains, so be careful when doing that. for the record, sending
              10 domains to msie 6 or firefox 2 works as expected.

    maxidle   this option allows inserted cookies to be ignored after some idle
              time. it only works with insert-mode cookies. when a cookie is
              sent to the client, the date this cookie was emitted is sent too.
              upon further presentations of this cookie, if the date is older
              than the delay indicated by the parameter (in seconds), it will
              be ignored. otherwise, it will be refreshed if needed when the
              response is sent to the client. this is particularly useful to
              prevent users who never close their browsers from remaining for
              too long on the same server (eg: after a farm size change). when
              this option is set and a cookie has no date, it is always
              accepted, but gets refreshed in the response. this maintains the
              ability for admins to access their sites. cookies that have a
              date in the future further than 24 hours are ignored. doing so
              lets admins fix timezone issues without risking kicking users off
              the site.

    maxlife   this option allows inserted cookies to be ignored after some life
              time, whether they're in use or not. it only works with insert
              mode cookies. when a cookie is first sent to the client, the date
              this cookie was emitted is sent too. upon further presentations
              of this cookie, if the date is older than the delay indicated by
              the parameter (in seconds), it will be ignored. if the cookie in
              the request has no date, it is accepted and a date will be set.
              cookies that have a date in the future further than 24 hours are
              ignored. doing so lets admins fix timezone issues without risking
              kicking users off the site. contrary to maxidle, this value is
              not refreshed, only the first visit date counts. both maxidle and
              maxlife may be used at the time. this is particularly useful to
              prevent users who never close their browsers from remaining for
              too long on the same server (eg: after a farm size change). this
              is stronger than the maxidle method in that it forces a
              redispatch after some absolute delay.

  there can be only one persistence cookie per http backend, and it can be
  declared in a defaults section. the value of the cookie will be the value
  indicated after the "cookie" keyword in a "server" statement. if no cookie
  is declared for a given server, the cookie is not set.

  examples :
        cookie jsessionid prefix
        cookie srv insert indirect nocache
        cookie srv insert postonly indirect
        cookie srv insert indirect nocache maxidle 30m maxlife 8h

  see also : "appsession", "balance source", "capture cookie", "server"
             and "ignore-persist".

declare capture [ request | response ] len <length>
  declares a capture slot.
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   no
  arguments:
    <length> is the length allowed for the capture.

  this declaration is only available in the frontend or listen section, but the
  reserved slot can be used in the backends. the "request" keyword allocates a
  capture slot for use in the request, and "response" allocates a capture slot
  for use in the response.

  see also: "capture-req", "capture-res" (sample converters),
            "http-request capture" and "http-response capture".

default-server [param*]
  change default options for a server in a backend
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments:
    <param*>  is a list of parameters for this server. the "default-server"
              keyword accepts an important number of options and has a complete
              section dedicated to it. please refer to section 5 for more
              details.

  example :
        default-server inter 1000 weight 13

  see also: "server" and section 5 about server options

default_backend <backend>
  specify the backend to use when no "use_backend" rule has been matched.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <backend> is the name of the backend to use.

  when doing content-switching between frontend and backends using the
  "use_backend" keyword, it is often useful to indicate which backend will be
  used when no rule has matched. it generally is the dynamic backend which
  will catch all undetermined requests.

  example :

        use_backend     dynamic  if  url_dyn
        use_backend     static   if  url_css url_img extension_img
        default_backend dynamic

  see also : "use_backend"

description <string>
  describe a listen, frontend or backend.
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments : string

  allows to add a sentence to describe the related object in the haproxy html
  stats page. the description will be printed on the right of the object name
  it describes.
  no need to backslash spaces in the <string> arguments.

disabled
  disable a proxy, frontend or backend.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  the "disabled" keyword is used to disable an instance, mainly in order to
  liberate a listening port or to temporarily disable a service. the instance
  will still be created and its configuration will be checked, but it will be
  created in the "stopped" state and will appear as such in the statistics. it
  will not receive any traffic nor will it send any health-checks or logs. it
  is possible to disable many instances at once by adding the "disabled"
  keyword in a "defaults" section.

  see also : "enabled"

dispatch <address>:<port>
  set a default server address
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes
  arguments :

    <address> is the ipv4 address of the default server. alternatively, a
              resolvable hostname is supported, but this name will be resolved
              during start-up.

    <ports>   is a mandatory port specification. all connections will be sent
              to this port, and it is not permitted to use port offsets as is
              possible with normal servers.

  the "dispatch" keyword designates a default server for use when no other
  server can take the connection. in the past it was used to forward non
  persistent connections to an auxiliary load balancer. due to its simple
  syntax, it has also been used for simple tcp relays. it is recommended not to
  use it for more clarity, and to use the "server" directive instead.

  see also : "server"

enabled
  enable a proxy, frontend or backend.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  the "enabled" keyword is used to explicitly enable an instance, when the
  defaults has been set to "disabled". this is very rarely used.

  see also : "disabled"

errorfile <code> <file>
  return a file contents instead of errors generated by haproxy
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <code>    is the http status code. currently, haproxy is capable of
              generating codes 200, 400, 403, 405, 408, 429, 500, 502, 503, and
              504.

    <file>    designates a file containing the full http response. it is
              recommended to follow the common practice of appending ".http" to
              the filename so that people do not confuse the response with html
              error pages, and to use absolute paths, since files are read
              before any chroot is performed.

  it is important to understand that this keyword is not meant to rewrite
  errors returned by the server, but errors detected and returned by haproxy.
  this is why the list of supported errors is limited to a small set.

  code 200 is emitted in response to requests matching a "monitor-uri" rule.

  the files are returned verbatim on the tcp socket. this allows any trick such
  as redirections to another url or site, as well as tricks to clean cookies,
  force enable or disable caching, etc... the package provides default error
  files returning the same contents as default errors.

  the files should not exceed the configured buffer size (bufsize), which
  generally is 8 or 16 kb, otherwise they will be truncated. it is also wise
  not to put any reference to local contents (eg: images) in order to avoid
  loops between the client and haproxy when all servers are down, causing an
  error to be returned instead of an image. for better http compliance, it is
  recommended that all header lines end with cr-lf and not lf alone.

  the files are read at the same time as the configuration and kept in memory.
  for this reason, the errors continue to be returned even when the process is
  chrooted, and no file change is considered while the process is running. a
  simple method for developing those files consists in associating them to the
  403 status code and interrogating a blocked url.

  see also : "errorloc", "errorloc302", "errorloc303"

  example :
        errorfile 400 /etc/haproxy/errorfiles/400badreq.http
        errorfile 408 /dev/null  # workaround chrome pre-connect bug
        errorfile 403 /etc/haproxy/errorfiles/403forbid.http
        errorfile 503 /etc/haproxy/errorfiles/503sorry.http

errorloc <code> <url>
errorloc302 <code> <url>
  return an http redirection to a url instead of errors generated by haproxy
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <code>    is the http status code. currently, haproxy is capable of
              generating codes 200, 400, 403, 408, 500, 502, 503, and 504.

    <url>     it is the exact contents of the "location" header. it may contain
              either a relative uri to an error page hosted on the same site,
              or an absolute uri designating an error page on another site.
              special care should be given to relative uris to avoid redirect
              loops if the uri itself may generate the same error (eg: 500).

  it is important to understand that this keyword is not meant to rewrite
  errors returned by the server, but errors detected and returned by haproxy.
  this is why the list of supported errors is limited to a small set.

  code 200 is emitted in response to requests matching a "monitor-uri" rule.

  note that both keyword return the http 302 status code, which tells the
  client to fetch the designated url using the same http method. this can be
  quite problematic in case of non-get methods such as post, because the url
  sent to the client might not be allowed for something other than get. to
  workaround this problem, please use "errorloc303" which send the http 303
  status code, indicating to the client that the url must be fetched with a get
  request.

  see also : "errorfile", "errorloc303"

errorloc303 <code> <url>
  return an http redirection to a url instead of errors generated by haproxy
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <code>    is the http status code. currently, haproxy is capable of
              generating codes 400, 403, 408, 500, 502, 503, and 504.

    <url>     it is the exact contents of the "location" header. it may contain
              either a relative uri to an error page hosted on the same site,
              or an absolute uri designating an error page on another site.
              special care should be given to relative uris to avoid redirect
              loops if the uri itself may generate the same error (eg: 500).

  it is important to understand that this keyword is not meant to rewrite
  errors returned by the server, but errors detected and returned by haproxy.
  this is why the list of supported errors is limited to a small set.

  code 200 is emitted in response to requests matching a "monitor-uri" rule.

  note that both keyword return the http 303 status code, which tells the
  client to fetch the designated url using the same http get method. this
  solves the usual problems associated with "errorloc" and the 302 code. it is
  possible that some very old browsers designed before http/1.1 do not support
  it, but no such problem has been reported till now.

  see also : "errorfile", "errorloc", "errorloc302"

email-alert from <emailaddr>
  declare the from email address to be used in both the envelope and header
  of email alerts.  this is the address that email alerts are sent from.
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  arguments :

    <emailaddr> is the from email address to use when sending email alerts

  also requires "email-alert mailers" and "email-alert to" to be set
  and if so sending email alerts is enabled for the proxy.

  see also : "email-alert level", "email-alert mailers",
             "email-alert myhostname", "email-alert to", section 3.6 about mailers.

email-alert level <level>
  declare the maximum log level of messages for which email alerts will be
  sent. this acts as a filter on the sending of email alerts.
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  arguments :

    <level> one of the 8 syslog levels:
              emerg alert crit err warning notice info  debug
            the above syslog levels are ordered from lowest to highest.

  by default level is alert

  also requires "email-alert from", "email-alert mailers" and
  "email-alert to" to be set and if so sending email alerts is enabled
  for the proxy.

  alerts are sent when :

  * an un-paused server is marked as down and <level> is alert or lower
  * a paused server is marked as down and <level> is notice or lower
  * a server is marked as up or enters the drain state and <level>
    is notice or lower
  * "option log-health-checks" is enabled, <level> is info or lower,
     and a health check status update occurs

  see also : "email-alert from", "email-alert mailers",
             "email-alert myhostname", "email-alert to",
             section 3.6 about mailers.

email-alert mailers <mailersect>
  declare the mailers to be used when sending email alerts
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  arguments :

    <mailersect> is the name of the mailers section to send email alerts.

  also requires "email-alert from" and "email-alert to" to be set
  and if so sending email alerts is enabled for the proxy.

  see also : "email-alert from", "email-alert level", "email-alert myhostname",
             "email-alert to", section 3.6 about mailers.

email-alert myhostname <hostname>
  declare the to hostname address to be used when communicating with
  mailers.
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  arguments :

    <emailaddr> is the to email address to use when sending email alerts

  by default the systems hostname is used.

  also requires "email-alert from", "email-alert mailers" and
  "email-alert to" to be set and if so sending email alerts is enabled
  for the proxy.

  see also : "email-alert from", "email-alert level", "email-alert mailers",
             "email-alert to", section 3.6 about mailers.

email-alert to <emailaddr>
  declare both the recipent address in the envelope and to address in the
  header of email alerts. this is the address that email alerts are sent to.
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  arguments :

    <emailaddr> is the to email address to use when sending email alerts

  also requires "email-alert mailers" and "email-alert to" to be set
  and if so sending email alerts is enabled for the proxy.

  see also : "email-alert from", "email-alert level", "email-alert mailers",
             "email-alert myhostname", section 3.6 about mailers.

force-persist { if | unless } <condition>
  declare a condition to force persistence on down servers
  may be used in sections:    defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   yes

  by default, requests are not dispatched to down servers. it is possible to
  force this using "option persist", but it is unconditional and redispatches
  to a valid server if "option redispatch" is set. that leaves with very little
  possibilities to force some requests to reach a server which is artificially
  marked down for maintenance operations.

  the "force-persist" statement allows one to declare various acl-based
  conditions which, when met, will cause a request to ignore the down status of
  a server and still try to connect to it. that makes it possible to start a
  server, still replying an error to the health checks, and run a specially
  configured browser to test the service. among the handy methods, one could
  use a specific source ip address, or a specific cookie. the cookie also has
  the advantage that it can easily be added/removed on the browser from a test
  page. once the service is validated, it is then possible to open the service
  to the world by returning a valid response to health checks.

  the forced persistence is enabled when an "if" condition is met, or unless an
  "unless" condition is met. the final redispatch is always disabled when this
  is used.

  see also : "option redispatch", "ignore-persist", "persist",
             and section 7 about acl usage.

fullconn <conns>
  specify at what backend load the servers will reach their maxconn
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <conns>   is the number of connections on the backend which will make the
              servers use the maximal number of connections.

  when a server has a "maxconn" parameter specified, it means that its number
  of concurrent connections will never go higher. additionally, if it has a
  "minconn" parameter, it indicates a dynamic limit following the backend's
  load. the server will then always accept at least <minconn> connections,
  never more than <maxconn>, and the limit will be on the ramp between both
  values when the backend has less than <conns> concurrent connections. this
  makes it possible to limit the load on the servers during normal loads, but
  push it further for important loads without overloading the servers during
  exceptional loads.

  since it's hard to get this value right, haproxy automatically sets it to
  10% of the sum of the maxconns of all frontends that may branch to this
  backend (based on "use_backend" and "default_backend" rules). that way it's
  safe to leave it unset. however, "use_backend" involving dynamic names are
  not counted since there is no way to know if they could match or not.

  example :
     # the servers will accept between 100 and 1000 concurrent connections each
     # and the maximum of 1000 will be reached when the backend reaches 10000
     # connections.
     backend dynamic
        fullconn   10000
        server     srv1   dyn1:80 minconn 100 maxconn 1000
        server     srv2   dyn2:80 minconn 100 maxconn 1000

  see also : "maxconn", "server"

grace <time>
  maintain a proxy operational for some time after a soft stop
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <time>    is the time (by default in milliseconds) for which the instance
              will remain operational with the frontend sockets still listening
              when a soft-stop is received via the sigusr1 signal.

  this may be used to ensure that the services disappear in a certain order.
  this was designed so that frontends which are dedicated to monitoring by an
  external equipment fail immediately while other ones remain up for the time
  needed by the equipment to detect the failure.

  note that currently, there is very little benefit in using this parameter,
  and it may in fact complicate the soft-reconfiguration process more than
  simplify it.

hash-type <method> <function> <modifier>
  specify a method to use for mapping hashes to servers
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <method> is the method used to select a server from the hash computed by
             the <function> :

      map-based   the hash table is a static array containing all alive servers.
                  the hashes will be very smooth, will consider weights, but
                  will be static in that weight changes while a server is up
                  will be ignored. this means that there will be no slow start.
                  also, since a server is selected by its position in the array,
                  most mappings are changed when the server count changes. this
                  means that when a server goes up or down, or when a server is
                  added to a farm, most connections will be redistributed to
                  different servers. this can be inconvenient with caches for
                  instance.

      consistent  the hash table is a tree filled with many occurrences of each
                  server. the hash key is looked up in the tree and the closest
                  server is chosen. this hash is dynamic, it supports changing
                  weights while the servers are up, so it is compatible with the
                  slow start feature. it has the advantage that when a server
                  goes up or down, only its associations are moved. when a
                  server is added to the farm, only a few part of the mappings
                  are redistributed, making it an ideal method for caches.
                  however, due to its principle, the distribution will never be
                  very smooth and it may sometimes be necessary to adjust a
                  server's weight or its id to get a more balanced distribution.
                  in order to get the same distribution on multiple load
                  balancers, it is important that all servers have the exact
                  same ids. note: consistent hash uses sdbm and avalanche if no
                  hash function is specified.

    <function> is the hash function to be used :

       sdbm   this function was created initially for sdbm (a public-domain
              reimplementation of ndbm) database library. it was found to do
              well in scrambling bits, causing better distribution of the keys
              and fewer splits. it also happens to be a good general hashing
              function with good distribution, unless the total server weight
              is a multiple of 64, in which case applying the avalanche
              modifier may help.

       djb2   this function was first proposed by dan bernstein many years ago
              on comp.lang.c. studies have shown that for certain workload this
              function provides a better distribution than sdbm. it generally
              works well with text-based inputs though it can perform extremely
              poorly with numeric-only input or when the total server weight is
              a multiple of 33, unless the avalanche modifier is also used.

       wt6    this function was designed for haproxy while testing other
              functions in the past. it is not as smooth as the other ones, but
              is much less sensible to the input data set or to the number of
              servers. it can make sense as an alternative to sdbm+avalanche or
              djb2+avalanche for consistent hashing or when hashing on numeric
              data such as a source ip address or a visitor identifier in a url
              parameter.

       crc32  this is the most common crc32 implementation as used in ethernet,
              gzip, png, etc. it is slower than the other ones but may provide
              a better distribution or less predictable results especially when
              used on strings.

    <modifier> indicates an optional method applied after hashing the key :

       avalanche   this directive indicates that the result from the hash
                   function above should not be used in its raw form but that
                   a 4-byte full avalanche hash must be applied first. the
                   purpose of this step is to mix the resulting bits from the
                   previous hash in order to avoid any undesired effect when
                   the input contains some limited values or when the number of
                   servers is a multiple of one of the hash's components (64
                   for sdbm, 33 for djb2). enabling avalanche tends to make the
                   result less predictable, but it's also not as smooth as when
                   using the original function. some testing might be needed
                   with some workloads. this hash is one of the many proposed
                   by bob jenkins.

  the default hash type is "map-based" and is recommended for most usages. the
  default function is "sdbm", the selection of a function should be based on
  the range of the values being hashed.

  see also : "balance", "server"

http-check disable-on-404
  enable a maintenance mode upon http/404 response to health-checks
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  when this option is set, a server which returns an http code 404 will be
  excluded from further load-balancing, but will still receive persistent
  connections. this provides a very convenient method for web administrators
  to perform a graceful shutdown of their servers. it is also important to note
  that a server which is detected as failed while it was in this mode will not
  generate an alert, just a notice. if the server responds 2xx or 3xx again, it
  will immediately be reinserted into the farm. the status on the stats page
  reports "nolb" for a server in this mode. it is important to note that this
  option only works in conjunction with the "httpchk" option. if this option
  is used with "http-check expect", then it has precedence over it so that 404
  responses will still be considered as soft-stop.

  see also : "option httpchk", "http-check expect"

http-check expect [!] <match> <pattern>
  make http health checks consider response contents or specific status codes
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <match>   is a keyword indicating how to look for a specific pattern in the
              response. the keyword may be one of "status", "rstatus",
              "string", or "rstring". the keyword may be preceded by an
              exclamation mark ("!") to negate the match. spaces are allowed
              between the exclamation mark and the keyword. see below for more
              details on the supported keywords.

    <pattern> is the pattern to look for. it may be a string or a regular
              expression. if the pattern contains spaces, they must be escaped
              with the usual backslash ('\').

  by default, "option httpchk" considers that response statuses 2xx and 3xx
  are valid, and that others are invalid. when "http-check expect" is used,
  it defines what is considered valid or invalid. only one "http-check"
  statement is supported in a backend. if a server fails to respond or times
  out, the check obviously fails. the available matches are :

    status <string> : test the exact string match for the http status code.
                      a health check response will be considered valid if the
                      response's status code is exactly this string. if the
                      "status" keyword is prefixed with "!", then the response
                      will be considered invalid if the status code matches.

    rstatus <regex> : test a regular expression for the http status code.
                      a health check response will be considered valid if the
                      response's status code matches the expression. if the
                      "rstatus" keyword is prefixed with "!", then the response
                      will be considered invalid if the status code matches.
                      this is mostly used to check for multiple codes.

    string <string> : test the exact string match in the http response body.
                      a health check response will be considered valid if the
                      response's body contains this exact string. if the
                      "string" keyword is prefixed with "!", then the response
                      will be considered invalid if the body contains this
                      string. this can be used to look for a mandatory word at
                      the end of a dynamic page, or to detect a failure when a
                      specific error appears on the check page (eg: a stack
                      trace).

    rstring <regex> : test a regular expression on the http response body.
                      a health check response will be considered valid if the
                      response's body matches this expression. if the "rstring"
                      keyword is prefixed with "!", then the response will be
                      considered invalid if the body matches the expression.
                      this can be used to look for a mandatory word at the end
                      of a dynamic page, or to detect a failure when a specific
                      error appears on the check page (eg: a stack trace).

  it is important to note that the responses will be limited to a certain size
  defined by the global "tune.chksize" option, which defaults to 16384 bytes.
  thus, too large responses may not contain the mandatory pattern when using
  "string" or "rstring". if a large response is absolutely required, it is
  possible to change the default max size by setting the global variable.
  however, it is worth keeping in mind that parsing very large responses can
  waste some cpu cycles, especially when regular expressions are used, and that
  it is always better to focus the checks on smaller resources.

  also "http-check expect" doesn't support http keep-alive. keep in mind that it
  will automatically append a "connection: close" header, meaning that this
  header should not be present in the request provided by "option httpchk".

  last, if "http-check expect" is combined with "http-check disable-on-404",
  then this last one has precedence when the server responds with 404.

  examples :
         # only accept status 200 as valid
         http-check expect status 200

         # consider sql errors as errors
         http-check expect ! string sql\ error

         # consider status 5xx only as errors
         http-check expect ! rstatus ^5

         # check that we have a correct hexadecimal tag before /html
         http-check expect rstring <!--tag:[0-9a-f]*</html>

  see also : "option httpchk", "http-check disable-on-404"

http-check send-state
  enable emission of a state header with http health checks
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  when this option is set, haproxy will systematically send a special header
  "x-haproxy-server-state" with a list of parameters indicating to each server
  how they are seen by haproxy. this can be used for instance when a server is
  manipulated without access to haproxy and the operator needs to know whether
  haproxy still sees it up or not, or if the server is the last one in a farm.

  the header is composed of fields delimited by semi-colons, the first of which
  is a word ("up", "down", "nolb"), possibly followed by a number of valid
  checks on the total number before transition, just as appears in the stats
  interface. next headers are in the form "<variable>=<value>", indicating in
  no specific order some values available in the stats interface :
    - a variable "address", containing the address of the backend server.
      this corresponds to the <address> field in the server declaration. for
      unix domain sockets, it will read "unix".

    - a variable "port", containing the port of the backend server. this
      corresponds to the <port> field in the server declaration. for unix
      domain sockets, it will read "unix".

    - a variable "name", containing the name of the backend followed by a slash
      ("/") then the name of the server. this can be used when a server is
      checked in multiple backends.

    - a variable "node" containing the name of the haproxy node, as set in the
      global "node" variable, otherwise the system's hostname if unspecified.

    - a variable "weight" indicating the weight of the server, a slash ("/")
      and the total weight of the farm (just counting usable servers). this
      helps to know if other servers are available to handle the load when this
      one fails.

    - a variable "scur" indicating the current number of concurrent connections
      on the server, followed by a slash ("/") then the total number of
      connections on all servers of the same backend.

    - a variable "qcur" indicating the current number of requests in the
      server's queue.

  example of a header received by the application server :
    >>>  x-haproxy-server-state: up 2/3; name=bck/srv2; node=lb1; weight=1/2; \
           scur=13/22; qcur=0

  see also : "option httpchk", "http-check disable-on-404"

http-request { allow | deny | tarpit | auth [realm <realm>] | redirect <rule> |
              add-header <name> <fmt> | set-header <name> <fmt> |
              capture <sample> [ len <length> | id <id> ] |
              del-header <name> | set-nice <nice> | set-log-level <level> |
              replace-header <name> <match-regex> <replace-fmt> |
              replace-value <name> <match-regex> <replace-fmt> |
              set-method <fmt> | set-path <fmt> | set-query <fmt> |
              set-uri <fmt> | set-tos <tos> | set-mark <mark> |
              add-acl(<file name>) <key fmt> |
              del-acl(<file name>) <key fmt> |
              del-map(<file name>) <key fmt> |
              set-map(<file name>) <key fmt> <value fmt> |
              set-var(<var name>) <expr> |
              { track-sc0 | track-sc1 | track-sc2 } <key> [table <table>] |
              lua <function name>
             }
             [ { if | unless } <condition> ]
  access control for layer 7 requests

  may be used in sections:   defaults | frontend | listen | backend
                                no    |    yes   |   yes  |   yes

  the http-request statement defines a set of rules which apply to layer 7
  processing. the rules are evaluated in their declaration order when they are
  met in a frontend, listen or backend section. any rule may optionally be
  followed by an acl-based condition, in which case it will only be evaluated
  if the condition is true.

  the first keyword is the rule's action. currently supported actions include :
    - "allow" : this stops the evaluation of the rules and lets the request
      pass the check. no further "http-request" rules are evaluated.

    - "deny" : this stops the evaluation of the rules and immediately rejects
      the request and emits an http 403 error. no further "http-request" rules
      are evaluated.

    - "tarpit" : this stops the evaluation of the rules and immediately blocks
      the request without responding for a delay specified by "timeout tarpit"
      or "timeout connect" if the former is not set. after that delay, if the
      client is still connected, an http error 500 is returned so that the
      client does not suspect it has been tarpitted. logs will report the flags
      "pt". the goal of the tarpit rule is to slow down robots during an attack
      when they're limited on the number of concurrent requests. it can be very
      efficient against very dumb robots, and will significantly reduce the
      load on firewalls compared to a "deny" rule. but when facing "correctly"
      developed robots, it can make things worse by forcing haproxy and the
      front firewall to support insane number of concurrent connections.

    - "auth" : this stops the evaluation of the rules and immediately responds
      with an http 401 or 407 error code to invite the user to present a valid
      user name and password. no further "http-request" rules are evaluated. an
      optional "realm" parameter is supported, it sets the authentication realm
      that is returned with the response (typically the application's name).

    - "redirect" : this performs an http redirection based on a redirect rule.
      this is exactly the same as the "redirect" statement except that it
      inserts a redirect rule which can be processed in the middle of other
      "http-request" rules and that these rules use the "log-format" strings.
      see the "redirect" keyword for the rule's syntax.

    - "add-header" appends an http header field whose name is specified in
      <name> and whose value is defined by <fmt> which follows the log-format
      rules (see custom log format in section 8.2.4). this is particularly
      useful to pass connection-specific information to the server (eg: the
      client's ssl certificate), or to combine several headers into one. this
      rule is not final, so it is possible to add other similar rules. note
      that header addition is performed immediately, so one rule might reuse
      the resulting header from a previous rule.

    - "set-header" does the same as "add-header" except that the header name
      is first removed if it existed. this is useful when passing security
      information to the server, where the header must not be manipulated by
      external users. note that the new value is computed before the removal so
      it is possible to concatenate a value to an existing header.

    - "del-header" removes all http header fields whose name is specified in
      <name>.

    - "replace-header" matches the regular expression in all occurrences of
      header field <name> according to <match-regex>, and replaces them with
      the <replace-fmt> argument. format characters are allowed in replace-fmt
      and work like in <fmt> arguments in "add-header". the match is only
      case-sensitive. it is important to understand that this action only
      considers whole header lines, regardless of the number of values they
      may contain. this usage is suited to headers naturally containing commas
      in their value, such as if-modified-since and so on.

      example:

        http-request replace-header cookie foo=([^;]*);(.*) foo=\1;ip=%bi;\2

      applied to:

        cookie: foo=foobar; expires=tue, 14-jun-2016 01:40:45 gmt;

      outputs:

        cookie: foo=foobar;ip=192.168.1.20; expires=tue, 14-jun-2016 01:40:45 gmt;

      assuming the backend ip is 192.168.1.20

    - "replace-value" works like "replace-header" except that it matches the
      regex against every comma-delimited value of the header field <name>
      instead of the entire header. this is suited for all headers which are
      allowed to carry more than one value. an example could be the accept
      header.

      example:

        http-request replace-value x-forwarded-for ^192\.168\.(.*)$ 172.16.\1

      applied to:

        x-forwarded-for: 192.168.10.1, 192.168.13.24, 10.0.0.37

      outputs:

        x-forwarded-for: 172.16.10.1, 172.16.13.24, 10.0.0.37

    - "set-method" rewrites the request method with the result of the
      evaluation of format string <fmt>. there should be very few valid reasons
      for having to do so as this is more likely to break something than to fix
      it.

    - "set-path" rewrites the request path with the result of the evaluation of
      format string <fmt>. the query string, if any, is left intact. if a
      scheme and authority is found before the path, they are left intact as
      well. if the request doesn't have a path ("*"), this one is replaced with
      the format. this can be used to prepend a directory component in front of
      a path for example. see also "set-query" and "set-uri".

      example :
          # prepend the host name before the path
          http-request set-path /%[hdr(host)]%[path]

    - "set-query" rewrites the request's query string which appears after the
      first question mark ("?") with the result of the evaluation of format
      string <fmt>. the part prior to the question mark is left intact. if the
      request doesn't contain a question mark and the new value is not empty,
      then one is added at the end of the uri, followed by the new value. if
      a question mark was present, it will never be removed even if the value
      is empty. this can be used to add or remove parameters from the query
      string. see also "set-query" and "set-uri".

      example :
          # replace "%3d" with "=" in the query string
          http-request set-query %[query,regsub(%3d,=,g)]

    - "set-uri" rewrites the request uri with the result of the evaluation of
      format string <fmt>. the scheme, authority, path and query string are all
      replaced at once. this can be used to rewrite hosts in front of proxies,
      or to perform complex modifications to the uri such as moving parts
      between the path and the query string. see also "set-path" and
      "set-query".

    - "set-nice" sets the "nice" factor of the current request being processed.
      it only has effect against the other requests being processed at the same
      time. the default value is 0, unless altered by the "nice" setting on the
      "bind" line. the accepted range is -1024..1024\. the higher the value, the
      nicest the request will be. lower values will make the request more
      important than other ones. this can be useful to improve the speed of
      some requests, or lower the priority of non-important requests. using
      this setting without prior experimentation can cause some major slowdown.

    - "set-log-level" is used to change the log level of the current request
      when a certain condition is met. valid levels are the 8 syslog levels
      (see the "log" keyword) plus the special level "silent" which disables
      logging for this request. this rule is not final so the last matching
      rule wins. this rule can be useful to disable health checks coming from
      another equipment.

    - "set-tos" is used to set the tos or dscp field value of packets sent to
      the client to the value passed in <tos> on platforms which support this.
      this value represents the whole 8 bits of the ip tos field, and can be
      expressed both in decimal or hexadecimal format (prefixed by "0x"). note
      that only the 6 higher bits are used in dscp or tos, and the two lower
      bits are always 0\. this can be used to adjust some routing behaviour on
      border routers based on some information from the request. see rfc 2474,
      2597, 3260 and 4594 for more information.

    - "set-mark" is used to set the netfilter mark on all packets sent to the
      client to the value passed in <mark> on platforms which support it. this
      value is an unsigned 32 bit value which can be matched by netfilter and
      by the routing table. it can be expressed both in decimal or hexadecimal
      format (prefixed by "0x"). this can be useful to force certain packets to
      take a different route (for example a cheaper network path for bulk
      downloads). this works on linux kernels 2.6.32 and above and requires
      admin privileges.

    - "add-acl" is used to add a new entry into an acl. the acl must be loaded
      from a file (even a dummy empty file). the file name of the acl to be
      updated is passed between parentheses. it takes one argument: <key fmt>,
      which follows log-format rules, to collect content of the new entry. it
      performs a lookup in the acl before insertion, to avoid duplicated (or
      more) values. this lookup is done by a linear search and can be expensive
      with large lists! it is the equivalent of the "add acl" command from the
      stats socket, but can be triggered by an http request.

    - "del-acl" is used to delete an entry from an acl. the acl must be loaded
      from a file (even a dummy empty file). the file name of the acl to be
      updated is passed between parentheses. it takes one argument: <key fmt>,
      which follows log-format rules, to collect content of the entry to delete.
      it is the equivalent of the "del acl" command from the stats socket, but
      can be triggered by an http request.

    - "del-map" is used to delete an entry from a map. the map must be loaded
      from a file (even a dummy empty file). the file name of the map to be
      updated is passed between parentheses. it takes one argument: <key fmt>,
      which follows log-format rules, to collect content of the entry to delete.
      it takes one argument: "file name" it is the equivalent of the "del map"
      command from the stats socket, but can be triggered by an http request.

    - "set-map" is used to add a new entry into a map. the map must be loaded
      from a file (even a dummy empty file). the file name of the map to be
      updated is passed between parentheses. it takes 2 arguments: <key fmt>,
      which follows log-format rules, used to collect map key, and <value fmt>,
      which follows log-format rules, used to collect content for the new entry.
      it performs a lookup in the map before insertion, to avoid duplicated (or
      more) values. this lookup is done by a linear search and can be expensive
      with large lists! it is the equivalent of the "set map" command from the
      stats socket, but can be triggered by an http request.

    - capture <sample> [ len <length> | id <id> ] :
      captures sample expression <sample> from the request buffer, and converts
      it to a string of at most <len> characters. the resulting string is
      stored into the next request "capture" slot, so it will possibly appear
      next to some captured http headers. it will then automatically appear in
      the logs, and it will be possible to extract it using sample fetch rules
      to feed it into headers or anything. the length should be limited given
      that this size will be allocated for each capture during the whole
      session life. please check section 7.3 (fetching samples) and "capture
      request header" for more information.

      if the keyword "id" is used instead of "len", the action tries to store
      the captured string in a previously declared capture slot. this is useful
      to run captures in backends. the slot id can be declared by a previous
      directive "http-request capture" or with the "declare capture" keyword.

    - { track-sc0 | track-sc1 | track-sc2 } <key> [table <table>] :
      enables tracking of sticky counters from current request. these rules
      do not stop evaluation and do not change default action. three sets of
      counters may be simultaneously tracked by the same connection. the first
      "track-sc0" rule executed enables tracking of the counters of the
      specified table as the first set. the first "track-sc1" rule executed
      enables tracking of the counters of the specified table as the second
      set. the first "track-sc2" rule executed enables tracking of the
      counters of the specified table as the third set. it is a recommended
      practice to use the first set of counters for the per-frontend counters
      and the second set for the per-backend ones. but this is just a
      guideline, all may be used everywhere.

      these actions take one or two arguments :
        <key>   is mandatory, and is a sample expression rule as described
                in section 7.3\. it describes what elements of the incoming
                request or connection will be analysed, extracted, combined,
                and used to select which table entry to update the counters.

        <table> is an optional table to be used instead of the default one,
                which is the stick-table declared in the current proxy. all
                the counters for the matches and updates for the key will
                then be performed in that table until the session ends.

      once a "track-sc*" rule is executed, the key is looked up in the table
      and if it is not found, an entry is allocated for it. then a pointer to
      that entry is kept during all the session's life, and this entry's
      counters are updated as often as possible, every time the session's
      counters are updated, and also systematically when the session ends.
      counters are only updated for events that happen after the tracking has
      been started. as an exception, connection counters and request counters
      are systematically updated so that they reflect useful information.

      if the entry tracks concurrent connection counters, one connection is
      counted for as long as the entry is tracked, and the entry will not
      expire during that time. tracking counters also provides a performance
      advantage over just checking the keys, because only one table lookup is
      performed for all acl checks that make use of it.

    - "lua" is used to run a lua function if the action is executed. the single
      parameter is the name of the function to run. the prototype of the
      function is documented in the api documentation.

    - set-var(<var-name>) <expr> :
      is used to set the contents of a variable. the variable is declared
      inline.

        <var-name> the name of the variable starts by an indication about its
                   scope. the allowed scopes are:
                     "sess" : the variable is shared with all the session,
                     "txn"  : the variable is shared with all the transaction
                              (request and response)
                     "req"  : the variable is shared only during the request
                              processing
                     "res"  : the variable is shared only during the response
                              processing.
                   this prefix is followed by a name. the separator is a '.'.
                   the name may only contain characters 'a-z', 'a-z', '0-9',
                   and '_'.

         <expr>    is a standard haproxy expression formed by a sample-fetch
                   followed by some converters.

      example:

         http-request set-var(req.my_var) req.fhdr(user-agent),lower

    - set-src <expr> :
      is used to set the source ip address to the value of specified
      expression. useful when a proxy in front of haproxy rewrites source ip,
      but provides the correct ip in a http header; or you want to mask
      source ip for privacy.

         <expr>    is a standard haproxy expression formed by a sample-fetch
                   followed by some converters.

      example:

         http-request set-src hdr(x-forwarded-for)
         http-request set-src src,ipmask(24)

      when set-src is successful, the source port is set to 0.

  there is no limit to the number of http-request statements per instance.

  it is important to know that http-request rules are processed very early in
  the http processing, just after "block" rules and before "reqdel" or "reqrep"
  rules. that way, headers added by "add-header"/"set-header" are visible by
  almost all further acl rules.

  example:
        acl nagios src 192.168.129.3
        acl local_net src 192.168.0.0/16
        acl auth_ok http_auth(l1)

        http-request allow if nagios
        http-request allow if local_net auth_ok
        http-request auth realm gimme if local_net auth_ok
        http-request deny

  example:
        acl auth_ok http_auth_group(l1) g1
        http-request auth unless auth_ok

  example:
        http-request set-header x-haproxy-current-date %t
        http-request set-header x-ssl                  %[ssl_fc]
        http-request set-header x-ssl-session_id       %[ssl_fc_session_id]
        http-request set-header x-ssl-client-verify    %[ssl_c_verify]
        http-request set-header x-ssl-client-dn        %{+q}[ssl_c_s_dn]
        http-request set-header x-ssl-client-cn        %{+q}[ssl_c_s_dn(cn)]
        http-request set-header x-ssl-issuer           %{+q}[ssl_c_i_dn]
        http-request set-header x-ssl-client-notbefore %{+q}[ssl_c_notbefore]
        http-request set-header x-ssl-client-notafter  %{+q}[ssl_c_notafter]

  example:
        acl key req.hdr(x-add-acl-key) -m found
        acl add path /addacl
        acl del path /delacl

        acl myhost hdr(host) -f myhost.lst

        http-request add-acl(myhost.lst) %[req.hdr(x-add-acl-key)] if key add
        http-request del-acl(myhost.lst) %[req.hdr(x-add-acl-key)] if key del

  example:
        acl value  req.hdr(x-value) -m found
        acl setmap path /setmap
        acl delmap path /delmap

        use_backend bk_appli if { hdr(host),map_str(map.lst) -m found }

        http-request set-map(map.lst) %[src] %[req.hdr(x-value)] if setmap value
        http-request del-map(map.lst) %[src]                     if delmap

  see also : "stats http-request", section 3.4 about userlists and section 7
             about acl usage.

http-response { allow | deny | add-header <name> <fmt> | set-nice <nice> |
                capture <sample> id <id> | redirect <rule> |
                set-header <name> <fmt> | del-header <name> |
                replace-header <name> <regex-match> <replace-fmt> |
                replace-value <name> <regex-match> <replace-fmt> |
                set-log-level <level> | set-mark <mark> | set-tos <tos> |
                add-acl(<file name>) <key fmt> |
                del-acl(<file name>) <key fmt> |
                del-map(<file name>) <key fmt> |
                set-map(<file name>) <key fmt> <value fmt> |
                set-var(<var-name>) <expr> |
                lua <function name>
              }
              [ { if | unless } <condition> ]
  access control for layer 7 responses

  may be used in sections:   defaults | frontend | listen | backend
                                no    |    yes   |   yes  |   yes

  the http-response statement defines a set of rules which apply to layer 7
  processing. the rules are evaluated in their declaration order when they are
  met in a frontend, listen or backend section. any rule may optionally be
  followed by an acl-based condition, in which case it will only be evaluated
  if the condition is true. since these rules apply on responses, the backend
  rules are applied first, followed by the frontend's rules.

  the first keyword is the rule's action. currently supported actions include :
    - "allow" : this stops the evaluation of the rules and lets the response
      pass the check. no further "http-response" rules are evaluated for the
      current section.

    - "deny" : this stops the evaluation of the rules and immediately rejects
      the response and emits an http 502 error. no further "http-response"
      rules are evaluated.

    - "add-header" appends an http header field whose name is specified in
      <name> and whose value is defined by <fmt> which follows the log-format
      rules (see custom log format in section 8.2.4). this may be used to send
      a cookie to a client for example, or to pass some internal information.
      this rule is not final, so it is possible to add other similar rules.
      note that header addition is performed immediately, so one rule might
      reuse the resulting header from a previous rule.

    - "set-header" does the same as "add-header" except that the header name
      is first removed if it existed. this is useful when passing security
      information to the server, where the header must not be manipulated by
      external users.

    - "del-header" removes all http header fields whose name is specified in
      <name>.

    - "replace-header" matches the regular expression in all occurrences of
      header field <name> according to <match-regex>, and replaces them with
      the <replace-fmt> argument. format characters are allowed in replace-fmt
      and work like in <fmt> arguments in "add-header". the match is only
      case-sensitive. it is important to understand that this action only
      considers whole header lines, regardless of the number of values they
      may contain. this usage is suited to headers naturally containing commas
      in their value, such as set-cookie, expires and so on.

      example:

        http-response replace-header set-cookie (c=[^;]*);(.*) \1;ip=%bi;\2

      applied to:

        set-cookie: c=1; expires=tue, 14-jun-2016 01:40:45 gmt

      outputs:

        set-cookie: c=1;ip=192.168.1.20; expires=tue, 14-jun-2016 01:40:45 gmt

      assuming the backend ip is 192.168.1.20.

    - "replace-value" works like "replace-header" except that it matches the
      regex against every comma-delimited value of the header field <name>
      instead of the entire header. this is suited for all headers which are
      allowed to carry more than one value. an example could be the accept
      header.

      example:

        http-response replace-value cache-control ^public$ private

      applied to:

        cache-control: max-age=3600, public

      outputs:

        cache-control: max-age=3600, private

    - "set-nice" sets the "nice" factor of the current request being processed.
      it only has effect against the other requests being processed at the same
      time. the default value is 0, unless altered by the "nice" setting on the
      "bind" line. the accepted range is -1024..1024\. the higher the value, the
      nicest the request will be. lower values will make the request more
      important than other ones. this can be useful to improve the speed of
      some requests, or lower the priority of non-important requests. using
      this setting without prior experimentation can cause some major slowdown.

    - "set-log-level" is used to change the log level of the current request
      when a certain condition is met. valid levels are the 8 syslog levels
      (see the "log" keyword) plus the special level "silent" which disables
      logging for this request. this rule is not final so the last matching
      rule wins. this rule can be useful to disable health checks coming from
      another equipment.

    - "set-tos" is used to set the tos or dscp field value of packets sent to
      the client to the value passed in <tos> on platforms which support this.
      this value represents the whole 8 bits of the ip tos field, and can be
      expressed both in decimal or hexadecimal format (prefixed by "0x"). note
      that only the 6 higher bits are used in dscp or tos, and the two lower
      bits are always 0\. this can be used to adjust some routing behaviour on
      border routers based on some information from the request. see rfc 2474,
      2597, 3260 and 4594 for more information.

    - "set-mark" is used to set the netfilter mark on all packets sent to the
      client to the value passed in <mark> on platforms which support it. this
      value is an unsigned 32 bit value which can be matched by netfilter and
      by the routing table. it can be expressed both in decimal or hexadecimal
      format (prefixed by "0x"). this can be useful to force certain packets to
      take a different route (for example a cheaper network path for bulk
      downloads). this works on linux kernels 2.6.32 and above and requires
      admin privileges.

    - "add-acl" is used to add a new entry into an acl. the acl must be loaded
      from a file (even a dummy empty file). the file name of the acl to be
      updated is passed between parentheses. it takes one argument: <key fmt>,
      which follows log-format rules, to collect content of the new entry. it
      performs a lookup in the acl before insertion, to avoid duplicated (or
      more) values. this lookup is done by a linear search and can be expensive
      with large lists! it is the equivalent of the "add acl" command from the
      stats socket, but can be triggered by an http response.

    - "del-acl" is used to delete an entry from an acl. the acl must be loaded
      from a file (even a dummy empty file). the file name of the acl to be
      updated is passed between parentheses. it takes one argument: <key fmt>,
      which follows log-format rules, to collect content of the entry to delete.
      it is the equivalent of the "del acl" command from the stats socket, but
      can be triggered by an http response.

    - "del-map" is used to delete an entry from a map. the map must be loaded
      from a file (even a dummy empty file). the file name of the map to be
      updated is passed between parentheses. it takes one argument: <key fmt>,
      which follows log-format rules, to collect content of the entry to delete.
      it takes one argument: "file name" it is the equivalent of the "del map"
      command from the stats socket, but can be triggered by an http response.

    - "set-map" is used to add a new entry into a map. the map must be loaded
      from a file (even a dummy empty file). the file name of the map to be
      updated is passed between parentheses. it takes 2 arguments: <key fmt>,
      which follows log-format rules, used to collect map key, and <value fmt>,
      which follows log-format rules, used to collect content for the new entry.
      it performs a lookup in the map before insertion, to avoid duplicated (or
      more) values. this lookup is done by a linear search and can be expensive
      with large lists! it is the equivalent of the "set map" command from the
      stats socket, but can be triggered by an http response.

    - "lua" is used to run a lua function if the action is executed. the single
      parameter is the name of the function to run. the prototype of the
      function is documented in the api documentation.

    - capture <sample> id <id> :
      captures sample expression <sample> from the response buffer, and converts
      it to a string. the resulting string is stored into the next request
      "capture" slot, so it will possibly appear next to some captured http
      headers. it will then automatically appear in the logs, and it will be
      possible to extract it using sample fetch rules to feed it into headers or
      anything. please check section 7.3 (fetching samples) and "capture
      response header" for more information.

      the keyword "id" is the id of the capture slot which is used for storing
      the string. the capture slot must be defined in an associated frontend.
      this is useful to run captures in backends. the slot id can be declared by
      a previous directive "http-response capture" or with the "declare capture"
      keyword.

    - "redirect" : this performs an http redirection based on a redirect rule.
      this supports a format string similarly to "http-request redirect" rules,
      with the exception that only the "location" type of redirect is possible
      on the response. see the "redirect" keyword for the rule's syntax. when
      a redirect rule is applied during a response, connections to the server
      are closed so that no data can be forwarded from the server to the client.

    - set-var(<var-name>) expr:
      is used to set the contents of a variable. the variable is declared
      inline.

        <var-name> the name of the variable starts by an indication about its
                   scope. the allowed scopes are:
                     "sess" : the variable is shared with all the session,
                     "txn"  : the variable is shared with all the transaction
                              (request and response)
                     "req"  : the variable is shared only during the request
                              processing
                     "res"  : the variable is shared only during the response
                              processing.
                   this prefix is followed by a name. the separator is a '.'.
                   the name may only contain characters 'a-z', 'a-z', '0-9',
                   and '_'.

         <expr>    is a standard haproxy expression formed by a sample-fetch
                   followed by some converters.

      example:

         http-response set-var(sess.last_redir) res.hdr(location)

  there is no limit to the number of http-response statements per instance.

  it is important to know that http-response rules are processed very early in
  the http processing, before "reqdel" or "reqrep" rules. that way, headers
  added by "add-header"/"set-header" are visible by almost all further acl
  rules.

  example:
         acl key_acl res.hdr(x-acl-key) -m found

         acl myhost hdr(host) -f myhost.lst

         http-response add-acl(myhost.lst) %[res.hdr(x-acl-key)] if key_acl
         http-response del-acl(myhost.lst) %[res.hdr(x-acl-key)] if key_acl

  example:
         acl value  res.hdr(x-value) -m found

         use_backend bk_appli if { hdr(host),map_str(map.lst) -m found }

         http-response set-map(map.lst) %[src] %[res.hdr(x-value)] if value
         http-response del-map(map.lst) %[src]                     if ! value

  see also : "http-request", section 3.4 about userlists and section 7 about
             acl usage.

http-send-name-header [<header>]
  add the server name to a request. use the header string given by <header>

  may be used in sections:   defaults | frontend | listen | backend
                               yes    |    no    |   yes  |   yes

  arguments :

    <header>  the header string to use to send the server name

  the "http-send-name-header" statement causes the name of the target
  server to be added to the headers of an http request.  the name
  is added with the header string proved.

  see also : "server"

id <value>
  set a persistent id to a proxy.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   yes
  arguments : none

  set a persistent id for the proxy. this id must be unique and positive.
  an unused id will automatically be assigned if unset. the first assigned
  value will be 1\. this id is currently only returned in statistics.

ignore-persist { if | unless } <condition>
  declare a condition to ignore persistence
  may be used in sections:    defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   yes

  by default, when cookie persistence is enabled, every requests containing
  the cookie are unconditionally persistent (assuming the target server is up
  and running).

  the "ignore-persist" statement allows one to declare various acl-based
  conditions which, when met, will cause a request to ignore persistence.
  this is sometimes useful to load balance requests for static files, which
  often don't require persistence. this can also be used to fully disable
  persistence for a specific user-agent (for example, some web crawler bots).

  combined with "appsession", it can also help reduce haproxy memory usage, as
  the appsession table won't grow if persistence is ignored.

  the persistence is ignored when an "if" condition is met, or unless an
  "unless" condition is met.

  see also : "force-persist", "cookie", and section 7 about acl usage.

log global
log <address> [len <length>] <facility> [<level> [<minlevel>]]
no log
  enable per-instance logging of events and traffic.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  prefix :
    no         should be used when the logger list must be flushed. for example,
               if you don't want to inherit from the default logger list. this
               prefix does not allow arguments.

  arguments :
    global     should be used when the instance's logging parameters are the
               same as the global ones. this is the most common usage. "global"
               replaces <address>, <facility> and <level> with those of the log
               entries found in the "global" section. only one "log global"
               statement may be used per instance, and this form takes no other
               parameter.

    <address>  indicates where to send the logs. it takes the same format as
               for the "global" section's logs, and can be one of :

               - an ipv4 address optionally followed by a colon (':') and a udp
                 port. if no port is specified, 514 is used by default (the
                 standard syslog port).

               - an ipv6 address followed by a colon (':') and optionally a udp
                 port. if no port is specified, 514 is used by default (the
                 standard syslog port).

               - a filesystem path to a unix domain socket, keeping in mind
                 considerations for chroot (be sure the path is accessible
                 inside the chroot) and uid/gid (be sure the path is
                 appropriately writeable).

              you may want to reference some environment variables in the
              address parameter, see section 2.3 about environment variables.

    <length>   is an optional maximum line length. log lines larger than this
               value will be truncated before being sent. the reason is that
               syslog servers act differently on log line length. all servers
               support the default value of 1024, but some servers simply drop
               larger lines while others do log them. if a server supports long
               lines, it may make sense to set this value here in order to avoid
               truncating long lines. similarly, if a server drops long lines,
               it is preferable to truncate them before sending them. accepted
               values are 80 to 65535 inclusive. the default value of 1024 is
               generally fine for all standard usages. some specific cases of
               long captures or json-formated logs may require larger values.

    <facility> must be one of the 24 standard syslog facilities :

                 kern   user   mail   daemon auth   syslog lpr    news
                 uucp   cron   auth2  ftp    ntp    audit  alert  cron2
                 local0 local1 local2 local3 local4 local5 local6 local7

    <level>    is optional and can be specified to filter outgoing messages. by
               default, all messages are sent. if a level is specified, only
               messages with a severity at least as important as this level
               will be sent. an optional minimum level can be specified. if it
               is set, logs emitted with a more severe level than this one will
               be capped to this level. this is used to avoid sending "emerg"
               messages on all terminals on some default syslog configurations.
               eight levels are known :

                 emerg  alert  crit   err    warning notice info  debug

  it is important to keep in mind that it is the frontend which decides what to
  log from a connection, and that in case of content switching, the log entries
  from the backend will be ignored. connections are logged at level "info".

  however, backend log declaration define how and where servers status changes
  will be logged. level "notice" will be used to indicate a server going up,
  "warning" will be used for termination signals and definitive service
  termination, and "alert" will be used for when a server goes down.

  note : according to rfc3164, messages are truncated to 1024 bytes before
         being emitted.

  example :
    log global
    log 127.0.0.1:514 local0 notice         # only send important events
    log 127.0.0.1:514 local0 notice notice  # same but limit output level
    log "${local_syslog}:514" local0 notice   # send to local server

log-format <string>
  specifies the log format string to use for traffic logs
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |    no

  this directive specifies the log format string that will be used for all logs
  resulting from traffic passing through the frontend using this line. if the
  directive is used in a defaults section, all subsequent frontends will use
  the same log format. please see section 8.2.4 which covers the log format
  string in depth.

log-tag <string>
  specifies the log tag to use for all outgoing logs
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

  sets the tag field in the syslog header to this string. it defaults to the
  log-tag set in the global section, otherwise the program name as launched
  from the command line, which usually is "haproxy". sometimes it can be useful
  to differentiate between multiple processes running on the same host, or to
  differentiate customer instances running in the same process. in the backend,
  logs about servers up/down will use this tag. as a hint, it can be convenient
  to set a log-tag related to a hosted customer in a defaults section then put
  all the frontends and backends for that customer, then start another customer
  in a new defaults section. see also the global "log-tag" directive.

max-keep-alive-queue <value>
  set the maximum server queue size for maintaining keep-alive connections
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |     no   |   yes  |   yes

  http keep-alive tries to reuse the same server connection whenever possible,
  but sometimes it can be counter-productive, for example if a server has a lot
  of connections while other ones are idle. this is especially true for static
  servers.

  the purpose of this setting is to set a threshold on the number of queued
  connections at which haproxy stops trying to reuse the same server and prefers
  to find another one. the default value, -1, means there is no limit. a value
  of zero means that keep-alive requests will never be queued. for very close
  servers which can be reached with a low latency and which are not sensible to
  breaking keep-alive, a low value is recommended (eg: local static server can
  use a value of 10 or less). for remote servers suffering from a high latency,
  higher values might be needed to cover for the latency and/or the cost of
  picking a different server.

  note that this has no impact on responses which are maintained to the same
  server consecutively to a 401 response. they will still go to the same server
  even if they have to be queued.

  see also : "option http-server-close", "option prefer-last-server", server
             "maxconn" and cookie persistence.

maxconn <conns>
  fix the maximum number of concurrent connections on a frontend
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <conns>   is the maximum number of concurrent connections the frontend will
              accept to serve. excess connections will be queued by the system
              in the socket's listen queue and will be served once a connection
              closes.

  if the system supports it, it can be useful on big sites to raise this limit
  very high so that haproxy manages connection queues, instead of leaving the
  clients with unanswered connection attempts. this value should not exceed the
  global maxconn. also, keep in mind that a connection contains two buffers
  of 8kb each, as well as some other data resulting in about 17 kb of ram being
  consumed per established connection. that means that a medium system equipped
  with 1gb of ram can withstand around 40000-50000 concurrent connections if
  properly tuned.

  also, when <conns> is set to large values, it is possible that the servers
  are not sized to accept such loads, and for this reason it is generally wise
  to assign them some reasonable connection limits.

  by default, this value is set to 2000.

  see also : "server", global section's "maxconn", "fullconn"

mode { tcp|http|health }
  set the running mode or protocol of the instance
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    tcp       the instance will work in pure tcp mode. a full-duplex connection
              will be established between clients and servers, and no layer 7
              examination will be performed. this is the default mode. it
              should be used for ssl, ssh, smtp, ...

    http      the instance will work in http mode. the client request will be
              analyzed in depth before connecting to any server. any request
              which is not rfc-compliant will be rejected. layer 7 filtering,
              processing and switching will be possible. this is the mode which
              brings haproxy most of its value.

    health    the instance will work in "health" mode. it will just reply "ok"
              to incoming connections and close the connection. alternatively,
              if the "httpchk" option is set, "http/1.0 200 ok" will be sent
              instead. nothing will be logged in either case. this mode is used
              to reply to external components health checks. this mode is
              deprecated and should not be used anymore as it is possible to do
              the same and even better by combining tcp or http modes with the
              "monitor" keyword.

  when doing content switching, it is mandatory that the frontend and the
  backend are in the same mode (generally http), otherwise the configuration
  will be refused.

  example :
     defaults http_instances
         mode http

  see also : "monitor", "monitor-net"

monitor fail { if | unless } <condition>
  add a condition to report a failure to a monitor http request.
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   no
  arguments :
    if <cond>     the monitor request will fail if the condition is satisfied,
                  and will succeed otherwise. the condition should describe a
                  combined test which must induce a failure if all conditions
                  are met, for instance a low number of servers both in a
                  backend and its backup.

    unless <cond> the monitor request will succeed only if the condition is
                  satisfied, and will fail otherwise. such a condition may be
                  based on a test on the presence of a minimum number of active
                  servers in a list of backends.

  this statement adds a condition which can force the response to a monitor
  request to report a failure. by default, when an external component queries
  the uri dedicated to monitoring, a 200 response is returned. when one of the
  conditions above is met, haproxy will return 503 instead of 200\. this is
  very useful to report a site failure to an external component which may base
  routing advertisements between multiple sites on the availability reported by
  haproxy. in this case, one would rely on an acl involving the "nbsrv"
  criterion. note that "monitor fail" only works in http mode. both status
  messages may be tweaked using "errorfile" or "errorloc" if needed.

  example:
     frontend www
        mode http
        acl site_dead nbsrv(dynamic) lt 2
        acl site_dead nbsrv(static)  lt 2
        monitor-uri   /site_alive
        monitor fail  if site_dead

  see also : "monitor-net", "monitor-uri", "errorfile", "errorloc"

monitor-net <source>
  declare a source network which is limited to monitor requests
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <source>  is the source ipv4 address or network which will only be able to
              get monitor responses to any request. it can be either an ipv4
              address, a host name, or an address followed by a slash ('/')
              followed by a mask.

  in tcp mode, any connection coming from a source matching <source> will cause
  the connection to be immediately closed without any log. this allows another
  equipment to probe the port and verify that it is still listening, without
  forwarding the connection to a remote server.

  in http mode, a connection coming from a source matching <source> will be
  accepted, the following response will be sent without waiting for a request,
  then the connection will be closed : "http/1.0 200 ok". this is normally
  enough for any front-end http probe to detect that the service is up and
  running without forwarding the request to a backend server. note that this
  response is sent in raw format, without any transformation. this is important
  as it means that it will not be ssl-encrypted on ssl listeners.

  monitor requests are processed very early, just after tcp-request connection
  acls which are the only ones able to block them. these connections are short
  lived and never wait for any data from the client. they cannot be logged, and
  it is the intended purpose. they are only used to report haproxy's health to
  an upper component, nothing more. please note that "monitor fail" rules do
  not apply to connections intercepted by "monitor-net".

  last, please note that only one "monitor-net" statement can be specified in
  a frontend. if more than one is found, only the last one will be considered.

  example :
    # addresses .252 and .253 are just probing us.
    frontend www
        monitor-net 192.168.0.252/31

  see also : "monitor fail", "monitor-uri"

monitor-uri <uri>
  intercept a uri used by external components' monitor requests
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <uri>     is the exact uri which we want to intercept to return haproxy's
              health status instead of forwarding the request.

  when an http request referencing <uri> will be received on a frontend,
  haproxy will not forward it nor log it, but instead will return either
  "http/1.0 200 ok" or "http/1.0 503 service unavailable", depending on failure
  conditions defined with "monitor fail". this is normally enough for any
  front-end http probe to detect that the service is up and running without
  forwarding the request to a backend server. note that the http method, the
  version and all headers are ignored, but the request must at least be valid
  at the http level. this keyword may only be used with an http-mode frontend.

  monitor requests are processed very early. it is not possible to block nor
  divert them using acls. they cannot be logged either, and it is the intended
  purpose. they are only used to report haproxy's health to an upper component,
  nothing more. however, it is possible to add any number of conditions using
  "monitor fail" and acls so that the result can be adjusted to whatever check
  can be imagined (most often the number of available servers in a backend).

  example :
    # use /haproxy_test to report haproxy's status
    frontend www
        mode http
        monitor-uri /haproxy_test

  see also : "monitor fail", "monitor-net"

option abortonclose
no option abortonclose
  enable or disable early dropping of aborted requests pending in queues.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |     no   |   yes  |   yes
  arguments : none

  in presence of very high loads, the servers will take some time to respond.
  the per-instance connection queue will inflate, and the response time will
  increase respective to the size of the queue times the average per-session
  response time. when clients will wait for more than a few seconds, they will
  often hit the "stop" button on their browser, leaving a useless request in
  the queue, and slowing down other users, and the servers as well, because the
  request will eventually be served, then aborted at the first error
  encountered while delivering the response.

  as there is no way to distinguish between a full stop and a simple output
  close on the client side, http agents should be conservative and consider
  that the client might only have closed its output channel while waiting for
  the response. however, this introduces risks of congestion when lots of users
  do the same, and is completely useless nowadays because probably no client at
  all will close the session while waiting for the response. some http agents
  support this behaviour (squid, apache, haproxy), and others do not (tux, most
  hardware-based load balancers). so the probability for a closed input channel
  to represent a user hitting the "stop" button is close to 100%, and the risk
  of being the single component to break rare but valid traffic is extremely
  low, which adds to the temptation to be able to abort a session early while
  still not served and not pollute the servers.

  in haproxy, the user can choose the desired behaviour using the option
  "abortonclose". by default (without the option) the behaviour is http
  compliant and aborted requests will be served. but when the option is
  specified, a session with an incoming channel closed will be aborted while
  it is still possible, either pending in the queue for a connection slot, or
  during the connection establishment if the server has not yet acknowledged
  the connection request. this considerably reduces the queue size and the load
  on saturated servers when users are tempted to click on stop, which in turn
  reduces the response time for other users.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "timeout queue" and server's "maxconn" and "maxqueue" parameters

option accept-invalid-http-request
no option accept-invalid-http-request
  enable or disable relaxing of http request parsing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  by default, haproxy complies with rfc7230 in terms of message parsing. this
  means that invalid characters in header names are not permitted and cause an
  error to be returned to the client. this is the desired behaviour as such
  forbidden characters are essentially used to build attacks exploiting server
  weaknesses, and bypass security filtering. sometimes, a buggy browser or
  server will emit invalid header names for whatever reason (configuration,
  implementation) and the issue will not be immediately fixed. in such a case,
  it is possible to relax haproxy's header name parser to accept any character
  even if that does not make sense, by specifying this option. similarly, the
  list of characters allowed to appear in a uri is well defined by rfc3986, and
  chars 0-31, 32 (space), 34 ('"'), 60 ('<'), 62 ('>'), 92 ('\'), 94 ('^'), 96
  ('`'), 123 ('{'), 124 ('|'), 125 ('}'), 127 (delete) and anything above are
  not allowed at all. haproxy always blocks a number of them (0..32, 127). the
  remaining ones are blocked by default unless this option is enabled. this
  option also relaxes the test on the http version, it allows http/0.9 requests
  to pass through (no version specified) and multiple digits for both the major
  and the minor version.

  this option should never be enabled by default as it hides application bugs
  and open security breaches. it should only be deployed after a problem has
  been confirmed.

  when this option is enabled, erroneous header names will still be accepted in
  requests, but the complete request will be captured in order to permit later
  analysis using the "show errors" request on the unix stats socket. similarly,
  requests containing invalid chars in the uri part will be logged. doing this
  also helps confirming that the issue has been solved.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option accept-invalid-http-response" and "show errors" on the
             stats socket.

option accept-invalid-http-response
no option accept-invalid-http-response
  enable or disable relaxing of http response parsing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |     no   |   yes  |   yes
  arguments : none

  by default, haproxy complies with rfc7230 in terms of message parsing. this
  means that invalid characters in header names are not permitted and cause an
  error to be returned to the client. this is the desired behaviour as such
  forbidden characters are essentially used to build attacks exploiting server
  weaknesses, and bypass security filtering. sometimes, a buggy browser or
  server will emit invalid header names for whatever reason (configuration,
  implementation) and the issue will not be immediately fixed. in such a case,
  it is possible to relax haproxy's header name parser to accept any character
  even if that does not make sense, by specifying this option. this option also
  relaxes the test on the http version format, it allows multiple digits for
  both the major and the minor version.

  this option should never be enabled by default as it hides application bugs
  and open security breaches. it should only be deployed after a problem has
  been confirmed.

  when this option is enabled, erroneous header names will still be accepted in
  responses, but the complete response will be captured in order to permit
  later analysis using the "show errors" request on the unix stats socket.
  doing this also helps confirming that the issue has been solved.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option accept-invalid-http-request" and "show errors" on the
             stats socket.

option allbackups
no option allbackups
  use either all backup servers at a time or only the first one
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |     no   |   yes  |   yes
  arguments : none

  by default, the first operational backup server gets all traffic when normal
  servers are all down. sometimes, it may be preferred to use multiple backups
  at once, because one will not be enough. when "option allbackups" is enabled,
  the load balancing will be performed among all backup servers when all normal
  ones are unavailable. the same load balancing algorithm will be used and the
  servers' weights will be respected. thus, there will not be any priority
  order between the backup servers anymore.

  this option is mostly used with static server farms dedicated to return a
  "sorry" page when an application is completely offline.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

option checkcache
no option checkcache
  analyze all server responses and block responses with cacheable cookies
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |     no   |   yes  |   yes
  arguments : none

  some high-level frameworks set application cookies everywhere and do not
  always let enough control to the developer to manage how the responses should
  be cached. when a session cookie is returned on a cacheable object, there is a
  high risk of session crossing or stealing between users traversing the same
  caches. in some situations, it is better to block the response than to let
  some sensitive session information go in the wild.

  the option "checkcache" enables deep inspection of all server responses for
  strict compliance with http specification in terms of cacheability. it
  carefully checks "cache-control", "pragma" and "set-cookie" headers in server
  response to check if there's a risk of caching a cookie on a client-side
  proxy. when this option is enabled, the only responses which can be delivered
  to the client are :
    - all those without "set-cookie" header ;
    - all those with a return code other than 200, 203, 206, 300, 301, 410,
      provided that the server has not set a "cache-control: public" header ;
    - all those that come from a post request, provided that the server has not
      set a 'cache-control: public' header ;
    - those with a 'pragma: no-cache' header
    - those with a 'cache-control: private' header
    - those with a 'cache-control: no-store' header
    - those with a 'cache-control: max-age=0' header
    - those with a 'cache-control: s-maxage=0' header
    - those with a 'cache-control: no-cache' header
    - those with a 'cache-control: no-cache="set-cookie"' header
    - those with a 'cache-control: no-cache="set-cookie,' header
      (allowing other fields after set-cookie)

  if a response doesn't respect these requirements, then it will be blocked
  just as if it was from an "rspdeny" filter, with an "http 502 bad gateway".
  the session state shows "ph--" meaning that the proxy blocked the response
  during headers processing. additionally, an alert will be sent in the logs so
  that admins are informed that there's something to be fixed.

  due to the high impact on the application, the application should be tested
  in depth with the option enabled before going to production. it is also a
  good practice to always activate it during tests, even if it is not used in
  production, as it will report potentially dangerous application behaviours.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

option clitcpka
no option clitcpka
  enable or disable the sending of tcp keepalive packets on the client side
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  when there is a firewall or any session-aware component between a client and
  a server, and when the protocol involves very long sessions with long idle
  periods (eg: remote desktops), there is a risk that one of the intermediate
  components decides to expire a session which has remained idle for too long.

  enabling socket-level tcp keep-alives makes the system regularly send packets
  to the other end of the connection, leaving it active. the delay between
  keep-alive probes is controlled by the system only and depends both on the
  operating system and its tuning parameters.

  it is important to understand that keep-alive packets are neither emitted nor
  received at the application level. it is only the network stacks which sees
  them. for this reason, even if one side of the proxy already uses keep-alives
  to maintain its connection alive, those keep-alive packets will not be
  forwarded to the other side of the proxy.

  please note that this has nothing to do with http keep-alive.

  using option "clitcpka" enables the emission of tcp keep-alive probes on the
  client side of a connection, which should help when session expirations are
  noticed between haproxy and a client.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option srvtcpka", "option tcpka"

option contstats
  enable continuous traffic statistics updates
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  by default, counters used for statistics calculation are incremented
  only when a session finishes. it works quite well when serving small
  objects, but with big ones (for example large images or archives) or
  with a/v streaming, a graph generated from haproxy counters looks like
  a hedgehog. with this option enabled counters get incremented continuously,
  during a whole session. recounting touches a hotpath directly so
  it is not enabled by default, as it has small performance impact (~0.5%).

option dontlog-normal
no option dontlog-normal
  enable or disable logging of normal, successful connections
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  there are large sites dealing with several thousand connections per second
  and for which logging is a major pain. some of them are even forced to turn
  logs off and cannot debug production issues. setting this option ensures that
  normal connections, those which experience no error, no timeout, no retry nor
  redispatch, will not be logged. this leaves disk space for anomalies. in http
  mode, the response status code is checked and return codes 5xx will still be
  logged.

  it is strongly discouraged to use this option as most of the time, the key to
  complex issues is in the normal logs which will not be logged here. if you
  need to separate logs, see the "log-separate-errors" option instead.

  see also : "log", "dontlognull", "log-separate-errors" and section 8 about
             logging.

option dontlognull
no option dontlognull
  enable or disable logging of null connections
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  in certain environments, there are components which will regularly connect to
  various systems to ensure that they are still alive. it can be the case from
  another load balancer as well as from monitoring systems. by default, even a
  simple port probe or scan will produce a log. if those connections pollute
  the logs too much, it is possible to enable option "dontlognull" to indicate
  that a connection on which no data has been transferred will not be logged,
  which typically corresponds to those probes. note that errors will still be
  returned to the client and accounted for in the stats. if this is not what is
  desired, option http-ignore-probes can be used instead.

  it is generally recommended not to use this option in uncontrolled
  environments (eg: internet), otherwise scans and other malicious activities
  would not be logged.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "log", "http-ignore-probes", "monitor-net", "monitor-uri", and
             section 8 about logging.

option forceclose
no option forceclose
  enable or disable active connection closing after response is transferred.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  some http servers do not necessarily close the connections when they receive
  the "connection: close" set by "option httpclose", and if the client does not
  close either, then the connection remains open till the timeout expires. this
  causes high number of simultaneous connections on the servers and shows high
  global session times in the logs.

  when this happens, it is possible to use "option forceclose". it will
  actively close the outgoing server channel as soon as the server has finished
  to respond and release some resources earlier than with "option httpclose".

  this option may also be combined with "option http-pretend-keepalive", which
  will disable sending of the "connection: close" header, but will still cause
  the connection to be closed once the whole response is received.

  this option disables and replaces any previous "option httpclose", "option
  http-server-close", "option http-keep-alive", or "option http-tunnel".

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option httpclose" and "option http-pretend-keepalive"

option forwardfor [ except <network> ] [ header <name> ] [ if-none ]
  enable insertion of the x-forwarded-for header to requests sent to servers
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <network> is an optional argument used to disable this option for sources
              matching <network>
    <name>    an optional argument to specify a different "x-forwarded-for"
              header name.

  since haproxy works in reverse-proxy mode, the servers see its ip address as
  their client address. this is sometimes annoying when the client's ip address
  is expected in server logs. to solve this problem, the well-known http header
  "x-forwarded-for" may be added by haproxy to all requests sent to the server.
  this header contains a value representing the client's ip address. since this
  header is always appended at the end of the existing header list, the server
  must be configured to always use the last occurrence of this header only. see
  the server's manual to find how to enable use of this standard header. note
  that only the last occurrence of the header must be used, since it is really
  possible that the client has already brought one.

  the keyword "header" may be used to supply a different header name to replace
  the default "x-forwarded-for". this can be useful where you might already
  have a "x-forwarded-for" header from a different application (eg: stunnel),
  and you need preserve it. also if your backend server doesn't use the
  "x-forwarded-for" header and requires different one (eg: zeus web servers
  require "x-cluster-client-ip").

  sometimes, a same haproxy instance may be shared between a direct client
  access and a reverse-proxy access (for instance when an ssl reverse-proxy is
  used to decrypt https traffic). it is possible to disable the addition of the
  header for a known source address or network by adding the "except" keyword
  followed by the network address. in this case, any source ip matching the
  network will not cause an addition of this header. most common uses are with
  private networks or 127.0.0.1.

  alternatively, the keyword "if-none" states that the header will only be
  added if it is not present. this should only be used in perfectly trusted
  environment, as this might cause a security issue if headers reaching haproxy
  are under the control of the end-user.

  this option may be specified either in the frontend or in the backend. if at
  least one of them uses it, the header will be added. note that the backend's
  setting of the header subargument takes precedence over the frontend's if
  both are defined. in the case of the "if-none" argument, if at least one of
  the frontend or the backend does not specify it, it wants the addition to be
  mandatory, so it wins.

  examples :
    # public http address also used by stunnel on the same machine
    frontend www
        mode http
        option forwardfor except 127.0.0.1  # stunnel already adds the header

    # those servers want the ip address in x-client
    backend www
        mode http
        option forwardfor header x-client

  see also : "option httpclose", "option http-server-close",
             "option forceclose", "option http-keep-alive"

option http-buffer-request
no option http-buffer-request
  enable or disable waiting for whole http request body before proceeding
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  it is sometimes desirable to wait for the body of an http request before
  taking a decision. this is what is being done by "balance url_param" for
  example. the first use case is to buffer requests from slow clients before
  connecting to the server. another use case consists in taking the routing
  decision based on the request body's contents. this option placed in a
  frontend or backend forces the http processing to wait until either the whole
  body is received, or the request buffer is full, or the first chunk is
  complete in case of chunked encoding. it can have undesired side effects with
  some applications abusing http by expecting unbufferred transmissions between
  the frontend and the backend, so this should definitely not be used by
  default.

  see also : "option http-no-delay"

option http-ignore-probes
no option http-ignore-probes
  enable or disable logging of null connections and request timeouts
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  recently some browsers started to implement a "pre-connect" feature
  consisting in speculatively connecting to some recently visited web sites
  just in case the user would like to visit them. this results in many
  connections being established to web sites, which end up in 408 request
  timeout if the timeout strikes first, or 400 bad request when the browser
  decides to close them first. these ones pollute the log and feed the error
  counters. there was already "option dontlognull" but it's insufficient in
  this case. instead, this option does the following things :
     - prevent any 400/408 message from being sent to the client if nothing
       was received over a connection before it was closed ;
     - prevent any log from being emitted in this situation ;
     - prevent any error counter from being incremented

  that way the empty connection is silently ignored. note that it is better
  not to use this unless it is clear that it is needed, because it will hide
  real problems. the most common reason for not receiving a request and seeing
  a 408 is due to an mtu inconsistency between the client and an intermediary
  element such as a vpn, which blocks too large packets. these issues are
  generally seen with post requests as well as get with large cookies. the logs
  are often the only way to detect them.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "log", "dontlognull", "errorfile", and section 8 about logging.

option http-keep-alive
no option http-keep-alive
  enable or disable http keep-alive from client to server
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  by default haproxy operates in keep-alive mode with regards to persistent
  connections: for each connection it processes each request and response, and
  leaves the connection idle on both sides between the end of a response and the
  start of a new request. this mode may be changed by several options such as
  "option http-server-close", "option forceclose", "option httpclose" or
  "option http-tunnel". this option allows to set back the keep-alive mode,
  which can be useful when another mode was used in a defaults section.

  setting "option http-keep-alive" enables http keep-alive mode on the client-
  and server- sides. this provides the lowest latency on the client side (slow
  network) and the fastest session reuse on the server side at the expense
  of maintaining idle connections to the servers. in general, it is possible
  with this option to achieve approximately twice the request rate that the
  "http-server-close" option achieves on small objects. there are mainly two
  situations where this option may be useful :

    - when the server is non-http compliant and authenticates the connection
      instead of requests (eg: ntlm authentication)

    - when the cost of establishing the connection to the server is significant
      compared to the cost of retrieving the associated object from the server.

  this last case can happen when the server is a fast static server of cache.
  in this case, the server will need to be properly tuned to support high enough
  connection counts because connections will last until the client sends another
  request.

  if the client request has to go to another backend or another server due to
  content switching or the load balancing algorithm, the idle connection will
  immediately be closed and a new one re-opened. option "prefer-last-server" is
  available to try optimize server selection so that if the server currently
  attached to an idle connection is usable, it will be used.

  in general it is preferred to use "option http-server-close" with application
  servers, and some static servers might benefit from "option http-keep-alive".

  at the moment, logs will not indicate whether requests came from the same
  session or not. the accept date reported in the logs corresponds to the end
  of the previous request, and the request time corresponds to the time spent
  waiting for a new request. the keep-alive request time is still bound to the
  timeout defined by "timeout http-keep-alive" or "timeout http-request" if
  not set.

  this option disables and replaces any previous "option httpclose", "option
  http-server-close", "option forceclose" or "option http-tunnel". when backend
  and frontend options differ, all of these 4 options have precedence over
  "option http-keep-alive".

  see also : "option forceclose", "option http-server-close",
             "option prefer-last-server", "option http-pretend-keepalive",
             "option httpclose", and "1.1\. the http transaction model".

option http-no-delay
no option http-no-delay
  instruct the system to favor low interactive delays over performance in http
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  in http, each payload is unidirectional and has no notion of interactivity.
  any agent is expected to queue data somewhat for a reasonably low delay.
  there are some very rare server-to-server applications that abuse the http
  protocol and expect the payload phase to be highly interactive, with many
  interleaved data chunks in both directions within a single request. this is
  absolutely not supported by the http specification and will not work across
  most proxies or servers. when such applications attempt to do this through
  haproxy, it works but they will experience high delays due to the network
  optimizations which favor performance by instructing the system to wait for
  enough data to be available in order to only send full packets. typical
  delays are around 200 ms per round trip. note that this only happens with
  abnormal uses. normal uses such as connect requests nor websockets are not
  affected.

  when "option http-no-delay" is present in either the frontend or the backend
  used by a connection, all such optimizations will be disabled in order to
  make the exchanges as fast as possible. of course this offers no guarantee on
  the functionality, as it may break at any other place. but if it works via
  haproxy, it will work as fast as possible. this option should never be used
  by default, and should never be used at all unless such a buggy application
  is discovered. the impact of using this option is an increase of bandwidth
  usage and cpu usage, which may significantly lower performance in high
  latency environments.

  see also : "option http-buffer-request"

option http-pretend-keepalive
no option http-pretend-keepalive
  define whether haproxy will announce keepalive to the server or not
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  when running with "option http-server-close" or "option forceclose", haproxy
  adds a "connection: close" header to the request forwarded to the server.
  unfortunately, when some servers see this header, they automatically refrain
  from using the chunked encoding for responses of unknown length, while this
  is totally unrelated. the immediate effect is that this prevents haproxy from
  maintaining the client connection alive. a second effect is that a client or
  a cache could receive an incomplete response without being aware of it, and
  consider the response complete.

  by setting "option http-pretend-keepalive", haproxy will make the server
  believe it will keep the connection alive. the server will then not fall back
  to the abnormal undesired above. when haproxy gets the whole response, it
  will close the connection with the server just as it would do with the
  "forceclose" option. that way the client gets a normal response and the
  connection is correctly closed on the server side.

  it is recommended not to enable this option by default, because most servers
  will more efficiently close the connection themselves after the last packet,
  and release its buffers slightly earlier. also, the added packet on the
  network could slightly reduce the overall peak performance. however it is
  worth noting that when this option is enabled, haproxy will have slightly
  less work to do. so if haproxy is the bottleneck on the whole architecture,
  enabling this option might save a few cpu cycles.

  this option may be set both in a frontend and in a backend. it is enabled if
  at least one of the frontend or backend holding a connection has it enabled.
  this option may be combined with "option httpclose", which will cause
  keepalive to be announced to the server and close to be announced to the
  client. this practice is discouraged though.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option forceclose", "option http-server-close", and
             "option http-keep-alive"

option http-server-close
no option http-server-close
  enable or disable http connection closing on the server side
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  by default haproxy operates in keep-alive mode with regards to persistent
  connections: for each connection it processes each request and response, and
  leaves the connection idle on both sides between the end of a response and
  the start of a new request. this mode may be changed by several options such
  as "option http-server-close", "option forceclose", "option httpclose" or
  "option http-tunnel". setting "option http-server-close" enables http
  connection-close mode on the server side while keeping the ability to support
  http keep-alive and pipelining on the client side.  this provides the lowest
  latency on the client side (slow network) and the fastest session reuse on
  the server side to save server resources, similarly to "option forceclose".
  it also permits non-keepalive capable servers to be served in keep-alive mode
  to the clients if they conform to the requirements of rfc2616\. please note
  that some servers do not always conform to those requirements when they see
  "connection: close" in the request. the effect will be that keep-alive will
  never be used. a workaround consists in enabling "option
  http-pretend-keepalive".

  at the moment, logs will not indicate whether requests came from the same
  session or not. the accept date reported in the logs corresponds to the end
  of the previous request, and the request time corresponds to the time spent
  waiting for a new request. the keep-alive request time is still bound to the
  timeout defined by "timeout http-keep-alive" or "timeout http-request" if
  not set.

  this option may be set both in a frontend and in a backend. it is enabled if
  at least one of the frontend or backend holding a connection has it enabled.
  it disables and replaces any previous "option httpclose", "option forceclose",
  "option http-tunnel" or "option http-keep-alive". please check section 4
  ("proxies") to see how this option combines with others when frontend and
  backend options differ.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option forceclose", "option http-pretend-keepalive",
             "option httpclose", "option http-keep-alive", and
             "1.1\. the http transaction model".

option http-tunnel
no option http-tunnel
  disable or enable http connection processing after first transaction
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  by default haproxy operates in keep-alive mode with regards to persistent
  connections: for each connection it processes each request and response, and
  leaves the connection idle on both sides between the end of a response and
  the start of a new request. this mode may be changed by several options such
  as "option http-server-close", "option forceclose", "option httpclose" or
  "option http-tunnel".

  option "http-tunnel" disables any http processing past the first request and
  the first response. this is the mode which was used by default in versions
  1.0 to 1.5-dev21\. it is the mode with the lowest processing overhead, which
  is normally not needed anymore unless in very specific cases such as when
  using an in-house protocol that looks like http but is not compatible, or
  just to log one request per client in order to reduce log size. note that
  everything which works at the http level, including header parsing/addition,
  cookie processing or content switching will only work for the first request
  and will be ignored after the first response.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option forceclose", "option http-server-close",
             "option httpclose", "option http-keep-alive", and
             "1.1\. the http transaction model".

option http-use-proxy-header
no option http-use-proxy-header
  make use of non-standard proxy-connection header instead of connection
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  while rfc2616 explicitly states that http/1.1 agents must use the
  connection header to indicate their wish of persistent or non-persistent
  connections, both browsers and proxies ignore this header for proxied
  connections and make use of the undocumented, non-standard proxy-connection
  header instead. the issue begins when trying to put a load balancer between
  browsers and such proxies, because there will be a difference between what
  haproxy understands and what the client and the proxy agree on.

  by setting this option in a frontend, haproxy can automatically switch to use
  that non-standard header if it sees proxied requests. a proxied request is
  defined here as one where the uri begins with neither a '/' nor a '*'. the
  choice of header only affects requests passing through proxies making use of
  one of the "httpclose", "forceclose" and "http-server-close" options. note
  that this option can only be specified in a frontend and will affect the
  request along its whole life.

  also, when this option is set, a request which requires authentication will
  automatically switch to use proxy authentication headers if it is itself a
  proxied request. that makes it possible to check or enforce authentication in
  front of an existing proxy.

  this option should normally never be used, except in front of a proxy.

  see also : "option httpclose", "option forceclose" and "option
             http-server-close".

option httpchk
option httpchk <uri>
option httpchk <method> <uri>
option httpchk <method> <uri> <version>
  enable http protocol to check on the servers health
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <method>  is the optional http method used with the requests. when not set,
              the "options" method is used, as it generally requires low server
              processing and is easy to filter out from the logs. any method
              may be used, though it is not recommended to invent non-standard
              ones.

    <uri>     is the uri referenced in the http requests. it defaults to " / "
              which is accessible by default on almost any server, but may be
              changed to any other uri. query strings are permitted.

    <version> is the optional http version string. it defaults to "http/1.0"
              but some servers might behave incorrectly in http 1.0, so turning
              it to http/1.1 may sometimes help. note that the host field is
              mandatory in http/1.1, and as a trick, it is possible to pass it
              after "\r\n" following the version string.

  by default, server health checks only consist in trying to establish a tcp
  connection. when "option httpchk" is specified, a complete http request is
  sent once the tcp connection is established, and responses 2xx and 3xx are
  considered valid, while all other ones indicate a server failure, including
  the lack of any response.

  the port and interval are specified in the server configuration.

  this option does not necessarily require an http backend, it also works with
  plain tcp backends. this is particularly useful to check simple scripts bound
  to some dedicated ports using the inetd daemon.

  examples :
      # relay https traffic to apache instance and check service availability
      # using http request "options * http/1.1" on port 80.
      backend https_relay
          mode tcp
          option httpchk options * http/1.1\r\nhost:\ www
          server apache1 192.168.1.1:443 check port 80

  see also : "option ssl-hello-chk", "option smtpchk", "option mysql-check",
             "option pgsql-check", "http-check" and the "check", "port" and
             "inter" server options.

option httpclose
no option httpclose
  enable or disable passive http connection closing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  by default haproxy operates in keep-alive mode with regards to persistent
  connections: for each connection it processes each request and response, and
  leaves the connection idle on both sides between the end of a response and
  the start of a new request. this mode may be changed by several options such
  as "option http-server-close", "option forceclose", "option httpclose" or
  "option http-tunnel".

  if "option httpclose" is set, haproxy will work in http tunnel mode and check
  if a "connection: close" header is already set in each direction, and will
  add one if missing. each end should react to this by actively closing the tcp
  connection after each transfer, thus resulting in a switch to the http close
  mode. any "connection" header different from "close" will also be removed.
  note that this option is deprecated since what it does is very cheap but not
  reliable. using "option http-server-close" or "option forceclose" is strongly
  recommended instead.

  it seldom happens that some servers incorrectly ignore this header and do not
  close the connection even though they reply "connection: close". for this
  reason, they are not compatible with older http 1.0 browsers. if this happens
  it is possible to use the "option forceclose" which actively closes the
  request connection once the server responds. option "forceclose" also
  releases the server connection earlier because it does not have to wait for
  the client to acknowledge it.

  this option may be set both in a frontend and in a backend. it is enabled if
  at least one of the frontend or backend holding a connection has it enabled.
  it disables and replaces any previous "option http-server-close",
  "option forceclose", "option http-keep-alive" or "option http-tunnel". please
  check section 4 ("proxies") to see how this option combines with others when
  frontend and backend options differ.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option forceclose", "option http-server-close" and
             "1.1\. the http transaction model".

option httplog [ clf ]
  enable logging of http request, session state and timers
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    clf       if the "clf" argument is added, then the output format will be
              the clf format instead of haproxy's default http format. you can
              use this when you need to feed haproxy's logs through a specific
              log analyser which only support the clf format and which is not
              extensible.

  by default, the log output format is very poor, as it only contains the
  source and destination addresses, and the instance name. by specifying
  "option httplog", each log line turns into a much richer format including,
  but not limited to, the http request, the connection timers, the session
  status, the connections numbers, the captured headers and cookies, the
  frontend, backend and server name, and of course the source address and
  ports.

  this option may be set either in the frontend or the backend.

  specifying only "option httplog" will automatically clear the 'clf' mode
  if it was set by default.

  see also :  section 8 about logging.

option http_proxy
no option http_proxy
  enable or disable plain http proxy mode
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  it sometimes happens that people need a pure http proxy which understands
  basic proxy requests without caching nor any fancy feature. in this case,
  it may be worth setting up an haproxy instance with the "option http_proxy"
  set. in this mode, no server is declared, and the connection is forwarded to
  the ip address and port found in the url after the "http://" scheme.

  no host address resolution is performed, so this only works when pure ip
  addresses are passed. since this option's usage perimeter is rather limited,
  it will probably be used only by experts who know they need exactly it. last,
  if the clients are susceptible of sending keep-alive requests, it will be
  needed to add "option httpclose" to ensure that all requests will correctly
  be analyzed.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  example :
    # this backend understands http proxy requests and forwards them directly.
    backend direct_forward
        option httpclose
        option http_proxy

  see also : "option httpclose"

option independent-streams
no option independent-streams
  enable or disable independent timeout processing for both directions
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |  yes
  arguments : none

  by default, when data is sent over a socket, both the write timeout and the
  read timeout for that socket are refreshed, because we consider that there is
  activity on that socket, and we have no other means of guessing if we should
  receive data or not.

  while this default behaviour is desirable for almost all applications, there
  exists a situation where it is desirable to disable it, and only refresh the
  read timeout if there are incoming data. this happens on sessions with large
  timeouts and low amounts of exchanged data such as telnet session. if the
  server suddenly disappears, the output data accumulates in the system's
  socket buffers, both timeouts are correctly refreshed, and there is no way
  to know the server does not receive them, so we don't timeout. however, when
  the underlying protocol always echoes sent data, it would be enough by itself
  to detect the issue using the read timeout. note that this problem does not
  happen with more verbose protocols because data won't accumulate long in the
  socket buffers.

  when this option is set on the frontend, it will disable read timeout updates
  on data sent to the client. there probably is little use of this case. when
  the option is set on the backend, it will disable read timeout updates on
  data sent to the server. doing so will typically break large http posts from
  slow lines, so use it with caution.

  note: older versions used to call this setting "option independent-streams"
        with a spelling mistake. this spelling is still supported but
        deprecated.

  see also : "timeout client", "timeout server" and "timeout tunnel"

option ldap-check
  use ldapv3 health checks for server testing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  it is possible to test that the server correctly talks ldapv3 instead of just
  testing that it accepts the tcp connection. when this option is set, an
  ldapv3 anonymous simple bind message is sent to the server, and the response
  is analyzed to find an ldapv3 bind response message.

  the server is considered valid only when the ldap response contains success
  resultcode (http://tools.ietf.org/html/rfc4511#section-4.1.9).

  logging of bind requests is server dependent see your documentation how to
  configure it.

  example :
        option ldap-check

  see also : "option httpchk"

option external-check
  use external processes for server health checks
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes

  it is possible to test the health of a server using an external command.
  this is achieved by running the executable set using "external-check
  command".

  requires the "external-check" global to be set.

  see also : "external-check", "external-check command", "external-check path"

option log-health-checks
no option log-health-checks
  enable or disable logging of health checks status updates
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |  yes
  arguments : none

  by default, failed health check are logged if server is up and successful
  health checks are logged if server is down, so the amount of additional
  information is limited.

  when this option is enabled, any change of the health check status or to
  the server's health will be logged, so that it becomes possible to know
  that a server was failing occasional checks before crashing, or exactly when
  it failed to respond a valid http status, then when the port started to
  reject connections, then when the server stopped responding at all.

  note that status changes not caused by health checks (eg: enable/disable on
  the cli) are intentionally not logged by this option.

  see also: "option httpchk", "option ldap-check", "option mysql-check",
            "option pgsql-check", "option redis-check", "option smtpchk",
            "option tcp-check", "log" and section 8 about logging.

option log-separate-errors
no option log-separate-errors
  change log level for non-completely successful connections
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  sometimes looking for errors in logs is not easy. this option makes haproxy
  raise the level of logs containing potentially interesting information such
  as errors, timeouts, retries, redispatches, or http status codes 5xx. the
  level changes from "info" to "err". this makes it possible to log them
  separately to a different file with most syslog daemons. be careful not to
  remove them from the original file, otherwise you would lose ordering which
  provides very important information.

  using this option, large sites dealing with several thousand connections per
  second may log normal traffic to a rotating buffer and only archive smaller
  error logs.

  see also : "log", "dontlognull", "dontlog-normal" and section 8 about
             logging.

option logasap
no option logasap
  enable or disable early logging of http requests
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments : none

  by default, http requests are logged upon termination so that the total
  transfer time and the number of bytes appear in the logs. when large objects
  are being transferred, it may take a while before the request appears in the
  logs. using "option logasap", the request gets logged as soon as the server
  sends the complete headers. the only missing information in the logs will be
  the total number of bytes which will indicate everything except the amount
  of data transferred, and the total time which will not take the transfer
  time into account. in such a situation, it's a good practice to capture the
  "content-length" response header so that the logs at least indicate how many
  bytes are expected to be transferred.

  examples :
      listen http_proxy 0.0.0.0:80
          mode http
          option httplog
          option logasap
          log 192.168.2.200 local3

    >>> feb  6 12:14:14 localhost \
          haproxy[14389]: 10.0.1.2:33317 [06/feb/2009:12:14:14.655] http-in \
          static/srv1 9/10/7/14/+30 200 +243 - - ---- 3/1/1/1/0 1/0 \
          "get /image.iso http/1.0"

  see also : "option httplog", "capture response header", and section 8 about
             logging.

option mysql-check [ user <username> [ post-41 ] ]
  use mysql health checks for server testing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <username> this is the username which will be used when connecting to mysql
               server.
    post-41    send post v4.1 client compatible checks

  if you specify a username, the check consists of sending two mysql packet,
  one client authentication packet, and one quit packet, to correctly close
  mysql session. we then parse the mysql handshake initialisation packet and/or
  error packet. it is a basic but useful test which does not produce error nor
  aborted connect on the server. however, it requires adding an authorization
  in the mysql table, like this :

      use mysql;
      insert into user (host,user) values ('<ip_of_haproxy>','<username>');
      flush privileges;

  if you don't specify a username (it is deprecated and not recommended), the
  check only consists in parsing the mysql handshake initialisation packet or
  error packet, we don't send anything in this mode. it was reported that it
  can generate lockout if check is too frequent and/or if there is not enough
  traffic. in fact, you need in this case to check mysql "max_connect_errors"
  value as if a connection is established successfully within fewer than mysql
  "max_connect_errors" attempts after a previous connection was interrupted,
  the error count for the host is cleared to zero. if haproxy's server get
  blocked, the "flush hosts" statement is the only way to unblock it.

  remember that this does not check database presence nor database consistency.
  to do this, you can use an external check with xinetd for example.

  the check requires mysql >=3.22, for older version, please use tcp check.

  most often, an incoming mysql server needs to see the client's ip address for
  various purposes, including ip privilege matching and connection logging.
  when possible, it is often wise to masquerade the client's ip address when
  connecting to the server using the "usesrc" argument of the "source" keyword,
  which requires the cttproxy feature to be compiled in, and the mysql server
  to route the client via the machine hosting haproxy.

  see also: "option httpchk"

option nolinger
no option nolinger
  enable or disable immediate session resource cleaning after close
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  when clients or servers abort connections in a dirty way (eg: they are
  physically disconnected), the session timeouts triggers and the session is
  closed. but it will remain in fin_wait1 state for some time in the system,
  using some resources and possibly limiting the ability to establish newer
  connections.

  when this happens, it is possible to activate "option nolinger" which forces
  the system to immediately remove any socket's pending data on close. thus,
  the session is instantly purged from the system's tables. this usually has
  side effects such as increased number of tcp resets due to old retransmits
  getting immediately rejected. some firewalls may sometimes complain about
  this too.

  for this reason, it is not recommended to use this option when not absolutely
  needed. you know that you need it when you have thousands of fin_wait1
  sessions on your system (time_wait ones do not count).

  this option may be used both on frontends and backends, depending on the side
  where it is required. use it on the frontend for clients, and on the backend
  for servers.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

option originalto [ except <network> ] [ header <name> ]
  enable insertion of the x-original-to header to requests sent to servers
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <network> is an optional argument used to disable this option for sources
              matching <network>
    <name>    an optional argument to specify a different "x-original-to"
              header name.

  since haproxy can work in transparent mode, every request from a client can
  be redirected to the proxy and haproxy itself can proxy every request to a
  complex squid environment and the destination host from so_original_dst will
  be lost. this is annoying when you want access rules based on destination ip
  addresses. to solve this problem, a new http header "x-original-to" may be
  added by haproxy to all requests sent to the server. this header contains a
  value representing the original destination ip address. since this must be
  configured to always use the last occurrence of this header only. note that
  only the last occurrence of the header must be used, since it is really
  possible that the client has already brought one.

  the keyword "header" may be used to supply a different header name to replace
  the default "x-original-to". this can be useful where you might already
  have a "x-original-to" header from a different application, and you need
  preserve it. also if your backend server doesn't use the "x-original-to"
  header and requires different one.

  sometimes, a same haproxy instance may be shared between a direct client
  access and a reverse-proxy access (for instance when an ssl reverse-proxy is
  used to decrypt https traffic). it is possible to disable the addition of the
  header for a known source address or network by adding the "except" keyword
  followed by the network address. in this case, any source ip matching the
  network will not cause an addition of this header. most common uses are with
  private networks or 127.0.0.1.

  this option may be specified either in the frontend or in the backend. if at
  least one of them uses it, the header will be added. note that the backend's
  setting of the header subargument takes precedence over the frontend's if
  both are defined.

  examples :
    # original destination address
    frontend www
        mode http
        option originalto except 127.0.0.1

    # those servers want the ip address in x-client-dst
    backend www
        mode http
        option originalto header x-client-dst

  see also : "option httpclose", "option http-server-close",
             "option forceclose"

option persist
no option persist
  enable or disable forced persistence on down servers
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  when an http request reaches a backend with a cookie which references a dead
  server, by default it is redispatched to another server. it is possible to
  force the request to be sent to the dead server first using "option persist"
  if absolutely needed. a common use case is when servers are under extreme
  load and spend their time flapping. in this case, the users would still be
  directed to the server they opened the session on, in the hope they would be
  correctly served. it is recommended to use "option redispatch" in conjunction
  with this option so that in the event it would not be possible to connect to
  the server at all (server definitely dead), the client would finally be
  redirected to another valid server.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option redispatch", "retries", "force-persist"

option pgsql-check [ user <username> ]
  use postgresql health checks for server testing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <username> this is the username which will be used when connecting to
               postgresql server.

  the check sends a postgresql startupmessage and waits for either
  authentication request or errorresponse message. it is a basic but useful
  test which does not produce error nor aborted connect on the server.
  this check is identical with the "mysql-check".

  see also: "option httpchk"

option prefer-last-server
no option prefer-last-server
  allow multiple load balanced requests to remain on the same server
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  when the load balancing algorithm in use is not deterministic, and a previous
  request was sent to a server to which haproxy still holds a connection, it is
  sometimes desirable that subsequent requests on a same session go to the same
  server as much as possible. note that this is different from persistence, as
  we only indicate a preference which haproxy tries to apply without any form
  of warranty. the real use is for keep-alive connections sent to servers. when
  this option is used, haproxy will try to reuse the same connection that is
  attached to the server instead of rebalancing to another server, causing a
  close of the connection. this can make sense for static file servers. it does
  not make much sense to use this in combination with hashing algorithms. note,
  haproxy already automatically tries to stick to a server which sends a 401 or
  to a proxy which sends a 407 (authentication required). this is mandatory for
  use with the broken ntlm authentication challenge, and significantly helps in
  troubleshooting some faulty applications. option prefer-last-server might be
  desirable in these environments as well, to avoid redistributing the traffic
  after every other response.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also: "option http-keep-alive"

option redispatch
option redispatch <interval>
no option redispatch
  enable or disable session redistribution in case of connection failure
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <interval> the optional integer value that controls how often redispatches
               occur when retrying connections. positive value p indicates a
               redispatch is desired on every pth retry, and negative value
               n indicate a redispath is desired on the nth retry prior to the
               last retry. for example, the default of -1 preserves the
               historical behaviour of redispatching on the last retry, a
               positive value of 1 would indicate a redispatch on every retry,
               and a positive value of 3 would indicate a redispatch on every
               third retry. you can disable redispatches with a value of 0.

  in http mode, if a server designated by a cookie is down, clients may
  definitely stick to it because they cannot flush the cookie, so they will not
  be able to access the service anymore.

  specifying "option redispatch" will allow the proxy to break their
  persistence and redistribute them to a working server.

  it also allows to retry connections to another server in case of multiple
  connection failures. of course, it requires having "retries" set to a nonzero
  value.

  this form is the preferred form, which replaces both the "redispatch" and
  "redisp" keywords.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "redispatch", "retries", "force-persist"

option redis-check
  use redis health checks for server testing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  it is possible to test that the server correctly talks redis protocol instead
  of just testing that it accepts the tcp connection. when this option is set,
  a ping redis command is sent to the server, and the response is analyzed to
  find the "+pong" response message.

  example :
        option redis-check

  see also : "option httpchk"

option smtpchk
option smtpchk <hello> <domain>
  use smtp health checks for server testing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <hello>   is an optional argument. it is the "hello" command to use. it can
              be either "helo" (for smtp) or "ehlo" (for estmp). all other
              values will be turned into the default command ("helo").

    <domain>  is the domain name to present to the server. it may only be
              specified (and is mandatory) if the hello command has been
              specified. by default, "localhost" is used.

  when "option smtpchk" is set, the health checks will consist in tcp
  connections followed by an smtp command. by default, this command is
  "helo localhost". the server's return code is analyzed and only return codes
  starting with a "2" will be considered as valid. all other responses,
  including a lack of response will constitute an error and will indicate a
  dead server.

  this test is meant to be used with smtp servers or relays. depending on the
  request, it is possible that some servers do not log each connection attempt,
  so you may want to experiment to improve the behaviour. using telnet on port
  25 is often easier than adjusting the configuration.

  most often, an incoming smtp server needs to see the client's ip address for
  various purposes, including spam filtering, anti-spoofing and logging. when
  possible, it is often wise to masquerade the client's ip address when
  connecting to the server using the "usesrc" argument of the "source" keyword,
  which requires the cttproxy feature to be compiled in.

  example :
        option smtpchk helo mydomain.org

  see also : "option httpchk", "source"

option socket-stats
no option socket-stats

  enable or disable collecting & providing separate statistics for each socket.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no

  arguments : none

option splice-auto
no option splice-auto
  enable or disable automatic kernel acceleration on sockets in both directions
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  when this option is enabled either on a frontend or on a backend, haproxy
  will automatically evaluate the opportunity to use kernel tcp splicing to
  forward data between the client and the server, in either direction. haproxy
  uses heuristics to estimate if kernel splicing might improve performance or
  not. both directions are handled independently. note that the heuristics used
  are not much aggressive in order to limit excessive use of splicing. this
  option requires splicing to be enabled at compile time, and may be globally
  disabled with the global option "nosplice". since splice uses pipes, using it
  requires that there are enough spare pipes.

  important note: kernel-based tcp splicing is a linux-specific feature which
  first appeared in kernel 2.6.25\. it offers kernel-based acceleration to
  transfer data between sockets without copying these data to user-space, thus
  providing noticeable performance gains and cpu cycles savings. since many
  early implementations are buggy, corrupt data and/or are inefficient, this
  feature is not enabled by default, and it should be used with extreme care.
  while it is not possible to detect the correctness of an implementation,
  2.6.29 is the first version offering a properly working implementation. in
  case of doubt, splicing may be globally disabled using the global "nosplice"
  keyword.

  example :
        option splice-auto

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option splice-request", "option splice-response", and global
             options "nosplice" and "maxpipes"

option splice-request
no option splice-request
  enable or disable automatic kernel acceleration on sockets for requests
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  when this option is enabled either on a frontend or on a backend, haproxy
  will use kernel tcp splicing whenever possible to forward data going from
  the client to the server. it might still use the recv/send scheme if there
  are no spare pipes left. this option requires splicing to be enabled at
  compile time, and may be globally disabled with the global option "nosplice".
  since splice uses pipes, using it requires that there are enough spare pipes.

  important note: see "option splice-auto" for usage limitations.

  example :
        option splice-request

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option splice-auto", "option splice-response", and global options
             "nosplice" and "maxpipes"

option splice-response
no option splice-response
  enable or disable automatic kernel acceleration on sockets for responses
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  when this option is enabled either on a frontend or on a backend, haproxy
  will use kernel tcp splicing whenever possible to forward data going from
  the server to the client. it might still use the recv/send scheme if there
  are no spare pipes left. this option requires splicing to be enabled at
  compile time, and may be globally disabled with the global option "nosplice".
  since splice uses pipes, using it requires that there are enough spare pipes.

  important note: see "option splice-auto" for usage limitations.

  example :
        option splice-response

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option splice-auto", "option splice-request", and global options
             "nosplice" and "maxpipes"

option srvtcpka
no option srvtcpka
  enable or disable the sending of tcp keepalive packets on the server side
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  when there is a firewall or any session-aware component between a client and
  a server, and when the protocol involves very long sessions with long idle
  periods (eg: remote desktops), there is a risk that one of the intermediate
  components decides to expire a session which has remained idle for too long.

  enabling socket-level tcp keep-alives makes the system regularly send packets
  to the other end of the connection, leaving it active. the delay between
  keep-alive probes is controlled by the system only and depends both on the
  operating system and its tuning parameters.

  it is important to understand that keep-alive packets are neither emitted nor
  received at the application level. it is only the network stacks which sees
  them. for this reason, even if one side of the proxy already uses keep-alives
  to maintain its connection alive, those keep-alive packets will not be
  forwarded to the other side of the proxy.

  please note that this has nothing to do with http keep-alive.

  using option "srvtcpka" enables the emission of tcp keep-alive probes on the
  server side of a connection, which should help when session expirations are
  noticed between haproxy and a server.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option clitcpka", "option tcpka"

option ssl-hello-chk
  use sslv3 client hello health checks for server testing
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  when some ssl-based protocols are relayed in tcp mode through haproxy, it is
  possible to test that the server correctly talks ssl instead of just testing
  that it accepts the tcp connection. when "option ssl-hello-chk" is set, pure
  sslv3 client hello messages are sent once the connection is established to
  the server, and the response is analyzed to find an ssl server hello message.
  the server is considered valid only when the response contains this server
  hello message.

  all servers tested till there correctly reply to sslv3 client hello messages,
  and most servers tested do not even log the requests containing only hello
  messages, which is appreciable.

  note that this check works even when ssl support was not built into haproxy
  because it forges the ssl message. when ssl support is available, it is best
  to use native ssl health checks instead of this one.

  see also: "option httpchk", "check-ssl"

option tcp-check
  perform health checks using tcp-check send/expect sequences
  may be used in sections:   defaults | frontend | listen | backend
                               yes    |    no    |   yes  |   yes

  this health check method is intended to be combined with "tcp-check" command
  lists in order to support send/expect types of health check sequences.

  tcp checks currently support 4 modes of operations :
    - no "tcp-check" directive : the health check only consists in a connection
      attempt, which remains the default mode.

    - "tcp-check send" or "tcp-check send-binary" only is mentioned : this is
      used to send a string along with a connection opening. with some
      protocols, it helps sending a "quit" message for example that prevents
      the server from logging a connection error for each health check. the
      check result will still be based on the ability to open the connection
      only.

    - "tcp-check expect" only is mentioned : this is used to test a banner.
      the connection is opened and haproxy waits for the server to present some
      contents which must validate some rules. the check result will be based
      on the matching between the contents and the rules. this is suited for
      pop, imap, smtp, ftp, ssh, telnet.

    - both "tcp-check send" and "tcp-check expect" are mentioned : this is
      used to test a hello-type protocol. haproxy sends a message, the server
      responds and its response is analysed. the check result will be based on
      the matching between the response contents and the rules. this is often
      suited for protocols which require a binding or a request/response model.
      ldap, mysql, redis and ssl are example of such protocols, though they
      already all have their dedicated checks with a deeper understanding of
      the respective protocols.
      in this mode, many questions may be sent and many answers may be
      analysed.

    a fifth mode can be used to insert comments in different steps of the
    script.

    for each tcp-check rule you create, you can add a "comment" directive,
    followed by a string. this string will be reported in the log and stderr
    in debug mode. it is useful to make user-friendly error reporting.
    the "comment" is of course optional.

  examples :
         # perform a pop check (analyse only server's banner)
         option tcp-check
         tcp-check expect string +ok\ pop3\ ready comment pop\ protocol

         # perform an imap check (analyse only server's banner)
         option tcp-check
         tcp-check expect string *\ ok\ imap4\ ready comment imap\ protocol

         # look for the redis master server after ensuring it speaks well
         # redis protocol, then it exits properly.
         # (send a command then analyse the response 3 times)
         option tcp-check
         tcp-check comment ping\ phase
         tcp-check send ping\r\n
         tcp-check expect +ponge
         tcp-check comment role\ check
         tcp-check send info\ replication\r\n
         tcp-check expect string role:master
         tcp-check comment quit\ phase
         tcp-check send quit\r\n
         tcp-check expect string +ok

         forge a http request, then analyse the response
         (send many headers before analyzing)
         option tcp-check
         tcp-check comment forge\ and\ send\ http\ request
         tcp-check send head\ /\ http/1.1\r\n
         tcp-check send host:\ www.mydomain.com\r\n
         tcp-check send user-agent:\ haproxy\ tcpcheck\r\n
         tcp-check send \r\n
         tcp-check expect rstring http/1\..\ (2..|3..) comment check\ http\ response

  see also : "tcp-check expect", "tcp-check send"

option tcp-smart-accept
no option tcp-smart-accept
  enable or disable the saving of one ack packet during the accept sequence
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |    no
  arguments : none

  when an http connection request comes in, the system acknowledges it on
  behalf of haproxy, then the client immediately sends its request, and the
  system acknowledges it too while it is notifying haproxy about the new
  connection. haproxy then reads the request and responds. this means that we
  have one tcp ack sent by the system for nothing, because the request could
  very well be acknowledged by haproxy when it sends its response.

  for this reason, in http mode, haproxy automatically asks the system to avoid
  sending this useless ack on platforms which support it (currently at least
  linux). it must not cause any problem, because the system will send it anyway
  after 40 ms if the response takes more time than expected to come.

  during complex network debugging sessions, it may be desirable to disable
  this optimization because delayed acks can make troubleshooting more complex
  when trying to identify where packets are delayed. it is then possible to
  fall back to normal behaviour by specifying "no option tcp-smart-accept".

  it is also possible to force it for non-http proxies by simply specifying
  "option tcp-smart-accept". for instance, it can make sense with some services
  such as smtp where the server speaks first.

  it is recommended to avoid forcing this option in a defaults section. in case
  of doubt, consider setting it back to automatic values by prepending the
  "default" keyword before it, or disabling it using the "no" keyword.

  see also : "option tcp-smart-connect"

option tcp-smart-connect
no option tcp-smart-connect
  enable or disable the saving of one ack packet during the connect sequence
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  on certain systems (at least linux), haproxy can ask the kernel not to
  immediately send an empty ack upon a connection request, but to directly
  send the buffer request instead. this saves one packet on the network and
  thus boosts performance. it can also be useful for some servers, because they
  immediately get the request along with the incoming connection.

  this feature is enabled when "option tcp-smart-connect" is set in a backend.
  it is not enabled by default because it makes network troubleshooting more
  complex.

  it only makes sense to enable it with protocols where the client speaks first
  such as http. in other situations, if there is no data to send in place of
  the ack, a normal ack is sent.

  if this option has been enabled in a "defaults" section, it can be disabled
  in a specific instance by prepending the "no" keyword before it.

  see also : "option tcp-smart-accept"

option tcpka
  enable or disable the sending of tcp keepalive packets on both sides
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  when there is a firewall or any session-aware component between a client and
  a server, and when the protocol involves very long sessions with long idle
  periods (eg: remote desktops), there is a risk that one of the intermediate
  components decides to expire a session which has remained idle for too long.

  enabling socket-level tcp keep-alives makes the system regularly send packets
  to the other end of the connection, leaving it active. the delay between
  keep-alive probes is controlled by the system only and depends both on the
  operating system and its tuning parameters.

  it is important to understand that keep-alive packets are neither emitted nor
  received at the application level. it is only the network stacks which sees
  them. for this reason, even if one side of the proxy already uses keep-alives
  to maintain its connection alive, those keep-alive packets will not be
  forwarded to the other side of the proxy.

  please note that this has nothing to do with http keep-alive.

  using option "tcpka" enables the emission of tcp keep-alive probes on both
  the client and server sides of a connection. note that this is meaningful
  only in "defaults" or "listen" sections. if this option is used in a
  frontend, only the client side will get keep-alives, and if this option is
  used in a backend, only the server side will get keep-alives. for this
  reason, it is strongly recommended to explicitly use "option clitcpka" and
  "option srvtcpka" when the configuration is split between frontends and
  backends.

  see also : "option clitcpka", "option srvtcpka"

option tcplog
  enable advanced logging of tcp connections with session state and timers
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  by default, the log output format is very poor, as it only contains the
  source and destination addresses, and the instance name. by specifying
  "option tcplog", each log line turns into a much richer format including, but
  not limited to, the connection timers, the session status, the connections
  numbers, the frontend, backend and server name, and of course the source
  address and ports. this option is useful for pure tcp proxies in order to
  find which of the client or server disconnects or times out. for normal http
  proxies, it's better to use "option httplog" which is even more complete.

  this option may be set either in the frontend or the backend.

  see also :  "option httplog", and section 8 about logging.

option transparent
no option transparent
  enable client-side transparent proxying
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  this option was introduced in order to provide layer 7 persistence to layer 3
  load balancers. the idea is to use the os's ability to redirect an incoming
  connection for a remote address to a local process (here haproxy), and let
  this process know what address was initially requested. when this option is
  used, sessions without cookies will be forwarded to the original destination
  ip address of the incoming request (which should match that of another
  equipment), while requests with cookies will still be forwarded to the
  appropriate server.

  note that contrary to a common belief, this option does not make haproxy
  present the client's ip to the server when establishing the connection.

  see also: the "usesrc" argument of the "source" keyword, and the
            "transparent" option of the "bind" keyword.

external-check command <command>
  executable to run when performing an external-check
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes

  arguments :
    <command> is the external command to run

  the arguments passed to the to the command are:

  <proxy_address> <proxy_port> <server_address> <server_port>

  the <proxy_address> and <proxy_port> are derived from the first listener
  that is either ipv4, ipv6 or a unix socket. in the case of a unix socket
  listener the proxy_address will be the path of the socket and the
  <proxy_port> will be the string "not_used". in a backend section, it's not
  possible to determine a listener, and both <proxy_address> and <proxy_port>
  will have the string value "not_used".

  some values are also provided through environment variables.

  environment variables :
    haproxy_proxy_addr      the first bind address if available (or empty if not
                            applicable, for example in a "backend" section).

    haproxy_proxy_id        the backend id.

    haproxy_proxy_name      the backend name.

    haproxy_proxy_port      the first bind port if available (or empty if not
                            applicable, for example in a "backend" section or
                            for a unix socket).

    haproxy_server_addr     the server address.

    haproxy_server_curconn  the current number of connections on the server.

    haproxy_server_id       the server id.

    haproxy_server_maxconn  the server max connections.

    haproxy_server_name     the server name.

    haproxy_server_port     the server port if available (or empty for a unix
                            socket).

    path                    the path environment variable used when executing
                            the command may be set using "external-check path".

  if the command executed and exits with a zero status then the check is
  considered to have passed, otherwise the check is considered to have
  failed.

  example :
        external-check command /bin/true

  see also : "external-check", "option external-check", "external-check path"

external-check path <path>
  the value of the path environment variable used when running an external-check
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes

  arguments :
    <path> is the path used when executing external command to run

  the default path is "".

  example :
        external-check path "/usr/bin:/bin"

  see also : "external-check", "option external-check",
             "external-check command"

persist rdp-cookie
persist rdp-cookie(<name>)
  enable rdp cookie-based persistence
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <name>    is the optional name of the rdp cookie to check. if omitted, the
              default cookie name "msts" will be used. there currently is no
              valid reason to change this name.

  this statement enables persistence based on an rdp cookie. the rdp cookie
  contains all information required to find the server in the list of known
  servers. so when this option is set in the backend, the request is analysed
  and if an rdp cookie is found, it is decoded. if it matches a known server
  which is still up (or if "option persist" is set), then the connection is
  forwarded to this server.

  note that this only makes sense in a tcp backend, but for this to work, the
  frontend must have waited long enough to ensure that an rdp cookie is present
  in the request buffer. this is the same requirement as with the "rdp-cookie"
  load-balancing method. thus it is highly recommended to put all statements in
  a single "listen" section.

  also, it is important to understand that the terminal server will emit this
  rdp cookie only if it is configured for "token redirection mode", which means
  that the "ip address redirection" option is disabled.

  example :
        listen tse-farm
            bind :3389
            # wait up to 5s for an rdp cookie in the request
            tcp-request inspect-delay 5s
            tcp-request content accept if rdp_cookie
            # apply rdp cookie persistence
            persist rdp-cookie
            # if server is unknown, let's balance on the same cookie.
            # alternatively, "balance leastconn" may be useful too.
            balance rdp-cookie
            server srv1 1.1.1.1:3389
            server srv2 1.1.1.2:3389

  see also : "balance rdp-cookie", "tcp-request", the "req_rdp_cookie" acl and
  the rdp_cookie pattern fetch function.

rate-limit sessions <rate>
  set a limit on the number of new sessions accepted per second on a frontend
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <rate>    the <rate> parameter is an integer designating the maximum number
              of new sessions per second to accept on the frontend.

  when the frontend reaches the specified number of new sessions per second, it
  stops accepting new connections until the rate drops below the limit again.
  during this time, the pending sessions will be kept in the socket's backlog
  (in system buffers) and haproxy will not even be aware that sessions are
  pending. when applying very low limit on a highly loaded service, it may make
  sense to increase the socket's backlog using the "backlog" keyword.

  this feature is particularly efficient at blocking connection-based attacks
  or service abuse on fragile servers. since the session rate is measured every
  millisecond, it is extremely accurate. also, the limit applies immediately,
  no delay is needed at all to detect the threshold.

  example : limit the connection rate on smtp to 10 per second max
        listen smtp
            mode tcp
            bind :25
            rate-limit sessions 10
            server 127.0.0.1:1025

  note : when the maximum rate is reached, the frontend's status is not changed
         but its sockets appear as "waiting" in the statistics if the
         "socket-stats" option is enabled.

  see also : the "backlog" keyword and the "fe_sess_rate" acl criterion.

redirect location <loc> [code <code>] <option> [{if | unless} <condition>]
redirect prefix   <pfx> [code <code>] <option> [{if | unless} <condition>]
redirect scheme   <sch> [code <code>] <option> [{if | unless} <condition>]
  return an http redirection if/unless a condition is matched
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes

  if/unless the condition is matched, the http request will lead to a redirect
  response. if no condition is specified, the redirect applies unconditionally.

  arguments :
    <loc>     with "redirect location", the exact value in <loc> is placed into
              the http "location" header. when used in an "http-request" rule,
              <loc> value follows the log-format rules and can include some
              dynamic values (see custom log format in section 8.2.4).

    <pfx>     with "redirect prefix", the "location" header is built from the
              concatenation of <pfx> and the complete uri path, including the
              query string, unless the "drop-query" option is specified (see
              below). as a special case, if <pfx> equals exactly "/", then
              nothing is inserted before the original uri. it allows one to
              redirect to the same url (for instance, to insert a cookie). when
              used in an "http-request" rule, <pfx> value follows the log-format
              rules and can include some dynamic values (see custom log format
              in section 8.2.4).

    <sch>     with "redirect scheme", then the "location" header is built by
              concatenating <sch> with "://" then the first occurrence of the
              "host" header, and then the uri path, including the query string
              unless the "drop-query" option is specified (see below). if no
              path is found or if the path is "*", then "/" is used instead. if
              no "host" header is found, then an empty host component will be
              returned, which most recent browsers interpret as redirecting to
              the same host. this directive is mostly used to redirect http to
              https. when used in an "http-request" rule, <sch> value follows
              the log-format rules and can include some dynamic values (see
              custom log format in section 8.2.4).

    <code>    the code is optional. it indicates which type of http redirection
              is desired. only codes 301, 302, 303, 307 and 308 are supported,
              with 302 used by default if no code is specified. 301 means
              "moved permanently", and a browser may cache the location. 302
              means "moved permanently" and means that the browser should not
              cache the redirection. 303 is equivalent to 302 except that the
              browser will fetch the location with a get method. 307 is just
              like 302 but makes it clear that the same method must be reused.
              likewise, 308 replaces 301 if the same method must be used.

    <option>  there are several options which can be specified to adjust the
              expected behaviour of a redirection :

      - "drop-query"
        when this keyword is used in a prefix-based redirection, then the
        location will be set without any possible query-string, which is useful
        for directing users to a non-secure page for instance. it has no effect
        with a location-type redirect.

      - "append-slash"
        this keyword may be used in conjunction with "drop-query" to redirect
        users who use a url not ending with a '/' to the same one with the '/'.
        it can be useful to ensure that search engines will only see one url.
        for this, a return code 301 is preferred.

      - "set-cookie name[=value]"
        a "set-cookie" header will be added with name (and optionally "=value")
        to the response. this is sometimes used to indicate that a user has
        been seen, for instance to protect against some types of dos. no other
        cookie option is added, so the cookie will be a session cookie. note
        that for a browser, a sole cookie name without an equal sign is
        different from a cookie with an equal sign.

      - "clear-cookie name[=]"
        a "set-cookie" header will be added with name (and optionally "="), but
        with the "max-age" attribute set to zero. this will tell the browser to
        delete this cookie. it is useful for instance on logout pages. it is
        important to note that clearing the cookie "name" will not remove a
        cookie set with "name=value". you have to clear the cookie "name=" for
        that, because the browser makes the difference.

  example: move the login url only to https.
        acl clear      dst_port  80
        acl secure     dst_port  8080
        acl login_page url_beg   /login
        acl logout     url_beg   /logout
        acl uid_given  url_reg   /login?userid=[^&]+
        acl cookie_set hdr_sub(cookie) seen=1

        redirect prefix   https://mysite.com set-cookie seen=1 if !cookie_set
        redirect prefix   https://mysite.com           if login_page !secure
        redirect prefix   http://mysite.com drop-query if login_page !uid_given
        redirect location http://mysite.com/           if !login_page secure
        redirect location / clear-cookie userid=       if logout

  example: send redirects for request for articles without a '/'.
        acl missing_slash path_reg ^/article/[^/]*$
        redirect code 301 prefix / drop-query append-slash if missing_slash

  example: redirect all http traffic to https when ssl is handled by haproxy.
        redirect scheme https if !{ ssl_fc }

  example: append 'www.' prefix in front of all hosts not having it
        http-request redirect code 301 location www.%[hdr(host)]%[req.uri] \
          unless { hdr_beg(host) -i www }

  see section 7 about acl usage.

redisp (deprecated)
redispatch (deprecated)
  enable or disable session redistribution in case of connection failure
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  in http mode, if a server designated by a cookie is down, clients may
  definitely stick to it because they cannot flush the cookie, so they will not
  be able to access the service anymore.

  specifying "redispatch" will allow the proxy to break their persistence and
  redistribute them to a working server.

  it also allows to retry last connection to another server in case of multiple
  connection failures. of course, it requires having "retries" set to a nonzero
  value.

  this form is deprecated, do not use it in any new configuration, use the new
  "option redispatch" instead.

  see also : "option redispatch"

reqadd  <string> [{if | unless} <cond>]
  add a header at the end of the http request
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <string>  is the complete line to be added. any space or known delimiter
              must be escaped using a backslash ('\'). please refer to section
              6 about http header manipulation for more information.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a new line consisting in <string> followed by a line feed will be added after
  the last header of an http request.

  header transformations only apply to traffic which passes through haproxy,
  and not to traffic generated by haproxy, such as health-checks or error
  responses.

  example : add "x-proto: ssl" to requests coming via port 81
     acl is-ssl  dst_port       81
     reqadd      x-proto:\ ssl  if is-ssl

  see also: "rspadd", section 6 about http header manipulation, and section 7
            about acls.

reqallow  <search> [{if | unless} <cond>]
reqiallow <search> [{if | unless} <cond>] (ignore case)
  definitely allow an http request if a line matches a regular expression
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              request line. this is an extended regular expression. parenthesis
              grouping is supported and no preliminary backslash is required.
              any space or known delimiter must be escaped using a backslash
              ('\'). the pattern applies to a full line at a time. the
              "reqallow" keyword strictly matches case while "reqiallow"
              ignores case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a request containing any line which matches extended regular expression
  <search> will mark the request as allowed, even if any later test would
  result in a deny. the test applies both to the request line and to request
  headers. keep in mind that urls in request line are case-sensitive while
  header names are not.

  it is easier, faster and more powerful to use acls to write access policies.
  reqdeny, reqallow and reqpass should be avoided in new designs.

  example :
     # allow www.* but refuse *.local
     reqiallow ^host:\ www\.
     reqideny  ^host:\ .*\.local

  see also: "reqdeny", "block", section 6 about http header manipulation, and
            section 7 about acls.

reqdel  <search> [{if | unless} <cond>]
reqidel <search> [{if | unless} <cond>]  (ignore case)
  delete all headers matching a regular expression in an http request
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              request line. this is an extended regular expression. parenthesis
              grouping is supported and no preliminary backslash is required.
              any space or known delimiter must be escaped using a backslash
              ('\'). the pattern applies to a full line at a time. the "reqdel"
              keyword strictly matches case while "reqidel" ignores case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  any header line matching extended regular expression <search> in the request
  will be completely deleted. most common use of this is to remove unwanted
  and/or dangerous headers or cookies from a request before passing it to the
  next servers.

  header transformations only apply to traffic which passes through haproxy,
  and not to traffic generated by haproxy, such as health-checks or error
  responses. keep in mind that header names are not case-sensitive.

  example :
     # remove x-forwarded-for header and server cookie
     reqidel ^x-forwarded-for:.*
     reqidel ^cookie:.*server=

  see also: "reqadd", "reqrep", "rspdel", section 6 about http header
            manipulation, and section 7 about acls.

reqdeny  <search> [{if | unless} <cond>]
reqideny <search> [{if | unless} <cond>]  (ignore case)
  deny an http request if a line matches a regular expression
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              request line. this is an extended regular expression. parenthesis
              grouping is supported and no preliminary backslash is required.
              any space or known delimiter must be escaped using a backslash
              ('\'). the pattern applies to a full line at a time. the
              "reqdeny" keyword strictly matches case while "reqideny" ignores
              case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a request containing any line which matches extended regular expression
  <search> will mark the request as denied, even if any later test would
  result in an allow. the test applies both to the request line and to request
  headers. keep in mind that urls in request line are case-sensitive while
  header names are not.

  a denied request will generate an "http 403 forbidden" response once the
  complete request has been parsed. this is consistent with what is practiced
  using acls.

  it is easier, faster and more powerful to use acls to write access policies.
  reqdeny, reqallow and reqpass should be avoided in new designs.

  example :
     # refuse *.local, then allow www.*
     reqideny  ^host:\ .*\.local
     reqiallow ^host:\ www\.

  see also: "reqallow", "rspdeny", "block", section 6 about http header
            manipulation, and section 7 about acls.

reqpass  <search> [{if | unless} <cond>]
reqipass <search> [{if | unless} <cond>]  (ignore case)
  ignore any http request line matching a regular expression in next rules
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              request line. this is an extended regular expression. parenthesis
              grouping is supported and no preliminary backslash is required.
              any space or known delimiter must be escaped using a backslash
              ('\'). the pattern applies to a full line at a time. the
              "reqpass" keyword strictly matches case while "reqipass" ignores
              case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a request containing any line which matches extended regular expression
  <search> will skip next rules, without assigning any deny or allow verdict.
  the test applies both to the request line and to request headers. keep in
  mind that urls in request line are case-sensitive while header names are not.

  it is easier, faster and more powerful to use acls to write access policies.
  reqdeny, reqallow and reqpass should be avoided in new designs.

  example :
     # refuse *.local, then allow www.*, but ignore "www.private.local"
     reqipass  ^host:\ www.private\.local
     reqideny  ^host:\ .*\.local
     reqiallow ^host:\ www\.

  see also: "reqallow", "reqdeny", "block", section 6 about http header
            manipulation, and section 7 about acls.

reqrep  <search> <string> [{if | unless} <cond>]
reqirep <search> <string> [{if | unless} <cond>]   (ignore case)
  replace a regular expression with a string in an http request line
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              request line. this is an extended regular expression. parenthesis
              grouping is supported and no preliminary backslash is required.
              any space or known delimiter must be escaped using a backslash
              ('\'). the pattern applies to a full line at a time. the "reqrep"
              keyword strictly matches case while "reqirep" ignores case.

    <string>  is the complete line to be added. any space or known delimiter
              must be escaped using a backslash ('\'). references to matched
              pattern groups are possible using the common \n form, with n
              being a single digit between 0 and 9\. please refer to section
              6 about http header manipulation for more information.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  any line matching extended regular expression <search> in the request (both
  the request line and header lines) will be completely replaced with <string>.
  most common use of this is to rewrite urls or domain names in "host" headers.

  header transformations only apply to traffic which passes through haproxy,
  and not to traffic generated by haproxy, such as health-checks or error
  responses. note that for increased readability, it is suggested to add enough
  spaces between the request and the response. keep in mind that urls in
  request line are case-sensitive while header names are not.

  example :
     # replace "/static/" with "/" at the beginning of any request path.
     reqrep ^([^\ :]*)\ /static/(.*)     \1\ /\2
     # replace "www.mydomain.com" with "www" in the host name.
     reqirep ^host:\ www.mydomain.com   host:\ www

  see also: "reqadd", "reqdel", "rsprep", "tune.bufsize", section 6 about
            http header manipulation, and section 7 about acls.

reqtarpit  <search> [{if | unless} <cond>]
reqitarpit <search> [{if | unless} <cond>]  (ignore case)
  tarpit an http request containing a line matching a regular expression
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              request line. this is an extended regular expression. parenthesis
              grouping is supported and no preliminary backslash is required.
              any space or known delimiter must be escaped using a backslash
              ('\'). the pattern applies to a full line at a time. the
              "reqtarpit" keyword strictly matches case while "reqitarpit"
              ignores case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a request containing any line which matches extended regular expression
  <search> will be tarpitted, which means that it will connect to nowhere, will
  be kept open for a pre-defined time, then will return an http error 500 so
  that the attacker does not suspect it has been tarpitted. the status 500 will
  be reported in the logs, but the completion flags will indicate "pt". the
  delay is defined by "timeout tarpit", or "timeout connect" if the former is
  not set.

  the goal of the tarpit is to slow down robots attacking servers with
  identifiable requests. many robots limit their outgoing number of connections
  and stay connected waiting for a reply which can take several minutes to
  come. depending on the environment and attack, it may be particularly
  efficient at reducing the load on the network and firewalls.

  examples :
     # ignore user-agents reporting any flavour of "mozilla" or "msie", but
     # block all others.
     reqipass   ^user-agent:\.*(mozilla|msie)
     reqitarpit ^user-agent:

     # block bad guys
     acl badguys src 10.1.0.3 172.16.13.20/28
     reqitarpit . if badguys

  see also: "reqallow", "reqdeny", "reqpass", section 6 about http header
            manipulation, and section 7 about acls.

retries <value>
  set the number of retries to perform on a server after a connection failure
  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <value>   is the number of times a connection attempt should be retried on
              a server when a connection either is refused or times out. the
              default value is 3.

  it is important to understand that this value applies to the number of
  connection attempts, not full requests. when a connection has effectively
  been established to a server, there will be no more retry.

  in order to avoid immediate reconnections to a server which is restarting,
  a turn-around timer of min("timeout connect", one second) is applied before
  a retry occurs.

  when "option redispatch" is set, the last retry may be performed on another
  server even if a cookie references a different server.

  see also : "option redispatch"

rspadd <string> [{if | unless} <cond>]
  add a header at the end of the http response
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <string>  is the complete line to be added. any space or known delimiter
              must be escaped using a backslash ('\'). please refer to section
              6 about http header manipulation for more information.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a new line consisting in <string> followed by a line feed will be added after
  the last header of an http response.

  header transformations only apply to traffic which passes through haproxy,
  and not to traffic generated by haproxy, such as health-checks or error
  responses.

  see also: "reqadd", section 6 about http header manipulation, and section 7
            about acls.

rspdel  <search> [{if | unless} <cond>]
rspidel <search> [{if | unless} <cond>]  (ignore case)
  delete all headers matching a regular expression in an http response
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              response line. this is an extended regular expression, so
              parenthesis grouping is supported and no preliminary backslash
              is required. any space or known delimiter must be escaped using
              a backslash ('\'). the pattern applies to a full line at a time.
              the "rspdel" keyword strictly matches case while "rspidel"
              ignores case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  any header line matching extended regular expression <search> in the response
  will be completely deleted. most common use of this is to remove unwanted
  and/or sensitive headers or cookies from a response before passing it to the
  client.

  header transformations only apply to traffic which passes through haproxy,
  and not to traffic generated by haproxy, such as health-checks or error
  responses. keep in mind that header names are not case-sensitive.

  example :
     # remove the server header from responses
     rspidel ^server:.*

  see also: "rspadd", "rsprep", "reqdel", section 6 about http header
            manipulation, and section 7 about acls.

rspdeny  <search> [{if | unless} <cond>]
rspideny <search> [{if | unless} <cond>]  (ignore case)
  block an http response if a line matches a regular expression
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              response line. this is an extended regular expression, so
              parenthesis grouping is supported and no preliminary backslash
              is required. any space or known delimiter must be escaped using
              a backslash ('\'). the pattern applies to a full line at a time.
              the "rspdeny" keyword strictly matches case while "rspideny"
              ignores case.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  a response containing any line which matches extended regular expression
  <search> will mark the request as denied. the test applies both to the
  response line and to response headers. keep in mind that header names are not
  case-sensitive.

  main use of this keyword is to prevent sensitive information leak and to
  block the response before it reaches the client. if a response is denied, it
  will be replaced with an http 502 error so that the client never retrieves
  any sensitive data.

  it is easier, faster and more powerful to use acls to write access policies.
  rspdeny should be avoided in new designs.

  example :
     # ensure that no content type matching ms-word will leak
     rspideny  ^content-type:\.*/ms-word

  see also: "reqdeny", "acl", "block", section 6 about http header manipulation
            and section 7 about acls.

rsprep  <search> <string> [{if | unless} <cond>]
rspirep <search> <string> [{if | unless} <cond>]  (ignore case)
  replace a regular expression with a string in an http response line
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <search>  is the regular expression applied to http headers and to the
              response line. this is an extended regular expression, so
              parenthesis grouping is supported and no preliminary backslash
              is required. any space or known delimiter must be escaped using
              a backslash ('\'). the pattern applies to a full line at a time.
              the "rsprep" keyword strictly matches case while "rspirep"
              ignores case.

    <string>  is the complete line to be added. any space or known delimiter
              must be escaped using a backslash ('\'). references to matched
              pattern groups are possible using the common \n form, with n
              being a single digit between 0 and 9\. please refer to section
              6 about http header manipulation for more information.

    <cond>    is an optional matching condition built from acls. it makes it
              possible to ignore this rule when other conditions are not met.

  any line matching extended regular expression <search> in the response (both
  the response line and header lines) will be completely replaced with
  <string>. most common use of this is to rewrite location headers.

  header transformations only apply to traffic which passes through haproxy,
  and not to traffic generated by haproxy, such as health-checks or error
  responses. note that for increased readability, it is suggested to add enough
  spaces between the request and the response. keep in mind that header names
  are not case-sensitive.

  example :
     # replace "location: 127.0.0.1:8080" with "location: www.mydomain.com"
     rspirep ^location:\ 127.0.0.1:8080    location:\ www.mydomain.com

  see also: "rspadd", "rspdel", "reqrep", section 6 about http header
            manipulation, and section 7 about acls.

server <name> <address>[:[port]] [param*]
  declare a server in a backend
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes
  arguments :
    <name>    is the internal name assigned to this server. this name will
              appear in logs and alerts.  if "http-send-name-header" is
              set, it will be added to the request header sent to the server.

    <address> is the ipv4 or ipv6 address of the server. alternatively, a
              resolvable hostname is supported, but this name will be resolved
              during start-up. address "0.0.0.0" or "*" has a special meaning.
              it indicates that the connection will be forwarded to the same ip
              address as the one from the client connection. this is useful in
              transparent proxy architectures where the client's connection is
              intercepted and haproxy must forward to the original destination
              address. this is more or less what the "transparent" keyword does
              except that with a server it's possible to limit concurrency and
              to report statistics. optionally, an address family prefix may be
              used before the address to force the family regardless of the
              address format, which can be useful to specify a path to a unix
              socket with no slash ('/'). currently supported prefixes are :
                    - 'ipv4@'  -> address is always ipv4
                    - 'ipv6@'  -> address is always ipv6
                    - 'unix@'  -> address is a path to a local unix socket
                    - 'abns@'  -> address is in abstract namespace (linux only)
              you may want to reference some environment variables in the
              address parameter, see section 2.3 about environment
              variables.

    <port>    is an optional port specification. if set, all connections will
              be sent to this port. if unset, the same port the client
              connected to will be used. the port may also be prefixed by a "+"
              or a "-". in this case, the server's port will be determined by
              adding this value to the client's port.

    <param*>  is a list of parameters for this server. the "server" keywords
              accepts an important number of options and has a complete section
              dedicated to it. please refer to section 5 for more details.

  examples :
        server first  10.1.1.1:1080 cookie first  check inter 1000
        server second 10.1.1.2:1080 cookie second check inter 1000
        server transp ipv4@
        server backup "${srv_backup}:1080" backup
        server www1_dc1 "${lan_dc1}.101:80"
        server www1_dc2 "${lan_dc2}.101:80"

  see also: "default-server", "http-send-name-header" and section 5 about
             server options

source <addr>[:<port>] [usesrc { <addr2>[:<port2>] | client | clientip } ]
source <addr>[:<port>] [usesrc { <addr2>[:<port2>] | hdr_ip(<hdr>[,<occ>]) } ]
source <addr>[:<port>] [interface <name>]
  set the source address for outgoing connections
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <addr>    is the ipv4 address haproxy will bind to before connecting to a
              server. this address is also used as a source for health checks.

              the default value of 0.0.0.0 means that the system will select
              the most appropriate address to reach its destination. optionally
              an address family prefix may be used before the address to force
              the family regardless of the address format, which can be useful
              to specify a path to a unix socket with no slash ('/'). currently
              supported prefixes are :
                - 'ipv4@' -> address is always ipv4
                - 'ipv6@' -> address is always ipv6
                - 'unix@' -> address is a path to a local unix socket
                - 'abns@' -> address is in abstract namespace (linux only)
              you may want to reference some environment variables in the address
              parameter, see section 2.3 about environment variables.

    <port>    is an optional port. it is normally not needed but may be useful
              in some very specific contexts. the default value of zero means
              the system will select a free port. note that port ranges are not
              supported in the backend. if you want to force port ranges, you
              have to specify them on each "server" line.

    <addr2>   is the ip address to present to the server when connections are
              forwarded in full transparent proxy mode. this is currently only
              supported on some patched linux kernels. when this address is
              specified, clients connecting to the server will be presented
              with this address, while health checks will still use the address
              <addr>.

    <port2>   is the optional port to present to the server when connections
              are forwarded in full transparent proxy mode (see <addr2> above).
              the default value of zero means the system will select a free
              port.

    <hdr>     is the name of a http header in which to fetch the ip to bind to.
              this is the name of a comma-separated header list which can
              contain multiple ip addresses. by default, the last occurrence is
              used. this is designed to work with the x-forwarded-for header
              and to automatically bind to the client's ip address as seen
              by previous proxy, typically stunnel. in order to use another
              occurrence from the last one, please see the <occ> parameter
              below. when the header (or occurrence) is not found, no binding
              is performed so that the proxy's default ip address is used. also
              keep in mind that the header name is case insensitive, as for any
              http header.

    <occ>     is the occurrence number of a value to be used in a multi-value
              header. this is to be used in conjunction with "hdr_ip(<hdr>)",
              in order to specify which occurrence to use for the source ip
              address. positive values indicate a position from the first
              occurrence, 1 being the first one. negative values indicate
              positions relative to the last one, -1 being the last one. this
              is helpful for situations where an x-forwarded-for header is set
              at the entry point of an infrastructure and must be used several
              proxy layers away. when this value is not specified, -1 is
              assumed. passing a zero here disables the feature.

    <name>    is an optional interface name to which to bind to for outgoing
              traffic. on systems supporting this features (currently, only
              linux), this allows one to bind all traffic to the server to
              this interface even if it is not the one the system would select
              based on routing tables. this should be used with extreme care.
              note that using this option requires root privileges.

  the "source" keyword is useful in complex environments where a specific
  address only is allowed to connect to the servers. it may be needed when a
  private address must be used through a public gateway for instance, and it is
  known that the system cannot determine the adequate source address by itself.

  an extension which is available on certain patched linux kernels may be used
  through the "usesrc" optional keyword. it makes it possible to connect to the
  servers with an ip address which does not belong to the system itself. this
  is called "full transparent proxy mode". for this to work, the destination
  servers have to route their traffic back to this address through the machine
  running haproxy, and ip forwarding must generally be enabled on this machine.

  in this "full transparent proxy" mode, it is possible to force a specific ip
  address to be presented to the servers. this is not much used in fact. a more
  common use is to tell haproxy to present the client's ip address. for this,
  there are two methods :

    - present the client's ip and port addresses. this is the most transparent
      mode, but it can cause problems when ip connection tracking is enabled on
      the machine, because a same connection may be seen twice with different
      states. however, this solution presents the huge advantage of not
      limiting the system to the 64k outgoing address+port couples, because all
      of the client ranges may be used.

    - present only the client's ip address and select a spare port. this
      solution is still quite elegant but slightly less transparent (downstream
      firewalls logs will not match upstream's). it also presents the downside
      of limiting the number of concurrent connections to the usual 64k ports.
      however, since the upstream and downstream ports are different, local ip
      connection tracking on the machine will not be upset by the reuse of the
      same session.

  note that depending on the transparent proxy technology used, it may be
  required to force the source address. in fact, cttproxy version 2 requires an
  ip address in <addr> above, and does not support setting of "0.0.0.0" as the
  ip address because it creates nat entries which much match the exact outgoing
  address. tproxy version 4 and some other kernel patches which work in pure
  forwarding mode generally will not have this limitation.

  this option sets the default source for all servers in the backend. it may
  also be specified in a "defaults" section. finer source address specification
  is possible at the server level using the "source" server option. refer to
  section 5 for more information.

  in order to work, "usesrc" requires root privileges.

  examples :
        backend private
            # connect to the servers using our 192.168.1.200 source address
            source 192.168.1.200

        backend transparent_ssl1
            # connect to the ssl farm from the client's source address
            source 192.168.1.200 usesrc clientip

        backend transparent_ssl2
            # connect to the ssl farm from the client's source address and port
            # not recommended if ip conntrack is present on the local machine.
            source 192.168.1.200 usesrc client

        backend transparent_ssl3
            # connect to the ssl farm from the client's source address. it
            # is more conntrack-friendly.
            source 192.168.1.200 usesrc clientip

        backend transparent_smtp
            # connect to the smtp farm from the client's source address/port
            # with tproxy version 4.
            source 0.0.0.0 usesrc clientip

        backend transparent_http
            # connect to the servers using the client's ip as seen by previous
            # proxy.
            source 0.0.0.0 usesrc hdr_ip(x-forwarded-for,-1)

  see also : the "source" server option in section 5, the tproxy patches for
             the linux kernel on www.balabit.com, the "bind" keyword.

srvtimeout <timeout> (deprecated)
  set the maximum inactivity time on the server side.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the inactivity timeout applies when the server is expected to acknowledge or
  send data. in http mode, this timeout is particularly important to consider
  during the first phase of the server's response, when it has to send the
  headers, as it directly represents the server's processing time for the
  request. to find out what value to put there, it's often good to start with
  what would be considered as unacceptable response times, then check the logs
  to observe the response time distribution, and adjust the value accordingly.

  the value is specified in milliseconds by default, but can be in any other
  unit if the number is suffixed by the unit, as specified at the top of this
  document. in tcp mode (and to a lesser extent, in http mode), it is highly
  recommended that the client timeout remains equal to the server timeout in
  order to avoid complex situations to debug. whatever the expected server
  response times, it is a good practice to cover at least one or several tcp
  packet losses by specifying timeouts that are slightly above multiples of 3
  seconds (eg: 4 or 5 seconds minimum).

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it. an unspecified timeout results in an infinite timeout, which
  is not recommended. such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of expired sessions in
  the system if the system's timeouts are not configured either.

  this parameter is provided for compatibility but is currently deprecated.
  please use "timeout server" instead.

  see also : "timeout server", "timeout tunnel", "timeout client" and
             "clitimeout".

stats admin { if | unless } <cond>
  enable statistics admin level if/unless a condition is matched
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes

  this statement enables the statistics admin level if/unless a condition is
  matched.

  the admin level allows to enable/disable servers from the web interface. by
  default, statistics page is read-only for security reasons.

  note : consider not using this feature in multi-process mode (nbproc > 1)
         unless you know what you do : memory is not shared between the
         processes, which can result in random behaviours.

  currently, the post request is limited to the buffer size minus the reserved
  buffer space, which means that if the list of servers is too long, the
  request won't be processed. it is recommended to alter few servers at a
  time.

  example :
    # statistics admin level only for localhost
    backend stats_localhost
        stats enable
        stats admin if localhost

  example :
    # statistics admin level always enabled because of the authentication
    backend stats_auth
        stats enable
        stats auth  admin:admin123
        stats admin if true

  example :
    # statistics admin level depends on the authenticated user
    userlist stats-auth
        group admin    users admin
        user  admin    insecure-password admin123
        group readonly users haproxy
        user  haproxy  insecure-password haproxy

    backend stats_auth
        stats enable
        acl auth       http_auth(stats-auth)
        acl auth_admin http_auth_group(stats-auth) admin
        stats http-request auth unless auth
        stats admin if auth_admin

  see also : "stats enable", "stats auth", "stats http-request", "nbproc",
             "bind-process", section 3.4 about userlists and section 7 about
             acl usage.

stats auth <user>:<passwd>
  enable statistics with authentication and grant access to an account
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <user>    is a user name to grant access to

    <passwd>  is the cleartext password associated to this user

  this statement enables statistics with default settings, and restricts access
  to declared users only. it may be repeated as many times as necessary to
  allow as many users as desired. when a user tries to access the statistics
  without a valid account, a "401 forbidden" response will be returned so that
  the browser asks the user to provide a valid user and password. the real
  which will be returned to the browser is configurable using "stats realm".

  since the authentication method is http basic authentication, the passwords
  circulate in cleartext on the network. thus, it was decided that the
  configuration file would also use cleartext passwords to remind the users
  that those ones should not be sensitive and not shared with any other account.

  it is also possible to reduce the scope of the proxies which appear in the
  report using "stats scope".

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats enable", "stats realm", "stats scope", "stats uri"

stats enable
  enable statistics reporting with default settings
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  this statement enables statistics reporting with default settings defined
  at build time. unless stated otherwise, these settings are used :
    - stats uri   : /haproxy?stats
    - stats realm : "haproxy statistics"
    - stats auth  : no authentication
    - stats scope : no restriction

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats auth", "stats realm", "stats uri"

stats hide-version
  enable statistics and hide haproxy version reporting
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  by default, the stats page reports some useful status information along with
  the statistics. among them is haproxy's version. however, it is generally
  considered dangerous to report precise version to anyone, as it can help them
  target known weaknesses with specific attacks. the "stats hide-version"
  statement removes the version from the statistics report. this is recommended
  for public sites or any site with a weak login/password.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats auth", "stats enable", "stats realm", "stats uri"

stats http-request { allow | deny | auth [realm <realm>] }
             [ { if | unless } <condition> ]
  access control for statistics

  may be used in sections:   defaults | frontend | listen | backend
                                no    |    no    |   yes  |   yes

  as "http-request", these set of options allow to fine control access to
  statistics. each option may be followed by if/unless and acl.
  first option with matched condition (or option without condition) is final.
  for "deny" a 403 error will be returned, for "allow" normal processing is
  performed, for "auth" a 401/407 error code is returned so the client
  should be asked to enter a username and password.

  there is no fixed limit to the number of http-request statements per
  instance.

  see also : "http-request", section 3.4 about userlists and section 7
             about acl usage.

stats realm <realm>
  enable statistics and set authentication realm
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <realm>   is the name of the http basic authentication realm reported to
              the browser. the browser uses it to display it in the pop-up
              inviting the user to enter a valid username and password.

  the realm is read as a single word, so any spaces in it should be escaped
  using a backslash ('\').

  this statement is useful only in conjunction with "stats auth" since it is
  only related to authentication.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats auth", "stats enable", "stats uri"

stats refresh <delay>
  enable statistics with automatic refresh
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <delay>   is the suggested refresh delay, specified in seconds, which will
              be returned to the browser consulting the report page. while the
              browser is free to apply any delay, it will generally respect it
              and refresh the page this every seconds. the refresh interval may
              be specified in any other non-default time unit, by suffixing the
              unit after the value, as explained at the top of this document.

  this statement is useful on monitoring displays with a permanent page
  reporting the load balancer's activity. when set, the html report page will
  include a link "refresh"/"stop refresh" so that the user can select whether
  he wants automatic refresh of the page or not.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats auth", "stats enable", "stats realm", "stats uri"

stats scope { <name> | "." }
  enable statistics and limit access scope
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <name>    is the name of a listen, frontend or backend section to be
              reported. the special name "." (a single dot) designates the
              section in which the statement appears.

  when this statement is specified, only the sections enumerated with this
  statement will appear in the report. all other ones will be hidden. this
  statement may appear as many times as needed if multiple sections need to be
  reported. please note that the name checking is performed as simple string
  comparisons, and that it is never checked that a give section name really
  exists.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats auth", "stats enable", "stats realm", "stats uri"

stats show-desc [ <desc> ]
  enable reporting of a description on the statistics page.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes

    <desc>    is an optional description to be reported. if unspecified, the
              description from global section is automatically used instead.

  this statement is useful for users that offer shared services to their
  customers, where node or description should be different for each customer.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.  by default description is not shown.

  example :
    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats show-desc master node for europe, asia, africa
        stats uri       /admin?stats
        stats refresh   5s

  see also: "show-node", "stats enable", "stats uri" and "description" in
            global section.

stats show-legends
  enable reporting additional information on the statistics page
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments : none

  enable reporting additional information on the statistics page :
    - cap: capabilities (proxy)
    - mode: one of tcp, http or health (proxy)
    - id: snmp id (proxy, socket, server)
    - ip (socket, server)
    - cookie (backend, server)

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.  default behaviour is not to show this information.

  see also: "stats enable", "stats uri".

stats show-node [ <name> ]
  enable reporting of a host name on the statistics page.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments:
    <name>    is an optional name to be reported. if unspecified, the
              node name from global section is automatically used instead.

  this statement is useful for users that offer shared services to their
  customers, where node or description might be different on a stats page
  provided for each customer.  default behaviour is not to show host name.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example:
    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats show-node europe-1
        stats uri       /admin?stats
        stats refresh   5s

  see also: "show-desc", "stats enable", "stats uri", and "node" in global
            section.

stats uri <prefix>
  enable statistics and define the uri prefix to access them
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <prefix>  is the prefix of any uri which will be redirected to stats. this
              prefix may contain a question mark ('?') to indicate part of a
              query string.

  the statistics uri is intercepted on the relayed traffic, so it appears as a
  page within the normal application. it is strongly advised to ensure that the
  selected uri will never appear in the application, otherwise it will never be
  possible to reach it in the application.

  the default uri compiled in haproxy is "/haproxy?stats", but this may be
  changed at build time, so it's better to always explicitly specify it here.
  it is generally a good idea to include a question mark in the uri so that
  intermediate proxies refrain from caching the results. also, since any string
  beginning with the prefix will be accepted as a stats request, the question
  mark helps ensuring that no valid uri will begin with the same words.

  it is sometimes very convenient to use "/" as the uri prefix, and put that
  statement in a "listen" instance of its own. that makes it easy to dedicate
  an address or a port to statistics only.

  though this statement alone is enough to enable statistics reporting, it is
  recommended to set all other settings in order to avoid relying on default
  unobvious parameters.

  example :
    # public access (limited to this backend only)
    backend public_www
        server srv1 192.168.0.1:80
        stats enable
        stats hide-version
        stats scope   .
        stats uri     /admin?stats
        stats realm   haproxy\ statistics
        stats auth    admin1:admin123
        stats auth    admin2:admin321

    # internal monitoring access (unlimited)
    backend private_monitoring
        stats enable
        stats uri     /admin?stats
        stats refresh 5s

  see also : "stats auth", "stats enable", "stats realm"

stick match <pattern> [table <table>] [{if | unless} <cond>]
  define a request pattern matching condition to stick a user to a server
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes

  arguments :
    <pattern>  is a sample expression rule as described in section 7.3\. it
               describes what elements of the incoming request or connection
               will be analysed in the hope to find a matching entry in a
               stickiness table. this rule is mandatory.

    <table>    is an optional stickiness table name. if unspecified, the same
               backend's table is used. a stickiness table is declared using
               the "stick-table" statement.

    <cond>     is an optional matching condition. it makes it possible to match
               on a certain criterion only when other conditions are met (or
               not met). for instance, it could be used to match on a source ip
               address except when a request passes through a known proxy, in
               which case we'd match on a header containing that ip address.

  some protocols or applications require complex stickiness rules and cannot
  always simply rely on cookies nor hashing. the "stick match" statement
  describes a rule to extract the stickiness criterion from an incoming request
  or connection. see section 7 for a complete list of possible patterns and
  transformation rules.

  the table has to be declared using the "stick-table" statement. it must be of
  a type compatible with the pattern. by default it is the one which is present
  in the same backend. it is possible to share a table with other backends by
  referencing it using the "table" keyword. if another table is referenced,
  the server's id inside the backends are used. by default, all server ids
  start at 1 in each backend, so the server ordering is enough. but in case of
  doubt, it is highly recommended to force server ids using their "id" setting.

  it is possible to restrict the conditions where a "stick match" statement
  will apply, using "if" or "unless" followed by a condition. see section 7 for
  acl based conditions.

  there is no limit on the number of "stick match" statements. the first that
  applies and matches will cause the request to be directed to the same server
  as was used for the request which created the entry. that way, multiple
  matches can be used as fallbacks.

  the stick rules are checked after the persistence cookies, so they will not
  affect stickiness if a cookie has already been used to select a server. that
  way, it becomes very easy to insert cookies and match on ip addresses in
  order to maintain stickiness between http and https.

  note : consider not using this feature in multi-process mode (nbproc > 1)
         unless you know what you do : memory is not shared between the
         processes, which can result in random behaviours.

  example :
    # forward smtp users to the same server they just used for pop in the
    # last 30 minutes
    backend pop
        mode tcp
        balance roundrobin
        stick store-request src
        stick-table type ip size 200k expire 30m
        server s1 192.168.1.1:110
        server s2 192.168.1.1:110

    backend smtp
        mode tcp
        balance roundrobin
        stick match src table pop
        server s1 192.168.1.1:25
        server s2 192.168.1.1:25

  see also : "stick-table", "stick on", "nbproc", "bind-process" and section 7
             about acls and samples fetching.

stick on <pattern> [table <table>] [{if | unless} <condition>]
  define a request pattern to associate a user to a server
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes

  note : this form is exactly equivalent to "stick match" followed by
         "stick store-request", all with the same arguments. please refer
         to both keywords for details. it is only provided as a convenience
         for writing more maintainable configurations.

  note : consider not using this feature in multi-process mode (nbproc > 1)
         unless you know what you do : memory is not shared between the
         processes, which can result in random behaviours.

  examples :
    # the following form ...
    stick on src table pop if !localhost

    # ...is strictly equivalent to this one :
    stick match src table pop if !localhost
    stick store-request src table pop if !localhost

    # use cookie persistence for http, and stick on source address for https as
    # well as http without cookie. share the same table between both accesses.
    backend http
        mode http
        balance roundrobin
        stick on src table https
        cookie srv insert indirect nocache
        server s1 192.168.1.1:80 cookie s1
        server s2 192.168.1.1:80 cookie s2

    backend https
        mode tcp
        balance roundrobin
        stick-table type ip size 200k expire 30m
        stick on src
        server s1 192.168.1.1:443
        server s2 192.168.1.1:443

  see also : "stick match", "stick store-request", "nbproc" and "bind-process".

stick store-request <pattern> [table <table>] [{if | unless} <condition>]
  define a request pattern used to create an entry in a stickiness table
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes

  arguments :
    <pattern>  is a sample expression rule as described in section 7.3\. it
               describes what elements of the incoming request or connection
               will be analysed, extracted and stored in the table once a
               server is selected.

    <table>    is an optional stickiness table name. if unspecified, the same
               backend's table is used. a stickiness table is declared using
               the "stick-table" statement.

    <cond>     is an optional storage condition. it makes it possible to store
               certain criteria only when some conditions are met (or not met).
               for instance, it could be used to store the source ip address
               except when the request passes through a known proxy, in which
               case we'd store a converted form of a header containing that ip
               address.

  some protocols or applications require complex stickiness rules and cannot
  always simply rely on cookies nor hashing. the "stick store-request" statement
  describes a rule to decide what to extract from the request and when to do
  it, in order to store it into a stickiness table for further requests to
  match it using the "stick match" statement. obviously the extracted part must
  make sense and have a chance to be matched in a further request. storing a
  client's ip address for instance often makes sense. storing an id found in a
  url parameter also makes sense. storing a source port will almost never make
  any sense because it will be randomly matched. see section 7 for a complete
  list of possible patterns and transformation rules.

  the table has to be declared using the "stick-table" statement. it must be of
  a type compatible with the pattern. by default it is the one which is present
  in the same backend. it is possible to share a table with other backends by
  referencing it using the "table" keyword. if another table is referenced,
  the server's id inside the backends are used. by default, all server ids
  start at 1 in each backend, so the server ordering is enough. but in case of
  doubt, it is highly recommended to force server ids using their "id" setting.

  it is possible to restrict the conditions where a "stick store-request"
  statement will apply, using "if" or "unless" followed by a condition. this
  condition will be evaluated while parsing the request, so any criteria can be
  used. see section 7 for acl based conditions.

  there is no limit on the number of "stick store-request" statements, but
  there is a limit of 8 simultaneous stores per request or response. this
  makes it possible to store up to 8 criteria, all extracted from either the
  request or the response, regardless of the number of rules. only the 8 first
  ones which match will be kept. using this, it is possible to feed multiple
  tables at once in the hope to increase the chance to recognize a user on
  another protocol or access method. using multiple store-request rules with
  the same table is possible and may be used to find the best criterion to rely
  on, by arranging the rules by decreasing preference order. only the first
  extracted criterion for a given table will be stored. all subsequent store-
  request rules referencing the same table will be skipped and their acls will
  not be evaluated.

  the "store-request" rules are evaluated once the server connection has been
  established, so that the table will contain the real server that processed
  the request.

  note : consider not using this feature in multi-process mode (nbproc > 1)
         unless you know what you do : memory is not shared between the
         processes, which can result in random behaviours.

  example :
    # forward smtp users to the same server they just used for pop in the
    # last 30 minutes
    backend pop
        mode tcp
        balance roundrobin
        stick store-request src
        stick-table type ip size 200k expire 30m
        server s1 192.168.1.1:110
        server s2 192.168.1.1:110

    backend smtp
        mode tcp
        balance roundrobin
        stick match src table pop
        server s1 192.168.1.1:25
        server s2 192.168.1.1:25

  see also : "stick-table", "stick on", "nbproc", "bind-process" and section 7
             about acls and sample fetching.

stick-table type {ip | integer | string [len <length>] | binary [len <length>]}
            size <size> [expire <expire>] [nopurge] [peers <peersect>]
            [store <data_type>]*
  configure the stickiness table for the current section
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes

  arguments :
    ip         a table declared with "type ip" will only store ipv4 addresses.
               this form is very compact (about 50 bytes per entry) and allows
               very fast entry lookup and stores with almost no overhead. this
               is mainly used to store client source ip addresses.

    ipv6       a table declared with "type ipv6" will only store ipv6 addresses.
               this form is very compact (about 60 bytes per entry) and allows
               very fast entry lookup and stores with almost no overhead. this
               is mainly used to store client source ip addresses.

    integer    a table declared with "type integer" will store 32bit integers
               which can represent a client identifier found in a request for
               instance.

    string     a table declared with "type string" will store substrings of up
               to <len> characters. if the string provided by the pattern
               extractor is larger than <len>, it will be truncated before
               being stored. during matching, at most <len> characters will be
               compared between the string in the table and the extracted
               pattern. when not specified, the string is automatically limited
               to 32 characters.

    binary     a table declared with "type binary" will store binary blocks
               of <len> bytes. if the block provided by the pattern
               extractor is larger than <len>, it will be truncated before
               being stored. if the block provided by the sample expression
               is shorter than <len>, it will be padded by 0\. when not
               specified, the block is automatically limited to 32 bytes.

    <length>   is the maximum number of characters that will be stored in a
               "string" type table (see type "string" above). or the number
               of bytes of the block in "binary" type table. be careful when
               changing this parameter as memory usage will proportionally
               increase.

    <size>     is the maximum number of entries that can fit in the table. this
               value directly impacts memory usage. count approximately
               50 bytes per entry, plus the size of a string if any. the size
               supports suffixes "k", "m", "g" for 2^10, 2^20 and 2^30 factors.

    [nopurge]  indicates that we refuse to purge older entries when the table
               is full. when not specified and the table is full when haproxy
               wants to store an entry in it, it will flush a few of the oldest
               entries in order to release some space for the new ones. this is
               most often the desired behaviour. in some specific cases, it
               be desirable to refuse new entries instead of purging the older
               ones. that may be the case when the amount of data to store is
               far above the hardware limits and we prefer not to offer access
               to new clients than to reject the ones already connected. when
               using this parameter, be sure to properly set the "expire"
               parameter (see below).

    <peersect> is the name of the peers section to use for replication. entries
               which associate keys to server ids are kept synchronized with
               the remote peers declared in this section. all entries are also
               automatically learned from the local peer (old process) during a
               soft restart.

               note : each peers section may be referenced only by tables
                      belonging to the same unique process.

    <expire>   defines the maximum duration of an entry in the table since it
               was last created, refreshed or matched. the expiration delay is
               defined using the standard time format, similarly as the various
               timeouts. the maximum duration is slightly above 24 days. see
               section 2.2 for more information. if this delay is not specified,
               the session won't automatically expire, but older entries will
               be removed once full. be sure not to use the "nopurge" parameter
               if not expiration delay is specified.

   <data_type> is used to store additional information in the stick-table. this
               may be used by acls in order to control various criteria related
               to the activity of the client matching the stick-table. for each
               item specified here, the size of each entry will be inflated so
               that the additional data can fit. several data types may be
               stored with an entry. multiple data types may be specified after
               the "store" keyword, as a comma-separated list. alternatively,
               it is possible to repeat the "store" keyword followed by one or
               several data types. except for the "server_id" type which is
               automatically detected and enabled, all data types must be
               explicitly declared to be stored. if an acl references a data
               type which is not stored, the acl will simply not match. some
               data types require an argument which must be passed just after
               the type between parenthesis. see below for the supported data
               types and their arguments.

  the data types that can be stored with an entry are the following :
    - server_id : this is an integer which holds the numeric id of the server a
      request was assigned to. it is used by the "stick match", "stick store",
      and "stick on" rules. it is automatically enabled when referenced.

    - gpc0 : first general purpose counter. it is a positive 32-bit integer
      integer which may be used for anything. most of the time it will be used
      to put a special tag on some entries, for instance to note that a
      specific behaviour was detected and must be known for future matches.

    - gpc0_rate(<period>) : increment rate of the first general purpose counter
      over a period. it is a positive 32-bit integer integer which may be used
      for anything. just like <gpc0>, it counts events, but instead of keeping
      a cumulative count, it maintains the rate at which the counter is
      incremented. most of the time it will be used to measure the frequency of
      occurrence of certain events (eg: requests to a specific url).

    - conn_cnt : connection count. it is a positive 32-bit integer which counts
      the absolute number of connections received from clients which matched
      this entry. it does not mean the connections were accepted, just that
      they were received.

    - conn_cur : current connections. it is a positive 32-bit integer which
      stores the concurrent connection counts for the entry. it is incremented
      once an incoming connection matches the entry, and decremented once the
      connection leaves. that way it is possible to know at any time the exact
      number of concurrent connections for an entry.

    - conn_rate(<period>) : frequency counter (takes 12 bytes). it takes an
      integer parameter <period> which indicates in milliseconds the length
      of the period over which the average is measured. it reports the average
      incoming connection rate over that period, in connections per period. the
      result is an integer which can be matched using acls.

    - sess_cnt : session count. it is a positive 32-bit integer which counts
      the absolute number of sessions received from clients which matched this
      entry. a session is a connection that was accepted by the layer 4 rules.

    - sess_rate(<period>) : frequency counter (takes 12 bytes). it takes an
      integer parameter <period> which indicates in milliseconds the length
      of the period over which the average is measured. it reports the average
      incoming session rate over that period, in sessions per period. the
      result is an integer which can be matched using acls.

    - http_req_cnt : http request count. it is a positive 32-bit integer which
      counts the absolute number of http requests received from clients which
      matched this entry. it does not matter whether they are valid requests or
      not. note that this is different from sessions when keep-alive is used on
      the client side.

    - http_req_rate(<period>) : frequency counter (takes 12 bytes). it takes an
      integer parameter <period> which indicates in milliseconds the length
      of the period over which the average is measured. it reports the average
      http request rate over that period, in requests per period. the result is
      an integer which can be matched using acls. it does not matter whether
      they are valid requests or not. note that this is different from sessions
      when keep-alive is used on the client side.

    - http_err_cnt : http error count. it is a positive 32-bit integer which
      counts the absolute number of http requests errors induced by clients
      which matched this entry. errors are counted on invalid and truncated
      requests, as well as on denied or tarpitted requests, and on failed
      authentications. if the server responds with 4xx, then the request is
      also counted as an error since it's an error triggered by the client
      (eg: vulnerability scan).

    - http_err_rate(<period>) : frequency counter (takes 12 bytes). it takes an
      integer parameter <period> which indicates in milliseconds the length
      of the period over which the average is measured. it reports the average
      http request error rate over that period, in requests per period (see
      http_err_cnt above for what is accounted as an error). the result is an
      integer which can be matched using acls.

    - bytes_in_cnt : client to server byte count. it is a positive 64-bit
      integer which counts the cumulated amount of bytes received from clients
      which matched this entry. headers are included in the count. this may be
      used to limit abuse of upload features on photo or video servers.

    - bytes_in_rate(<period>) : frequency counter (takes 12 bytes). it takes an
      integer parameter <period> which indicates in milliseconds the length
      of the period over which the average is measured. it reports the average
      incoming bytes rate over that period, in bytes per period. it may be used
      to detect users which upload too much and too fast. warning: with large
      uploads, it is possible that the amount of uploaded data will be counted
      once upon termination, thus causing spikes in the average transfer speed
      instead of having a smooth one. this may partially be smoothed with
      "option contstats" though this is not perfect yet. use of byte_in_cnt is
      recommended for better fairness.

    - bytes_out_cnt : server to client byte count. it is a positive 64-bit
      integer which counts the cumulated amount of bytes sent to clients which
      matched this entry. headers are included in the count. this may be used
      to limit abuse of bots sucking the whole site.

    - bytes_out_rate(<period>) : frequency counter (takes 12 bytes). it takes
      an integer parameter <period> which indicates in milliseconds the length
      of the period over which the average is measured. it reports the average
      outgoing bytes rate over that period, in bytes per period. it may be used
      to detect users which download too much and too fast. warning: with large
      transfers, it is possible that the amount of transferred data will be
      counted once upon termination, thus causing spikes in the average
      transfer speed instead of having a smooth one. this may partially be
      smoothed with "option contstats" though this is not perfect yet. use of
      byte_out_cnt is recommended for better fairness.

  there is only one stick-table per proxy. at the moment of writing this doc,
  it does not seem useful to have multiple tables per proxy. if this happens
  to be required, simply create a dummy backend with a stick-table in it and
  reference it.

  it is important to understand that stickiness based on learning information
  has some limitations, including the fact that all learned associations are
  lost upon restart. in general it can be good as a complement but not always
  as an exclusive stickiness.

  last, memory requirements may be important when storing many data types.
  indeed, storing all indicators above at once in each entry requires 116 bytes
  per entry, or 116 mb for a 1-million entries table. this is definitely not
  something that can be ignored.

  example:
        # keep track of counters of up to 1 million ip addresses over 5 minutes
        # and store a general purpose counter and the average connection rate
        # computed over a sliding window of 30 seconds.
        stick-table type ip size 1m expire 5m store gpc0,conn_rate(30s)

  see also : "stick match", "stick on", "stick store-request", section 2.2
             about time format and section 7 about acls.

stick store-response <pattern> [table <table>] [{if | unless} <condition>]
  define a request pattern used to create an entry in a stickiness table
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes

  arguments :
    <pattern>  is a sample expression rule as described in section 7.3\. it
               describes what elements of the response or connection will
               be analysed, extracted and stored in the table once a
               server is selected.

    <table>    is an optional stickiness table name. if unspecified, the same
               backend's table is used. a stickiness table is declared using
               the "stick-table" statement.

    <cond>     is an optional storage condition. it makes it possible to store
               certain criteria only when some conditions are met (or not met).
               for instance, it could be used to store the ssl session id only
               when the response is a ssl server hello.

  some protocols or applications require complex stickiness rules and cannot
  always simply rely on cookies nor hashing. the "stick store-response"
  statement  describes a rule to decide what to extract from the response and
  when to do it, in order to store it into a stickiness table for further
  requests to match it using the "stick match" statement. obviously the
  extracted part must make sense and have a chance to be matched in a further
  request. storing an id found in a header of a response makes sense.
  see section 7 for a complete list of possible patterns and transformation
  rules.

  the table has to be declared using the "stick-table" statement. it must be of
  a type compatible with the pattern. by default it is the one which is present
  in the same backend. it is possible to share a table with other backends by
  referencing it using the "table" keyword. if another table is referenced,
  the server's id inside the backends are used. by default, all server ids
  start at 1 in each backend, so the server ordering is enough. but in case of
  doubt, it is highly recommended to force server ids using their "id" setting.

  it is possible to restrict the conditions where a "stick store-response"
  statement will apply, using "if" or "unless" followed by a condition. this
  condition will be evaluated while parsing the response, so any criteria can
  be used. see section 7 for acl based conditions.

  there is no limit on the number of "stick store-response" statements, but
  there is a limit of 8 simultaneous stores per request or response. this
  makes it possible to store up to 8 criteria, all extracted from either the
  request or the response, regardless of the number of rules. only the 8 first
  ones which match will be kept. using this, it is possible to feed multiple
  tables at once in the hope to increase the chance to recognize a user on
  another protocol or access method. using multiple store-response rules with
  the same table is possible and may be used to find the best criterion to rely
  on, by arranging the rules by decreasing preference order. only the first
  extracted criterion for a given table will be stored. all subsequent store-
  response rules referencing the same table will be skipped and their acls will
  not be evaluated. however, even if a store-request rule references a table, a
  store-response rule may also use the same table. this means that each table
  may learn exactly one element from the request and one element from the
  response at once.

  the table will contain the real server that processed the request.

  example :
    # learn ssl session id from both request and response and create affinity.
    backend https
        mode tcp
        balance roundrobin
        # maximum ssl session id length is 32 bytes.
        stick-table type binary len 32 size 30k expire 30m

        acl clienthello req_ssl_hello_type 1
        acl serverhello rep_ssl_hello_type 2

        # use tcp content accepts to detects ssl client and server hello.
        tcp-request inspect-delay 5s
        tcp-request content accept if clienthello

        # no timeout on response inspect delay by default.
        tcp-response content accept if serverhello

        # ssl session id (sslid) may be present on a client or server hello.
        # its length is coded on 1 byte at offset 43 and its value starts
        # at offset 44.

        # match and learn on request if client hello.
        stick on payload_lv(43,1) if clienthello

        # learn on response if server hello.
        stick store-response payload_lv(43,1) if serverhello

        server s1 192.168.1.1:443
        server s2 192.168.1.1:443

  see also : "stick-table", "stick on", and section 7 about acls and pattern
             extraction.

tcp-check connect [params*]
  opens a new connection
  may be used in sections:   defaults | frontend | listen | backend
                               no     |    no    |   yes  |   yes

  when an application lies on more than a single tcp port or when haproxy
  load-balance many services in a single backend, it makes sense to probe all
  the services individually before considering a server as operational.

  when there are no tcp port configured on the server line neither server port
  directive, then the 'tcp-check connect port <port>' must be the first step
  of the sequence.

  in a tcp-check ruleset a 'connect' is required, it is also mandatory to start
  the ruleset with a 'connect' rule. purpose is to ensure admin know what they
  do.

  parameters :
    they are optional and can be used to describe how haproxy should open and
    use the tcp connection.

    port      if not set, check port or server port is used.
              it tells haproxy where to open the connection to.
              <port> must be a valid tcp port source integer, from 1 to 65535.

    send-proxy   send a proxy protocol string

    ssl          opens a ciphered connection

    examples:
         # check http and https services on a server.
         # first open port 80 thanks to server line port directive, then
         # tcp-check opens port 443, ciphered and run a request on it:
         option tcp-check
         tcp-check connect
         tcp-check send get\ /\ http/1.0\r\n
         tcp-check send host:\ haproxy.1wt.eu\r\n
         tcp-check send \r\n
         tcp-check expect rstring (2..|3..)
         tcp-check connect port 443 ssl
         tcp-check send get\ /\ http/1.0\r\n
         tcp-check send host:\ haproxy.1wt.eu\r\n
         tcp-check send \r\n
         tcp-check expect rstring (2..|3..)
         server www 10.0.0.1 check port 80

         # check both pop and imap from a single server:
         option tcp-check
         tcp-check connect port 110
         tcp-check expect string +ok\ pop3\ ready
         tcp-check connect port 143
         tcp-check expect string *\ ok\ imap4\ ready
         server mail 10.0.0.1 check

  see also : "option tcp-check", "tcp-check send", "tcp-check expect"

tcp-check expect [!] <match> <pattern>
  specify data to be collected and analysed during a generic health check
  may be used in sections:   defaults | frontend | listen | backend
                               no     |    no    |   yes  |   yes

  arguments :
    <match>   is a keyword indicating how to look for a specific pattern in the
              response. the keyword may be one of "string", "rstring" or
              binary.
              the keyword may be preceded by an exclamation mark ("!") to negate
              the match. spaces are allowed between the exclamation mark and the
              keyword. see below for more details on the supported keywords.

    <pattern> is the pattern to look for. it may be a string or a regular
              expression. if the pattern contains spaces, they must be escaped
              with the usual backslash ('\').
              if the match is set to binary, then the pattern must be passed as
              a serie of hexadecimal digits in an even number. each sequence of
              two digits will represent a byte. the hexadecimal digits may be
              used upper or lower case.

  the available matches are intentionally similar to their http-check cousins :

    string <string> : test the exact string matches in the response buffer.
                      a health check response will be considered valid if the
                      response's buffer contains this exact string. if the
                      "string" keyword is prefixed with "!", then the response
                      will be considered invalid if the body contains this
                      string. this can be used to look for a mandatory pattern
                      in a protocol response, or to detect a failure when a
                      specific error appears in a protocol banner.

    rstring <regex> : test a regular expression on the response buffer.
                      a health check response will be considered valid if the
                      response's buffer matches this expression. if the
                      "rstring" keyword is prefixed with "!", then the response
                      will be considered invalid if the body matches the
                      expression.

    binary <hexstring> : test the exact string in its hexadecimal form matches
                         in the response buffer. a health check response will
                         be considered valid if the response's buffer contains
                         this exact hexadecimal string.
                         purpose is to match data on binary protocols.

  it is important to note that the responses will be limited to a certain size
  defined by the global "tune.chksize" option, which defaults to 16384 bytes.
  thus, too large responses may not contain the mandatory pattern when using
  "string", "rstring" or binary. if a large response is absolutely required, it
  is possible to change the default max size by setting the global variable.
  however, it is worth keeping in mind that parsing very large responses can
  waste some cpu cycles, especially when regular expressions are used, and that
  it is always better to focus the checks on smaller resources. also, in its
  current state, the check will not find any string nor regex past a null
  character in the response. similarly it is not possible to request matching
  the null character.

  examples :
         # perform a pop check
         option tcp-check
         tcp-check expect string +ok\ pop3\ ready

         # perform an imap check
         option tcp-check
         tcp-check expect string *\ ok\ imap4\ ready

         # look for the redis master server
         option tcp-check
         tcp-check send ping\r\n
         tcp-check expect +pong
         tcp-check send info\ replication\r\n
         tcp-check expect string role:master
         tcp-check send quit\r\n
         tcp-check expect string +ok

  see also : "option tcp-check", "tcp-check connect", "tcp-check send",
             "tcp-check send-binary", "http-check expect", tune.chksize

tcp-check send <data>
  specify a string to be sent as a question during a generic health check
  may be used in sections:   defaults | frontend | listen | backend
                               no     |    no    |   yes  |   yes

    <data> : the data to be sent as a question during a generic health check
             session. for now, <data> must be a string.

  examples :
         # look for the redis master server
         option tcp-check
         tcp-check send info\ replication\r\n
         tcp-check expect string role:master

  see also : "option tcp-check", "tcp-check connect", "tcp-check expect",
             "tcp-check send-binary", tune.chksize

tcp-check send-binary <hexastring>
  specify an hexa digits string to be sent as a binary question during a raw
  tcp health check
  may be used in sections:   defaults | frontend | listen | backend
                               no     |    no    |   yes  |   yes

    <data> : the data to be sent as a question during a generic health check
             session. for now, <data> must be a string.
    <hexastring> : test the exact string in its hexadecimal form matches in the
                   response buffer. a health check response will be considered
                   valid if the response's buffer contains this exact
                   hexadecimal string.
                   purpose is to send binary data to ask on binary protocols.

  examples :
         # redis check in binary
         option tcp-check
         tcp-check send-binary 50494e470d0a # ping\r\n
         tcp-check expect binary 2b504f4e47 # +pong

  see also : "option tcp-check", "tcp-check connect", "tcp-check expect",
             "tcp-check send", tune.chksize

tcp-request connection <action> [{if | unless} <condition>]
  perform an action on an incoming connection depending on a layer 4 condition
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   no
  arguments :
    <action>    defines the action to perform if the condition applies. valid
                actions include : "accept", "reject", "track-sc0", "track-sc1",
                "track-sc2", and "expect-proxy". see below for more details.

    <condition> is a standard layer4-only acl-based condition (see section 7).

  immediately after acceptance of a new incoming connection, it is possible to
  evaluate some conditions to decide whether this connection must be accepted
  or dropped or have its counters tracked. those conditions cannot make use of
  any data contents because the connection has not been read from yet, and the
  buffers are not yet allocated. this is used to selectively and very quickly
  accept or drop connections from various sources with a very low overhead. if
  some contents need to be inspected in order to take the decision, the
  "tcp-request content" statements must be used instead.

  the "tcp-request connection" rules are evaluated in their exact declaration
  order. if no rule matches or if there is no rule, the default action is to
  accept the incoming connection. there is no specific limit to the number of
  rules which may be inserted.

  four types of actions are supported :
    - accept :
        accepts the connection if the condition is true (when used with "if")
        or false (when used with "unless"). the first such rule executed ends
        the rules evaluation.

    - reject :
        rejects the connection if the condition is true (when used with "if")
        or false (when used with "unless"). the first such rule executed ends
        the rules evaluation. rejected connections do not even become a
        session, which is why they are accounted separately for in the stats,
        as "denied connections". they are not considered for the session
        rate-limit and are not logged either. the reason is that these rules
        should only be used to filter extremely high connection rates such as
        the ones encountered during a massive ddos attack. under these extreme
        conditions, the simple action of logging each event would make the
        system collapse and would considerably lower the filtering capacity. if
        logging is absolutely desired, then "tcp-request content" rules should
        be used instead.

    - expect-proxy layer4 :
        configures the client-facing connection to receive a proxy protocol
        header before any byte is read from the socket. this is equivalent to
        having the "accept-proxy" keyword on the "bind" line, except that using
        the tcp rule allows the proxy protocol to be accepted only for certain
        ip address ranges using an acl. this is convenient when multiple layers
        of load balancers are passed through by traffic coming from public
        hosts.

    - capture <sample> len <length> :
        this only applies to "tcp-request content" rules. it captures sample
        expression <sample> from the request buffer, and converts it to a
        string of at most <len> characters. the resulting string is stored into
        the next request "capture" slot, so it will possibly appear next to
        some captured http headers. it will then automatically appear in the
        logs, and it will be possible to extract it using sample fetch rules to
        feed it into headers or anything. the length should be limited given
        that this size will be allocated for each capture during the whole
        session life. please check section 7.3 (fetching samples) and "capture
        request header" for more information.

    - { track-sc0 | track-sc1 | track-sc2 } <key> [table <table>] :
        enables tracking of sticky counters from current connection. these
        rules do not stop evaluation and do not change default action. 3 sets
        of counters may be simultaneously tracked by the same connection. the
        first "track-sc0" rule executed enables tracking of the counters of the
        specified table as the first set. the first "track-sc1" rule executed
        enables tracking of the counters of the specified table as the second
        set. the first "track-sc2" rule executed enables tracking of the
        counters of the specified table as the third set. it is a recommended
        practice to use the first set of counters for the per-frontend counters
        and the second set for the per-backend ones. but this is just a
        guideline, all may be used everywhere.

        these actions take one or two arguments :
          <key>   is mandatory, and is a sample expression rule as described
                  in section 7.3\. it describes what elements of the incoming
                  request or connection will be analysed, extracted, combined,
                  and used to select which table entry to update the counters.
                  note that "tcp-request connection" cannot use content-based
                  fetches.

         <table>  is an optional table to be used instead of the default one,
                  which is the stick-table declared in the current proxy. all
                  the counters for the matches and updates for the key will
                  then be performed in that table until the session ends.

        once a "track-sc*" rule is executed, the key is looked up in the table
        and if it is not found, an entry is allocated for it. then a pointer to
        that entry is kept during all the session's life, and this entry's
        counters are updated as often as possible, every time the session's
        counters are updated, and also systematically when the session ends.
        counters are only updated for events that happen after the tracking has
        been started. for example, connection counters will not be updated when
        tracking layer 7 information, since the connection event happens before
        layer7 information is extracted.

        if the entry tracks concurrent connection counters, one connection is
        counted for as long as the entry is tracked, and the entry will not
        expire during that time. tracking counters also provides a performance
        advantage over just checking the keys, because only one table lookup is
        performed for all acl checks that make use of it.

  note that the "if/unless" condition is optional. if no condition is set on
  the action, it is simply performed unconditionally. that can be useful for
  "track-sc*" actions as well as for changing the default action to a reject.

  example: accept all connections from white-listed hosts, reject too fast
           connection without counting them, and track accepted connections.
           this results in connection rate being capped from abusive sources.

        tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
        tcp-request connection reject if { src_conn_rate gt 10 }
        tcp-request connection track-sc0 src

  example: accept all connections from white-listed hosts, count all other
           connections and reject too fast ones. this results in abusive ones
           being blocked as long as they don't slow down.

        tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
        tcp-request connection track-sc0 src
        tcp-request connection reject if { sc0_conn_rate gt 10 }

  example: enable the proxy protocol for traffic coming from all known proxies.

        tcp-request connection expect-proxy layer4 if { src -f proxies.lst }

  see section 7 about acl usage.

  see also : "tcp-request content", "stick-table"

tcp-request content <action> [{if | unless} <condition>]
  perform an action on a new session depending on a layer 4-7 condition
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <action>    defines the action to perform if the condition applies. valid
                actions include : "accept", "reject", "track-sc0", "track-sc1",
                "track-sc2", "capture" and "lua". see "tcp-request connection"
                above for their signification.

    <condition> is a standard layer 4-7 acl-based condition (see section 7).

  a request's contents can be analysed at an early stage of request processing
  called "tcp content inspection". during this stage, acl-based rules are
  evaluated every time the request contents are updated, until either an
  "accept" or a "reject" rule matches, or the tcp request inspection delay
  expires with no matching rule.

  the first difference between these rules and "tcp-request connection" rules
  is that "tcp-request content" rules can make use of contents to take a
  decision. most often, these decisions will consider a protocol recognition or
  validity. the second difference is that content-based rules can be used in
  both frontends and backends. in case of http keep-alive with the client, all
  tcp-request content rules are evaluated again, so haproxy keeps a record of
  what sticky counters were assigned by a "tcp-request connection" versus a
  "tcp-request content" rule, and flushes all the content-related ones after
  processing an http request, so that they may be evaluated again by the rules
  being evaluated again for the next request. this is of particular importance
  when the rule tracks some l7 information or when it is conditioned by an
  l7-based acl, since tracking may change between requests.

  content-based rules are evaluated in their exact declaration order. if no
  rule matches or if there is no rule, the default action is to accept the
  contents. there is no specific limit to the number of rules which may be
  inserted.

  four types of actions are supported :
    - accept : the request is accepted
    - reject : the request is rejected and the connection is closed
    - capture : the specified sample expression is captured
    - { track-sc0 | track-sc1 | track-sc2 } <key> [table <table>]
    - lua <function>
    - set-var(<var-name>) <expr>

  they have the same meaning as their counter-parts in "tcp-request connection"
  so please refer to that section for a complete description.

  while there is nothing mandatory about it, it is recommended to use the
  track-sc0 in "tcp-request connection" rules, track-sc1 for "tcp-request
  content" rules in the frontend, and track-sc2 for "tcp-request content"
  rules in the backend, because that makes the configuration more readable
  and easier to troubleshoot, but this is just a guideline and all counters
  may be used everywhere.

  note that the "if/unless" condition is optional. if no condition is set on
  the action, it is simply performed unconditionally. that can be useful for
  "track-sc*" actions as well as for changing the default action to a reject.

  it is perfectly possible to match layer 7 contents with "tcp-request content"
  rules, since http-specific acl matches are able to preliminarily parse the
  contents of a buffer before extracting the required data. if the buffered
  contents do not parse as a valid http message, then the acl does not match.
  the parser which is involved there is exactly the same as for all other http
  processing, so there is no risk of parsing something differently. in an http
  backend connected to from an http frontend, it is guaranteed that http
  contents will always be immediately present when the rule is evaluated first.

  tracking layer7 information is also possible provided that the information
  are present when the rule is processed. the rule processing engine is able to
  wait until the inspect delay expires when the data to be tracked is not yet
  available.

  the "lua" keyword is followed by a lua function name. it is used to run a lua
  function if the action is executed. the single parameter is the name of the
  function to run. the prototype of the function is documented in the api
  documentation.

  the "set-var" is used to set the content of a variable. the variable is
  declared inline.

    <var-name> the name of the variable starts by an indication about its scope.
               the allowed scopes are:
                 "sess" : the variable is shared with all the session,
                 "txn"  : the variable is shared with all the transaction
                          (request and response)
                 "req"  : the variable is shared only during the request
                          processing
                 "res"  : the variable is shared only during the response
                          processing.
               this prefix is followed by a name. the separator is a '.'.
               the name may only contain characters 'a-z', 'a-z', '0-9' and '_'.

    <expr>     is a standard haproxy expression formed by a sample-fetch
               followed by some converters.

  example:

        tcp-request content set-var(sess.my_var) src

  example:
        # accept http requests containing a host header saying "example.com"
        # and reject everything else.
        acl is_host_com hdr(host) -i example.com
        tcp-request inspect-delay 30s
        tcp-request content accept if is_host_com
        tcp-request content reject

  example:
        # reject smtp connection if client speaks first
        tcp-request inspect-delay 30s
        acl content_present req_len gt 0
        tcp-request content reject if content_present

        # forward https connection only if client speaks
        tcp-request inspect-delay 30s
        acl content_present req_len gt 0
        tcp-request content accept if content_present
        tcp-request content reject

  example:
        # track the last ip from x-forwarded-for
        tcp-request inspect-delay 10s
        tcp-request content track-sc0 hdr(x-forwarded-for,-1)

  example:
        # track request counts per "base" (concatenation of host+url)
        tcp-request inspect-delay 10s
        tcp-request content track-sc0 base table req-rate

  example: track per-frontend and per-backend counters, block abusers at the
           frontend when the backend detects abuse.

        frontend http
            # use general purpose couter 0 in sc0 as a global abuse counter
            # protecting all our sites
            stick-table type ip size 1m expire 5m store gpc0
            tcp-request connection track-sc0 src
            tcp-request connection reject if { sc0_get_gpc0 gt 0 }
            ...
            use_backend http_dynamic if { path_end .php }

        backend http_dynamic
            # if a source makes too fast requests to this dynamic site (tracked
            # by sc1), block it globally in the frontend.
            stick-table type ip size 1m expire 5m store http_req_rate(10s)
            acl click_too_fast sc1_http_req_rate gt 10
            acl mark_as_abuser sc0_inc_gpc0 gt 0
            tcp-request content track-sc1 src
            tcp-request content reject if click_too_fast mark_as_abuser

  see section 7 about acl usage.

  see also : "tcp-request connection", "tcp-request inspect-delay"

tcp-request inspect-delay <timeout>
  set the maximum allowed time to wait for data during content inspection
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    yes   |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  people using haproxy primarily as a tcp relay are often worried about the
  risk of passing any type of protocol to a server without any analysis. in
  order to be able to analyze the request contents, we must first withhold
  the data then analyze them. this statement simply enables withholding of
  data for at most the specified amount of time.

  tcp content inspection applies very early when a connection reaches a
  frontend, then very early when the connection is forwarded to a backend. this
  means that a connection may experience a first delay in the frontend and a
  second delay in the backend if both have tcp-request rules.

  note that when performing content inspection, haproxy will evaluate the whole
  rules for every new chunk which gets in, taking into account the fact that
  those data are partial. if no rule matches before the aforementioned delay,
  a last check is performed upon expiration, this time considering that the
  contents are definitive. if no delay is set, haproxy will not wait at all
  and will immediately apply a verdict based on the available information.
  obviously this is unlikely to be very useful and might even be racy, so such
  setups are not recommended.

  as soon as a rule matches, the request is released and continues as usual. if
  the timeout is reached and no rule matches, the default policy will be to let
  it pass through unaffected.

  for most protocols, it is enough to set it to a few seconds, as most clients
  send the full request immediately upon connection. add 3 or more seconds to
  cover tcp retransmits but that's all. for some protocols, it may make sense
  to use large values, for instance to ensure that the client never talks
  before the server (eg: smtp), or to wait for a client to talk before passing
  data to the server (eg: ssl). note that the client timeout must cover at
  least the inspection delay, otherwise it will expire first. if the client
  closes the connection or if the buffer is full, the delay immediately expires
  since the contents will not be able to change anymore.

  see also : "tcp-request content accept", "tcp-request content reject",
             "timeout client".

tcp-response content <action> [{if | unless} <condition>]
  perform an action on a session response depending on a layer 4-7 condition
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes
  arguments :
    <action>    defines the action to perform if the condition applies. valid
                actions include : "accept", "close", "reject", "lua".

    <condition> is a standard layer 4-7 acl-based condition (see section 7).

  response contents can be analysed at an early stage of response processing
  called "tcp content inspection". during this stage, acl-based rules are
  evaluated every time the response contents are updated, until either an
  "accept", "close" or a "reject" rule matches, or a tcp response inspection
  delay is set and expires with no matching rule.

  most often, these decisions will consider a protocol recognition or validity.

  content-based rules are evaluated in their exact declaration order. if no
  rule matches or if there is no rule, the default action is to accept the
  contents. there is no specific limit to the number of rules which may be
  inserted.

  two types of actions are supported :
    - accept :
        accepts the response if the condition is true (when used with "if")
        or false (when used with "unless"). the first such rule executed ends
        the rules evaluation.

    - close :
        immediately closes the connection with the server if the condition is
        true (when used with "if"), or false (when used with "unless"). the
        first such rule executed ends the rules evaluation. the main purpose of
        this action is to force a connection to be finished between a client
        and a server after an exchange when the application protocol expects
        some long time outs to elapse first. the goal is to eliminate idle
        connections which take significant resources on servers with certain
        protocols.

    - reject :
        rejects the response if the condition is true (when used with "if")
        or false (when used with "unless"). the first such rule executed ends
        the rules evaluation. rejected session are immediately closed.

    - lua <function>
        executes lua.

    - set-var(<var-name>) <expr>
        sets a variable.

  note that the "if/unless" condition is optional. if no condition is set on
  the action, it is simply performed unconditionally. that can be useful for
  for changing the default action to a reject.

  it is perfectly possible to match layer 7 contents with "tcp-response
  content" rules, but then it is important to ensure that a full response has
  been buffered, otherwise no contents will match. in order to achieve this,
  the best solution involves detecting the http protocol during the inspection
  period.

  the "lua" keyword is followed by a lua function name. it is used to run a lua
  function if the action is executed. the single parameter is the name of the
  function to run. the prototype of the function is documented in the api
  documentation.

  the "set-var" is used to set the content of a variable. the variable is
  declared inline.

    <var-name> the name of the variable starts by an indication about its scope.
               the allowed scopes are:
                 "sess" : the variable is shared with all the session,
                 "txn"  : the variable is shared with all the transaction
                          (request and response)
                 "req"  : the variable is shared only during the request
                          processing
                 "res"  : the variable is shared only during the response
                          processing.
               this prefix is followed by a name. the separator is a '.'.
               the name may only contain characters 'a-z', 'a-z', '0-9' and '_'.

    <expr>     is a standard haproxy expression formed by a sample-fetch
               followed by some converters.

  example:

        tcp-request content set-var(sess.my_var) src

  see section 7 about acl usage.

  see also : "tcp-request content", "tcp-response inspect-delay"

tcp-response inspect-delay <timeout>
  set the maximum allowed time to wait for a response during content inspection
  may be used in sections :   defaults | frontend | listen | backend
                                 no    |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  see also : "tcp-response content", "tcp-request inspect-delay".

timeout check <timeout>
  set additional check timeout, but only after a connection has been already
  established.

  may be used in sections:    defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments:
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  if set, haproxy uses min("timeout connect", "inter") as a connect timeout
  for check and "timeout check" as an additional read timeout. the "min" is
  used so that people running with *very* long "timeout connect" (eg. those
  who needed this due to the queue or tarpit) do not slow down their checks.
  (please also note that there is no valid reason to have such long connect
  timeouts, because "timeout queue" and "timeout tarpit" can always be used to
  avoid that).

  if "timeout check" is not set haproxy uses "inter" for complete check
  timeout (connect + read) exactly like all <1.3.15 version.

  in most cases check request is much simpler and faster to handle than normal
  requests and people may want to kick out laggy servers so this timeout should
  be smaller than "timeout server".

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it.

  see also: "timeout connect", "timeout queue", "timeout server",
            "timeout tarpit".

timeout client <timeout>
timeout clitimeout <timeout> (deprecated)
  set the maximum inactivity time on the client side.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the inactivity timeout applies when the client is expected to acknowledge or
  send data. in http mode, this timeout is particularly important to consider
  during the first phase, when the client sends the request, and during the
  response while it is reading data sent by the server. the value is specified
  in milliseconds by default, but can be in any other unit if the number is
  suffixed by the unit, as specified at the top of this document. in tcp mode
  (and to a lesser extent, in http mode), it is highly recommended that the
  client timeout remains equal to the server timeout in order to avoid complex
  situations to debug. it is a good practice to cover one or several tcp packet
  losses by specifying timeouts that are slightly above multiples of 3 seconds
  (eg: 4 or 5 seconds). if some long-lived sessions are mixed with short-lived
  sessions (eg: websocket and http), it's worth considering "timeout tunnel",
  which overrides "timeout client" and "timeout server" for tunnels, as well as
  "timeout client-fin" for half-closed connections.

  this parameter is specific to frontends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it. an unspecified timeout results in an infinite timeout, which
  is not recommended. such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of expired sessions in
  the system if the system's timeouts are not configured either.

  this parameter replaces the old, deprecated "clitimeout". it is recommended
  to use it to write new configurations. the form "timeout clitimeout" is
  provided only by backwards compatibility but its use is strongly discouraged.

  see also : "clitimeout", "timeout server", "timeout tunnel".

timeout client-fin <timeout>
  set the inactivity timeout on the client side for half-closed connections.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   no
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the inactivity timeout applies when the client is expected to acknowledge or
  send data while one direction is already shut down. this timeout is different
  from "timeout client" in that it only applies to connections which are closed
  in one direction. this is particularly useful to avoid keeping connections in
  fin_wait state for too long when clients do not disconnect cleanly. this
  problem is particularly common long connections such as rdp or websocket.
  note that this timeout can override "timeout tunnel" when a connection shuts
  down in one direction.

  this parameter is specific to frontends, but can be specified once for all in
  "defaults" sections. by default it is not set, so half-closed connections
  will use the other timeouts (timeout.client or timeout.tunnel).

  see also : "timeout client", "timeout server-fin", and "timeout tunnel".

timeout connect <timeout>
timeout contimeout <timeout> (deprecated)
  set the maximum time to wait for a connection attempt to a server to succeed.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  if the server is located on the same lan as haproxy, the connection should be
  immediate (less than a few milliseconds). anyway, it is a good practice to
  cover one or several tcp packet losses by specifying timeouts that are
  slightly above multiples of 3 seconds (eg: 4 or 5 seconds). by default, the
  connect timeout also presets both queue and tarpit timeouts to the same value
  if these have not been specified.

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it. an unspecified timeout results in an infinite timeout, which
  is not recommended. such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of failed sessions in
  the system if the system's timeouts are not configured either.

  this parameter replaces the old, deprecated "contimeout". it is recommended
  to use it to write new configurations. the form "timeout contimeout" is
  provided only by backwards compatibility but its use is strongly discouraged.

  see also: "timeout check", "timeout queue", "timeout server", "contimeout",
            "timeout tarpit".

timeout http-keep-alive <timeout>
  set the maximum allowed time to wait for a new http request to appear
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  by default, the time to wait for a new request in case of keep-alive is set
  by "timeout http-request". however this is not always convenient because some
  people want very short keep-alive timeouts in order to release connections
  faster, and others prefer to have larger ones but still have short timeouts
  once the request has started to present itself.

  the "http-keep-alive" timeout covers these needs. it will define how long to
  wait for a new http request to start coming after a response was sent. once
  the first byte of request has been seen, the "http-request" timeout is used
  to wait for the complete request to come. note that empty lines prior to a
  new request do not refresh the timeout and are not counted as a new request.

  there is also another difference between the two timeouts : when a connection
  expires during timeout http-keep-alive, no error is returned, the connection
  just closes. if the connection expires in "http-request" while waiting for a
  connection to complete, a http 408 error is returned.

  in general it is optimal to set this value to a few tens to hundreds of
  milliseconds, to allow users to fetch all objects of a page at once but
  without waiting for further clicks. also, if set to a very small value (eg:
  1 millisecond) it will probably only accept pipelined requests but not the
  non-pipelined ones. it may be a nice trade-off for very large sites running
  with tens to hundreds of thousands of clients.

  if this parameter is not set, the "http-request" timeout applies, and if both
  are not set, "timeout client" still applies at the lower level. it should be
  set in the frontend to take effect, unless the frontend is in tcp mode, in
  which case the http backend's timeout will be used.

  see also : "timeout http-request", "timeout client".

timeout http-request <timeout>
  set the maximum allowed time to wait for a complete http request
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  in order to offer dos protection, it may be required to lower the maximum
  accepted time to receive a complete http request without affecting the client
  timeout. this helps protecting against established connections on which
  nothing is sent. the client timeout cannot offer a good protection against
  this abuse because it is an inactivity timeout, which means that if the
  attacker sends one character every now and then, the timeout will not
  trigger. with the http request timeout, no matter what speed the client
  types, the request will be aborted if it does not complete in time. when the
  timeout expires, an http 408 response is sent to the client to inform it
  about the problem, and the connection is closed. the logs will report
  termination codes "cr". some recent browsers are having problems with this
  standard, well-documented behaviour, so it might be needed to hide the 408
  code using "option http-ignore-probes" or "errorfile 408 /dev/null". see
  more details in the explanations of the "cr" termination code in section 8.5.

  note that this timeout only applies to the header part of the request, and
  not to any data. as soon as the empty line is received, this timeout is not
  used anymore. it is used again on keep-alive connections to wait for a second
  request if "timeout http-keep-alive" is not set.

  generally it is enough to set it to a few seconds, as most clients send the
  full request immediately upon connection. add 3 or more seconds to cover tcp
  retransmits but that's all. setting it to very low values (eg: 50 ms) will
  generally work on local networks as long as there are no packet losses. this
  will prevent people from sending bare http requests using telnet.

  if this parameter is not set, the client timeout still applies between each
  chunk of the incoming request. it should be set in the frontend to take
  effect, unless the frontend is in tcp mode, in which case the http backend's
  timeout will be used.

  see also : "errorfile", "http-ignore-probes", "timeout http-keep-alive", and
             "timeout client".

timeout queue <timeout>
  set the maximum time to wait in the queue for a connection slot to be free
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  when a server's maxconn is reached, connections are left pending in a queue
  which may be server-specific or global to the backend. in order not to wait
  indefinitely, a timeout is applied to requests pending in the queue. if the
  timeout is reached, it is considered that the request will almost never be
  served, so it is dropped and a 503 error is returned to the client.

  the "timeout queue" statement allows to fix the maximum time for a request to
  be left pending in a queue. if unspecified, the same value as the backend's
  connection timeout ("timeout connect") is used, for backwards compatibility
  with older versions with no "timeout queue" parameter.

  see also : "timeout connect", "contimeout".

timeout server <timeout>
timeout srvtimeout <timeout> (deprecated)
  set the maximum inactivity time on the server side.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the inactivity timeout applies when the server is expected to acknowledge or
  send data. in http mode, this timeout is particularly important to consider
  during the first phase of the server's response, when it has to send the
  headers, as it directly represents the server's processing time for the
  request. to find out what value to put there, it's often good to start with
  what would be considered as unacceptable response times, then check the logs
  to observe the response time distribution, and adjust the value accordingly.

  the value is specified in milliseconds by default, but can be in any other
  unit if the number is suffixed by the unit, as specified at the top of this
  document. in tcp mode (and to a lesser extent, in http mode), it is highly
  recommended that the client timeout remains equal to the server timeout in
  order to avoid complex situations to debug. whatever the expected server
  response times, it is a good practice to cover at least one or several tcp
  packet losses by specifying timeouts that are slightly above multiples of 3
  seconds (eg: 4 or 5 seconds minimum). if some long-lived sessions are mixed
  with short-lived sessions (eg: websocket and http), it's worth considering
  "timeout tunnel", which overrides "timeout client" and "timeout server" for
  tunnels.

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it. an unspecified timeout results in an infinite timeout, which
  is not recommended. such a usage is accepted and works but reports a warning
  during startup because it may results in accumulation of expired sessions in
  the system if the system's timeouts are not configured either.

  this parameter replaces the old, deprecated "srvtimeout". it is recommended
  to use it to write new configurations. the form "timeout srvtimeout" is
  provided only by backwards compatibility but its use is strongly discouraged.

  see also : "srvtimeout", "timeout client" and "timeout tunnel".

timeout server-fin <timeout>
  set the inactivity timeout on the server side for half-closed connections.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the inactivity timeout applies when the server is expected to acknowledge or
  send data while one direction is already shut down. this timeout is different
  from "timeout server" in that it only applies to connections which are closed
  in one direction. this is particularly useful to avoid keeping connections in
  fin_wait state for too long when a remote server does not disconnect cleanly.
  this problem is particularly common long connections such as rdp or websocket.
  note that this timeout can override "timeout tunnel" when a connection shuts
  down in one direction. this setting was provided for completeness, but in most
  situations, it should not be needed.

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. by default it is not set, so half-closed connections
  will use the other timeouts (timeout.server or timeout.tunnel).

  see also : "timeout client-fin", "timeout server", and "timeout tunnel".

timeout tarpit <timeout>
  set the duration for which tarpitted connections will be maintained
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  arguments :
    <timeout> is the tarpit duration specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  when a connection is tarpitted using "reqtarpit", it is maintained open with
  no activity for a certain amount of time, then closed. "timeout tarpit"
  defines how long it will be maintained open.

  the value is specified in milliseconds by default, but can be in any other
  unit if the number is suffixed by the unit, as specified at the top of this
  document. if unspecified, the same value as the backend's connection timeout
  ("timeout connect") is used, for backwards compatibility with older versions
  with no "timeout tarpit" parameter.

  see also : "timeout connect", "contimeout".

timeout tunnel <timeout>
  set the maximum inactivity time on the client and server side for tunnels.
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments :
    <timeout> is the timeout value specified in milliseconds by default, but
              can be in any other unit if the number is suffixed by the unit,
              as explained at the top of this document.

  the tunnel timeout applies when a bidirectional connection is established
  between a client and a server, and the connection remains inactive in both
  directions. this timeout supersedes both the client and server timeouts once
  the connection becomes a tunnel. in tcp, this timeout is used as soon as no
  analyser remains attached to either connection (eg: tcp content rules are
  accepted). in http, this timeout is used when a connection is upgraded (eg:
  when switching to the websocket protocol, or forwarding a connect request
  to a proxy), or after the first response when no keepalive/close option is
  specified.

  since this timeout is usually used in conjunction with long-lived connections,
  it usually is a good idea to also set "timeout client-fin" to handle the
  situation where a client suddenly disappears from the net and does not
  acknowledge a close, or sends a shutdown and does not acknowledge pending
  data anymore. this can happen in lossy networks where firewalls are present,
  and is detected by the presence of large amounts of sessions in a fin_wait
  state.

  the value is specified in milliseconds by default, but can be in any other
  unit if the number is suffixed by the unit, as specified at the top of this
  document. whatever the expected normal idle time, it is a good practice to
  cover at least one or several tcp packet losses by specifying timeouts that
  are slightly above multiples of 3 seconds (eg: 4 or 5 seconds minimum).

  this parameter is specific to backends, but can be specified once for all in
  "defaults" sections. this is in fact one of the easiest solutions not to
  forget about it.

  example :
        defaults http
            option http-server-close
            timeout connect 5s
            timeout client 30s
            timeout client-fin 30s
            timeout server 30s
            timeout tunnel  1h    # timeout to use with websocket and connect

  see also : "timeout client", "timeout client-fin", "timeout server".

transparent (deprecated)
  enable client-side transparent proxying
  may be used in sections :   defaults | frontend | listen | backend
                                 yes   |    no    |   yes  |   yes
  arguments : none

  this keyword was introduced in order to provide layer 7 persistence to layer
  3 load balancers. the idea is to use the os's ability to redirect an incoming
  connection for a remote address to a local process (here haproxy), and let
  this process know what address was initially requested. when this option is
  used, sessions without cookies will be forwarded to the original destination
  ip address of the incoming request (which should match that of another
  equipment), while requests with cookies will still be forwarded to the
  appropriate server.

  the "transparent" keyword is deprecated, use "option transparent" instead.

  note that contrary to a common belief, this option does not make haproxy
  present the client's ip to the server when establishing the connection.

  see also: "option transparent"

unique-id-format <string>
  generate a unique id for each request.
  may be used in sections :   defaults | frontend | listen | backend
                                  yes  |    yes   |   yes  |   no
  arguments :
    <string>   is a log-format string.

  this keyword creates a id for each request using the custom log format. a
  unique id is useful to trace a request passing through many components of
  a complex infrastructure. the newly created id may also be logged using the
  %id tag the log-format string.

  the format should be composed from elements that are guaranteed to be
  unique when combined together. for instance, if multiple haproxy instances
  are involved, it might be important to include the node name. it is often
  needed to log the incoming connection's source and destination addresses
  and ports. note that since multiple requests may be performed over the same
  connection, including a request counter may help differentiate them.
  similarly, a timestamp may protect against a rollover of the counter.
  logging the process id will avoid collisions after a service restart.

  it is recommended to use hexadecimal notation for many fields since it
  makes them more compact and saves space in logs.

  example:

        unique-id-format %{+x}o\ %ci:%cp_%fi:%fp_%ts_%rt:%pid

        will generate:

               7f000001:8296_7f00001e:1f90_4f7b0a69_0003:790a

  see also: "unique-id-header"

unique-id-header <name>
  add a unique id header in the http request.
  may be used in sections :   defaults | frontend | listen | backend
                                  yes  |    yes   |   yes  |   no
  arguments :
    <name>   is the name of the header.

  add a unique-id header in the http request sent to the server, using the
  unique-id-format. it can't work if the unique-id-format doesn't exist.

  example:

        unique-id-format %{+x}o\ %ci:%cp_%fi:%fp_%ts_%rt:%pid
        unique-id-header x-unique-id

        will generate:

           x-unique-id: 7f000001:8296_7f00001e:1f90_4f7b0a69_0003:790a

    see also: "unique-id-format"

use_backend <backend> [{if | unless} <condition>]
  switch to a specific backend if/unless an acl-based condition is matched.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    yes   |   yes  |   no
  arguments :
    <backend>   is the name of a valid backend or "listen" section, or a
                "log-format" string resolving to a backend name.

    <condition> is a condition composed of acls, as described in section 7\. if
                it is omitted, the rule is unconditionally applied.

  when doing content-switching, connections arrive on a frontend and are then
  dispatched to various backends depending on a number of conditions. the
  relation between the conditions and the backends is described with the
  "use_backend" keyword. while it is normally used with http processing, it can
  also be used in pure tcp, either without content using stateless acls (eg:
  source address validation) or combined with a "tcp-request" rule to wait for
  some payload.

  there may be as many "use_backend" rules as desired. all of these rules are
  evaluated in their declaration order, and the first one which matches will
  assign the backend.

  in the first form, the backend will be used if the condition is met. in the
  second form, the backend will be used if the condition is not met. if no
  condition is valid, the backend defined with "default_backend" will be used.
  if no default backend is defined, either the servers in the same section are
  used (in case of a "listen" section) or, in case of a frontend, no server is
  used and a 503 service unavailable response is returned.

  note that it is possible to switch from a tcp frontend to an http backend. in
  this case, either the frontend has already checked that the protocol is http,
  and backend processing will immediately follow, or the backend will wait for
  a complete http request to get in. this feature is useful when a frontend
  must decode several protocols on a unique port, one of them being http.

  when <backend> is a simple name, it is resolved at configuration time, and an
  error is reported if the specified backend does not exist. if <backend> is
  a log-format string instead, no check may be done at configuration time, so
  the backend name is resolved dynamically at run time. if the resulting
  backend name does not correspond to any valid backend, no other rule is
  evaluated, and the default_backend directive is applied instead. note that
  when using dynamic backend names, it is highly recommended to use a prefix
  that no other backend uses in order to ensure that an unauthorized backend
  cannot be forced from the request.

  it is worth mentioning that "use_backend" rules with an explicit name are
  used to detect the association between frontends and backends to compute the
  backend's "fullconn" setting. this cannot be done for dynamic names.

  see also: "default_backend", "tcp-request", "fullconn", "log-format", and
            section 7 about acls.

use-server <server> if <condition>
use-server <server> unless <condition>
  only use a specific server if/unless an acl-based condition is matched.
  may be used in sections :   defaults | frontend | listen | backend
                                  no   |    no    |   yes  |   yes
  arguments :
    <server>    is the name of a valid server in the same backend section.

    <condition> is a condition composed of acls, as described in section 7.

  by default, connections which arrive to a backend are load-balanced across
  the available servers according to the configured algorithm, unless a
  persistence mechanism such as a cookie is used and found in the request.

  sometimes it is desirable to forward a particular request to a specific
  server without having to declare a dedicated backend for this server. this
  can be achieved using the "use-server" rules. these rules are evaluated after
  the "redirect" rules and before evaluating cookies, and they have precedence
  on them. there may be as many "use-server" rules as desired. all of these
  rules are evaluated in their declaration order, and the first one which
  matches will assign the server.

  if a rule designates a server which is down, and "option persist" is not used
  and no force-persist rule was validated, it is ignored and evaluation goes on
  with the next rules until one matches.

  in the first form, the server will be used if the condition is met. in the
  second form, the server will be used if the condition is not met. if no
  condition is valid, the processing continues and the server will be assigned
  according to other persistence mechanisms.

  note that even if a rule is matched, cookie processing is still performed but
  does not assign the server. this allows prefixed cookies to have their prefix
  stripped.

  the "use-server" statement works both in http and tcp mode. this makes it
  suitable for use with content-based inspection. for instance, a server could
  be selected in a farm according to the tls sni field. and if these servers
  have their weight set to zero, they will not be used for other traffic.

  example :
     # intercept incoming tls requests based on the sni field
     use-server www if { req_ssl_sni -i www.example.com }
     server     www 192.168.0.1:443 weight 0
     use-server mail if { req_ssl_sni -i mail.example.com }
     server     mail 192.168.0.1:587 weight 0
     use-server imap if { req_ssl_sni -i imap.example.com }
     server     mail 192.168.0.1:993 weight 0
     # all the rest is forwarded to this server
     server  default 192.168.0.2:443 check

  see also: "use_backend", section 5 about server and section 7 about acls.

5\. bind and server options
--------------------------

the "bind", "server" and "default-server" keywords support a number of settings
depending on some build options and on the system haproxy was built on. these
settings generally each consist in one word sometimes followed by a value,
written on the same line as the "bind" or "server" line. all these options are
described in this section.

5.1\. bind options
-----------------

the "bind" keyword supports a certain number of settings which are all passed
as arguments on the same line. the order in which those arguments appear makes
no importance, provided that they appear after the bind address. all of these
parameters are optional. some of them consist in a single words (booleans),
while other ones expect a value after them. in this case, the value must be
provided immediately after the setting name.

the currently supported settings are the following ones.

accept-proxy
  enforces the use of the proxy protocol over any connection accepted by any of
  the sockets declared on the same line. versions 1 and 2 of the proxy protocol
  are supported and correctly detected. the proxy protocol dictates the layer
  3/4 addresses of the incoming connection to be used everywhere an address is
  used, with the only exception of "tcp-request connection" rules which will
  only see the real connection address. logs will reflect the addresses
  indicated in the protocol, unless it is violated, in which case the real
  address will still be used.  this keyword combined with support from external
  components can be used as an efficient and reliable alternative to the
  x-forwarded-for mechanism which is not always reliable and not even always
  usable. see also "tcp-request connection expect-proxy" for a finer-grained
  setting of which client is allowed to use the protocol.

alpn <protocols>
  this enables the tls alpn extension and advertises the specified protocol
  list as supported on top of alpn. the protocol list consists in a comma-
  delimited list of protocol names, for instance: "http/1.1,http/1.0" (without
  quotes). this requires that the ssl library is build with support for tls
  extensions enabled (check with haproxy -vv). the alpn extension replaces the
  initial npn extension.

backlog <backlog>
  sets the socket's backlog to this value. if unspecified, the frontend's
  backlog is used instead, which generally defaults to the maxconn value.

ecdhe <named curve>
  this setting is only available when support for openssl was built in. it sets
  the named curve (rfc 4492) used to generate ecdh ephemeral keys. by default,
  used named curve is prime256v1.

ca-file <cafile>
  this setting is only available when support for openssl was built in. it
  designates a pem file from which to load ca certificates used to verify
  client's certificate.

ca-ignore-err [all|<errorid>,...]
  this setting is only available when support for openssl was built in.
  sets a comma separated list of errorids to ignore during verify at depth > 0.
  if set to 'all', all errors are ignored. ssl handshake is not aborted if an
  error is ignored.

ca-sign-file <cafile>
  this setting is only available when support for openssl was built in. it
  designates a pem file containing both the ca certificate and the ca private
  key used to create and sign server's certificates. this is a mandatory
  setting when the dynamic generation of certificates is enabled. see
  'generate-certificates' for details.

ca-sign-passphrase <passphrase>
  this setting is only available when support for openssl was built in. it is
  the ca private key passphrase. this setting is optional and used only when
  the dynamic generation of certificates is enabled. see
  'generate-certificates' for details.

ciphers <ciphers>
  this setting is only available when support for openssl was built in. it sets
  the string describing the list of cipher algorithms ("cipher suite") that are
  negotiated during the ssl/tls handshake. the format of the string is defined
  in "man 1 ciphers" from openssl man pages, and can be for instance a string
  such as "aes:all:!anull:!enull:+rc4:@strength" (without quotes).

crl-file <crlfile>
  this setting is only available when support for openssl was built in. it
  designates a pem file from which to load certificate revocation list used
  to verify client's certificate.

crt <cert>
  this setting is only available when support for openssl was built in. it
  designates a pem file containing both the required certificates and any
  associated private keys. this file can be built by concatenating multiple
  pem files into one (e.g. cat cert.pem key.pem > combined.pem). if your ca
  requires an intermediate certificate, this can also be concatenated into this
  file.

  if the openssl used supports diffie-hellman, parameters present in this file
  are loaded.

  if a directory name is used instead of a pem file, then all files found in
  that directory will be loaded in alphabetic order unless their name ends with
  '.issuer', '.ocsp' or '.sctl' (reserved extensions). this directive may be
  specified multiple times in order to load certificates from multiple files or
  directories. the certificates will be presented to clients who provide a
  valid tls server name indication field matching one of their cn or alt
  subjects.  wildcards are supported, where a wildcard character '*' is used
  instead of the first hostname component (eg: *.example.org matches
  www.example.org but not www.sub.example.org).

  if no sni is provided by the client or if the ssl library does not support
  tls extensions, or if the client provides an sni hostname which does not
  match any certificate, then the first loaded certificate will be presented.
  this means that when loading certificates from a directory, it is highly
  recommended to load the default one first as a file or to ensure that it will
  always be the first one in the directory.

  note that the same cert may be loaded multiple times without side effects.

  some cas (such as godaddy) offer a drop down list of server types that do not
  include haproxy when obtaining a certificate. if this happens be sure to
  choose a webserver that the ca believes requires an intermediate ca (for
  godaddy, selection apache tomcat will get the correct bundle, but many
  others, e.g. nginx, result in a wrong bundle that will not work for some
  clients).

  for each pem file, haproxy checks for the presence of file at the same path
  suffixed by ".ocsp". if such file is found, support for the tls certificate
  status request extension (also known as "ocsp stapling") is automatically
  enabled. the content of this file is optional. if not empty, it must contain
  a valid ocsp response in der format. in order to be valid an ocsp response
  must comply with the following rules: it has to indicate a good status,
  it has to be a single response for the certificate of the pem file, and it
  has to be valid at the moment of addition. if these rules are not respected
  the ocsp response is ignored and a warning is emitted. in order to  identify
  which certificate an ocsp response applies to, the issuer's certificate is
  necessary. if the issuer's certificate is not found in the pem file, it will
  be loaded from a file at the same path as the pem file suffixed by ".issuer"
  if it exists otherwise it will fail with an error.

  for each pem file, haproxy also checks for the presence of file at the same
  path suffixed by ".sctl". if such file is found, support for certificate
  transparency (rfc6962) tls extension is enabled. the file must contain a
  valid signed certificate timestamp list, as described in rfc. file is parsed
  to check basic syntax, but no signatures are verified.

crt-ignore-err <errors>
  this setting is only available when support for openssl was built in. sets a
  comma separated list of errorids to ignore during verify at depth == 0\.  if
  set to 'all', all errors are ignored. ssl handshake is not aborted if an error
  is ignored.

crt-list <file>
  this setting is only available when support for openssl was built in. it
  designates a list of pem file with an optional list of sni filter per
  certificate, with the following format for each line :

        <crtfile> [[!]<snifilter> ...]

  wildcards are supported in the sni filter. negative filter are also supported,
  only useful in combination with a wildcard filter to exclude a particular sni.
  the certificates will be presented to clients who provide a valid tls server
  name indication field matching one of the sni filters. if no sni filter is
  specified, the cn and alt subjects are used. this directive may be specified
  multiple times. see the "crt" option for more information. the default
  certificate is still needed to meet openssl expectations. if it is not used,
  the 'strict-sni' option may be used.

defer-accept
  is an optional keyword which is supported only on certain linux kernels. it
  states that a connection will only be accepted once some data arrive on it,
  or at worst after the first retransmit. this should be used only on protocols
  for which the client talks first (eg: http). it can slightly improve
  performance by ensuring that most of the request is already available when
  the connection is accepted. on the other hand, it will not be able to detect
  connections which don't talk. it is important to note that this option is
  broken in all kernels up to 2.6.31, as the connection is never accepted until
  the client talks. this can cause issues with front firewalls which would see
  an established connection while the proxy will only see it in syn_recv. this
  option is only supported on tcpv4/tcpv6 sockets and ignored by other ones.

force-sslv3
  this option enforces use of sslv3 only on ssl connections instantiated from
  this listener. sslv3 is generally less expensive than the tls counterparts
  for high connection rates. this option is also available on global statement
  "ssl-default-bind-options". see also "no-tlsv*" and "no-sslv3".

force-tlsv10
  this option enforces use of tlsv1.0 only on ssl connections instantiated from
  this listener. this option is also available on global statement
  "ssl-default-bind-options". see also "no-tlsv*" and "no-sslv3".

force-tlsv11
  this option enforces use of tlsv1.1 only on ssl connections instantiated from
  this listener. this option is also available on global statement
  "ssl-default-bind-options". see also "no-tlsv*", and "no-sslv3".

force-tlsv12
  this option enforces use of tlsv1.2 only on ssl connections instantiated from
  this listener. this option is also available on global statement
  "ssl-default-bind-options". see also "no-tlsv*", and "no-sslv3".

generate-certificates
  this setting is only available when support for openssl was built in. it
  enables the dynamic ssl certificates generation. a ca certificate and its
  private key are necessary (see 'ca-sign-file'). when haproxy is configured as
  a transparent forward proxy, ssl requests generate errors because of a common
  name mismatch on the certificate presented to the client. with this option
  enabled, haproxy will try to forge a certificate using the sni hostname
  indicated by the client. this is done only if no certificate matches the sni
  hostname (see 'crt-list'). if an error occurs, the default certificate is
  used, else the 'strict-sni' option is set.
  it can also be used when haproxy is configured as a reverse proxy to ease the
  deployment of an architecture with many backends.

  creating a ssl certificate is an expensive operation, so a lru cache is used
  to store forged certificates (see 'tune.ssl.ssl-ctx-cache-size'). it
  increases the haproxy's memroy footprint to reduce latency when the same
  certificate is used many times.

gid <gid>
  sets the group of the unix sockets to the designated system gid. it can also
  be set by default in the global section's "unix-bind" statement. note that
  some platforms simply ignore this. this setting is equivalent to the "group"
  setting except that the group id is used instead of its name. this setting is
  ignored by non unix sockets.

group <group>
  sets the group of the unix sockets to the designated system group. it can
  also be set by default in the global section's "unix-bind" statement. note
  that some platforms simply ignore this. this setting is equivalent to the
  "gid" setting except that the group name is used instead of its gid. this
  setting is ignored by non unix sockets.

id <id>
  fixes the socket id. by default, socket ids are automatically assigned, but
  sometimes it is more convenient to fix them to ease monitoring. this value
  must be strictly positive and unique within the listener/frontend. this
  option can only be used when defining only a single socket.

interface <interface>
  restricts the socket to a specific interface. when specified, only packets
  received from that particular interface are processed by the socket. this is
  currently only supported on linux. the interface must be a primary system
  interface, not an aliased interface. it is also possible to bind multiple
  frontends to the same address if they are bound to different interfaces. note
  that binding to a network interface requires root privileges. this parameter
  is only compatible with tcpv4/tcpv6 sockets.

level <level>
  this setting is used with the stats sockets only to restrict the nature of
  the commands that can be issued on the socket. it is ignored by other
  sockets. <level> can be one of :
  - "user" is the least privileged level ; only non-sensitive stats can be
    read, and no change is allowed. it would make sense on systems where it
    is not easy to restrict access to the socket.
  - "operator" is the default level and fits most common uses. all data can
    be read, and only non-sensitive changes are permitted (eg: clear max
    counters).
  - "admin" should be used with care, as everything is permitted (eg: clear
    all counters).

maxconn <maxconn>
  limits the sockets to this number of concurrent connections. extraneous
  connections will remain in the system's backlog until a connection is
  released. if unspecified, the limit will be the same as the frontend's
  maxconn. note that in case of port ranges or multiple addresses, the same
  value will be applied to each socket. this setting enables different
  limitations on expensive sockets, for instance ssl entries which may easily
  eat all memory.

mode <mode>
  sets the octal mode used to define access permissions on the unix socket. it
  can also be set by default in the global section's "unix-bind" statement.
  note that some platforms simply ignore this. this setting is ignored by non
  unix sockets.

mss <maxseg>
  sets the tcp maximum segment size (mss) value to be advertised on incoming
  connections. this can be used to force a lower mss for certain specific
  ports, for instance for connections passing through a vpn. note that this
  relies on a kernel feature which is theoretically supported under linux but
  was buggy in all versions prior to 2.6.28\. it may or may not work on other
  operating systems. it may also not change the advertised value but change the
  effective size of outgoing segments. the commonly advertised value for tcpv4
  over ethernet networks is 1460 = 1500(mtu) - 40(ip+tcp). if this value is
  positive, it will be used as the advertised mss. if it is negative, it will
  indicate by how much to reduce the incoming connection's advertised mss for
  outgoing segments. this parameter is only compatible with tcp v4/v6 sockets.

name <name>
  sets an optional name for these sockets, which will be reported on the stats
  page.

nice <nice>
  sets the 'niceness' of connections initiated from the socket. value must be
  in the range -1024..1024 inclusive, and defaults to zero. positive values
  means that such connections are more friendly to others and easily offer
  their place in the scheduler. on the opposite, negative values mean that
  connections want to run with a higher priority than others. the difference
  only happens under high loads when the system is close to saturation.
  negative values are appropriate for low-latency or administration services,
  and high values are generally recommended for cpu intensive tasks such as ssl
  processing or bulk transfers which are less sensible to latency. for example,
  it may make sense to use a positive value for an smtp socket and a negative
  one for an rdp socket.

no-sslv3
  this setting is only available when support for openssl was built in. it
  disables support for sslv3 on any sockets instantiated from the listener when
  ssl is supported. note that sslv2 is forced disabled in the code and cannot
  be enabled using any configuration option. this option is also available on
  global statement "ssl-default-bind-options". see also "force-tls*",
  and "force-sslv3".

no-tls-tickets
  this setting is only available when support for openssl was built in. it
  disables the stateless session resumption (rfc 5077 tls ticket
  extension) and force to use stateful session resumption. stateless
  session resumption is more expensive in cpu usage. this option is also
  available on global statement "ssl-default-bind-options".

no-tlsv10
  this setting is only available when support for openssl was built in. it
  disables support for tlsv1.0 on any sockets instantiated from the listener
  when ssl is supported. note that sslv2 is forced disabled in the code and
  cannot be enabled using any configuration option. this option is also
  available on global statement "ssl-default-bind-options". see also
  "force-tlsv*", and "force-sslv3".

no-tlsv11
  this setting is only available when support for openssl was built in. it
  disables support for tlsv1.1 on any sockets instantiated from the listener
  when ssl is supported. note that sslv2 is forced disabled in the code and
  cannot be enabled using any configuration option. this option is also
  available on global statement "ssl-default-bind-options". see also
  "force-tlsv*", and "force-sslv3".

no-tlsv12
  this setting is only available when support for openssl was built in. it
  disables support for tlsv1.2 on any sockets instantiated from the listener
  when ssl is supported. note that sslv2 is forced disabled in the code and
  cannot be enabled using any configuration option. this option is also
  available on global statement "ssl-default-bind-options". see also
  "force-tlsv*", and "force-sslv3".

npn <protocols>
  this enables the npn tls extension and advertises the specified protocol list
  as supported on top of npn. the protocol list consists in a comma-delimited
  list of protocol names, for instance: "http/1.1,http/1.0" (without quotes).
  this requires that the ssl library is build with support for tls extensions
  enabled (check with haproxy -vv). note that the npn extension has been
  replaced with the alpn extension (see the "alpn" keyword).

process [ all | odd | even | <number 1-64>[-<number 1-64>] ]
  this restricts the list of processes on which this listener is allowed to
  run. it does not enforce any process but eliminates those which do not match.
  if the frontend uses a "bind-process" setting, the intersection between the
  two is applied. if in the end the listener is not allowed to run on any
  remaining process, a warning is emitted, and the listener will either run on
  the first process of the listener if a single process was specified, or on
  all of its processes if multiple processes were specified. for the unlikely
  case where several ranges are needed, this directive may be repeated. the
  main purpose of this directive is to be used with the stats sockets and have
  one different socket per process. the second purpose is to have multiple bind
  lines sharing the same ip:port but not the same process in a listener, so
  that the system can distribute the incoming connections into multiple queues
  and allow a smoother inter-process load balancing. currently linux 3.9 and
  above is known for supporting this. see also "bind-process" and "nbproc".

ssl
  this setting is only available when support for openssl was built in. it
  enables ssl deciphering on connections instantiated from this listener. a
  certificate is necessary (see "crt" above). all contents in the buffers will
  appear in clear text, so that acls and http processing will only have access
  to deciphered contents.

strict-sni
  this setting is only available when support for openssl was built in. the
  ssl/tls negotiation is allow only if the client provided an sni which match
  a certificate. the default certificate is not used.
  see the "crt" option for more information.

tcp-ut <delay>
  sets the tcp user timeout for all incoming connections instanciated from this
  listening socket. this option is available on linux since version 2.6.37\. it
  allows haproxy to configure a timeout for sockets which contain data not
  receiving an acknoledgement for the configured delay. this is especially
  useful on long-lived connections experiencing long idle periods such as
  remote terminals or database connection pools, where the client and server
  timeouts must remain high to allow a long period of idle, but where it is
  important to detect that the client has disappeared in order to release all
  resources associated with its connection (and the server's session). the
  argument is a delay expressed in milliseconds by default. this only works
  for regular tcp connections, and is ignored for other protocols.

tfo
  is an optional keyword which is supported only on linux kernels >= 3.7\. it
  enables tcp fast open on the listening socket, which means that clients which
  support this feature will be able to send a request and receive a response
  during the 3-way handshake starting from second connection, thus saving one
  round-trip after the first connection. this only makes sense with protocols
  that use high connection rates and where each round trip matters. this can
  possibly cause issues with many firewalls which do not accept data on syn
  packets, so this option should only be enabled once well tested. this option
  is only supported on tcpv4/tcpv6 sockets and ignored by other ones. you may
  need to build haproxy with use_tfo=1 if your libc doesn't define
  tcp_fastopen.

tls-ticket-keys <keyfile>
  sets the tls ticket keys file to load the keys from. the keys need to be 48
  bytes long, encoded with base64 (ex. openssl rand -base64 48). number of keys
  is specified by the tls_tickets_no build option (default 3) and at least as
  many keys need to be present in the file. last tls_tickets_no keys will be
  used for decryption and the penultimate one for encryption. this enables easy
  key rotation by just appending new key to the file and reloading the process.
  keys must be periodically rotated (ex. every 12h) or perfect forward secrecy
  is compromised. it is also a good idea to keep the keys off any permanent
  storage such as hard drives (hint: use tmpfs and don't swap those files).
  lifetime hint can be changed using tune.ssl.timeout.

transparent
  is an optional keyword which is supported only on certain linux kernels. it
  indicates that the addresses will be bound even if they do not belong to the
  local machine, and that packets targeting any of these addresses will be
  intercepted just as if the addresses were locally configured. this normally
  requires that ip forwarding is enabled. caution! do not use this with the
  default address '*', as it would redirect any traffic for the specified port.
  this keyword is available only when haproxy is built with use_linux_tproxy=1.
  this parameter is only compatible with tcpv4 and tcpv6 sockets, depending on
  kernel version. some distribution kernels include backports of the feature,
  so check for support with your vendor.

v4v6
  is an optional keyword which is supported only on most recent systems
  including linux kernels >= 2.4.21\. it is used to bind a socket to both ipv4
  and ipv6 when it uses the default address. doing so is sometimes necessary
  on systems which bind to ipv6 only by default. it has no effect on non-ipv6
  sockets, and is overridden by the "v6only" option.

v6only
  is an optional keyword which is supported only on most recent systems
  including linux kernels >= 2.4.21\. it is used to bind a socket to ipv6 only
  when it uses the default address. doing so is sometimes preferred to doing it
  system-wide as it is per-listener. it has no effect on non-ipv6 sockets and
  has precedence over the "v4v6" option.

uid <uid>
  sets the owner of the unix sockets to the designated system uid. it can also
  be set by default in the global section's "unix-bind" statement. note that
  some platforms simply ignore this. this setting is equivalent to the "user"
  setting except that the user numeric id is used instead of its name. this
  setting is ignored by non unix sockets.

user <user>
  sets the owner of the unix sockets to the designated system user. it can also
  be set by default in the global section's "unix-bind" statement. note that
  some platforms simply ignore this. this setting is equivalent to the "uid"
  setting except that the user name is used instead of its uid. this setting is
  ignored by non unix sockets.

verify [none|optional|required]
  this setting is only available when support for openssl was built in. if set
  to 'none', client certificate is not requested. this is the default. in other
  cases, a client certificate is requested. if the client does not provide a
  certificate after the request and if 'verify' is set to 'required', then the
  handshake is aborted, while it would have succeeded if set to 'optional'. the
  certificate provided by the client is always verified using cas from
  'ca-file' and optional crls from 'crl-file'. on verify failure the handshake
  is aborted, regardless of the 'verify' option, unless the error code exactly
  matches one of those listed with 'ca-ignore-err' or 'crt-ignore-err'.

5.2\. server and default-server options
------------------------------------

the "server" and "default-server" keywords support a certain number of settings
which are all passed as arguments on the server line. the order in which those
arguments appear does not count, and they are all optional. some of those
settings are single words (booleans) while others expect one or several values
after them. in this case, the values must immediately follow the setting name.
except default-server, all those settings must be specified after the server's
address if they are used:

  server <name> <address>[:port] [settings ...]
  default-server [settings ...]

the currently supported settings are the following ones.

addr <ipv4|ipv6>
  using the "addr" parameter, it becomes possible to use a different ip address
  to send health-checks. on some servers, it may be desirable to dedicate an ip
  address to specific component able to perform complex tests which are more
  suitable to health-checks than the application. this parameter is ignored if
  the "check" parameter is not set. see also the "port" parameter.

  supported in default-server: no

agent-check
  enable an auxiliary agent check which is run independently of a regular
  health check. an agent health check is performed by making a tcp connection
  to the port set by the "agent-port" parameter and reading an ascii string.
  the string is made of a series of words delimited by spaces, tabs or commas
  in any order, optionally terminated by '\r' and/or '\n', each consisting of :

  - an ascii representation of a positive integer percentage, e.g. "75%".
    values in this format will set the weight proportional to the initial
    weight of a server as configured when haproxy starts. note that a zero
    weight is reported on the stats page as "drain" since it has the same
    effect on the server (it's removed from the lb farm).

  - the word "ready". this will turn the server's administrative state to the
    ready mode, thus cancelling any drain or maint state

  - the word "drain". this will turn the server's administrative state to the
    drain mode, thus it will not accept any new connections other than those
    that are accepted via persistence.

  - the word "maint". this will turn the server's administrative state to the
    maint mode, thus it will not accept any new connections at all, and health
    checks will be stopped.

  - the words "down", "failed", or "stopped", optionally followed by a
    description string after a sharp ('#'). all of these mark the server's
    operating state as down, but since the word itself is reported on the stats
    page, the difference allows an administrator to know if the situation was
    expected or not : the service may intentionally be stopped, may appear up
    but fail some validity tests, or may be seen as down (eg: missing process,
    or port not responding).

  - the word "up" sets back the server's operating state as up if health checks
    also report that the service is accessible.

  parameters which are not advertised by the agent are not changed. for
  example, an agent might be designed to monitor cpu usage and only report a
  relative weight and never interact with the operating status. similarly, an
  agent could be designed as an end-user interface with 3 radio buttons
  allowing an administrator to change only the administrative state. however,
  it is important to consider that only the agent may revert its own actions,
  so if a server is set to drain mode or to down state using the agent, the
  agent must implement the other equivalent actions to bring the service into
  operations again.

  failure to connect to the agent is not considered an error as connectivity
  is tested by the regular health check which is enabled by the "check"
  parameter. warning though, it is not a good idea to stop an agent after it
  reports "down", since only an agent reporting "up" will be able to turn the
  server up again. note that the cli on the unix stats socket is also able to
  force an agent's result in order to workaround a bogus agent if needed.

  requires the "agent-port" parameter to be set. see also the "agent-inter"
  parameter.

  supported in default-server: no

agent-inter <delay>
  the "agent-inter" parameter sets the interval between two agent checks
  to <delay> milliseconds. if left unspecified, the delay defaults to 2000 ms.

  just as with every other time-based parameter, it may be entered in any
  other explicit unit among { us, ms, s, m, h, d }. the "agent-inter"
  parameter also serves as a timeout for agent checks "timeout check" is
  not set. in order to reduce "resonance" effects when multiple servers are
  hosted on the same hardware, the agent and health checks of all servers
  are started with a small time offset between them. it is also possible to
  add some random noise in the agent and health checks interval using the
  global "spread-checks" keyword. this makes sense for instance when a lot
  of backends use the same servers.

  see also the "agent-check" and "agent-port" parameters.

  supported in default-server: yes

agent-port <port>
  the "agent-port" parameter sets the tcp port used for agent checks.

  see also the "agent-check" and "agent-inter" parameters.

  supported in default-server: yes

backup
  when "backup" is present on a server line, the server is only used in load
  balancing when all other non-backup servers are unavailable. requests coming
  with a persistence cookie referencing the server will always be served
  though. by default, only the first operational backup server is used, unless
  the "allbackups" option is set in the backend. see also the "allbackups"
  option.

  supported in default-server: no

ca-file <cafile>
  this setting is only available when support for openssl was built in. it
  designates a pem file from which to load ca certificates used to verify
  server's certificate.

  supported in default-server: no

check
  this option enables health checks on the server. by default, a server is
  always considered available. if "check" is set, the server is available when
  accepting periodic tcp connections, to ensure that it is really able to serve
  requests. the default address and port to send the tests to are those of the
  server, and the default source is the same as the one defined in the
  backend. it is possible to change the address using the "addr" parameter, the
  port using the "port" parameter, the source address using the "source"
  address, and the interval and timers using the "inter", "rise" and "fall"
  parameters. the request method is define in the backend using the "httpchk",
  "smtpchk", "mysql-check", "pgsql-check" and "ssl-hello-chk" options. please
  refer to those options and parameters for more information.

  supported in default-server: no

check-send-proxy
  this option forces emission of a proxy protocol line with outgoing health
  checks, regardless of whether the server uses send-proxy or not for the
  normal traffic. by default, the proxy protocol is enabled for health checks
  if it is already enabled for normal traffic and if no "port" nor "addr"
  directive is present. however, if such a directive is present, the
  "check-send-proxy" option needs to be used to force the use of the
  protocol. see also the "send-proxy" option for more information.

  supported in default-server: no

check-ssl
  this option forces encryption of all health checks over ssl, regardless of
  whether the server uses ssl or not for the normal traffic. this is generally
  used when an explicit "port" or "addr" directive is specified and ssl health
  checks are not inherited. it is important to understand that this option
  inserts an ssl transport layer below the checks, so that a simple tcp connect
  check becomes an ssl connect, which replaces the old ssl-hello-chk. the most
  common use is to send https checks by combining "httpchk" with ssl checks.
  all ssl settings are common to health checks and traffic (eg: ciphers).
  see the "ssl" option for more information.

  supported in default-server: no

ciphers <ciphers>
  this option sets the string describing the list of cipher algorithms that is
  is negotiated during the ssl/tls handshake with the server. the format of the
  string is defined in "man 1 ciphers". when ssl is used to communicate with
  servers on the local network, it is common to see a weaker set of algorithms
  than what is used over the internet. doing so reduces cpu usage on both the
  server and haproxy while still keeping it compatible with deployed software.
  some algorithms such as rc4-sha1 are reasonably cheap. if no security at all
  is needed and just connectivity, using des can be appropriate.

  supported in default-server: no

cookie <value>
  the "cookie" parameter sets the cookie value assigned to the server to
  <value>. this value will be checked in incoming requests, and the first
  operational server possessing the same value will be selected. in return, in
  cookie insertion or rewrite modes, this value will be assigned to the cookie
  sent to the client. there is nothing wrong in having several servers sharing
  the same cookie value, and it is in fact somewhat common between normal and
  backup servers. see also the "cookie" keyword in backend section.

  supported in default-server: no

crl-file <crlfile>
  this setting is only available when support for openssl was built in. it
  designates a pem file from which to load certificate revocation list used
  to verify server's certificate.

  supported in default-server: no

crt <cert>
  this setting is only available when support for openssl was built in.
  it designates a pem file from which to load both a certificate and the
  associated private key. this file can be built by concatenating both pem
  files into one. this certificate will be sent if the server send a client
  certificate request.

  supported in default-server: no

disabled
  the "disabled" keyword starts the server in the "disabled" state. that means
  that it is marked down in maintenance mode, and no connection other than the
  ones allowed by persist mode will reach it. it is very well suited to setup
  new servers, because normal traffic will never reach them, while it is still
  possible to test the service by making use of the force-persist mechanism.

  supported in default-server: no

error-limit <count>
  if health observing is enabled, the "error-limit" parameter specifies the
  number of consecutive errors that triggers event selected by the "on-error"
  option. by default it is set to 10 consecutive errors.

  supported in default-server: yes

  see also the "check", "error-limit" and "on-error".

fall <count>
  the "fall" parameter states that a server will be considered as dead after
  <count> consecutive unsuccessful health checks. this value defaults to 3 if
  unspecified. see also the "check", "inter" and "rise" parameters.

  supported in default-server: yes

force-sslv3
  this option enforces use of sslv3 only when ssl is used to communicate with
  the server. sslv3 is generally less expensive than the tls counterparts for
  high connection rates. this option is also available on global statement
  "ssl-default-server-options". see also "no-tlsv*", "no-sslv3".

  supported in default-server: no

force-tlsv10
  this option enforces use of tlsv1.0 only when ssl is used to communicate with
  the server. this option is also available on global statement
  "ssl-default-server-options". see also "no-tlsv*", "no-sslv3".

  supported in default-server: no

force-tlsv11
  this option enforces use of tlsv1.1 only when ssl is used to communicate with
  the server. this option is also available on global statement
  "ssl-default-server-options". see also "no-tlsv*", "no-sslv3".

  supported in default-server: no

force-tlsv12
  this option enforces use of tlsv1.2 only when ssl is used to communicate with
  the server. this option is also available on global statement
  "ssl-default-server-options". see also "no-tlsv*", "no-sslv3".

  supported in default-server: no

id <value>
  set a persistent id for the server. this id must be positive and unique for
  the proxy. an unused id will automatically be assigned if unset. the first
  assigned value will be 1\. this id is currently only returned in statistics.

  supported in default-server: no

inter <delay>
fastinter <delay>
downinter <delay>
  the "inter" parameter sets the interval between two consecutive health checks
  to <delay> milliseconds. if left unspecified, the delay defaults to 2000 ms.
  it is also possible to use "fastinter" and "downinter" to optimize delays
  between checks depending on the server state :

             server state            |             interval used
    ---------------------------------+-----------------------------------------
     up 100% (non-transitional)      | "inter"
    ---------------------------------+-----------------------------------------
     transitionally up (going down), |
     transitionally down (going up), | "fastinter" if set, "inter" otherwise.
     or yet unchecked.               |
    ---------------------------------+-----------------------------------------
     down 100% (non-transitional)    | "downinter" if set, "inter" otherwise.
    ---------------------------------+-----------------------------------------

  just as with every other time-based parameter, they can be entered in any
  other explicit unit among { us, ms, s, m, h, d }. the "inter" parameter also
  serves as a timeout for health checks sent to servers if "timeout check" is
  not set. in order to reduce "resonance" effects when multiple servers are
  hosted on the same hardware, the agent and health checks of all servers
  are started with a small time offset between them. it is also possible to
  add some random noise in the agent and health checks interval using the
  global "spread-checks" keyword. this makes sense for instance when a lot
  of backends use the same servers.

  supported in default-server: yes

maxconn <maxconn>
  the "maxconn" parameter specifies the maximal number of concurrent
  connections that will be sent to this server. if the number of incoming
  concurrent requests goes higher than this value, they will be queued, waiting
  for a connection to be released. this parameter is very important as it can
  save fragile servers from going down under extreme loads. if a "minconn"
  parameter is specified, the limit becomes dynamic. the default value is "0"
  which means unlimited. see also the "minconn" and "maxqueue" parameters, and
  the backend's "fullconn" keyword.

  supported in default-server: yes

maxqueue <maxqueue>
  the "maxqueue" parameter specifies the maximal number of connections which
  will wait in the queue for this server. if this limit is reached, next
  requests will be redispatched to other servers instead of indefinitely
  waiting to be served. this will break persistence but may allow people to
  quickly re-log in when the server they try to connect to is dying. the
  default value is "0" which means the queue is unlimited. see also the
  "maxconn" and "minconn" parameters.

  supported in default-server: yes

minconn <minconn>
  when the "minconn" parameter is set, the maxconn limit becomes a dynamic
  limit following the backend's load. the server will always accept at least
  <minconn> connections, never more than <maxconn>, and the limit will be on
  the ramp between both values when the backend has less than <fullconn>
  concurrent connections. this makes it possible to limit the load on the
  server during normal loads, but push it further for important loads without
  overloading the server during exceptional loads. see also the "maxconn"
  and "maxqueue" parameters, as well as the "fullconn" backend keyword.

  supported in default-server: yes

no-ssl-reuse
  this option disables ssl session reuse when ssl is used to communicate with
  the server. it will force the server to perform a full handshake for every
  new connection. it's probably only useful for benchmarking, troubleshooting,
  and for paranoid users.

  supported in default-server: no

no-sslv3
  this option disables support for sslv3 when ssl is used to communicate with
  the server. note that sslv2 is disabled in the code and cannot be enabled
  using any configuration option. see also "force-sslv3", "force-tlsv*".

  supported in default-server: no

no-tls-tickets
  this setting is only available when support for openssl was built in. it
  disables the stateless session resumption (rfc 5077 tls ticket
  extension) and force to use stateful session resumption. stateless
  session resumption is more expensive in cpu usage for servers. this option
  is also available on global statement "ssl-default-server-options".

  supported in default-server: no

no-tlsv10
  this option disables support for tlsv1.0 when ssl is used to communicate with
  the server. note that sslv2 is disabled in the code and cannot be enabled
  using any configuration option. tlsv1 is more expensive than sslv3 so it
  often makes sense to disable it when communicating with local servers. this
  option is also available on global statement "ssl-default-server-options".
  see also "force-sslv3", "force-tlsv*".

  supported in default-server: no

no-tlsv11
  this option disables support for tlsv1.1 when ssl is used to communicate with
  the server. note that sslv2 is disabled in the code and cannot be enabled
  using any configuration option. tlsv1 is more expensive than sslv3 so it
  often makes sense to disable it when communicating with local servers. this
  option is also available on global statement "ssl-default-server-options".
  see also "force-sslv3", "force-tlsv*".

  supported in default-server: no

no-tlsv12
  this option disables support for tlsv1.2 when ssl is used to communicate with
  the server. note that sslv2 is disabled in the code and cannot be enabled
  using any configuration option. tlsv1 is more expensive than sslv3 so it
  often makes sense to disable it when communicating with local servers. this
  option is also available on global statement "ssl-default-server-options".
  see also "force-sslv3", "force-tlsv*".

  supported in default-server: no

non-stick
  never add connections allocated to this sever to a stick-table.
  this may be used in conjunction with backup to ensure that
  stick-table persistence is disabled for backup servers.

  supported in default-server: no

observe <mode>
  this option enables health adjusting based on observing communication with
  the server. by default this functionality is disabled and enabling it also
  requires to enable health checks. there are two supported modes: "layer4" and
  "layer7". in layer4 mode, only successful/unsuccessful tcp connections are
  significant. in layer7, which is only allowed for http proxies, responses
  received from server are verified, like valid/wrong http code, unparsable
  headers, a timeout, etc. valid status codes include 100 to 499, 501 and 505.

  supported in default-server: no

  see also the "check", "on-error" and "error-limit".

on-error <mode>
  select what should happen when enough consecutive errors are detected.
  currently, four modes are available:
  - fastinter: force fastinter
  - fail-check: simulate a failed check, also forces fastinter (default)
  - sudden-death: simulate a pre-fatal failed health check, one more failed
    check will mark a server down, forces fastinter
  - mark-down: mark the server immediately down and force fastinter

  supported in default-server: yes

  see also the "check", "observe" and "error-limit".

on-marked-down <action>
  modify what occurs when a server is marked down.
  currently one action is available:
  - shutdown-sessions: shutdown peer sessions. when this setting is enabled,
    all connections to the server are immediately terminated when the server
    goes down. it might be used if the health check detects more complex cases
    than a simple connection status, and long timeouts would cause the service
    to remain unresponsive for too long a time. for instance, a health check
    might detect that a database is stuck and that there's no chance to reuse
    existing connections anymore. connections killed this way are logged with
    a 'd' termination code (for "down").

  actions are disabled by default

  supported in default-server: yes

on-marked-up <action>
  modify what occurs when a server is marked up.
  currently one action is available:
  - shutdown-backup-sessions: shutdown sessions on all backup servers. this is
    done only if the server is not in backup state and if it is not disabled
    (it must have an effective weight > 0). this can be used sometimes to force
    an active server to take all the traffic back after recovery when dealing
    with long sessions (eg: ldap, sql, ...). doing this can cause more trouble
    than it tries to solve (eg: incomplete transactions), so use this feature
    with extreme care. sessions killed because a server comes up are logged
    with an 'u' termination code (for "up").

  actions are disabled by default

  supported in default-server: yes

port <port>
  using the "port" parameter, it becomes possible to use a different port to
  send health-checks. on some servers, it may be desirable to dedicate a port
  to a specific component able to perform complex tests which are more suitable
  to health-checks than the application. it is common to run a simple script in
  inetd for instance. this parameter is ignored if the "check" parameter is not
  set. see also the "addr" parameter.

  supported in default-server: yes

redir <prefix>
  the "redir" parameter enables the redirection mode for all get and head
  requests addressing this server. this means that instead of having haproxy
  forward the request to the server, it will send an "http 302" response with
  the "location" header composed of this prefix immediately followed by the
  requested uri beginning at the leading '/' of the path component. that means
  that no trailing slash should be used after <prefix>. all invalid requests
  will be rejected, and all non-get or head requests will be normally served by
  the server. note that since the response is completely forged, no header
  mangling nor cookie insertion is possible in the response. however, cookies in
  requests are still analysed, making this solution completely usable to direct
  users to a remote location in case of local disaster. main use consists in
  increasing bandwidth for static servers by having the clients directly
  connect to them. note: never use a relative location here, it would cause a
  loop between the client and haproxy!

  example :  server srv1 192.168.1.1:80 redir http://image1.mydomain.com check

  supported in default-server: no

rise <count>
  the "rise" parameter states that a server will be considered as operational
  after <count> consecutive successful health checks. this value defaults to 2
  if unspecified. see also the "check", "inter" and "fall" parameters.

  supported in default-server: yes

resolve-prefer <family>
  when dns resolution is enabled for a server and multiple ip addresses from
  different families are returned, haproxy will prefer using an ip address
  from the family mentioned in the "resolve-prefer" parameter.
  available families: "ipv4" and "ipv6"

  default value: ipv4

  example: server s1 app1.domain.com:80 resolvers mydns resolve-prefer ipv6

resolvers <id>
  points to an existing "resolvers" section to resolve current server's
  hostname.

  example: server s1 app1.domain.com:80 resolvers mydns

  see also chapter 5.3

send-proxy
  the "send-proxy" parameter enforces use of the proxy protocol over any
  connection established to this server. the proxy protocol informs the other
  end about the layer 3/4 addresses of the incoming connection, so that it can
  know the client's address or the public address it accessed to, whatever the
  upper layer protocol. for connections accepted by an "accept-proxy" listener,
  the advertised address will be used. only tcpv4 and tcpv6 address families
  are supported. other families such as unix sockets, will report an unknown
  family. servers using this option can fully be chained to another instance of
  haproxy listening with an "accept-proxy" setting. this setting must not be
  used if the server isn't aware of the protocol. when health checks are sent
  to the server, the proxy protocol is automatically used when this option is
  set, unless there is an explicit "port" or "addr" directive, in which case an
  explicit "check-send-proxy" directive would also be needed to use the proxy
  protocol. see also the "accept-proxy" option of the "bind" keyword.

  supported in default-server: no

send-proxy-v2
  the "send-proxy-v2" parameter enforces use of the proxy protocol version 2
  over any connection established to this server. the proxy protocol informs
  the other end about the layer 3/4 addresses of the incoming connection, so
  that it can know the client's address or the public address it accessed to,
  whatever the upper layer protocol. this setting must not be used if the
  server isn't aware of this version of the protocol. see also the "send-proxy"
  option of the "bind" keyword.

  supported in default-server: no

send-proxy-v2-ssl
  the "send-proxy-v2-ssl" parameter enforces use of the proxy protocol version
  2 over any connection established to this server. the proxy protocol informs
  the other end about the layer 3/4 addresses of the incoming connection, so
  that it can know the client's address or the public address it accessed to,
  whatever the upper layer protocol. in addition, the ssl information extension
  of the proxy protocol is added to the proxy protocol header. this setting
  must not be used if the server isn't aware of this version of the protocol.
  see also the "send-proxy-v2" option of the "bind" keyword.

  supported in default-server: no

send-proxy-v2-ssl-cn
  the "send-proxy-v2-ssl" parameter enforces use of the proxy protocol version
  2 over any connection established to this server. the proxy protocol informs
  the other end about the layer 3/4 addresses of the incoming connection, so
  that it can know the client's address or the public address it accessed to,
  whatever the upper layer protocol. in addition, the ssl information extension
  of the proxy protocol, along along with the common name from the subject of
  the client certificate (if any), is added to the proxy protocol header. this
  setting must not be used if the server isn't aware of this version of the
  protocol. see also the "send-proxy-v2" option of the "bind" keyword.

  supported in default-server: no

slowstart <start_time_in_ms>
  the "slowstart" parameter for a server accepts a value in milliseconds which
  indicates after how long a server which has just come back up will run at
  full speed. just as with every other time-based parameter, it can be entered
  in any other explicit unit among { us, ms, s, m, h, d }. the speed grows
  linearly from 0 to 100% during this time. the limitation applies to two
  parameters :

  - maxconn: the number of connections accepted by the server will grow from 1
    to 100% of the usual dynamic limit defined by (minconn,maxconn,fullconn).

  - weight: when the backend uses a dynamic weighted algorithm, the weight
    grows linearly from 1 to 100%. in this case, the weight is updated at every
    health-check. for this reason, it is important that the "inter" parameter
    is smaller than the "slowstart", in order to maximize the number of steps.

  the slowstart never applies when haproxy starts, otherwise it would cause
  trouble to running servers. it only applies when a server has been previously
  seen as failed.

  supported in default-server: yes

sni <expression>
  the "sni" parameter evaluates the sample fetch expression, converts it to a
  string and uses the result as the host name sent in the sni tls extension to
  the server. a typical use case is to send the sni received from the client in
  a bridged https scenario, using the "ssl_fc_sni" sample fetch for the
  expression, though alternatives such as req.hdr(host) can also make sense.

  supported in default-server: no

source <addr>[:<pl>[-<ph>]] [usesrc { <addr2>[:<port2>] | client | clientip } ]
source <addr>[:<port>] [usesrc { <addr2>[:<port2>] | hdr_ip(<hdr>[,<occ>]) } ]
source <addr>[:<pl>[-<ph>]] [interface <name>] ...
  the "source" parameter sets the source address which will be used when
  connecting to the server. it follows the exact same parameters and principle
  as the backend "source" keyword, except that it only applies to the server
  referencing it. please consult the "source" keyword for details.

  additionally, the "source" statement on a server line allows one to specify a
  source port range by indicating the lower and higher bounds delimited by a
  dash ('-'). some operating systems might require a valid ip address when a
  source port range is specified. it is permitted to have the same ip/range for
  several servers. doing so makes it possible to bypass the maximum of 64k
  total concurrent connections. the limit will then reach 64k connections per
  server.

  supported in default-server: no

ssl
  this option enables ssl ciphering on outgoing connections to the server. it
  is critical to verify server certificates using "verify" when using ssl to
  connect to servers, otherwise the communication is prone to trivial man in
  the-middle attacks rendering ssl useless. when this option is used, health
  checks are automatically sent in ssl too unless there is a "port" or an
  "addr" directive indicating the check should be sent to a different location.
  see the "check-ssl" option to force ssl health checks.

  supported in default-server: no

track [<proxy>/]<server>
  this option enables ability to set the current state of the server by tracking
  another one. it is possible to track a server which itself tracks another
  server, provided that at the end of the chain, a server has health checks
  enabled. if <proxy> is omitted the current one is used. if disable-on-404 is
  used, it has to be enabled on both proxies.

  supported in default-server: no

verify [none|required]
  this setting is only available when support for openssl was built in. if set
  to 'none', server certificate is not verified. in the other case, the
  certificate provided by the server is verified using cas from 'ca-file'
  and optional crls from 'crl-file'. if 'ssl_server_verify' is not specified
  in global  section, this is the default. on verify failure the handshake
  is aborted. it is critically important to verify server certificates when
  using ssl to connect to servers, otherwise the communication is prone to
  trivial man-in-the-middle attacks rendering ssl totally useless.

  supported in default-server: no

verifyhost <hostname>
  this setting is only available when support for openssl was built in, and
  only takes effect if 'verify required' is also specified. when set, the
  hostnames in the subject and subjectalternatenames of the certificate
  provided by the server are checked. if none of the hostnames in the
  certificate match the specified hostname, the handshake is aborted. the
  hostnames in the server-provided certificate may include wildcards.

  supported in default-server: no

weight <weight>
  the "weight" parameter is used to adjust the server's weight relative to
  other servers. all servers will receive a load proportional to their weight
  relative to the sum of all weights, so the higher the weight, the higher the
  load. the default weight is 1, and the maximal value is 256\. a value of 0
  means the server will not participate in load-balancing but will still accept
  persistent connections. if this parameter is used to distribute the load
  according to server's capacity, it is recommended to start with values which
  can both grow and shrink, for instance between 10 and 100 to leave enough
  room above and below for later adjustments.

  supported in default-server: yes

5.3\. server ip address resolution using dns
-------------------------------------------

haproxy allows using a host name to be resolved to find out what is the server
ip address. by default, haproxy resolves the name when parsing the
configuration, at startup.
this is not sufficient in some cases, such as in amazon where a server's ip
can change after a reboot or an elb virtual ip can change based on current
workload.
this chapter describes how haproxy can be configured to process server's name
resolution at run time.
whether run time server name resolution has been enable or not, haproxy will
carry on doing the first resolution when parsing the configuration.

5.3.1\. global overview
----------------------

as we've seen in introduction, name resolution in haproxy occurs at two
different steps of the process life:

  1\. when starting up, haproxy parses the server line definition and matches a
     host name. it uses libc functions to get the host name resolved. this
     resolution relies on /etc/resolv.conf file.

  2\. at run time, when haproxy gets prepared to run a health check on a server,
     it verifies if the current name resolution is still considered as valid.
     if not, it processes a new resolution, in parallel of the health check.

a few other events can trigger a name resolution at run time:
  - when a server's health check ends up in a connection timeout: this may be
    because the server has a new ip address. so we need to trigger a name
    resolution to know this new ip.

a few things important to notice:
  - all the name servers are queried in the mean time. haproxy will process the
    first valid response.

  - a resolution is considered as invalid (nx, timeout, refused), when all the
    servers return an error.

5.3.2\. the resolvers section
----------------------------

this section is dedicated to host information related to name resolution in
haproxy.
there can be as many as resolvers section as needed. each section can contain
many name servers.

resolvers <resolvers id>
  creates a new name server list labelled <resolvers id>

a resolvers section accept the following parameters:

nameserver <id> <ip>:<port>
  dns server description:
    <id>   : label of the server, should be unique
    <ip>   : ip address of the server
    <port> : port where the dns service actually runs

hold <status> <period>
  defines <period> during which the last name resolution should be kept based
  on last resolution <status>
    <status> : last name resolution status. only "valid" is accepted for now.
    <period> : interval between two successive name resolution when the last
               answer was in <status>. it follows the haproxy time format.
               <period> is in milliseconds by default.

  default value is 10s for "valid".

  note: since the name resolution is triggered by the health checks, a new
        resolution is triggered after <period> modulo the <inter> parameter of
        the healch check.

resolve_retries <nb>
  defines the number <nb> of queries to send to resolve a server name before
  giving up.
  default value: 3

timeout <event> <time>
  defines timeouts related to name resolution
     <event> : the event on which the <time> timeout period applies to.
               events available are:
               - retry: time between two dns queries, when no response have
                        been received.
                        default value: 1s
     <time>  : time related to the event. it follows the haproxy time format.
               <time> is expressed in milliseconds.

example of a resolvers section (with default values):

   resolvers mydns
     nameserver dns1 10.0.0.1:53
     nameserver dns2 10.0.0.2:53
     resolve_retries       3
     timeout retry         1s
     hold valid           10s

6\. http header manipulation
---------------------------

in http mode, it is possible to rewrite, add or delete some of the request and
response headers based on regular expressions. it is also possible to block a
request or a response if a particular header matches a regular expression,
which is enough to stop most elementary protocol attacks, and to protect
against information leak from the internal network.

if haproxy encounters an "informational response" (status code 1xx), it is able
to process all rsp* rules which can allow, deny, rewrite or delete a header,
but it will refuse to add a header to any such messages as this is not
http-compliant. the reason for still processing headers in such responses is to
stop and/or fix any possible information leak which may happen, for instance
because another downstream equipment would unconditionally add a header, or if
a server name appears there. when such messages are seen, normal processing
still occurs on the next non-informational messages.

this section covers common usage of the following keywords, described in detail
in section 4.2 :

  - reqadd     <string>
  - reqallow   <search>
  - reqiallow  <search>
  - reqdel     <search>
  - reqidel    <search>
  - reqdeny    <search>
  - reqideny   <search>
  - reqpass    <search>
  - reqipass   <search>
  - reqrep     <search> <replace>
  - reqirep    <search> <replace>
  - reqtarpit  <search>
  - reqitarpit <search>
  - rspadd     <string>
  - rspdel     <search>
  - rspidel    <search>
  - rspdeny    <search>
  - rspideny   <search>
  - rsprep     <search> <replace>
  - rspirep    <search> <replace>

with all these keywords, the same conventions are used. the <search> parameter
is a posix extended regular expression (regex) which supports grouping through
parenthesis (without the backslash). spaces and other delimiters must be
prefixed with a backslash ('\') to avoid confusion with a field delimiter.
other characters may be prefixed with a backslash to change their meaning :

  \t   for a tab
  \r   for a carriage return (cr)
  \n   for a new line (lf)
  \    to mark a space and differentiate it from a delimiter
  \#   to mark a sharp and differentiate it from a comment
  \\   to use a backslash in a regex
  \\\\ to use a backslash in the text (*2 for regex, *2 for haproxy)
  \xxx to write the ascii hex code xx as in the c language

the <replace> parameter contains the string to be used to replace the largest
portion of text matching the regex. it can make use of the special characters
above, and can reference a substring which is delimited by parenthesis in the
regex, by writing a backslash ('\') immediately followed by one digit from 0 to
9 indicating the group position (0 designating the entire line). this practice
is very common to users of the "sed" program.

the <string> parameter represents the string which will systematically be added
after the last header line. it can also use special character sequences above.

notes related to these keywords :
---------------------------------
  - these keywords are not always convenient to allow/deny based on header
    contents. it is strongly recommended to use acls with the "block" keyword
    instead, resulting in far more flexible and manageable rules.

  - lines are always considered as a whole. it is not possible to reference
    a header name only or a value only. this is important because of the way
    headers are written (notably the number of spaces after the colon).

  - the first line is always considered as a header, which makes it possible to
    rewrite or filter http requests uris or response codes, but in turn makes
    it harder to distinguish between headers and request line. the regex prefix
    ^[^\ \t]*[\ \t] matches any http method followed by a space, and the prefix
    ^[^ \t:]*: matches any header name followed by a colon.

  - for performances reasons, the number of characters added to a request or to
    a response is limited at build time to values between 1 and 4 kb. this
    should normally be far more than enough for most usages. if it is too short
    on occasional usages, it is possible to gain some space by removing some
    useless headers before adding new ones.

  - keywords beginning with "reqi" and "rspi" are the same as their counterpart
    without the 'i' letter except that they ignore case when matching patterns.

  - when a request passes through a frontend then a backend, all req* rules
    from the frontend will be evaluated, then all req* rules from the backend
    will be evaluated. the reverse path is applied to responses.

  - req* statements are applied after "block" statements, so that "block" is
    always the first one, but before "use_backend" in order to permit rewriting
    before switching.

7\. using acls and fetching samples
----------------------------------

haproxy is capable of extracting data from request or response streams, from
client or server information, from tables, environmental information etc...
the action of extracting such data is called fetching a sample. once retrieved,
these samples may be used for various purposes such as a key to a stick-table,
but most common usages consist in matching them against predefined constant
data called patterns.

7.1\. acl basics
---------------

the use of access control lists (acl) provides a flexible solution to perform
content switching and generally to take decisions based on content extracted
from the request, the response or any environmental status. the principle is
simple :

  - extract a data sample from a stream, table or the environment
  - optionally apply some format conversion to the extracted sample
  - apply one or multiple pattern matching methods on this sample
  - perform actions only when a pattern matches the sample

the actions generally consist in blocking a request, selecting a backend, or
adding a header.

in order to define a test, the "acl" keyword is used. the syntax is :

   acl <aclname> <criterion> [flags] [operator] [<value>] ...

this creates a new acl <aclname> or completes an existing one with new tests.
those tests apply to the portion of request/response specified in <criterion>
and may be adjusted with optional flags [flags]. some criteria also support
an operator which may be specified before the set of values. optionally some
conversion operators may be applied to the sample, and they will be specified
as a comma-delimited list of keywords just after the first keyword. the values
are of the type supported by the criterion, and are separated by spaces.

acl names must be formed from upper and lower case letters, digits, '-' (dash),
'_' (underscore) , '.' (dot) and ':' (colon). acl names are case-sensitive,
which means that "my_acl" and "my_acl" are two different acls.

there is no enforced limit to the number of acls. the unused ones do not affect
performance, they just consume a small amount of memory.

the criterion generally is the name of a sample fetch method, or one of its acl
specific declinations. the default test method is implied by the output type of
this sample fetch method. the acl declinations can describe alternate matching
methods of a same sample fetch method. the sample fetch methods are the only
ones supporting a conversion.

sample fetch methods return data which can be of the following types :
  - boolean
  - integer (signed or unsigned)
  - ipv4 or ipv6 address
  - string
  - data block

converters transform any of these data into any of these. for example, some
converters might convert a string to a lower-case string while other ones
would turn a string to an ipv4 address, or apply a netmask to an ip address.
the resulting sample is of the type of the last converter applied to the list,
which defaults to the type of the sample fetch method.

each sample or converter returns data of a specific type, specified with its
keyword in this documentation. when an acl is declared using a standard sample
fetch method, certain types automatically involved a default matching method
which are summarized in the table below :

   +---------------------+-----------------+
   | sample or converter | default         |
   |    output type      | matching method |
   +---------------------+-----------------+
   | boolean             | bool            |
   +---------------------+-----------------+
   | integer             | int             |
   +---------------------+-----------------+
   | ip                  | ip              |
   +---------------------+-----------------+
   | string              | str             |
   +---------------------+-----------------+
   | binary              | none, use "-m"  |
   +---------------------+-----------------+

note that in order to match a binary samples, it is mandatory to specify a
matching method, see below.

the acl engine can match these types against patterns of the following types :
  - boolean
  - integer or integer range
  - ip address / network
  - string (exact, substring, suffix, prefix, subdir, domain)
  - regular expression
  - hex block

the following acl flags are currently supported :

   -i : ignore case during matching of all subsequent patterns.
   -f : load patterns from a file.
   -m : use a specific pattern matching method
   -n : forbid the dns resolutions
   -m : load the file pointed by -f like a map file.
   -u : force the unique id of the acl
   -- : force end of flags. useful when a string looks like one of the flags.

the "-f" flag is followed by the name of a file from which all lines will be
read as individual values. it is even possible to pass multiple "-f" arguments
if the patterns are to be loaded from multiple files. empty lines as well as
lines beginning with a sharp ('#') will be ignored. all leading spaces and tabs
will be stripped. if it is absolutely necessary to insert a valid pattern
beginning with a sharp, just prefix it with a space so that it is not taken for
a comment. depending on the data type and match method, haproxy may load the
lines into a binary tree, allowing very fast lookups. this is true for ipv4 and
exact string matching. in this case, duplicates will automatically be removed.

the "-m" flag allows an acl to use a map file. if this flag is set, the file is
parsed as two column file. the first column contains the patterns used by the
acl, and the second column contain the samples. the sample can be used later by
a map. this can be useful in some rare cases where an acl would just be used to
check for the existence of a pattern in a map before a mapping is applied.

the "-u" flag forces the unique id of the acl. this unique id is used with the
socket interface to identify acl and dynamically change its values. note that a
file is always identified by its name even if an id is set.

also, note that the "-i" flag applies to subsequent entries and not to entries
loaded from files preceding it. for instance :

    acl valid-ua hdr(user-agent) -f exact-ua.lst -i -f generic-ua.lst test

in this example, each line of "exact-ua.lst" will be exactly matched against
the "user-agent" header of the request. then each line of "generic-ua" will be
case-insensitively matched. then the word "test" will be insensitively matched
as well.

the "-m" flag is used to select a specific pattern matching method on the input
sample. all acl-specific criteria imply a pattern matching method and generally
do not need this flag. however, this flag is useful with generic sample fetch
methods to describe how they're going to be matched against the patterns. this
is required for sample fetches which return data type for which there is no
obvious matching method (eg: string or binary). when "-m" is specified and
followed by a pattern matching method name, this method is used instead of the
default one for the criterion. this makes it possible to match contents in ways
that were not initially planned, or with sample fetch methods which return a
string. the matching method also affects the way the patterns are parsed.

the "-n" flag forbids the dns resolutions. it is used with the load of ip files.
by default, if the parser cannot parse ip address it considers that the parsed
string is maybe a domain name and try dns resolution. the flag "-n" disable this
resolution. it is useful for detecting malformed ip lists. note that if the dns
server is not reachable, the haproxy configuration parsing may last many minutes
waiting fir the timeout. during this time no error messages are displayed. the
flag "-n" disable this behavior. note also that during the runtime, this
function is disabled for the dynamic acl modifications.

there are some restrictions however. not all methods can be used with all
sample fetch methods. also, if "-m" is used in conjunction with "-f", it must
be placed first. the pattern matching method must be one of the following :

  - "found" : only check if the requested sample could be found in the stream,
              but do not compare it against any pattern. it is recommended not
              to pass any pattern to avoid confusion. this matching method is
              particularly useful to detect presence of certain contents such
              as headers, cookies, etc... even if they are empty and without
              comparing them to anything nor counting them.

  - "bool"  : check the value as a boolean. it can only be applied to fetches
              which return a boolean or integer value, and takes no pattern.
              value zero or false does not match, all other values do match.

  - "int"   : match the value as an integer. it can be used with integer and
              boolean samples. boolean false is integer 0, true is integer 1.

  - "ip"    : match the value as an ipv4 or ipv6 address. it is compatible
              with ip address samples only, so it is implied and never needed.

  - "bin"   : match the contents against an hexadecimal string representing a
              binary sequence. this may be used with binary or string samples.

  - "len"   : match the sample's length as an integer. this may be used with
              binary or string samples.

  - "str"   : exact match : match the contents against a string. this may be
              used with binary or string samples.

  - "sub"   : substring match : check that the contents contain at least one of
              the provided string patterns. this may be used with binary or
              string samples.

  - "reg"   : regex match : match the contents against a list of regular
              expressions. this may be used with binary or string samples.

  - "beg"   : prefix match : check that the contents begin like the provided
              string patterns. this may be used with binary or string samples.

  - "end"   : suffix match : check that the contents end like the provided
              string patterns. this may be used with binary or string samples.

  - "dir"   : subdir match : check that a slash-delimited portion of the
              contents exactly matches one of the provided string patterns.
              this may be used with binary or string samples.

  - "dom"   : domain match : check that a dot-delimited portion of the contents
              exactly match one of the provided string patterns. this may be
              used with binary or string samples.

for example, to quickly detect the presence of cookie "jsessionid" in an http
request, it is possible to do :

    acl jsess_present cook(jsessionid) -m found

in order to apply a regular expression on the 500 first bytes of data in the
buffer, one would use the following acl :

    acl script_tag payload(0,500) -m reg -i <script>

on systems where the regex library is much slower when using "-i", it is
possible to convert the sample to lowercase before matching, like this :

    acl script_tag payload(0,500),lower -m reg <script>

all acl-specific criteria imply a default matching method. most often, these
criteria are composed by concatenating the name of the original sample fetch
method and the matching method. for example, "hdr_beg" applies the "beg" match
to samples retrieved using the "hdr" fetch method. since all acl-specific
criteria rely on a sample fetch method, it is always possible instead to use
the original sample fetch method and the explicit matching method using "-m".

if an alternate match is specified using "-m" on an acl-specific criterion,
the matching method is simply applied to the underlying sample fetch method.
for example, all acls below are exact equivalent :

    acl short_form  hdr_beg(host)        www.
    acl alternate1  hdr_beg(host) -m beg www.
    acl alternate2  hdr_dom(host) -m beg www.
    acl alternate3  hdr(host)     -m beg www.

the table below summarizes the compatibility matrix between sample or converter
types and the pattern types to fetch against. it indicates for each compatible
combination the name of the matching method to be used, surrounded with angle
brackets ">" and "<" when the method is the default one and will work by
default without "-m".

                           +-------------------------------------------------+
                           |                input sample type                |
    +----------------------+---------+---------+---------+---------+---------+
    |     pattern type     | boolean | integer |   ip    | string  | binary  |
    +----------------------+---------+---------+---------+---------+---------+
    | none (presence only) |  found  |  found  |  found  |  found  |  found  |
    +----------------------+---------+---------+---------+---------+---------+
    | none (boolean value) |>  bool <|   bool  |         |   bool  |         |
    +----------------------+---------+---------+---------+---------+---------+
    | integer (value)      |   int   |>  int  <|   int   |   int   |         |
    +----------------------+---------+---------+---------+---------+---------+
    | integer (length)     |   len   |   len   |   len   |   len   |   len   |
    +----------------------+---------+---------+---------+---------+---------+
    | ip address           |         |         |>   ip  <|    ip   |    ip   |
    +----------------------+---------+---------+---------+---------+---------+
    | exact string         |   str   |   str   |   str   |>  str  <|   str   |
    +----------------------+---------+---------+---------+---------+---------+
    | prefix               |   beg   |   beg   |   beg   |   beg   |   beg   |
    +----------------------+---------+---------+---------+---------+---------+
    | suffix               |   end   |   end   |   end   |   end   |   end   |
    +----------------------+---------+---------+---------+---------+---------+
    | substring            |   sub   |   sub   |   sub   |   sub   |   sub   |
    +----------------------+---------+---------+---------+---------+---------+
    | subdir               |   dir   |   dir   |   dir   |   dir   |   dir   |
    +----------------------+---------+---------+---------+---------+---------+
    | domain               |   dom   |   dom   |   dom   |   dom   |   dom   |
    +----------------------+---------+---------+---------+---------+---------+
    | regex                |   reg   |   reg   |   reg   |   reg   |   reg   |
    +----------------------+---------+---------+---------+---------+---------+
    | hex block            |         |         |         |   bin   |   bin   |
    +----------------------+---------+---------+---------+---------+---------+

7.1.1\. matching booleans
------------------------

in order to match a boolean, no value is needed and all values are ignored.
boolean matching is used by default for all fetch methods of type "boolean".
when boolean matching is used, the fetched value is returned as-is, which means
that a boolean "true" will always match and a boolean "false" will never match.

boolean matching may also be enforced using "-m bool" on fetch methods which
return an integer value. then, integer value 0 is converted to the boolean
"false" and all other values are converted to "true".

7.1.2\. matching integers
------------------------

integer matching applies by default to integer fetch methods. it can also be
enforced on boolean fetches using "-m int". in this case, "false" is converted
to the integer 0, and "true" is converted to the integer 1.

integer matching also supports integer ranges and operators. note that integer
matching only applies to positive values. a range is a value expressed with a
lower and an upper bound separated with a colon, both of which may be omitted.

for instance, "1024:65535" is a valid range to represent a range of
unprivileged ports, and "1024:" would also work. "0:1023" is a valid
representation of privileged ports, and ":1023" would also work.

as a special case, some acl functions support decimal numbers which are in fact
two integers separated by a dot. this is used with some version checks for
instance. all integer properties apply to those decimal numbers, including
ranges and operators.

for an easier usage, comparison operators are also supported. note that using
operators with ranges does not make much sense and is strongly discouraged.
similarly, it does not make much sense to perform order comparisons with a set
of values.

available operators for integer matching are :

  eq : true if the tested value equals at least one value
  ge : true if the tested value is greater than or equal to at least one value
  gt : true if the tested value is greater than at least one value
  le : true if the tested value is less than or equal to at least one value
  lt : true if the tested value is less than at least one value

for instance, the following acl matches any negative content-length header :

  acl negative-length hdr_val(content-length) lt 0

this one matches ssl versions between 3.0 and 3.1 (inclusive) :

  acl sslv3 req_ssl_ver 3:3.1

7.1.3\. matching strings
-----------------------

string matching applies to string or binary fetch methods, and exists in 6
different forms :

  - exact match     (-m str) : the extracted string must exactly match the
    patterns ;

  - substring match (-m sub) : the patterns are looked up inside the
    extracted string, and the acl matches if any of them is found inside ;

  - prefix match    (-m beg) : the patterns are compared with the beginning of
    the extracted string, and the acl matches if any of them matches.

  - suffix match    (-m end) : the patterns are compared with the end of the
    extracted string, and the acl matches if any of them matches.

  - subdir match    (-m sub) : the patterns are looked up inside the extracted
    string, delimited with slashes ("/"), and the acl matches if any of them
    matches.

  - domain match    (-m dom) : the patterns are looked up inside the extracted
    string, delimited with dots ("."), and the acl matches if any of them
    matches.

string matching applies to verbatim strings as they are passed, with the
exception of the backslash ("\") which makes it possible to escape some
characters such as the space. if the "-i" flag is passed before the first
string, then the matching will be performed ignoring the case. in order
to match the string "-i", either set it second, or pass the "--" flag
before the first string. same applies of course to match the string "--".

7.1.4\. matching regular expressions (regexes)
---------------------------------------------

just like with string matching, regex matching applies to verbatim strings as
they are passed, with the exception of the backslash ("\") which makes it
possible to escape some characters such as the space. if the "-i" flag is
passed before the first regex, then the matching will be performed ignoring
the case. in order to match the string "-i", either set it second, or pass
the "--" flag before the first string. same principle applies of course to
match the string "--".

7.1.5\. matching arbitrary data blocks
-------------------------------------

it is possible to match some extracted samples against a binary block which may
not safely be represented as a string. for this, the patterns must be passed as
a series of hexadecimal digits in an even number, when the match method is set
to binary. each sequence of two digits will represent a byte. the hexadecimal
digits may be used upper or lower case.

example :
    # match "hello\n" in the input stream (\x48 \x65 \x6c \x6c \x6f \x0a)
    acl hello payload(0,6) -m bin 48656c6c6f0a

7.1.6\. matching ipv4 and ipv6 addresses
---------------------------------------

ipv4 addresses values can be specified either as plain addresses or with a
netmask appended, in which case the ipv4 address matches whenever it is
within the network. plain addresses may also be replaced with a resolvable
host name, but this practice is generally discouraged as it makes it more
difficult to read and debug configurations. if hostnames are used, you should
at least ensure that they are present in /etc/hosts so that the configuration
does not depend on any random dns match at the moment the configuration is
parsed.

ipv6 may be entered in their usual form, with or without a netmask appended.
only bit counts are accepted for ipv6 netmasks. in order to avoid any risk of
trouble with randomly resolved ip addresses, host names are never allowed in
ipv6 patterns.

haproxy is also able to match ipv4 addresses with ipv6 addresses in the
following situations :
  - tested address is ipv4, pattern address is ipv4, the match applies
    in ipv4 using the supplied mask if any.
  - tested address is ipv6, pattern address is ipv6, the match applies
    in ipv6 using the supplied mask if any.
  - tested address is ipv6, pattern address is ipv4, the match applies in ipv4
    using the pattern's mask if the ipv6 address matches with 2002:ipv4::,
    ::ipv4 or ::ffff:ipv4, otherwise it fails.
  - tested address is ipv4, pattern address is ipv6, the ipv4 address is first
    converted to ipv6 by prefixing ::ffff: in front of it, then the match is
    applied in ipv6 using the supplied ipv6 mask.

7.2\. using acls to form conditions
----------------------------------

some actions are only performed upon a valid condition. a condition is a
combination of acls with operators. 3 operators are supported :

  - and (implicit)
  - or  (explicit with the "or" keyword or the "||" operator)
  - negation with the exclamation mark ("!")

a condition is formed as a disjunctive form:

   [!]acl1 [!]acl2 ... [!]acln  { or [!]acl1 [!]acl2 ... [!]acln } ...

such conditions are generally used after an "if" or "unless" statement,
indicating when the condition will trigger the action.

for instance, to block http requests to the "*" url with methods other than
"options", as well as post requests without content-length, and get or head
requests with a content-length greater than 0, and finally every request which
is not either get/head/post/options !

   acl missing_cl hdr_cnt(content-length) eq 0
   block if http_url_star !meth_options || meth_post missing_cl
   block if meth_get http_content
   block unless meth_get or meth_post or meth_options

to select a different backend for requests to static contents on the "www" site
and to every request on the "img", "video", "download" and "ftp" hosts :

   acl url_static  path_beg         /static /images /img /css
   acl url_static  path_end         .gif .png .jpg .css .js
   acl host_www    hdr_beg(host) -i www
   acl host_static hdr_beg(host) -i img. video. download. ftp.

   # now use backend "static" for all static-only hosts, and for static urls
   # of host "www". use backend "www" for the rest.
   use_backend static if host_static or host_www url_static
   use_backend www    if host_www

it is also possible to form rules using "anonymous acls". those are unnamed acl
expressions that are built on the fly without needing to be declared. they must
be enclosed between braces, with a space before and after each brace (because
the braces must be seen as independent words). example :

   the following rule :

       acl missing_cl hdr_cnt(content-length) eq 0
       block if meth_post missing_cl

   can also be written that way :

       block if meth_post { hdr_cnt(content-length) eq 0 }

it is generally not recommended to use this construct because it's a lot easier
to leave errors in the configuration when written that way. however, for very
simple rules matching only one source ip address for instance, it can make more
sense to use them than to declare acls with random names. another example of
good use is the following :

   with named acls :

        acl site_dead nbsrv(dynamic) lt 2
        acl site_dead nbsrv(static)  lt 2
        monitor fail  if site_dead

   with anonymous acls :

        monitor fail if { nbsrv(dynamic) lt 2 } || { nbsrv(static) lt 2 }

see section 4.2 for detailed help on the "block" and "use_backend" keywords.

7.3\. fetching samples
---------------------

historically, sample fetch methods were only used to retrieve data to match
against patterns using acls. with the arrival of stick-tables, a new class of
sample fetch methods was created, most often sharing the same syntax as their
acl counterpart. these sample fetch methods are also known as "fetches". as
of now, acls and fetches have converged. all acl fetch methods have been made
available as fetch methods, and acls may use any sample fetch method as well.

this section details all available sample fetch methods and their output type.
some sample fetch methods have deprecated aliases that are used to maintain
compatibility with existing configurations. they are then explicitly marked as
deprecated and should not be used in new setups.

the acl derivatives are also indicated when available, with their respective
matching methods. these ones all have a well defined default pattern matching
method, so it is never necessary (though allowed) to pass the "-m" option to
indicate how the sample will be matched using acls.

as indicated in the sample type versus matching compatibility matrix above,
when using a generic sample fetch method in an acl, the "-m" option is
mandatory unless the sample type is one of boolean, integer, ipv4 or ipv6\. when
the same keyword exists as an acl keyword and as a standard fetch method, the
acl engine will automatically pick the acl-only one by default.

some of these keywords support one or multiple mandatory arguments, and one or
multiple optional arguments. these arguments are strongly typed and are checked
when the configuration is parsed so that there is no risk of running with an
incorrect argument (eg: an unresolved backend name). fetch function arguments
are passed between parenthesis and are delimited by commas.  when an argument
is optional, it will be indicated below between square brackets ('[ ]'). when
all arguments are optional, the parenthesis may be omitted.

thus, the syntax of a standard sample fetch method is one of the following :
   - name
   - name(arg1)
   - name(arg1,arg2)

7.3.1\. converters
-----------------

sample fetch methods may be combined with transformations to be applied on top
of the fetched sample (also called "converters"). these combinations form what
is called "sample expressions" and the result is a "sample". initially this
was only supported by "stick on" and "stick store-request" directives but this
has now be extended to all places where samples may be used (acls, log-format,
unique-id-format, add-header, ...).

these transformations are enumerated as a series of specific keywords after the
sample fetch method. these keywords may equally be appended immediately after
the fetch keyword's argument, delimited by a comma. these keywords can also
support some arguments (eg: a netmask) which must be passed in parenthesis.

a certain category of converters are bitwise and arithmetic operators which
support performing basic operations on integers. some bitwise operations are
supported (and, or, xor, cpl) and some arithmetic operations are supported
(add, sub, mul, div, mod, neg). some comparators are provided (odd, even, not,
bool) which make it possible to report a match without having to write an acl.

the currently available list of transformation keywords include :

add(<value>)
  adds <value> to the input value of type signed integer, and returns the
  result as a signed integer. <value> can be a numeric value or a variable
  name. the name of the variable starts by an indication about its scope. the
  allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

and(<value>)
  performs a bitwise "and" between <value> and the input value of type signed
  integer, and returns the result as an signed integer. <value> can be a
  numeric value or a variable name. the name of the variable starts by an
  indication about its scope. the allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

base64
  converts a binary input sample to a base64 string. it is used to log or
  transfer binary content in a way that can be reliably transferred (eg:
  an ssl id can be copied in a header).

bool
  returns a boolean true if the input value of type signed integer is
  non-null, otherwise returns false. used in conjunction with and(), it can be
  used to report true/false for bit testing on input values (eg: verify the
  presence of a flag).

bytes(<offset>[,<length>])
  extracts some bytes from an input binary sample. the result is a binary
  sample starting at an offset (in bytes) of the original sample and
  optionnaly truncated at the given length.

cpl
  takes the input value of type signed integer, applies a ones-complement
  (flips all bits) and returns the result as an signed integer.

crc32([<avalanche>])
  hashes a binary input sample into an unsigned 32-bit quantity using the crc32
  hash function. optionally, it is possible to apply a full avalanche hash
  function to the output if the optional <avalanche> argument equals 1\. this
  converter uses the same functions as used by the various hash-based load
  balancing algorithms, so it will provide exactly the same results. it is
  provided for compatibility with other software which want a crc32 to be
  computed on some input keys, so it follows the most common implementation as
  found in ethernet, gzip, png, etc... it is slower than the other algorithms
  but may provide a better or at least less predictable distribution. it must
  not be used for security purposes as a 32-bit hash is trivial to break. see
  also "djb2", "sdbm", "wt6" and the "hash-type" directive.

da-csv(<prop>[,<prop>*])
  asks the deviceatlas converter to identify the user agent string passed on
  input, and to emit a string made of the concatenation of the properties
  enumerated in argument, delimited by the separator defined by the global
  keyword "deviceatlas-property-separator", or by default the pipe character
  ('|'). there's a limit of 5 different properties imposed by the haproxy
  configuration language.

  example:
    frontend www
	bind *:8881
	default_backend servers
	http-request set-header x-deviceatlas-data %[req.fhdr(user-agent),da-csv(primaryhardwaretype,osname,osversion,browsername,browserversion)]

debug
  this converter is used as debug tool. it dumps on screen the content and the
  type of the input sample. the sample is returned as is on its output. this
  converter only exists when haproxy was built with debugging enabled.

div(<value>)
  divides the input value of type signed integer by <value>, and returns the
  result as an signed integer. if <value> is null, the largest unsigned
  integer is returned (typically 2^63-1). <value> can be a numeric value or a
  variable name. the name of the variable starts by an indication about it
  scope. the scope allowed are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

djb2([<avalanche>])
  hashes a binary input sample into an unsigned 32-bit quantity using the djb2
  hash function. optionally, it is possible to apply a full avalanche hash
  function to the output if the optional <avalanche> argument equals 1\. this
  converter uses the same functions as used by the various hash-based load
  balancing algorithms, so it will provide exactly the same results. it is
  mostly intended for debugging, but can be used as a stick-table entry to
  collect rough statistics. it must not be used for security purposes as a
  32-bit hash is trivial to break. see also "crc32", "sdbm", "wt6" and the
  "hash-type" directive.

even
  returns a boolean true if the input value of type signed integer is even
  otherwise returns false. it is functionally equivalent to "not,and(1),bool".

field(<index>,<delimiters>)
  extracts the substring at the given index considering given delimiters from
  an input string. indexes start at 1 and delimiters are a string formatted
  list of chars.

hex
  converts a binary input sample to an hex string containing two hex digits per
  input byte. it is used to log or transfer hex dumps of some binary input data
  in a way that can be reliably transferred (eg: an ssl id can be copied in a
  header).

http_date([<offset>])
  converts an integer supposed to contain a date since epoch to a string
  representing this date in a format suitable for use in http header fields. if
  an offset value is specified, then it is a number of seconds that is added to
  the date before the conversion is operated. this is particularly useful to
  emit date header fields, expires values in responses when combined with a
  positive offset, or last-modified values when the offset is negative.

in_table(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, a boolean false
  is returned. otherwise a boolean true is returned. this can be used to verify
  the presence of a certain key in a table tracking some elements (eg: whether
  or not a source ip address or an authorization header was already seen).

ipmask(<mask>)
  apply a mask to an ipv4 address, and use the result for lookups and storage.
  this can be used to make all hosts within a certain mask to share the same
  table entries and as such use the same server. the mask can be passed in
  dotted form (eg: 255.255.255.0) or in cidr form (eg: 24).

json([<input-code>])
  escapes the input string and produces an ascii ouput string ready to use as a
  json string. the converter tries to decode the input string according to the
  <input-code> parameter. it can be "ascii", "utf8", "utf8s", "utf8"" or
  "utf8ps". the "ascii" decoder never fails. the "utf8" decoder detects 3 types
  of errors:
   - bad utf-8 sequence (lone continuation byte, bad number of continuation
     bytes, ...)
   - invalid range (the decoded value is within a utf-8 prohibited range),
   - code overlong (the value is encoded with more bytes than necessary).

  the utf-8 json encoding can produce a "too long value" error when the utf-8
  character is greater than 0xffff because the json string escape specification
  only authorizes 4 hex digits for the value encoding. the utf-8 decoder exists
  in 4 variants designated by a combination of two suffix letters : "p" for
  "permissive" and "s" for "silently ignore". the behaviors of the decoders
  are :
   - "ascii"  : never fails ;
   - "utf8"   : fails on any detected errors ;
   - "utf8s"  : never fails, but removes characters corresponding to errors ;
   - "utf8p"  : accepts and fixes the overlong errors, but fails on any other
                error ;
   - "utf8ps" : never fails, accepts and fixes the overlong errors, but removes
                characters corresponding to the other errors.

  this converter is particularly useful for building properly escaped json for
  logging to servers which consume json-formated traffic logs.

  example:
     capture request header user-agent len 150
     capture request header host len 15
     log-format {"ip":"%[src]","user-agent":"%[capture.req.hdr(1),json]"}

  input request from client 127.0.0.1:
     get / http/1.0
     user-agent: very "ugly" ua 1/2

  output log:
     {"ip":"127.0.0.1","user-agent":"very \"ugly\" ua 1\/2"}

language(<value>[,<default>])
  returns the value with the highest q-factor from a list as extracted from the
  "accept-language" header using "req.fhdr". values with no q-factor have a
  q-factor of 1\. values with a q-factor of 0 are dropped. only values which
  belong to the list of semi-colon delimited <values> will be considered. the
  argument <value> syntax is "lang[;lang[;lang[;...]]]". if no value matches the
  given list and a default value is provided, it is returned. note that language
  names may have a variant after a dash ('-'). if this variant is present in the
  list, it will be matched, but if it is not, only the base language is checked.
  the match is case-sensitive, and the output string is always one of those
  provided in arguments.  the ordering of arguments is meaningless, only the
  ordering of the values in the request counts, as the first value among
  multiple sharing the same q-factor is used.

  example :

    # this configuration switches to the backend matching a
    # given language based on the request :

    acl es req.fhdr(accept-language),language(es;fr;en) -m str es
    acl fr req.fhdr(accept-language),language(es;fr;en) -m str fr
    acl en req.fhdr(accept-language),language(es;fr;en) -m str en
    use_backend spanish if es
    use_backend french  if fr
    use_backend english if en
    default_backend choose_your_language

lower
  convert a string sample to lower case. this can only be placed after a string
  sample fetch function or after a transformation keyword returning a string
  type. the result is of type string.

ltime(<format>[,<offset>])
  converts an integer supposed to contain a date since epoch to a string
  representing this date in local time using a format defined by the <format>
  string using strftime(3). the purpose is to allow any date format to be used
  in logs. an optional <offset> in seconds may be applied to the input date
  (positive or negative). see the strftime() man page for the format supported
  by your operating system. see also the utime converter.

  example :

      # emit two colons, one with the local time and another with ip:port
      # eg:  20140710162350 127.0.0.1:57325
      log-format %[date,ltime(%y%m%d%h%m%s)]\ %ci:%cp

map(<map_file>[,<default_value>])
map_<match_type>(<map_file>[,<default_value>])
map_<match_type>_<output_type>(<map_file>[,<default_value>])
  search the input value from <map_file> using the <match_type> matching method,
  and return the associated value converted to the type <output_type>. if the
  input value cannot be found in the <map_file>, the converter returns the
  <default_value>. if the <default_value> is not set, the converter fails and
  acts as if no input value could be fetched. if the <match_type> is not set, it
  defaults to "str". likewise, if the <output_type> is not set, it defaults to
  "str". for convenience, the "map" keyword is an alias for "map_str" and maps a
  string to another string.

  it is important to avoid overlapping between the keys : ip addresses and
  strings are stored in trees, so the first of the finest match will be used.
  other keys are stored in lists, so the first matching occurrence will be used.

  the following array contains the list of all map functions avalaible sorted by
  input type, match type and output type.

  input type | match method | output type str | output type int | output type ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | str          | map_str         | map_str_int     | map_str_ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | beg          | map_beg         | map_beg_int     | map_end_ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | sub          | map_sub         | map_sub_int     | map_sub_ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | dir          | map_dir         | map_dir_int     | map_dir_ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | dom          | map_dom         | map_dom_int     | map_dom_ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | end          | map_end         | map_end_int     | map_end_ip
  -----------+--------------+-----------------+-----------------+---------------
    str      | reg          | map_reg         | map_reg_int     | map_reg_ip
  -----------+--------------+-----------------+-----------------+---------------
    int      | int          | map_int         | map_int_int     | map_int_ip
  -----------+--------------+-----------------+-----------------+---------------
    ip       | ip           | map_ip          | map_ip_int      | map_ip_ip
  -----------+--------------+-----------------+-----------------+---------------

  the file contains one key + value per line. lines which start with '#' are
  ignored, just like empty lines. leading tabs and spaces are stripped. the key
  is then the first "word" (series of non-space/tabs characters), and the value
  is what follows this series of space/tab till the end of the line excluding
  trailing spaces/tabs.

  example :

     # this is a comment and is ignored
        2.22.246.0/23    united kingdom      \n
     <-><-----------><--><------------><---->
      |       |       |         |        `- trailing spaces ignored
      |       |       |         `---------- value
      |       |       `-------------------- middle spaces ignored
      |       `---------------------------- key
      `------------------------------------ leading spaces ignored

mod(<value>)
  divides the input value of type signed integer by <value>, and returns the
  remainder as an signed integer. if <value> is null, then zero is returned.
  <value> can be a numeric value or a variable name. the name of the variable
  starts by an indication about its scope. the allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

mul(<value>)
  multiplies the input value of type signed integer by <value>, and returns
  the product as an signed integer. in case of overflow, the largest possible
  value for the sign is returned so that the operation doesn't wrap around.
  <value> can be a numeric value or a variable name. the name of the variable
  starts by an indication about its scope. the allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

neg
  takes the input value of type signed integer, computes the opposite value,
  and returns the remainder as an signed integer. 0 is identity. this operator
  is provided for reversed subtracts : in order to subtract the input from a
  constant, simply perform a "neg,add(value)".

not
  returns a boolean false if the input value of type signed integer is
  non-null, otherwise returns true. used in conjunction with and(), it can be
  used to report true/false for bit testing on input values (eg: verify the
  absence of a flag).

odd
  returns a boolean true if the input value of type signed integer is odd
  otherwise returns false. it is functionally equivalent to "and(1),bool".

or(<value>)
  performs a bitwise "or" between <value> and the input value of type signed
  integer, and returns the result as an signed integer. <value> can be a
  numeric value or a variable name. the name of the variable starts by an
  indication about its scope. the allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

regsub(<regex>,<subst>[,<flags>])
  applies a regex-based substitution to the input string. it does the same
  operation as the well-known "sed" utility with "s/<regex>/<subst>/". by
  default it will replace in the input string the first occurrence of the
  largest part matching the regular expression <regex> with the substitution
  string <subst>. it is possible to replace all occurrences instead by adding
  the flag "g" in the third argument <flags>. it is also possible to make the
  regex case insensitive by adding the flag "i" in <flags>. since <flags> is a
  string, it is made up from the concatenation of all desired flags. thus if
  both "i" and "g" are desired, using "gi" or "ig" will have the same effect.
  it is important to note that due to the current limitations of the
  configuration parser, some characters such as closing parenthesis or comma
  are not possible to use in the arguments. the first use of this converter is
  to replace certain characters or sequence of characters with other ones.

  example :

     # de-duplicate "/" in header "x-path".
     # input:  x-path: /////a///b/c/xzxyz/
     # output: x-path: /a/b/c/xzxyz/
     http-request set-header x-path %[hdr(x-path),regsub(/+,/,g)]

capture-req(<id>)
  capture the string entry in the request slot <id> and returns the entry as
  is. if the slot doesn't exist, the capture fails silently.

  see also: "declare capture", "http-request capture",
            "http-response capture", "req.hdr.capture" and
            "res.hdr.capture" (sample fetches).

capture-res(<id>)
  capture the string entry in the response slot <id> and returns the entry as
  is. if the slot doesn't exist, the capture fails silently.

  see also: "declare capture", "http-request capture",
            "http-response capture", "req.hdr.capture" and
            "res.hdr.capture" (sample fetches).

sdbm([<avalanche>])
  hashes a binary input sample into an unsigned 32-bit quantity using the sdbm
  hash function. optionally, it is possible to apply a full avalanche hash
  function to the output if the optional <avalanche> argument equals 1\. this
  converter uses the same functions as used by the various hash-based load
  balancing algorithms, so it will provide exactly the same results. it is
  mostly intended for debugging, but can be used as a stick-table entry to
  collect rough statistics. it must not be used for security purposes as a
  32-bit hash is trivial to break. see also "crc32", "djb2", "wt6" and the
  "hash-type" directive.

set-var(<var name>)
  sets a variable with the input content and return the content on the output as
  is. the variable keep the value and the associated input type. the name of the
  variable starts by an indication about it scope. the scope allowed are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

sub(<value>)
  subtracts <value> from the input value of type signed integer, and returns
  the result as an signed integer. note: in order to subtract the input from
  a constant, simply perform a "neg,add(value)". <value> can be a numeric value
  or a variable name. the name of the variable starts by an indication about its
  scope. the allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

table_bytes_in_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the average client-to-server
  bytes rate associated with the input sample in the designated table, measured
  in amount of bytes over the period configured in the table. see also the
  sc_bytes_in_rate sample fetch keyword.

table_bytes_out_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the average server-to-client
  bytes rate associated with the input sample in the designated table, measured
  in amount of bytes over the period configured in the table. see also the
  sc_bytes_out_rate sample fetch keyword.

table_conn_cnt(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the cumulated amount of incoming
  connections associated with the input sample in the designated table. see
  also the sc_conn_cnt sample fetch keyword.

table_conn_cur(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the current amount of concurrent
  tracked connections associated with the input sample in the designated table.
  see also the sc_conn_cur sample fetch keyword.

table_conn_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the average incoming connection
  rate associated with the input sample in the designated table. see also the
  sc_conn_rate sample fetch keyword.

table_gpc0(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the current value of the first
  general purpose counter associated with the input sample in the designated
  table. see also the sc_get_gpc0 sample fetch keyword.

table_gpc0_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the frequency which the gpc0
  counter was incremented over the configured period in the table, associated
  with the input sample in the designated table. see also the sc_get_gpc0_rate
  sample fetch keyword.

table_http_err_cnt(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the cumulated amount of http
  errors associated with the input sample in the designated table. see also the
  sc_http_err_cnt sample fetch keyword.

table_http_err_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the average rate of http errors associated with the
  input sample in the designated table, measured in amount of errors over the
  period configured in the table. see also the sc_http_err_rate sample fetch
  keyword.

table_http_req_cnt(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the cumulated amount of http
  requests associated with the input sample in the designated table. see also
  the sc_http_req_cnt sample fetch keyword.

table_http_req_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the average rate of http requests associated with the
  input sample in the designated table, measured in amount of requests over the
  period configured in the table. see also the sc_http_req_rate sample fetch
  keyword.

table_kbytes_in(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the cumulated amount of client-
  to-server data associated with the input sample in the designated table,
  measured in kilobytes. the test is currently performed on 32-bit integers,
  which limits values to 4 terabytes. see also the sc_kbytes_in sample fetch
  keyword.

table_kbytes_out(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the cumulated amount of server-
  to-client data associated with the input sample in the designated table,
  measured in kilobytes. the test is currently performed on 32-bit integers,
  which limits values to 4 terabytes. see also the sc_kbytes_out sample fetch
  keyword.

table_server_id(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the server id associated with
  the input sample in the designated table. a server id is associated to a
  sample by a "stick" rule when a connection to a server succeeds. a server id
  zero means that no server is associated with this key.

table_sess_cnt(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the cumulated amount of incoming
  sessions associated with the input sample in the designated table. note that
  a session here refers to an incoming connection being accepted by the
  "tcp-request connection" rulesets. see also the sc_sess_cnt sample fetch
  keyword.

table_sess_rate(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the average incoming session
  rate associated with the input sample in the designated table. note that a
  session here refers to an incoming connection being accepted by the
  "tcp-request connection" rulesets. see also the sc_sess_rate sample fetch
  keyword.

table_trackers(<table>)
  uses the string representation of the input sample to perform a look up in
  the specified table. if the key is not found in the table, integer value zero
  is returned. otherwise the converter returns the current amount of concurrent
  connections tracking the same key as the input sample in the designated
  table. it differs from table_conn_cur in that it does not rely on any stored
  information but on the table's reference count (the "use" value which is
  returned by "show table" on the cli). this may sometimes be more suited for
  layer7 tracking. it can be used to tell a server how many concurrent
  connections there are from a given address for example. see also the
  sc_trackers sample fetch keyword.

upper
  convert a string sample to upper case. this can only be placed after a string
  sample fetch function or after a transformation keyword returning a string
  type. the result is of type string.

url_dec
  takes an url-encoded string provided as input and returns the decoded
  version as output. the input and the output are of type string.

utime(<format>[,<offset>])
  converts an integer supposed to contain a date since epoch to a string
  representing this date in utc time using a format defined by the <format>
  string using strftime(3). the purpose is to allow any date format to be used
  in logs. an optional <offset> in seconds may be applied to the input date
  (positive or negative). see the strftime() man page for the format supported
  by your operating system. see also the ltime converter.

  example :

      # emit two colons, one with the utc time and another with ip:port
      # eg:  20140710162350 127.0.0.1:57325
      log-format %[date,utime(%y%m%d%h%m%s)]\ %ci:%cp

word(<index>,<delimiters>)
  extracts the nth word considering given delimiters from an input string.
  indexes start at 1 and delimiters are a string formatted list of chars.

wt6([<avalanche>])
  hashes a binary input sample into an unsigned 32-bit quantity using the wt6
  hash function. optionally, it is possible to apply a full avalanche hash
  function to the output if the optional <avalanche> argument equals 1\. this
  converter uses the same functions as used by the various hash-based load
  balancing algorithms, so it will provide exactly the same results. it is
  mostly intended for debugging, but can be used as a stick-table entry to
  collect rough statistics. it must not be used for security purposes as a
  32-bit hash is trivial to break. see also "crc32", "djb2", "sdbm", and the
  "hash-type" directive.

xor(<value>)
  performs a bitwise "xor" (exclusive or) between <value> and the input value
  of type signed integer, and returns the result as an signed integer.
  <value> can be a numeric value or a variable name. the name of the variable
  starts by an indication about its scope. the allowed scopes are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

7.3.2\. fetching samples from internal states
--------------------------------------------

a first set of sample fetch methods applies to internal information which does
not even relate to any client information. these ones are sometimes used with
"monitor-fail" directives to report an internal status to external watchers.
the sample fetch methods described in this section are usable anywhere.

always_false : boolean
  always returns the boolean "false" value. it may be used with acls as a
  temporary replacement for another one when adjusting configurations.

always_true : boolean
  always returns the boolean "true" value. it may be used with acls as a
  temporary replacement for another one when adjusting configurations.

avg_queue([<backend>]) : integer
  returns the total number of queued connections of the designated backend
  divided by the number of active servers. the current backend is used if no
  backend is specified. this is very similar to "queue" except that the size of
  the farm is considered, in order to give a more accurate measurement of the
  time it may take for a new connection to be processed. the main usage is with
  acl to return a sorry page to new users when it becomes certain they will get
  a degraded service, or to pass to the backend servers in a header so that
  they decide to work in degraded mode or to disable some functions to speed up
  the processing a bit. note that in the event there would not be any active
  server anymore, twice the number of queued connections would be considered as
  the measured value. this is a fair estimate, as we expect one server to get
  back soon anyway, but we still prefer to send new traffic to another backend
  if in better shape. see also the "queue", "be_conn", and "be_sess_rate"
  sample fetches.

be_conn([<backend>]) : integer
  applies to the number of currently established connections on the backend,
  possibly including the connection being evaluated. if no backend name is
  specified, the current one is used. but it is also possible to check another
  backend. it can be used to use a specific farm when the nominal one is full.
  see also the "fe_conn", "queue" and "be_sess_rate" criteria.

be_sess_rate([<backend>]) : integer
  returns an integer value corresponding to the sessions creation rate on the
  backend, in number of new sessions per second. this is used with acls to
  switch to an alternate backend when an expensive or fragile one reaches too
  high a session rate, or to limit abuse of service (eg. prevent sucking of an
  online dictionary). it can also be useful to add this element to logs using a
  log-format directive.

  example :
        # redirect to an error page if the dictionary is requested too often
        backend dynamic
            mode http
            acl being_scanned be_sess_rate gt 100
            redirect location /denied.html if being_scanned

bin(<hexa>) : bin
  returns a binary chain. the input is the hexadecimal representation
  of the string.

bool(<bool>) : bool
  returns a boolean value. <bool> can be 'true', 'false', '1' or '0'.
  'false' and '0' are the same. 'true' and '1' are the same.

connslots([<backend>]) : integer
  returns an integer value corresponding to the number of connection slots
  still available in the backend, by totaling the maximum amount of
  connections on all servers and the maximum queue size. this is probably only
  used with acls.

  the basic idea here is to be able to measure the number of connection "slots"
  still available (connection + queue), so that anything beyond that (intended
  usage; see "use_backend" keyword) can be redirected to a different backend.

  'connslots' = number of available server connection slots, + number of
  available server queue slots.

  note that while "fe_conn" may be used, "connslots" comes in especially
  useful when you have a case of traffic going to one single ip, splitting into
  multiple backends (perhaps using acls to do name-based load balancing) and
  you want to be able to differentiate between different backends, and their
  available "connslots".  also, whereas "nbsrv" only measures servers that are
  actually *down*, this fetch is more fine-grained and looks into the number of
  available connection slots as well. see also "queue" and "avg_queue".

  other caveats and notes: at this point in time, the code does not take care
  of dynamic connections. also, if any of the server maxconn, or maxqueue is 0,
  then this fetch clearly does not make sense, in which case the value returned
  will be -1.

date([<offset>]) : integer
  returns the current date as the epoch (number of seconds since 01/01/1970).
  if an offset value is specified, then it is a number of seconds that is added
  to the current date before returning the value. this is particularly useful
  to compute relative dates, as both positive and negative offsets are allowed.
  it is useful combined with the http_date converter.

  example :

     # set an expires header to now+1 hour in every response
     http-response set-header expires %[date(3600),http_date]

env(<name>) : string
  returns a string containing the value of environment variable <name>. as a
  reminder, environment variables are per-process and are sampled when the
  process starts. this can be useful to pass some information to a next hop
  server, or with acls to take specific action when the process is started a
  certain way.

  examples :
      # pass the via header to next hop with the local hostname in it
      http-request add-header via 1.1\ %[env(hostname)]

      # reject cookie-less requests when the stop environment variable is set
      http-request deny if !{ cook(sessionid) -m found } { env(stop) -m found }

fe_conn([<frontend>]) : integer
  returns the number of currently established connections on the frontend,
  possibly including the connection being evaluated. if no frontend name is
  specified, the current one is used. but it is also possible to check another
  frontend. it can be used to return a sorry page before hard-blocking, or to
  use a specific backend to drain new requests when the farm is considered
  full.  this is mostly used with acls but can also be used to pass some
  statistics to servers in http headers. see also the "dst_conn", "be_conn",
  "fe_sess_rate" fetches.

fe_sess_rate([<frontend>]) : integer
  returns an integer value corresponding to the sessions creation rate on the
  frontend, in number of new sessions per second. this is used with acls to
  limit the incoming session rate to an acceptable range in order to prevent
  abuse of service at the earliest moment, for example when combined with other
  layer 4 acls in order to force the clients to wait a bit for the rate to go
  down below the limit. it can also be useful to add this element to logs using
  a log-format directive. see also the "rate-limit sessions" directive for use
  in frontends.

  example :
        # this frontend limits incoming mails to 10/s with a max of 100
        # concurrent connections. we accept any connection below 10/s, and
        # force excess clients to wait for 100 ms. since clients are limited to
        # 100 max, there cannot be more than 10 incoming mails per second.
        frontend mail
            bind :25
            mode tcp
            maxconn 100
            acl too_fast fe_sess_rate ge 10
            tcp-request inspect-delay 100ms
            tcp-request content accept if ! too_fast
            tcp-request content accept if wait_end

int(<integer>) : signed integer
  returns a signed integer.

ipv4(<ipv4>) : ipv4
  returns an ipv4.

ipv6(<ipv6>) : ipv6
  returns an ipv6.

meth(<method>) : method
  returns a method.

nbproc : integer
  returns an integer value corresponding to the number of processes that were
  started (it equals the global "nbproc" setting). this is useful for logging
  and debugging purposes.

nbsrv([<backend>]) : integer
  returns an integer value corresponding to the number of usable servers of
  either the current backend or the named backend. this is mostly used with
  acls but can also be useful when added to logs. this is normally used to
  switch to an alternate backend when the number of servers is too low to
  to handle some load. it is useful to report a failure when combined with
  "monitor fail".

proc : integer
  returns an integer value corresponding to the position of the process calling
  the function, between 1 and global.nbproc. this is useful for logging and
  debugging purposes.

queue([<backend>]) : integer
  returns the total number of queued connections of the designated backend,
  including all the connections in server queues. if no backend name is
  specified, the current one is used, but it is also possible to check another
  one. this is useful with acls or to pass statistics to backend servers. this
  can be used to take actions when queuing goes above a known level, generally
  indicating a surge of traffic or a massive slowdown on the servers. one
  possible action could be to reject new users but still accept old ones. see
  also the "avg_queue", "be_conn", and "be_sess_rate" fetches.

rand([<range>]) : integer
  returns a random integer value within a range of <range> possible values,
  starting at zero. if the range is not specified, it defaults to 2^32, which
  gives numbers between 0 and 4294967295\. it can be useful to pass some values
  needed to take some routing decisions for example, or just for debugging
  purposes. this random must not be used for security purposes.

srv_conn([<backend>/]<server>) : integer
  returns an integer value corresponding to the number of currently established
  connections on the designated server, possibly including the connection being
  evaluated. if <backend> is omitted, then the server is looked up in the
  current backend. it can be used to use a specific farm when one server is
  full, or to inform the server about our view of the number of active
  connections with it. see also the "fe_conn", "be_conn" and "queue" fetch
  methods.

srv_is_up([<backend>/]<server>) : boolean
  returns true when the designated server is up, and false when it is either
  down or in maintenance mode. if <backend> is omitted, then the server is
  looked up in the current backend. it is mainly used to take action based on
  an external status reported via a health check (eg: a geographical site's
  availability). another possible use which is more of a hack consists in
  using dummy servers as boolean variables that can be enabled or disabled from
  the cli, so that rules depending on those acls can be tweaked in realtime.

srv_sess_rate([<backend>/]<server>) : integer
  returns an integer corresponding to the sessions creation rate on the
  designated server, in number of new sessions per second. if <backend> is
  omitted, then the server is looked up in the current backend. this is mostly
  used with acls but can make sense with logs too. this is used to switch to an
  alternate backend when an expensive or fragile one reaches too high a session
  rate, or to limit abuse of service (eg. prevent latent requests from
  overloading servers).

  example :
        # redirect to a separate back
        acl srv1_full srv_sess_rate(be1/srv1) gt 50
        acl srv2_full srv_sess_rate(be1/srv2) gt 50
        use_backend be2 if srv1_full or srv2_full

stopping : boolean
  returns true if the process calling the function is currently stopping. this
  can be useful for logging, or for relaxing certain checks or helping close
  certain connections upon graceful shutdown.

str(<string>) : string
  returns a string.

table_avl([<table>]) : integer
  returns the total number of available entries in the current proxy's
  stick-table or in the designated stick-table. see also table_cnt.

table_cnt([<table>]) : integer
  returns the total number of entries currently in use in the current proxy's
  stick-table or in the designated stick-table. see also src_conn_cnt and
  table_avl for other entry counting methods.

var(<var-name>) : undefined
  returns a variable with the stored type. if the variable is not set, the
  sample fetch fails. the name of the variable starts by an indication about its
  scope. the scope allowed are:
    "sess" : the variable is shared with all the session,
    "txn"  : the variable is shared with all the transaction (request and
             response),
    "req"  : the variable is shared only during the request processing,
    "res"  : the variable is shared only during the response processing.
  this prefix is followed by a name. the separator is a '.'. the name may only
  contain characters 'a-z', 'a-z', '0-9' and '_'.

7.3.3\. fetching samples at layer 4
----------------------------------

the layer 4 usually describes just the transport layer which in haproxy is
closest to the connection, where no content is yet made available. the fetch
methods described here are usable as low as the "tcp-request connection" rule
sets unless they require some future information. those generally include
tcp/ip addresses and ports, as well as elements from stick-tables related to
the incoming connection. for retrieving a value from a sticky counters, the
counter number can be explicitly set as 0, 1, or 2 using the pre-defined
"sc0_", "sc1_", or "sc2_" prefix, or it can be specified as the first integer
argument when using the "sc_" prefix. an optional table may be specified with
the "sc*" form, in which case the currently tracked key will be looked up into
this alternate table instead of the table currently being tracked.

be_id : integer
  returns an integer containing the current backend's id. it can be used in
  frontends with responses to check which backend processed the request.

dst : ip
  this is the destination ipv4 address of the connection on the client side,
  which is the address the client connected to. it can be useful when running
  in transparent mode. it is of type ip and works on both ipv4 and ipv6 tables.
  on ipv6 tables, ipv4 address is mapped to its ipv6 equivalent, according to
  rfc 4291.

dst_conn : integer
  returns an integer value corresponding to the number of currently established
  connections on the same socket including the one being evaluated. it is
  normally used with acls but can as well be used to pass the information to
  servers in an http header or in logs. it can be used to either return a sorry
  page before hard-blocking, or to use a specific backend to drain new requests
  when the socket is considered saturated. this offers the ability to assign
  different limits to different listening ports or addresses. see also the
  "fe_conn" and "be_conn" fetches.

dst_port : integer
  returns an integer value corresponding to the destination tcp port of the
  connection on the client side, which is the port the client connected to.
  this might be used when running in transparent mode, when assigning dynamic
  ports to some clients for a whole application session, to stick all users to
  a same server, or to pass the destination port information to a server using
  an http header.

fe_id : integer
  returns an integer containing the current frontend's id. it can be used in
  backends to check from which backend it was called, or to stick all users
  coming via a same frontend to the same server.

sc_bytes_in_rate(<ctr>[,<table>]) : integer
sc0_bytes_in_rate([<table>]) : integer
sc1_bytes_in_rate([<table>]) : integer
sc2_bytes_in_rate([<table>]) : integer
  returns the average client-to-server bytes rate from the currently tracked
  counters, measured in amount of bytes over the period configured in the
  table. see also src_bytes_in_rate.

sc_bytes_out_rate(<ctr>[,<table>]) : integer
sc0_bytes_out_rate([<table>]) : integer
sc1_bytes_out_rate([<table>]) : integer
sc2_bytes_out_rate([<table>]) : integer
  returns the average server-to-client bytes rate from the currently tracked
  counters, measured in amount of bytes over the period configured in the
  table. see also src_bytes_out_rate.

sc_clr_gpc0(<ctr>[,<table>]) : integer
sc0_clr_gpc0([<table>]) : integer
sc1_clr_gpc0([<table>]) : integer
sc2_clr_gpc0([<table>]) : integer
  clears the first general purpose counter associated to the currently tracked
  counters, and returns its previous value. before the first invocation, the
  stored value is zero, so first invocation will always return zero. this is
  typically used as a second acl in an expression in order to mark a connection
  when a first acl was verified :

        # block if 5 consecutive requests continue to come faster than 10 sess
        # per second, and reset the counter as soon as the traffic slows down.
        acl abuse sc0_http_req_rate gt 10
        acl kill  sc0_inc_gpc0 gt 5
        acl save  sc0_clr_gpc0 ge 0
        tcp-request connection accept if !abuse save
        tcp-request connection reject if abuse kill

sc_conn_cnt(<ctr>[,<table>]) : integer
sc0_conn_cnt([<table>]) : integer
sc1_conn_cnt([<table>]) : integer
sc2_conn_cnt([<table>]) : integer
  returns the cumulated number of incoming connections from currently tracked
  counters. see also src_conn_cnt.

sc_conn_cur(<ctr>[,<table>]) : integer
sc0_conn_cur([<table>]) : integer
sc1_conn_cur([<table>]) : integer
sc2_conn_cur([<table>]) : integer
  returns the current amount of concurrent connections tracking the same
  tracked counters. this number is automatically incremented when tracking
  begins and decremented when tracking stops. see also src_conn_cur.

sc_conn_rate(<ctr>[,<table>]) : integer
sc0_conn_rate([<table>]) : integer
sc1_conn_rate([<table>]) : integer
sc2_conn_rate([<table>]) : integer
  returns the average connection rate from the currently tracked counters,
  measured in amount of connections over the period configured in the table.
  see also src_conn_rate.

sc_get_gpc0(<ctr>[,<table>]) : integer
sc0_get_gpc0([<table>]) : integer
sc1_get_gpc0([<table>]) : integer
sc2_get_gpc0([<table>]) : integer
  returns the value of the first general purpose counter associated to the
  currently tracked counters. see also src_get_gpc0 and sc/sc0/sc1/sc2_inc_gpc0.

sc_gpc0_rate(<ctr>[,<table>]) : integer
sc0_gpc0_rate([<table>]) : integer
sc1_gpc0_rate([<table>]) : integer
sc2_gpc0_rate([<table>]) : integer
  returns the average increment rate of the first general purpose counter
  associated to the currently tracked counters. it reports the frequency
  which the gpc0 counter was incremented over the configured period. see also
  src_gpc0_rate, sc/sc0/sc1/sc2_get_gpc0, and sc/sc0/sc1/sc2_inc_gpc0\. note
  that the "gpc0_rate" counter must be stored in the stick-table for a value to
  be returned, as "gpc0" only holds the event count.

sc_http_err_cnt(<ctr>[,<table>]) : integer
sc0_http_err_cnt([<table>]) : integer
sc1_http_err_cnt([<table>]) : integer
sc2_http_err_cnt([<table>]) : integer
  returns the cumulated number of http errors from the currently tracked
  counters. this includes the both request errors and 4xx error responses.
  see also src_http_err_cnt.

sc_http_err_rate(<ctr>[,<table>]) : integer
sc0_http_err_rate([<table>]) : integer
sc1_http_err_rate([<table>]) : integer
sc2_http_err_rate([<table>]) : integer
  returns the average rate of http errors from the currently tracked counters,
  measured in amount of errors over the period configured in the table. this
  includes the both request errors and 4xx error responses. see also
  src_http_err_rate.

sc_http_req_cnt(<ctr>[,<table>]) : integer
sc0_http_req_cnt([<table>]) : integer
sc1_http_req_cnt([<table>]) : integer
sc2_http_req_cnt([<table>]) : integer
  returns the cumulated number of http requests from the currently tracked
  counters. this includes every started request, valid or not. see also
  src_http_req_cnt.

sc_http_req_rate(<ctr>[,<table>]) : integer
sc0_http_req_rate([<table>]) : integer
sc1_http_req_rate([<table>]) : integer
sc2_http_req_rate([<table>]) : integer
  returns the average rate of http requests from the currently tracked
  counters, measured in amount of requests over the period configured in
  the table. this includes every started request, valid or not. see also
  src_http_req_rate.

sc_inc_gpc0(<ctr>[,<table>]) : integer
sc0_inc_gpc0([<table>]) : integer
sc1_inc_gpc0([<table>]) : integer
sc2_inc_gpc0([<table>]) : integer
  increments the first general purpose counter associated to the currently
  tracked counters, and returns its new value. before the first invocation,
  the stored value is zero, so first invocation will increase it to 1 and will
  return 1\. this is typically used as a second acl in an expression in order
  to mark a connection when a first acl was verified :

        acl abuse sc0_http_req_rate gt 10
        acl kill  sc0_inc_gpc0 gt 0
        tcp-request connection reject if abuse kill

sc_kbytes_in(<ctr>[,<table>]) : integer
sc0_kbytes_in([<table>]) : integer
sc1_kbytes_in([<table>]) : integer
sc2_kbytes_in([<table>]) : integer
  returns the total amount of client-to-server data from the currently tracked
  counters, measured in kilobytes. the test is currently performed on 32-bit
  integers, which limits values to 4 terabytes. see also src_kbytes_in.

sc_kbytes_out(<ctr>[,<table>]) : integer
sc0_kbytes_out([<table>]) : integer
sc1_kbytes_out([<table>]) : integer
sc2_kbytes_out([<table>]) : integer
  returns the total amount of server-to-client data from the currently tracked
  counters, measured in kilobytes. the test is currently performed on 32-bit
  integers, which limits values to 4 terabytes. see also src_kbytes_out.

sc_sess_cnt(<ctr>[,<table>]) : integer
sc0_sess_cnt([<table>]) : integer
sc1_sess_cnt([<table>]) : integer
sc2_sess_cnt([<table>]) : integer
  returns the cumulated number of incoming connections that were transformed
  into sessions, which means that they were accepted by a "tcp-request
  connection" rule, from the currently tracked counters. a backend may count
  more sessions than connections because each connection could result in many
  backend sessions if some http keep-alive is performed over the connection
  with the client. see also src_sess_cnt.

sc_sess_rate(<ctr>[,<table>]) : integer
sc0_sess_rate([<table>]) : integer
sc1_sess_rate([<table>]) : integer
sc2_sess_rate([<table>]) : integer
  returns the average session rate from the currently tracked counters,
  measured in amount of sessions over the period configured in the table. a
  session is a connection that got past the early "tcp-request connection"
  rules. a backend may count more sessions than connections because each
  connection could result in many backend sessions if some http keep-alive is
  performed over the connection with the client. see also src_sess_rate.

sc_tracked(<ctr>[,<table>]) : boolean
sc0_tracked([<table>]) : boolean
sc1_tracked([<table>]) : boolean
sc2_tracked([<table>]) : boolean
  returns true if the designated session counter is currently being tracked by
  the current session. this can be useful when deciding whether or not we want
  to set some values in a header passed to the server.

sc_trackers(<ctr>[,<table>]) : integer
sc0_trackers([<table>]) : integer
sc1_trackers([<table>]) : integer
sc2_trackers([<table>]) : integer
  returns the current amount of concurrent connections tracking the same
  tracked counters. this number is automatically incremented when tracking
  begins and decremented when tracking stops. it differs from sc0_conn_cur in
  that it does not rely on any stored information but on the table's reference
  count (the "use" value which is returned by "show table" on the cli). this
  may sometimes be more suited for layer7 tracking. it can be used to tell a
  server how many concurrent connections there are from a given address for
  example.

so_id : integer
  returns an integer containing the current listening socket's id. it is useful
  in frontends involving many "bind" lines, or to stick all users coming via a
  same socket to the same server.

src : ip
  this is the source ipv4 address of the client of the session.  it is of type
  ip and works on both ipv4 and ipv6 tables. on ipv6 tables, ipv4 addresses are
  mapped to their ipv6 equivalent, according to rfc 4291\. note that it is the
  tcp-level source address which is used, and not the address of a client
  behind a proxy. however if the "accept-proxy" bind directive is used, it can
  be the address of a client behind another proxy-protocol compatible component
  for all rule sets except "tcp-request connection" which sees the real address.

  example:
       # add an http header in requests with the originating address' country
       http-request set-header x-country %[src,map_ip(geoip.lst)]

src_bytes_in_rate([<table>]) : integer
  returns the average bytes rate from the incoming connection's source address
  in the current proxy's stick-table or in the designated stick-table, measured
  in amount of bytes over the period configured in the table. if the address is
  not found, zero is returned. see also sc/sc0/sc1/sc2_bytes_in_rate.

src_bytes_out_rate([<table>]) : integer
  returns the average bytes rate to the incoming connection's source address in
  the current proxy's stick-table or in the designated stick-table, measured in
  amount of bytes over the period configured in the table. if the address is
  not found, zero is returned. see also sc/sc0/sc1/sc2_bytes_out_rate.

src_clr_gpc0([<table>]) : integer
  clears the first general purpose counter associated to the incoming
  connection's source address in the current proxy's stick-table or in the
  designated stick-table, and returns its previous value. if the address is not
  found, an entry is created and 0 is returned. this is typically used as a
  second acl in an expression in order to mark a connection when a first acl
  was verified :

        # block if 5 consecutive requests continue to come faster than 10 sess
        # per second, and reset the counter as soon as the traffic slows down.
        acl abuse src_http_req_rate gt 10
        acl kill  src_inc_gpc0 gt 5
        acl save  src_clr_gpc0 ge 0
        tcp-request connection accept if !abuse save
        tcp-request connection reject if abuse kill

src_conn_cnt([<table>]) : integer
  returns the cumulated number of connections initiated from the current
  incoming connection's source address in the current proxy's stick-table or in
  the designated stick-table. if the address is not found, zero is returned.
  see also sc/sc0/sc1/sc2_conn_cnt.

src_conn_cur([<table>]) : integer
  returns the current amount of concurrent connections initiated from the
  current incoming connection's source address in the current proxy's
  stick-table or in the designated stick-table. if the address is not found,
  zero is returned. see also sc/sc0/sc1/sc2_conn_cur.

src_conn_rate([<table>]) : integer
  returns the average connection rate from the incoming connection's source
  address in the current proxy's stick-table or in the designated stick-table,
  measured in amount of connections over the period configured in the table. if
  the address is not found, zero is returned. see also sc/sc0/sc1/sc2_conn_rate.

src_get_gpc0([<table>]) : integer
  returns the value of the first general purpose counter associated to the
  incoming connection's source address in the current proxy's stick-table or in
  the designated stick-table. if the address is not found, zero is returned.
  see also sc/sc0/sc1/sc2_get_gpc0 and src_inc_gpc0.

src_gpc0_rate([<table>]) : integer
  returns the average increment rate of the first general purpose counter
  associated to the incoming connection's source address in the current proxy's
  stick-table or in the designated stick-table. it reports the frequency
  which the gpc0 counter was incremented over the configured period. see also
  sc/sc0/sc1/sc2_gpc0_rate, src_get_gpc0, and sc/sc0/sc1/sc2_inc_gpc0\. note
  that the "gpc0_rate" counter must be stored in the stick-table for a value to
  be returned, as "gpc0" only holds the event count.

src_http_err_cnt([<table>]) : integer
  returns the cumulated number of http errors from the incoming connection's
  source address in the current proxy's stick-table or in the designated
  stick-table. this includes the both request errors and 4xx error responses.
  see also sc/sc0/sc1/sc2_http_err_cnt. if the address is not found, zero is
  returned.

src_http_err_rate([<table>]) : integer
  returns the average rate of http errors from the incoming connection's source
  address in the current proxy's stick-table or in the designated stick-table,
  measured in amount of errors over the period configured in the table. this
  includes the both request errors and 4xx error responses. if the address is
  not found, zero is returned. see also sc/sc0/sc1/sc2_http_err_rate.

src_http_req_cnt([<table>]) : integer
  returns the cumulated number of http requests from the incoming connection's
  source address in the current proxy's stick-table or in the designated stick-
  table. this includes every started request, valid or not. if the address is
  not found, zero is returned. see also sc/sc0/sc1/sc2_http_req_cnt.

src_http_req_rate([<table>]) : integer
  returns the average rate of http requests from the incoming connection's
  source address in the current proxy's stick-table or in the designated stick-
  table, measured in amount of requests over the period configured in the
  table. this includes every started request, valid or not. if the address is
  not found, zero is returned. see also sc/sc0/sc1/sc2_http_req_rate.

src_inc_gpc0([<table>]) : integer
  increments the first general purpose counter associated to the incoming
  connection's source address in the current proxy's stick-table or in the
  designated stick-table, and returns its new value. if the address is not
  found, an entry is created and 1 is returned. see also sc0/sc2/sc2_inc_gpc0.
  this is typically used as a second acl in an expression in order to mark a
  connection when a first acl was verified :

        acl abuse src_http_req_rate gt 10
        acl kill  src_inc_gpc0 gt 0
        tcp-request connection reject if abuse kill

src_kbytes_in([<table>]) : integer
  returns the total amount of data received from the incoming connection's
  source address in the current proxy's stick-table or in the designated
  stick-table, measured in kilobytes. if the address is not found, zero is
  returned. the test is currently performed on 32-bit integers, which limits
  values to 4 terabytes. see also sc/sc0/sc1/sc2_kbytes_in.

src_kbytes_out([<table>]) : integer
  returns the total amount of data sent to the incoming connection's source
  address in the current proxy's stick-table or in the designated stick-table,
  measured in kilobytes. if the address is not found, zero is returned. the
  test is currently performed on 32-bit integers, which limits values to 4
  terabytes. see also sc/sc0/sc1/sc2_kbytes_out.

src_port : integer
  returns an integer value corresponding to the tcp source port of the
  connection on the client side, which is the port the client connected from.
  usage of this function is very limited as modern protocols do not care much
  about source ports nowadays.

src_sess_cnt([<table>]) : integer
  returns the cumulated number of connections initiated from the incoming
  connection's source ipv4 address in the current proxy's stick-table or in the
  designated stick-table, that were transformed into sessions, which means that
  they were accepted by "tcp-request" rules. if the address is not found, zero
  is returned. see also sc/sc0/sc1/sc2_sess_cnt.

src_sess_rate([<table>]) : integer
  returns the average session rate from the incoming connection's source
  address in the current proxy's stick-table or in the designated stick-table,
  measured in amount of sessions over the period configured in the table. a
  session is a connection that went past the early "tcp-request" rules. if the
  address is not found, zero is returned. see also sc/sc0/sc1/sc2_sess_rate.

src_updt_conn_cnt([<table>]) : integer
  creates or updates the entry associated to the incoming connection's source
  address in the current proxy's stick-table or in the designated stick-table.
  this table must be configured to store the "conn_cnt" data type, otherwise
  the match will be ignored. the current count is incremented by one, and the
  expiration timer refreshed. the updated count is returned, so this match
  can't return zero. this was used to reject service abusers based on their
  source address. note: it is recommended to use the more complete "track-sc*"
  actions in "tcp-request" rules instead.

  example :
        # this frontend limits incoming ssh connections to 3 per 10 second for
        # each source address, and rejects excess connections until a 10 second
        # silence is observed. at most 20 addresses are tracked.
        listen ssh
            bind :22
            mode tcp
            maxconn 100
            stick-table type ip size 20 expire 10s store conn_cnt
            tcp-request content reject if { src_updt_conn_cnt gt 3 }
            server local 127.0.0.1:22

srv_id : integer
  returns an integer containing the server's id when processing the response.
  while it's almost only used with acls, it may be used for logging or
  debugging.

7.3.4\. fetching samples at layer 5
----------------------------------

the layer 5 usually describes just the session layer which in haproxy is
closest to the session once all the connection handshakes are finished, but
when no content is yet made available. the fetch methods described here are
usable as low as the "tcp-request content" rule sets unless they require some
future information. those generally include the results of ssl negotiations.

ssl_bc : boolean
  returns true when the back connection was made via an ssl/tls transport
  layer and is locally deciphered. this means the outgoing connection was made
  other a server with the "ssl" option.

ssl_bc_alg_keysize : integer
  returns the symmetric cipher key size supported in bits when the outgoing
  connection was made over an ssl/tls transport layer.

ssl_bc_cipher : string
  returns the name of the used cipher when the outgoing connection was made
  over an ssl/tls transport layer.

ssl_bc_protocol : string
  returns the name of the used protocol when the outgoing connection was made
  over an ssl/tls transport layer.

ssl_bc_unique_id : binary
  when the outgoing connection was made over an ssl/tls transport layer,
  returns the tls unique id as defined in rfc5929 section 3\. the unique id
  can be encoded to base64 using the converter: "ssl_bc_unique_id,base64".

ssl_bc_session_id : binary
  returns the ssl id of the back connection when the outgoing connection was
  made over an ssl/tls transport layer. it is useful to log if we want to know
  if session was reused or not.

ssl_bc_use_keysize : integer
  returns the symmetric cipher key size used in bits when the outgoing
  connection was made over an ssl/tls transport layer.

ssl_c_ca_err : integer
  when the incoming connection was made over an ssl/tls transport layer,
  returns the id of the first error detected during verification of the client
  certificate at depth > 0, or 0 if no error was encountered during this
  verification process. please refer to your ssl library's documentation to
  find the exhaustive list of error codes.

ssl_c_ca_err_depth : integer
  when the incoming connection was made over an ssl/tls transport layer,
  returns the depth in the ca chain of the first error detected during the
  verification of the client certificate. if no error is encountered, 0 is
  returned.

ssl_c_der : binary
  returns the der formatted certificate presented by the client when the
  incoming connection was made over an ssl/tls transport layer. when used for
  an acl, the value(s) to match against can be passed in hexadecimal form.

ssl_c_err : integer
  when the incoming connection was made over an ssl/tls transport layer,
  returns the id of the first error detected during verification at depth 0, or
  0 if no error was encountered during this verification process. please refer
  to your ssl library's documentation to find the exhaustive list of error
  codes.

ssl_c_i_dn([<entry>[,<occ>]]) : string
  when the incoming connection was made over an ssl/tls transport layer,
  returns the full distinguished name of the issuer of the certificate
  presented by the client when no <entry> is specified, or the value of the
  first given entry found from the beginning of the dn. if a positive/negative
  occurrence number is specified as the optional second argument, it returns
  the value of the nth given entry value from the beginning/end of the dn.
  for instance, "ssl_c_i_dn(ou,2)" the second organization unit, and
  "ssl_c_i_dn(cn)" retrieves the common name.

ssl_c_key_alg : string
  returns the name of the algorithm used to generate the key of the certificate
  presented by the client when the incoming connection was made over an ssl/tls
  transport layer.

ssl_c_notafter : string
  returns the end date presented by the client as a formatted string
  yymmddhhmmss[z] when the incoming connection was made over an ssl/tls
  transport layer.

ssl_c_notbefore : string
  returns the start date presented by the client as a formatted string
  yymmddhhmmss[z] when the incoming connection was made over an ssl/tls
  transport layer.

ssl_c_s_dn([<entry>[,<occ>]]) : string
  when the incoming connection was made over an ssl/tls transport layer,
  returns the full distinguished name of the subject of the certificate
  presented by the client when no <entry> is specified, or the value of the
  first given entry found from the beginning of the dn. if a positive/negative
  occurrence number is specified as the optional second argument, it returns
  the value of the nth given entry value from the beginning/end of the dn.
  for instance, "ssl_c_s_dn(ou,2)" the second organization unit, and
  "ssl_c_s_dn(cn)" retrieves the common name.

ssl_c_serial : binary
  returns the serial of the certificate presented by the client when the
  incoming connection was made over an ssl/tls transport layer. when used for
  an acl, the value(s) to match against can be passed in hexadecimal form.

ssl_c_sha1 : binary
  returns the sha-1 fingerprint of the certificate presented by the client when
  the incoming connection was made over an ssl/tls transport layer. this can be
  used to stick a client to a server, or to pass this information to a server.
  note that the output is binary, so if you want to pass that signature to the
  server, you need to encode it in hex or base64, such as in the example below:

     http-request set-header x-ssl-client-sha1 %[ssl_c_sha1,hex]

ssl_c_sig_alg : string
  returns the name of the algorithm used to sign the certificate presented by
  the client when the incoming connection was made over an ssl/tls transport
  layer.

ssl_c_used : boolean
  returns true if current ssl session uses a client certificate even if current
  connection uses ssl session resumption. see also "ssl_fc_has_crt".

ssl_c_verify : integer
  returns the verify result error id when the incoming connection was made over
  an ssl/tls transport layer, otherwise zero if no error is encountered. please
  refer to your ssl library's documentation for an exhaustive list of error
  codes.

ssl_c_version : integer
  returns the version of the certificate presented by the client when the
  incoming connection was made over an ssl/tls transport layer.

ssl_f_der : binary
  returns the der formatted certificate presented by the frontend when the
  incoming connection was made over an ssl/tls transport layer. when used for
  an acl, the value(s) to match against can be passed in hexadecimal form.

ssl_f_i_dn([<entry>[,<occ>]]) : string
  when the incoming connection was made over an ssl/tls transport layer,
  returns the full distinguished name of the issuer of the certificate
  presented by the frontend when no <entry> is specified, or the value of the
  first given entry found from the beginning of the dn. if a positive/negative
  occurrence number is specified as the optional second argument, it returns
  the value of the nth given entry value from the beginning/end of the dn.
  for instance, "ssl_f_i_dn(ou,2)" the second organization unit, and
  "ssl_f_i_dn(cn)" retrieves the common name.

ssl_f_key_alg : string
  returns the name of the algorithm used to generate the key of the certificate
  presented by the frontend when the incoming connection was made over an
  ssl/tls transport layer.

ssl_f_notafter : string
  returns the end date presented by the frontend as a formatted string
  yymmddhhmmss[z] when the incoming connection was made over an ssl/tls
  transport layer.

ssl_f_notbefore : string
  returns the start date presented by the frontend as a formatted string
  yymmddhhmmss[z] when the incoming connection was made over an ssl/tls
  transport layer.

ssl_f_s_dn([<entry>[,<occ>]]) : string
  when the incoming connection was made over an ssl/tls transport layer,
  returns the full distinguished name of the subject of the certificate
  presented by the frontend when no <entry> is specified, or the value of the
  first given entry found from the beginning of the dn. if a positive/negative
  occurrence number is specified as the optional second argument, it returns
  the value of the nth given entry value from the beginning/end of the dn.
  for instance, "ssl_f_s_dn(ou,2)" the second organization unit, and
  "ssl_f_s_dn(cn)" retrieves the common name.

ssl_f_serial : binary
  returns the serial of the certificate presented by the frontend when the
  incoming connection was made over an ssl/tls transport layer. when used for
  an acl, the value(s) to match against can be passed in hexadecimal form.

ssl_f_sha1 : binary
  returns the sha-1 fingerprint of the certificate presented by the frontend
  when the incoming connection was made over an ssl/tls transport layer. this
  can be used to know which certificate was chosen using sni.

ssl_f_sig_alg : string
  returns the name of the algorithm used to sign the certificate presented by
  the frontend when the incoming connection was made over an ssl/tls transport
  layer.

ssl_f_version : integer
  returns the version of the certificate presented by the frontend when the
  incoming connection was made over an ssl/tls transport layer.

ssl_fc : boolean
  returns true when the front connection was made via an ssl/tls transport
  layer and is locally deciphered. this means it has matched a socket declared
  with a "bind" line having the "ssl" option.

  example :
        # this passes "x-proto: https" to servers when client connects over ssl
        listen http-https
            bind :80
            bind :443 ssl crt /etc/haproxy.pem
            http-request add-header x-proto https if { ssl_fc }

ssl_fc_alg_keysize : integer
  returns the symmetric cipher key size supported in bits when the incoming
  connection was made over an ssl/tls transport layer.

ssl_fc_alpn : string
  this extracts the application layer protocol negotiation field from an
  incoming connection made via a tls transport layer and locally deciphered by
  haproxy. the result is a string containing the protocol name advertised by
  the client. the ssl library must have been built with support for tls
  extensions enabled (check haproxy -vv). note that the tls alpn extension is
  not advertised unless the "alpn" keyword on the "bind" line specifies a
  protocol list. also, nothing forces the client to pick a protocol from this
  list, any other one may be requested. the tls alpn extension is meant to
  replace the tls npn extension. see also "ssl_fc_npn".

ssl_fc_cipher : string
  returns the name of the used cipher when the incoming connection was made
  over an ssl/tls transport layer.

ssl_fc_has_crt : boolean
  returns true if a client certificate is present in an incoming connection over
  ssl/tls transport layer. useful if 'verify' statement is set to 'optional'.
  note: on ssl session resumption with session id or tls ticket, client
  certificate is not present in the current connection but may be retrieved
  from the cache or the ticket. so prefer "ssl_c_used" if you want to check if
  current ssl session uses a client certificate.

ssl_fc_has_sni : boolean
  this checks for the presence of a server name indication tls extension (sni)
  in an incoming connection was made over an ssl/tls transport layer. returns
  true when the incoming connection presents a tls sni field. this requires
  that the ssl library is build with support for tls extensions enabled (check
  haproxy -vv).

ssl_fc_is_resumed: boolean
  returns true if the ssl/tls session has been resumed through the use of
  ssl session cache or tls tickets.

ssl_fc_npn : string
  this extracts the next protocol negotiation field from an incoming connection
  made via a tls transport layer and locally deciphered by haproxy. the result
  is a string containing the protocol name advertised by the client. the ssl
  library must have been built with support for tls extensions enabled (check
  haproxy -vv). note that the tls npn extension is not advertised unless the
  "npn" keyword on the "bind" line specifies a protocol list. also, nothing
  forces the client to pick a protocol from this list, any other one may be
  requested. please note that the tls npn extension was replaced with alpn.

ssl_fc_protocol : string
  returns the name of the used protocol when the incoming connection was made
  over an ssl/tls transport layer.

ssl_fc_unique_id : binary
  when the incoming connection was made over an ssl/tls transport layer,
  returns the tls unique id as defined in rfc5929 section 3\. the unique id
  can be encoded to base64 using the converter: "ssl_bc_unique_id,base64".

ssl_fc_session_id : binary
  returns the ssl id of the front connection when the incoming connection was
  made over an ssl/tls transport layer. it is useful to stick a given client to
  a server. it is important to note that some browsers refresh their session id
  every few minutes.

ssl_fc_sni : string
  this extracts the server name indication tls extension (sni) field from an
  incoming connection made via an ssl/tls transport layer and locally
  deciphered by haproxy. the result (when present) typically is a string
  matching the https host name (253 chars or less). the ssl library must have
  been built with support for tls extensions enabled (check haproxy -vv).

  this fetch is different from "req_ssl_sni" above in that it applies to the
  connection being deciphered by haproxy and not to ssl contents being blindly
  forwarded. see also "ssl_fc_sni_end" and "ssl_fc_sni_reg" below. this
  requires that the ssl library is build with support for tls extensions
  enabled (check haproxy -vv).

  acl derivatives :
    ssl_fc_sni_end : suffix match
    ssl_fc_sni_reg : regex match

ssl_fc_use_keysize : integer
  returns the symmetric cipher key size used in bits when the incoming
  connection was made over an ssl/tls transport layer.

7.3.5\. fetching samples from buffer contents (layer 6)
------------------------------------------------------

fetching samples from buffer contents is a bit different from the previous
sample fetches above because the sampled data are ephemeral. these data can
only be used when they're available and will be lost when they're forwarded.
for this reason, samples fetched from buffer contents during a request cannot
be used in a response for example. even while the data are being fetched, they
can change. sometimes it is necessary to set some delays or combine multiple
sample fetch methods to ensure that the expected data are complete and usable,
for example through tcp request content inspection. please see the "tcp-request
content" keyword for more detailed information on the subject.

payload(<offset>,<length>) : binary (deprecated)
  this is an alias for "req.payload" when used in the context of a request (eg:
  "stick on", "stick match"), and for "res.payload" when used in the context of
  a response such as in "stick store response".

payload_lv(<offset1>,<length>[,<offset2>]) : binary (deprecated)
  this is an alias for "req.payload_lv" when used in the context of a request
  (eg: "stick on", "stick match"), and for "res.payload_lv" when used in the
  context of a response such as in "stick store response".

req.len : integer
req_len : integer (deprecated)
  returns an integer value corresponding to the number of bytes present in the
  request buffer. this is mostly used in acl. it is important to understand
  that this test does not return false as long as the buffer is changing. this
  means that a check with equality to zero will almost always immediately match
  at the beginning of the session, while a test for more data will wait for
  that data to come in and return false only when haproxy is certain that no
  more data will come in. this test was designed to be used with tcp request
  content inspection.

req.payload(<offset>,<length>) : binary
  this extracts a binary block of <length> bytes and starting at byte <offset>
  in the request buffer. as a special case, if the <length> argument is zero,
  the the whole buffer from <offset> to the end is extracted. this can be used
  with acls in order to check for the presence of some content in a buffer at
  any location.

  acl alternatives :
    payload(<offset>,<length>) : hex binary match

req.payload_lv(<offset1>,<length>[,<offset2>]) : binary
  this extracts a binary block whose size is specified at <offset1> for <length>
  bytes, and which starts at <offset2> if specified or just after the length in
  the request buffer. the <offset2> parameter also supports relative offsets if
  prepended with a '+' or '-' sign.

  acl alternatives :
    payload_lv(<offset1>,<length>[,<offset2>]) : hex binary match

  example : please consult the example from the "stick store-response" keyword.

req.proto_http : boolean
req_proto_http : boolean (deprecated)
  returns true when data in the request buffer look like http and correctly
  parses as such. it is the same parser as the common http request parser which
  is used so there should be no surprises. the test does not match until the
  request is complete, failed or timed out. this test may be used to report the
  protocol in tcp logs, but the biggest use is to block tcp request analysis
  until a complete http request is present in the buffer, for example to track
  a header.

  example:
        # track request counts per "base" (concatenation of host+url)
        tcp-request inspect-delay 10s
        tcp-request content reject if !http
        tcp-request content track-sc0 base table req-rate

req.rdp_cookie([<name>]) : string
rdp_cookie([<name>]) : string (deprecated)
  when the request buffer looks like the rdp protocol, extracts the rdp cookie
  <name>, or any cookie if unspecified. the parser only checks for the first
  cookie, as illustrated in the rdp protocol specification. the cookie name is
  case insensitive. generally the "msts" cookie name will be used, as it can
  contain the user name of the client connecting to the server if properly
  configured on the client. the "mstshash" cookie is often used as well for
  session stickiness to servers.

  this differs from "balance rdp-cookie" in that any balancing algorithm may be
  used and thus the distribution of clients to backend servers is not linked to
  a hash of the rdp cookie. it is envisaged that using a balancing algorithm
  such as "balance roundrobin" or "balance leastconn" will lead to a more even
  distribution of clients to backend servers than the hash used by "balance
  rdp-cookie".

  acl derivatives :
    req_rdp_cookie([<name>]) : exact string match

  example :
   listen tse-farm
       bind 0.0.0.0:3389
       # wait up to 5s for an rdp cookie in the request
       tcp-request inspect-delay 5s
       tcp-request content accept if rdp_cookie
       # apply rdp cookie persistence
       persist rdp-cookie
       # persist based on the mstshash cookie
       # this is only useful makes sense if
       # balance rdp-cookie is not used
       stick-table type string size 204800
       stick on req.rdp_cookie(mstshash)
       server srv1 1.1.1.1:3389
       server srv1 1.1.1.2:3389

  see also : "balance rdp-cookie", "persist rdp-cookie", "tcp-request" and the
  "req_rdp_cookie" acl.

req.rdp_cookie_cnt([name]) : integer
rdp_cookie_cnt([name]) : integer (deprecated)
  tries to parse the request buffer as rdp protocol, then returns an integer
  corresponding to the number of rdp cookies found. if an optional cookie name
  is passed, only cookies matching this name are considered. this is mostly
  used in acl.

  acl derivatives :
    req_rdp_cookie_cnt([<name>]) : integer match

req.ssl_ec_ext : boolean
  returns a boolean identifying if client sent the supported elliptic curves
  extension as defined in rfc4492, section 5.1\. within the ssl clienthello
  message. this can be used to present ecc compatible clients with ec certificate
  and to use rsa for all others, on the same ip address. note that this only
  applies to raw contents found in the request buffer and not to contents
  deciphered via an ssl data layer, so this will not work with "bind" lines
  having the "ssl" option.

req.ssl_hello_type : integer
req_ssl_hello_type : integer (deprecated)
  returns an integer value containing the type of the ssl hello message found
  in the request buffer if the buffer contains data that parse as a complete
  ssl (v3 or superior) client hello message. note that this only applies to raw
  contents found in the request buffer and not to contents deciphered via an
  ssl data layer, so this will not work with "bind" lines having the "ssl"
  option. this is mostly used in acl to detect presence of an ssl hello message
  that is supposed to contain an ssl session id usable for stickiness.

req.ssl_sni : string
req_ssl_sni : string (deprecated)
  returns a string containing the value of the server name tls extension sent
  by a client in a tls stream passing through the request buffer if the buffer
  contains data that parse as a complete ssl (v3 or superior) client hello
  message. note that this only applies to raw contents found in the request
  buffer and not to contents deciphered via an ssl data layer, so this will not
  work with "bind" lines having the "ssl" option. sni normally contains the
  name of the host the client tries to connect to (for recent browsers). sni is
  useful for allowing or denying access to certain hosts when ssl/tls is used
  by the client. this test was designed to be used with tcp request content
  inspection. if content switching is needed, it is recommended to first wait
  for a complete client hello (type 1), like in the example below. see also
  "ssl_fc_sni".

  acl derivatives :
    req_ssl_sni : exact string match

  examples :
     # wait for a client hello for at most 5 seconds
     tcp-request inspect-delay 5s
     tcp-request content accept if { req_ssl_hello_type 1 }
     use_backend bk_allow if { req_ssl_sni -f allowed_sites }
     default_backend bk_sorry_page

res.ssl_hello_type : integer
rep_ssl_hello_type : integer (deprecated)
  returns an integer value containing the type of the ssl hello message found
  in the response buffer if the buffer contains data that parses as a complete
  ssl (v3 or superior) hello message. note that this only applies to raw
  contents found in the response buffer and not to contents deciphered via an
  ssl data layer, so this will not work with "server" lines having the "ssl"
  option. this is mostly used in acl to detect presence of an ssl hello message
  that is supposed to contain an ssl session id usable for stickiness.

req.ssl_ver : integer
req_ssl_ver : integer (deprecated)
  returns an integer value containing the version of the ssl/tls protocol of a
  stream present in the request buffer. both sslv2 hello messages and sslv3
  messages are supported. tlsv1 is announced as ssl version 3.1\. the value is
  composed of the major version multiplied by 65536, added to the minor
  version. note that this only applies to raw contents found in the request
  buffer and not to contents deciphered via an ssl data layer, so this will not
  work with "bind" lines having the "ssl" option. the acl version of the test
  matches against a decimal notation in the form major.minor (eg: 3.1). this
  fetch is mostly used in acl.

  acl derivatives :
    req_ssl_ver : decimal match

res.len : integer
  returns an integer value corresponding to the number of bytes present in the
  response buffer. this is mostly used in acl. it is important to understand
  that this test does not return false as long as the buffer is changing. this
  means that a check with equality to zero will almost always immediately match
  at the beginning of the session, while a test for more data will wait for
  that data to come in and return false only when haproxy is certain that no
  more data will come in. this test was designed to be used with tcp response
  content inspection.

res.payload(<offset>,<length>) : binary
  this extracts a binary block of <length> bytes and starting at byte <offset>
  in the response buffer. as a special case, if the <length> argument is zero,
  the the whole buffer from <offset> to the end is extracted. this can be used
  with acls in order to check for the presence of some content in a buffer at
  any location.

res.payload_lv(<offset1>,<length>[,<offset2>]) : binary
  this extracts a binary block whose size is specified at <offset1> for <length>
  bytes, and which starts at <offset2> if specified or just after the length in
  the response buffer. the <offset2> parameter also supports relative offsets
  if prepended with a '+' or '-' sign.

  example : please consult the example from the "stick store-response" keyword.

wait_end : boolean
  this fetch either returns true when the inspection period is over, or does
  not fetch. it is only used in acls, in conjunction with content analysis to
  avoid returning a wrong verdict early.  it may also be used to delay some
  actions, such as a delayed reject for some special addresses. since it either
  stops the rules evaluation or immediately returns true, it is recommended to
  use this acl as the last one in a rule.  please note that the default acl
  "wait_end" is always usable without prior declaration. this test was designed
  to be used with tcp request content inspection.

  examples :
     # delay every incoming request by 2 seconds
     tcp-request inspect-delay 2s
     tcp-request content accept if wait_end

     # don't immediately tell bad guys they are rejected
     tcp-request inspect-delay 10s
     acl goodguys src 10.0.0.0/24
     acl badguys  src 10.0.1.0/24
     tcp-request content accept if goodguys
     tcp-request content reject if badguys wait_end
     tcp-request content reject

7.3.6\. fetching http samples (layer 7)
--------------------------------------

it is possible to fetch samples from http contents, requests and responses.
this application layer is also called layer 7\. it is only possible to fetch the
data in this section when a full http request or response has been parsed from
its respective request or response buffer. this is always the case with all
http specific rules and for sections running with "mode http". when using tcp
content inspection, it may be necessary to support an inspection delay in order
to let the request or response come in first. these fetches may require a bit
more cpu resources than the layer 4 ones, but not much since the request and
response are indexed.

base : string
  this returns the concatenation of the first host header and the path part of
  the request, which starts at the first slash and ends before the question
  mark. it can be useful in virtual hosted environments to detect url abuses as
  well as to improve shared caches efficiency. using this with a limited size
  stick table also allows one to collect statistics about most commonly
  requested objects by host/path. with acls it can allow simple content
  switching rules involving the host and the path at the same time, such as
  "www.example.com/favicon.ico". see also "path" and "uri".

  acl derivatives :
    base     : exact string match
    base_beg : prefix match
    base_dir : subdir match
    base_dom : domain match
    base_end : suffix match
    base_len : length match
    base_reg : regex match
    base_sub : substring match

base32 : integer
  this returns a 32-bit hash of the value returned by the "base" fetch method
  above. this is useful to track per-url activity on high traffic sites without
  having to store all urls. instead a shorter hash is stored, saving a lot of
  memory. the output type is an unsigned integer. the hash function used is
  sdbm with full avalanche on the output. technically, base32 is exactly equal
  to "base,sdbm(1)".

base32+src : binary
  this returns the concatenation of the base32 fetch above and the src fetch
  below. the resulting type is of type binary, with a size of 8 or 20 bytes
  depending on the source address family. this can be used to track per-ip,
  per-url counters.

capture.req.hdr(<idx>) : string
  this extracts the content of the header captured by the "capture request
  header", idx is the position of the capture keyword in the configuration.
  the first entry is an index of 0\. see also: "capture request header".

capture.req.method : string
  this extracts the method of an http request. it can be used in both request
  and response. unlike "method", it can be used in both request and response
  because it's allocated.

capture.req.uri : string
  this extracts the request's uri, which starts at the first slash and ends
  before the first space in the request (without the host part). unlike "path"
  and "url", it can be used in both request and response because it's
  allocated.

capture.req.ver : string
  this extracts the request's http version and returns either "http/1.0" or
  "http/1.1". unlike "req.ver", it can be used in both request, response, and
  logs because it relies on a persistent flag.

capture.res.hdr(<idx>) : string
  this extracts the content of the header captured by the "capture response
  header", idx is the position of the capture keyword in the configuration.
  the first entry is an index of 0.
  see also: "capture response header"

capture.res.ver : string
  this extracts the response's http version and returns either "http/1.0" or
  "http/1.1". unlike "res.ver", it can be used in logs because it relies on a
  persistent flag.

req.body : binary
  this returns the http request's available body as a block of data. it
  requires that the request body has been buffered made available using
  "option http-buffer-request". in case of chunked-encoded body, currently only
  the first chunk is analyzed.

req.body_param([<name>) : string
  this fetch assumes that the body of the post request is url-encoded. the user
  can check if the "content-type" contains the value
  "application/x-www-form-urlencoded". this extracts the first occurrence of the
  parameter <name> in the body, which ends before '&'. the parameter name is
  case-sensitive. if no name is given, any parameter will match, and the first
  one will be returned. the result is a string corresponding to the value of the
  parameter <name> as presented in the request body (no url decoding is
  performed). note that the acl version of this fetch iterates over multiple
  parameters and will iteratively report all parameters values if no name is
  given.

req.body_len : integer
  this returns the length of the http request's available body in bytes. it may
  be lower than the advertised length if the body is larger than the buffer. it
  requires that the request body has been buffered made available using
  "option http-buffer-request".

req.body_size : integer
  this returns the advertised length of the http request's body in bytes. it
  will represent the advertised content-length header, or the size of the first
  chunk in case of chunked encoding. in order to parse the chunks, it requires
  that the request body has been buffered made available using
  "option http-buffer-request".

req.cook([<name>]) : string
cook([<name>]) : string (deprecated)
  this extracts the last occurrence of the cookie name <name> on a "cookie"
  header line from the request, and returns its value as string. if no name is
  specified, the first cookie value is returned. when used with acls, all
  matching cookies are evaluated. spaces around the name and the value are
  ignored as requested by the cookie header specification (rfc6265). the cookie
  name is case-sensitive. empty cookies are valid, so an empty cookie may very
  well return an empty value if it is present. use the "found" match to detect
  presence. use the res.cook() variant for response cookies sent by the server.

  acl derivatives :
    cook([<name>])     : exact string match
    cook_beg([<name>]) : prefix match
    cook_dir([<name>]) : subdir match
    cook_dom([<name>]) : domain match
    cook_end([<name>]) : suffix match
    cook_len([<name>]) : length match
    cook_reg([<name>]) : regex match
    cook_sub([<name>]) : substring match

req.cook_cnt([<name>]) : integer
cook_cnt([<name>]) : integer (deprecated)
  returns an integer value representing the number of occurrences of the cookie
  <name> in the request, or all cookies if <name> is not specified.

req.cook_val([<name>]) : integer
cook_val([<name>]) : integer (deprecated)
  this extracts the last occurrence of the cookie name <name> on a "cookie"
  header line from the request, and converts its value to an integer which is
  returned. if no name is specified, the first cookie value is returned. when
  used in acls, all matching names are iterated over until a value matches.

cookie([<name>]) : string (deprecated)
  this extracts the last occurrence of the cookie name <name> on a "cookie"
  header line from the request, or a "set-cookie" header from the response, and
  returns its value as a string. a typical use is to get multiple clients
  sharing a same profile use the same server. this can be similar to what
  "appsession" does with the "request-learn" statement, but with support for
  multi-peer synchronization and state keeping across restarts. if no name is
  specified, the first cookie value is returned. this fetch should not be used
  anymore and should be replaced by req.cook() or res.cook() instead as it
  ambiguously uses the direction based on the context where it is used.
  see also : "appsession".

hdr([<name>[,<occ>]]) : string
  this is equivalent to req.hdr() when used on requests, and to res.hdr() when
  used on responses. please refer to these respective fetches for more details.
  in case of doubt about the fetch direction, please use the explicit ones.
  note that contrary to the hdr() sample fetch method, the hdr_* acl keywords
  unambiguously apply to the request headers.

req.fhdr(<name>[,<occ>]) : string
  this extracts the last occurrence of header <name> in an http request. when
  used from an acl, all occurrences are iterated over until a match is found.
  optionally, a specific occurrence might be specified as a position number.
  positive values indicate a position from the first occurrence, with 1 being
  the first one. negative values indicate positions relative to the last one,
  with -1 being the last one. it differs from req.hdr() in that any commas
  present in the value are returned and are not used as delimiters. this is
  sometimes useful with headers such as user-agent.

req.fhdr_cnt([<name>]) : integer
  returns an integer value representing the number of occurrences of request
  header field name <name>, or the total number of header fields if <name> is
  not specified. contrary to its req.hdr_cnt() cousin, this function returns
  the number of full line headers and does not stop on commas.

req.hdr([<name>[,<occ>]]) : string
  this extracts the last occurrence of header <name> in an http request. when
  used from an acl, all occurrences are iterated over until a match is found.
  optionally, a specific occurrence might be specified as a position number.
  positive values indicate a position from the first occurrence, with 1 being
  the first one. negative values indicate positions relative to the last one,
  with -1 being the last one. a typical use is with the x-forwarded-for header
  once converted to ip, associated with an ip stick-table. the function
  considers any comma as a delimiter for distinct values. if full-line headers
  are desired instead, use req.fhdr(). please carefully check rfc2616 to know
  how certain headers are supposed to be parsed. also, some of them are case
  insensitive (eg: connection).

  acl derivatives :
    hdr([<name>[,<occ>]])     : exact string match
    hdr_beg([<name>[,<occ>]]) : prefix match
    hdr_dir([<name>[,<occ>]]) : subdir match
    hdr_dom([<name>[,<occ>]]) : domain match
    hdr_end([<name>[,<occ>]]) : suffix match
    hdr_len([<name>[,<occ>]]) : length match
    hdr_reg([<name>[,<occ>]]) : regex match
    hdr_sub([<name>[,<occ>]]) : substring match

req.hdr_cnt([<name>]) : integer
hdr_cnt([<header>]) : integer (deprecated)
  returns an integer value representing the number of occurrences of request
  header field name <name>, or the total number of header field values if
  <name> is not specified. it is important to remember that one header line may
  count as several headers if it has several values. the function considers any
  comma as a delimiter for distinct values. if full-line headers are desired
  instead, req.fhdr_cnt() should be used instead. with acls, it can be used to
  detect presence, absence or abuse of a specific header, as well as to block
  request smuggling attacks by rejecting requests which contain more than one
  of certain headers. see "req.hdr" for more information on header matching.

req.hdr_ip([<name>[,<occ>]]) : ip
hdr_ip([<name>[,<occ>]]) : ip (deprecated)
  this extracts the last occurrence of header <name> in an http request,
  converts it to an ipv4 or ipv6 address and returns this address. when used
  with acls, all occurrences are checked, and if <name> is omitted, every value
  of every header is checked. optionally, a specific occurrence might be
  specified as a position number. positive values indicate a position from the
  first occurrence, with 1 being the first one.  negative values indicate
  positions relative to the last one, with -1 being the last one. a typical use
  is with the x-forwarded-for and x-client-ip headers.

req.hdr_val([<name>[,<occ>]]) : integer
hdr_val([<name>[,<occ>]]) : integer (deprecated)
  this extracts the last occurrence of header <name> in an http request, and
  converts it to an integer value. when used with acls, all occurrences are
  checked, and if <name> is omitted, every value of every header is checked.
  optionally, a specific occurrence might be specified as a position number.
  positive values indicate a position from the first occurrence, with 1 being
  the first one. negative values indicate positions relative to the last one,
  with -1 being the last one. a typical use is with the x-forwarded-for header.

http_auth(<userlist>) : boolean
  returns a boolean indicating whether the authentication data received from
  the client match a username & password stored in the specified userlist. this
  fetch function is not really useful outside of acls. currently only http
  basic auth is supported.

http_auth_group(<userlist>) : string
  returns a string corresponding to the user name found in the authentication
  data received from the client if both the user name and password are valid
  according to the specified userlist. the main purpose is to use it in acls
  where it is then checked whether the user belongs to any group within a list.
  this fetch function is not really useful outside of acls. currently only http
  basic auth is supported.

  acl derivatives :
    http_auth_group(<userlist>) : group ...
    returns true when the user extracted from the request and whose password is
    valid according to the specified userlist belongs to at least one of the
    groups.

http_first_req : boolean
  returns true when the request being processed is the first one of the
  connection. this can be used to add or remove headers that may be missing
  from some requests when a request is not the first one, or to help grouping
  requests in the logs.

method : integer + string
  returns an integer value corresponding to the method in the http request. for
  example, "get" equals 1 (check sources to establish the matching). value 9
  means "other method" and may be converted to a string extracted from the
  stream. this should not be used directly as a sample, this is only meant to
  be used from acls, which transparently convert methods from patterns to these
  integer + string values. some predefined acl already check for most common
  methods.

  acl derivatives :
    method : case insensitive method match

  example :
      # only accept get and head requests
      acl valid_method method get head
      http-request deny if ! valid_method

path : string
  this extracts the request's url path, which starts at the first slash and
  ends before the question mark (without the host part). a typical use is with
  prefetch-capable caches, and with portals which need to aggregate multiple
  information from databases and keep them in caches. note that with outgoing
  caches, it would be wiser to use "url" instead. with acls, it's typically
  used to match exact file names (eg: "/login.php"), or directory parts using
  the derivative forms. see also the "url" and "base" fetch methods.

  acl derivatives :
    path     : exact string match
    path_beg : prefix match
    path_dir : subdir match
    path_dom : domain match
    path_end : suffix match
    path_len : length match
    path_reg : regex match
    path_sub : substring match

query : string
  this extracts the request's query string, which starts after the first
  question mark. if no question mark is present, this fetch returns nothing. if
  a question mark is present but nothing follows, it returns an empty string.
  this means it's possible to easily know whether a query string is present
  using the "found" matching method. this fetch is the completemnt of "path"
  which stops before the question mark.

req.hdr_names([<delim>]) : string
  this builds a string made from the concatenation of all header names as they
  appear in the request when the rule is evaluated. the default delimiter is
  the comma (',') but it may be overridden as an optional argument <delim>. in
  this case, only the first character of <delim> is considered.

req.ver : string
req_ver : string (deprecated)
  returns the version string from the http request, for example "1.1". this can
  be useful for logs, but is mostly there for acl. some predefined acl already
  check for versions 1.0 and 1.1.

  acl derivatives :
    req_ver : exact string match

res.comp : boolean
  returns the boolean "true" value if the response has been compressed by
  haproxy, otherwise returns boolean "false". this may be used to add
  information in the logs.

res.comp_algo : string
  returns a string containing the name of the algorithm used if the response
  was compressed by haproxy, for example : "deflate". this may be used to add
  some information in the logs.

res.cook([<name>]) : string
scook([<name>]) : string (deprecated)
  this extracts the last occurrence of the cookie name <name> on a "set-cookie"
  header line from the response, and returns its value as string. if no name is
  specified, the first cookie value is returned.

  acl derivatives :
    scook([<name>] : exact string match

res.cook_cnt([<name>]) : integer
scook_cnt([<name>]) : integer (deprecated)
  returns an integer value representing the number of occurrences of the cookie
  <name> in the response, or all cookies if <name> is not specified. this is
  mostly useful when combined with acls to detect suspicious responses.

res.cook_val([<name>]) : integer
scook_val([<name>]) : integer (deprecated)
  this extracts the last occurrence of the cookie name <name> on a "set-cookie"
  header line from the response, and converts its value to an integer which is
  returned. if no name is specified, the first cookie value is returned.

res.fhdr([<name>[,<occ>]]) : string
  this extracts the last occurrence of header <name> in an http response, or of
  the last header if no <name> is specified. optionally, a specific occurrence
  might be specified as a position number. positive values indicate a position
  from the first occurrence, with 1 being the first one. negative values
  indicate positions relative to the last one, with -1 being the last one. it
  differs from res.hdr() in that any commas present in the value are returned
  and are not used as delimiters. if this is not desired, the res.hdr() fetch
  should be used instead. this is sometimes useful with headers such as date or
  expires.

res.fhdr_cnt([<name>]) : integer
  returns an integer value representing the number of occurrences of response
  header field name <name>, or the total number of header fields if <name> is
  not specified. contrary to its res.hdr_cnt() cousin, this function returns
  the number of full line headers and does not stop on commas. if this is not
  desired, the res.hdr_cnt() fetch should be used instead.

res.hdr([<name>[,<occ>]]) : string
shdr([<name>[,<occ>]]) : string (deprecated)
  this extracts the last occurrence of header <name> in an http response, or of
  the last header if no <name> is specified. optionally, a specific occurrence
  might be specified as a position number. positive values indicate a position
  from the first occurrence, with 1 being the first one. negative values
  indicate positions relative to the last one, with -1 being the last one. this
  can be useful to learn some data into a stick-table. the function considers
  any comma as a delimiter for distinct values. if this is not desired, the
  res.fhdr() fetch should be used instead.

  acl derivatives :
    shdr([<name>[,<occ>]])     : exact string match
    shdr_beg([<name>[,<occ>]]) : prefix match
    shdr_dir([<name>[,<occ>]]) : subdir match
    shdr_dom([<name>[,<occ>]]) : domain match
    shdr_end([<name>[,<occ>]]) : suffix match
    shdr_len([<name>[,<occ>]]) : length match
    shdr_reg([<name>[,<occ>]]) : regex match
    shdr_sub([<name>[,<occ>]]) : substring match

res.hdr_cnt([<name>]) : integer
shdr_cnt([<name>]) : integer (deprecated)
  returns an integer value representing the number of occurrences of response
  header field name <name>, or the total number of header fields if <name> is
  not specified. the function considers any comma as a delimiter for distinct
  values. if this is not desired, the res.fhdr_cnt() fetch should be used
  instead.

res.hdr_ip([<name>[,<occ>]]) : ip
shdr_ip([<name>[,<occ>]]) : ip (deprecated)
  this extracts the last occurrence of header <name> in an http response,
  convert it to an ipv4 or ipv6 address and returns this address. optionally, a
  specific occurrence might be specified as a position number. positive values
  indicate a position from the first occurrence, with 1 being the first one.
  negative values indicate positions relative to the last one, with -1 being
  the last one. this can be useful to learn some data into a stick table.

res.hdr_names([<delim>]) : string
  this builds a string made from the concatenation of all header names as they
  appear in the response when the rule is evaluated. the default delimiter is
  the comma (',') but it may be overridden as an optional argument <delim>. in
  this case, only the first character of <delim> is considered.

res.hdr_val([<name>[,<occ>]]) : integer
shdr_val([<name>[,<occ>]]) : integer (deprecated)
  this extracts the last occurrence of header <name> in an http response, and
  converts it to an integer value. optionally, a specific occurrence might be
  specified as a position number. positive values indicate a position from the
  first occurrence, with 1 being the first one. negative values indicate
  positions relative to the last one, with -1 being the last one. this can be
  useful to learn some data into a stick table.

res.ver : string
resp_ver : string (deprecated)
  returns the version string from the http response, for example "1.1". this
  can be useful for logs, but is mostly there for acl.

  acl derivatives :
    resp_ver : exact string match

set-cookie([<name>]) : string (deprecated)
  this extracts the last occurrence of the cookie name <name> on a "set-cookie"
  header line from the response and uses the corresponding value to match. this
  can be comparable to what "appsession" does with default options, but with
  support for multi-peer synchronization and state keeping across restarts.

  this fetch function is deprecated and has been superseded by the "res.cook"
  fetch. this keyword will disappear soon.

  see also : "appsession"

status : integer
  returns an integer containing the http status code in the http response, for
  example, 302\. it is mostly used within acls and integer ranges, for example,
  to remove any location header if the response is not a 3xx.

url : string
  this extracts the request's url as presented in the request. a typical use is
  with prefetch-capable caches, and with portals which need to aggregate
  multiple information from databases and keep them in caches. with acls, using
  "path" is preferred over using "url", because clients may send a full url as
  is normally done with proxies. the only real use is to match "*" which does
  not match in "path", and for which there is already a predefined acl. see
  also "path" and "base".

  acl derivatives :
    url     : exact string match
    url_beg : prefix match
    url_dir : subdir match
    url_dom : domain match
    url_end : suffix match
    url_len : length match
    url_reg : regex match
    url_sub : substring match

url_ip : ip
  this extracts the ip address from the request's url when the host part is
  presented as an ip address. its use is very limited. for instance, a
  monitoring system might use this field as an alternative for the source ip in
  order to test what path a given source address would follow, or to force an
  entry in a table for a given source address. with acls it can be used to
  restrict access to certain systems through a proxy, for example when combined
  with option "http_proxy".

url_port : integer
  this extracts the port part from the request's url. note that if the port is
  not specified in the request, port 80 is assumed. with acls it can be used to
  restrict access to certain systems through a proxy, for example when combined
  with option "http_proxy".

urlp([<name>[,<delim>]]) : string
url_param([<name>[,<delim>]]) : string
  this extracts the first occurrence of the parameter <name> in the query
  string, which begins after either '?' or <delim>, and which ends before '&',
  ';' or <delim>. the parameter name is case-sensitive. if no name is given,
  any parameter will match, and the first one will be returned. the result is
  a string corresponding to the value of the parameter <name> as presented in
  the request (no url decoding is performed). this can be used for session
  stickiness based on a client id, to extract an application cookie passed as a
  url parameter, or in acls to apply some checks. note that the acl version of
  this fetch iterates over multiple parameters and will iteratively report all
  parameters values if no name is given

  acl derivatives :
    urlp(<name>[,<delim>])     : exact string match
    urlp_beg(<name>[,<delim>]) : prefix match
    urlp_dir(<name>[,<delim>]) : subdir match
    urlp_dom(<name>[,<delim>]) : domain match
    urlp_end(<name>[,<delim>]) : suffix match
    urlp_len(<name>[,<delim>]) : length match
    urlp_reg(<name>[,<delim>]) : regex match
    urlp_sub(<name>[,<delim>]) : substring match

  example :
      # match http://example.com/foo?phpsessionid=some_id
      stick on urlp(phpsessionid)
      # match http://example.com/foo;jsessionid=some_id
      stick on urlp(jsessionid,;)

urlp_val([<name>[,<delim>])] : integer
  see "urlp" above. this one extracts the url parameter <name> in the request
  and converts it to an integer value. this can be used for session stickiness
  based on a user id for example, or with acls to match a page number or price.

7.4\. pre-defined acls
---------------------

some predefined acls are hard-coded so that they do not have to be declared in
every frontend which needs them. they all have their names in upper case in
order to avoid confusion. their equivalence is provided below.

acl name          equivalent to                usage
---------------+-----------------------------+---------------------------------
false            always_false                  never match
http             req_proto_http                match if protocol is valid http
http_1.0         req_ver 1.0                   match http version 1.0
http_1.1         req_ver 1.1                   match http version 1.1
http_content     hdr_val(content-length) gt 0  match an existing content-length
http_url_abs     url_reg ^[^/:]*://            match absolute url with scheme
http_url_slash   url_beg /                     match url beginning with "/"
http_url_star    url     *                     match url equal to "*"
localhost        src 127.0.0.1/8               match connection from local host
meth_connect     method  connect               match http connect method
meth_get         method  get head              match http get or head method
meth_head        method  head                  match http head method
meth_options     method  options               match http options method
meth_post        method  post                  match http post method
meth_trace       method  trace                 match http trace method
rdp_cookie       req_rdp_cookie_cnt gt 0       match presence of an rdp cookie
req_content      req_len gt 0                  match data in the request buffer
true             always_true                   always match
wait_end         wait_end                      wait for end of content analysis
---------------+-----------------------------+---------------------------------

8\. logging
----------

one of haproxy's strong points certainly lies is its precise logs. it probably
provides the finest level of information available for such a product, which is
very important for troubleshooting complex environments. standard information
provided in logs include client ports, tcp/http state timers, precise session
state at termination and precise termination cause, information about decisions
to direct traffic to a server, and of course the ability to capture arbitrary
headers.

in order to improve administrators reactivity, it offers a great transparency
about encountered problems, both internal and external, and it is possible to
send logs to different sources at the same time with different level filters :

  - global process-level logs (system errors, start/stop, etc..)
  - per-instance system and internal errors (lack of resource, bugs, ...)
  - per-instance external troubles (servers up/down, max connections)
  - per-instance activity (client connections), either at the establishment or
    at the termination.
  - per-request control of log-level, eg:
        http-request set-log-level silent if sensitive_request

the ability to distribute different levels of logs to different log servers
allow several production teams to interact and to fix their problems as soon
as possible. for example, the system team might monitor system-wide errors,
while the application team might be monitoring the up/down for their servers in
real time, and the security team might analyze the activity logs with one hour
delay.

8.1\. log levels
---------------

tcp and http connections can be logged with information such as the date, time,
source ip address, destination address, connection duration, response times,
http request, http return code, number of bytes transmitted, conditions
in which the session ended, and even exchanged cookies values. for example
track a particular user's problems. all messages may be sent to up to two
syslog servers. check the "log" keyword in section 4.2 for more information
about log facilities.

8.2\. log formats
----------------

haproxy supports 5 log formats. several fields are common between these formats
and will be detailed in the following sections. a few of them may vary
slightly with the configuration, due to indicators specific to certain
options. the supported formats are as follows :

  - the default format, which is very basic and very rarely used. it only
    provides very basic information about the incoming connection at the moment
    it is accepted : source ip:port, destination ip:port, and frontend-name.
    this mode will eventually disappear so it will not be described to great
    extents.

  - the tcp format, which is more advanced. this format is enabled when "option
    tcplog" is set on the frontend. haproxy will then usually wait for the
    connection to terminate before logging. this format provides much richer
    information, such as timers, connection counts, queue size, etc... this
    format is recommended for pure tcp proxies.

  - the http format, which is the most advanced for http proxying. this format
    is enabled when "option httplog" is set on the frontend. it provides the
    same information as the tcp format with some http-specific fields such as
    the request, the status code, and captures of headers and cookies. this
    format is recommended for http proxies.

  - the clf http format, which is equivalent to the http format, but with the
    fields arranged in the same order as the clf format. in this mode, all
    timers, captures, flags, etc... appear one per field after the end of the
    common fields, in the same order they appear in the standard http format.

  - the custom log format, allows you to make your own log line.

next sections will go deeper into details for each of these formats. format
specification will be performed on a "field" basis. unless stated otherwise, a
field is a portion of text delimited by any number of spaces. since syslog
servers are susceptible of inserting fields at the beginning of a line, it is
always assumed that the first field is the one containing the process name and
identifier.

note : since log lines may be quite long, the log examples in sections below
       might be broken into multiple lines. the example log lines will be
       prefixed with 3 closing angle brackets ('>>>') and each time a log is
       broken into multiple lines, each non-final line will end with a
       backslash ('\') and the next line will start indented by two characters.

8.2.1\. default log format
-------------------------

this format is used when no specific option is set. the log is emitted as soon
as the connection is accepted. one should note that this currently is the only
format which logs the request's destination ip and ports.

  example :
        listen www
            mode http
            log global
            server srv1 127.0.0.1:8000

    >>> feb  6 12:12:09 localhost \
          haproxy[14385]: connect from 10.0.1.2:33312 to 10.0.3.31:8012 \
          (www/http)

  field   format                                extract from the example above
      1   process_name '[' pid ']:'                            haproxy[14385]:
      2   'connect from'                                          connect from
      3   source_ip ':' source_port                             10.0.1.2:33312
      4   'to'                                                              to
      5   destination_ip ':' destination_port                   10.0.3.31:8012
      6   '(' frontend_name '/' mode ')'                            (www/http)

detailed fields description :
  - "source_ip" is the ip address of the client which initiated the connection.
  - "source_port" is the tcp port of the client which initiated the connection.
  - "destination_ip" is the ip address the client connected to.
  - "destination_port" is the tcp port the client connected to.
  - "frontend_name" is the name of the frontend (or listener) which received
    and processed the connection.
  - "mode is the mode the frontend is operating (tcp or http).

in case of a unix socket, the source and destination addresses are marked as
"unix:" and the ports reflect the internal id of the socket which accepted the
connection (the same id as reported in the stats).

it is advised not to use this deprecated format for newer installations as it
will eventually disappear.

8.2.2\. tcp log format
---------------------

the tcp format is used when "option tcplog" is specified in the frontend, and
is the recommended format for pure tcp proxies. it provides a lot of precious
information for troubleshooting. since this format includes timers and byte
counts, the log is normally emitted at the end of the session. it can be
emitted earlier if "option logasap" is specified, which makes sense in most
environments with long sessions such as remote terminals. sessions which match
the "monitor" rules are never logged. it is also possible not to emit logs for
sessions for which no data were exchanged between the client and the server, by
specifying "option dontlognull" in the frontend. successful connections will
not be logged if "option dontlog-normal" is specified in the frontend. a few
fields may slightly vary depending on some configuration options, those are
marked with a star ('*') after the field name below.

  example :
        frontend fnt
            mode tcp
            option tcplog
            log global
            default_backend bck

        backend bck
            server srv1 127.0.0.1:8000

    >>> feb  6 12:12:56 localhost \
          haproxy[14387]: 10.0.1.2:33313 [06/feb/2009:12:12:51.443] fnt \
          bck/srv1 0/0/5007 212 -- 0/0/0/0/3 0/0

  field   format                                extract from the example above
      1   process_name '[' pid ']:'                            haproxy[14387]:
      2   client_ip ':' client_port                             10.0.1.2:33313
      3   '[' accept_date ']'                       [06/feb/2009:12:12:51.443]
      4   frontend_name                                                    fnt
      5   backend_name '/' server_name                                bck/srv1
      6   tw '/' tc '/' tt*                                           0/0/5007
      7   bytes_read*                                                      212
      8   termination_state                                                 --
      9   actconn '/' feconn '/' beconn '/' srv_conn '/' retries*    0/0/0/0/3
     10   srv_queue '/' backend_queue                                      0/0

detailed fields description :
  - "client_ip" is the ip address of the client which initiated the tcp
    connection to haproxy. if the connection was accepted on a unix socket
    instead, the ip address would be replaced with the word "unix". note that
    when the connection is accepted on a socket configured with "accept-proxy"
    and the proxy protocol is correctly used, then the logs will reflect the
    forwarded connection's information.

  - "client_port" is the tcp port of the client which initiated the connection.
    if the connection was accepted on a unix socket instead, the port would be
    replaced with the id of the accepting socket, which is also reported in the
    stats interface.

  - "accept_date" is the exact date when the connection was received by haproxy
    (which might be very slightly different from the date observed on the
    network if there was some queuing in the system's backlog). this is usually
    the same date which may appear in any upstream firewall's log.

  - "frontend_name" is the name of the frontend (or listener) which received
    and processed the connection.

  - "backend_name" is the name of the backend (or listener) which was selected
    to manage the connection to the server. this will be the same as the
    frontend if no switching rule has been applied, which is common for tcp
    applications.

  - "server_name" is the name of the last server to which the connection was
    sent, which might differ from the first one if there were connection errors
    and a redispatch occurred. note that this server belongs to the backend
    which processed the request. if the connection was aborted before reaching
    a server, "<nosrv>" is indicated instead of a server name.

  - "tw" is the total time in milliseconds spent waiting in the various queues.
    it can be "-1" if the connection was aborted before reaching the queue.
    see "timers" below for more details.

  - "tc" is the total time in milliseconds spent waiting for the connection to
    establish to the final server, including retries. it can be "-1" if the
    connection was aborted before a connection could be established. see
    "timers" below for more details.

  - "tt" is the total time in milliseconds elapsed between the accept and the
    last close. it covers all possible processing. there is one exception, if
    "option logasap" was specified, then the time counting stops at the moment
    the log is emitted. in this case, a '+' sign is prepended before the value,
    indicating that the final one will be larger. see "timers" below for more
    details.

  - "bytes_read" is the total number of bytes transmitted from the server to
    the client when the log is emitted. if "option logasap" is specified, the
    this value will be prefixed with a '+' sign indicating that the final one
    may be larger. please note that this value is a 64-bit counter, so log
    analysis tools must be able to handle it without overflowing.

  - "termination_state" is the condition the session was in when the session
    ended. this indicates the session state, which side caused the end of
    session to happen, and for what reason (timeout, error, ...). the normal
    flags should be "--", indicating the session was closed by either end with
    no data remaining in buffers. see below "session state at disconnection"
    for more details.

  - "actconn" is the total number of concurrent connections on the process when
    the session was logged. it is useful to detect when some per-process system
    limits have been reached. for instance, if actconn is close to 512 when
    multiple connection errors occur, chances are high that the system limits
    the process to use a maximum of 1024 file descriptors and that all of them
    are used. see section 3 "global parameters" to find how to tune the system.

  - "feconn" is the total number of concurrent connections on the frontend when
    the session was logged. it is useful to estimate the amount of resource
    required to sustain high loads, and to detect when the frontend's "maxconn"
    has been reached. most often when this value increases by huge jumps, it is
    because there is congestion on the backend servers, but sometimes it can be
    caused by a denial of service attack.

  - "beconn" is the total number of concurrent connections handled by the
    backend when the session was logged. it includes the total number of
    concurrent connections active on servers as well as the number of
    connections pending in queues. it is useful to estimate the amount of
    additional servers needed to support high loads for a given application.
    most often when this value increases by huge jumps, it is because there is
    congestion on the backend servers, but sometimes it can be caused by a
    denial of service attack.

  - "srv_conn" is the total number of concurrent connections still active on
    the server when the session was logged. it can never exceed the server's
    configured "maxconn" parameter. if this value is very often close or equal
    to the server's "maxconn", it means that traffic regulation is involved a
    lot, meaning that either the server's maxconn value is too low, or that
    there aren't enough servers to process the load with an optimal response
    time. when only one of the server's "srv_conn" is high, it usually means
    that this server has some trouble causing the connections to take longer to
    be processed than on other servers.

  - "retries" is the number of connection retries experienced by this session
    when trying to connect to the server. it must normally be zero, unless a
    server is being stopped at the same moment the connection was attempted.
    frequent retries generally indicate either a network problem between
    haproxy and the server, or a misconfigured system backlog on the server
    preventing new connections from being queued. this field may optionally be
    prefixed with a '+' sign, indicating that the session has experienced a
    redispatch after the maximal retry count has been reached on the initial
    server. in this case, the server name appearing in the log is the one the
    connection was redispatched to, and not the first one, though both may
    sometimes be the same in case of hashing for instance. so as a general rule
    of thumb, when a '+' is present in front of the retry count, this count
    should not be attributed to the logged server.

  - "srv_queue" is the total number of requests which were processed before
    this one in the server queue. it is zero when the request has not gone
    through the server queue. it makes it possible to estimate the approximate
    server's response time by dividing the time spent in queue by the number of
    requests in the queue. it is worth noting that if a session experiences a
    redispatch and passes through two server queues, their positions will be
    cumulated. a request should not pass through both the server queue and the
    backend queue unless a redispatch occurs.

  - "backend_queue" is the total number of requests which were processed before
    this one in the backend's global queue. it is zero when the request has not
    gone through the global queue. it makes it possible to estimate the average
    queue length, which easily translates into a number of missing servers when
    divided by a server's "maxconn" parameter. it is worth noting that if a
    session experiences a redispatch, it may pass twice in the backend's queue,
    and then both positions will be cumulated. a request should not pass
    through both the server queue and the backend queue unless a redispatch
    occurs.

8.2.3\. http log format
----------------------

the http format is the most complete and the best suited for http proxies. it
is enabled by when "option httplog" is specified in the frontend. it provides
the same level of information as the tcp format with additional features which
are specific to the http protocol. just like the tcp format, the log is usually
emitted at the end of the session, unless "option logasap" is specified, which
generally only makes sense for download sites. a session which matches the
"monitor" rules will never logged. it is also possible not to log sessions for
which no data were sent by the client by specifying "option dontlognull" in the
frontend. successful connections will not be logged if "option dontlog-normal"
is specified in the frontend.

most fields are shared with the tcp log, some being different. a few fields may
slightly vary depending on some configuration options. those ones are marked
with a star ('*') after the field name below.

  example :
        frontend http-in
            mode http
            option httplog
            log global
            default_backend bck

        backend static
            server srv1 127.0.0.1:8000

    >>> feb  6 12:14:14 localhost \
          haproxy[14389]: 10.0.1.2:33317 [06/feb/2009:12:14:14.655] http-in \
          static/srv1 10/0/30/69/109 200 2750 - - ---- 1/1/1/1/0 0/0 {1wt.eu} \
          {} "get /index.html http/1.1"

  field   format                                extract from the example above
      1   process_name '[' pid ']:'                            haproxy[14389]:
      2   client_ip ':' client_port                             10.0.1.2:33317
      3   '[' accept_date ']'                       [06/feb/2009:12:14:14.655]
      4   frontend_name                                                http-in
      5   backend_name '/' server_name                             static/srv1
      6   tq '/' tw '/' tc '/' tr '/' tt*                       10/0/30/69/109
      7   status_code                                                      200
      8   bytes_read*                                                     2750
      9   captured_request_cookie                                            -
     10   captured_response_cookie                                           -
     11   termination_state                                               ----
     12   actconn '/' feconn '/' beconn '/' srv_conn '/' retries*    1/1/1/1/0
     13   srv_queue '/' backend_queue                                      0/0
     14   '{' captured_request_headers* '}'                   {haproxy.1wt.eu}
     15   '{' captured_response_headers* '}'                                {}
     16   '"' http_request '"'                      "get /index.html http/1.1"

detailed fields description :
  - "client_ip" is the ip address of the client which initiated the tcp
    connection to haproxy. if the connection was accepted on a unix socket
    instead, the ip address would be replaced with the word "unix". note that
    when the connection is accepted on a socket configured with "accept-proxy"
    and the proxy protocol is correctly used, then the logs will reflect the
    forwarded connection's information.

  - "client_port" is the tcp port of the client which initiated the connection.
    if the connection was accepted on a unix socket instead, the port would be
    replaced with the id of the accepting socket, which is also reported in the
    stats interface.

  - "accept_date" is the exact date when the tcp connection was received by
    haproxy (which might be very slightly different from the date observed on
    the network if there was some queuing in the system's backlog). this is
    usually the same date which may appear in any upstream firewall's log. this
    does not depend on the fact that the client has sent the request or not.

  - "frontend_name" is the name of the frontend (or listener) which received
    and processed the connection.

  - "backend_name" is the name of the backend (or listener) which was selected
    to manage the connection to the server. this will be the same as the
    frontend if no switching rule has been applied.

  - "server_name" is the name of the last server to which the connection was
    sent, which might differ from the first one if there were connection errors
    and a redispatch occurred. note that this server belongs to the backend
    which processed the request. if the request was aborted before reaching a
    server, "<nosrv>" is indicated instead of a server name. if the request was
    intercepted by the stats subsystem, "<stats>" is indicated instead.

  - "tq" is the total time in milliseconds spent waiting for the client to send
    a full http request, not counting data. it can be "-1" if the connection
    was aborted before a complete request could be received. it should always
    be very small because a request generally fits in one single packet. large
    times here generally indicate network trouble between the client and
    haproxy. see "timers" below for more details.

  - "tw" is the total time in milliseconds spent waiting in the various queues.
    it can be "-1" if the connection was aborted before reaching the queue.
    see "timers" below for more details.

  - "tc" is the total time in milliseconds spent waiting for the connection to
    establish to the final server, including retries. it can be "-1" if the
    request was aborted before a connection could be established. see "timers"
    below for more details.

  - "tr" is the total time in milliseconds spent waiting for the server to send
    a full http response, not counting data. it can be "-1" if the request was
    aborted before a complete response could be received. it generally matches
    the server's processing time for the request, though it may be altered by
    the amount of data sent by the client to the server. large times here on
    "get" requests generally indicate an overloaded server. see "timers" below
    for more details.

  - "tt" is the total time in milliseconds elapsed between the accept and the
    last close. it covers all possible processing. there is one exception, if
    "option logasap" was specified, then the time counting stops at the moment
    the log is emitted. in this case, a '+' sign is prepended before the value,
    indicating that the final one will be larger. see "timers" below for more
    details.

  - "status_code" is the http status code returned to the client. this status
    is generally set by the server, but it might also be set by haproxy when
    the server cannot be reached or when its response is blocked by haproxy.

  - "bytes_read" is the total number of bytes transmitted to the client when
    the log is emitted. this does include http headers. if "option logasap" is
    specified, the this value will be prefixed with a '+' sign indicating that
    the final one may be larger. please note that this value is a 64-bit
    counter, so log analysis tools must be able to handle it without
    overflowing.

  - "captured_request_cookie" is an optional "name=value" entry indicating that
    the client had this cookie in the request. the cookie name and its maximum
    length are defined by the "capture cookie" statement in the frontend
    configuration. the field is a single dash ('-') when the option is not
    set. only one cookie may be captured, it is generally used to track session
    id exchanges between a client and a server to detect session crossing
    between clients due to application bugs. for more details, please consult
    the section "capturing http headers and cookies" below.

  - "captured_response_cookie" is an optional "name=value" entry indicating
    that the server has returned a cookie with its response. the cookie name
    and its maximum length are defined by the "capture cookie" statement in the
    frontend configuration. the field is a single dash ('-') when the option is
    not set. only one cookie may be captured, it is generally used to track
    session id exchanges between a client and a server to detect session
    crossing between clients due to application bugs. for more details, please
    consult the section "capturing http headers and cookies" below.

  - "termination_state" is the condition the session was in when the session
    ended. this indicates the session state, which side caused the end of
    session to happen, for what reason (timeout, error, ...), just like in tcp
    logs, and information about persistence operations on cookies in the last
    two characters. the normal flags should begin with "--", indicating the
    session was closed by either end with no data remaining in buffers. see
    below "session state at disconnection" for more details.

  - "actconn" is the total number of concurrent connections on the process when
    the session was logged. it is useful to detect when some per-process system
    limits have been reached. for instance, if actconn is close to 512 or 1024
    when multiple connection errors occur, chances are high that the system
    limits the process to use a maximum of 1024 file descriptors and that all
    of them are used. see section 3 "global parameters" to find how to tune the
    system.

  - "feconn" is the total number of concurrent connections on the frontend when
    the session was logged. it is useful to estimate the amount of resource
    required to sustain high loads, and to detect when the frontend's "maxconn"
    has been reached. most often when this value increases by huge jumps, it is
    because there is congestion on the backend servers, but sometimes it can be
    caused by a denial of service attack.

  - "beconn" is the total number of concurrent connections handled by the
    backend when the session was logged. it includes the total number of
    concurrent connections active on servers as well as the number of
    connections pending in queues. it is useful to estimate the amount of
    additional servers needed to support high loads for a given application.
    most often when this value increases by huge jumps, it is because there is
    congestion on the backend servers, but sometimes it can be caused by a
    denial of service attack.

  - "srv_conn" is the total number of concurrent connections still active on
    the server when the session was logged. it can never exceed the server's
    configured "maxconn" parameter. if this value is very often close or equal
    to the server's "maxconn", it means that traffic regulation is involved a
    lot, meaning that either the server's maxconn value is too low, or that
    there aren't enough servers to process the load with an optimal response
    time. when only one of the server's "srv_conn" is high, it usually means
    that this server has some trouble causing the requests to take longer to be
    processed than on other servers.

  - "retries" is the number of connection retries experienced by this session
    when trying to connect to the server. it must normally be zero, unless a
    server is being stopped at the same moment the connection was attempted.
    frequent retries generally indicate either a network problem between
    haproxy and the server, or a misconfigured system backlog on the server
    preventing new connections from being queued. this field may optionally be
    prefixed with a '+' sign, indicating that the session has experienced a
    redispatch after the maximal retry count has been reached on the initial
    server. in this case, the server name appearing in the log is the one the
    connection was redispatched to, and not the first one, though both may
    sometimes be the same in case of hashing for instance. so as a general rule
    of thumb, when a '+' is present in front of the retry count, this count
    should not be attributed to the logged server.

  - "srv_queue" is the total number of requests which were processed before
    this one in the server queue. it is zero when the request has not gone
    through the server queue. it makes it possible to estimate the approximate
    server's response time by dividing the time spent in queue by the number of
    requests in the queue. it is worth noting that if a session experiences a
    redispatch and passes through two server queues, their positions will be
    cumulated. a request should not pass through both the server queue and the
    backend queue unless a redispatch occurs.

  - "backend_queue" is the total number of requests which were processed before
    this one in the backend's global queue. it is zero when the request has not
    gone through the global queue. it makes it possible to estimate the average
    queue length, which easily translates into a number of missing servers when
    divided by a server's "maxconn" parameter. it is worth noting that if a
    session experiences a redispatch, it may pass twice in the backend's queue,
    and then both positions will be cumulated. a request should not pass
    through both the server queue and the backend queue unless a redispatch
    occurs.

  - "captured_request_headers" is a list of headers captured in the request due
    to the presence of the "capture request header" statement in the frontend.
    multiple headers can be captured, they will be delimited by a vertical bar
    ('|'). when no capture is enabled, the braces do not appear, causing a
    shift of remaining fields. it is important to note that this field may
    contain spaces, and that using it requires a smarter log parser than when
    it's not used. please consult the section "capturing http headers and
    cookies" below for more details.

  - "captured_response_headers" is a list of headers captured in the response
    due to the presence of the "capture response header" statement in the
    frontend. multiple headers can be captured, they will be delimited by a
    vertical bar ('|'). when no capture is enabled, the braces do not appear,
    causing a shift of remaining fields. it is important to note that this
    field may contain spaces, and that using it requires a smarter log parser
    than when it's not used. please consult the section "capturing http headers
    and cookies" below for more details.

  - "http_request" is the complete http request line, including the method,
    request and http version string. non-printable characters are encoded (see
    below the section "non-printable characters"). this is always the last
    field, and it is always delimited by quotes and is the only one which can
    contain quotes. if new fields are added to the log format, they will be
    added before this field. this field might be truncated if the request is
    huge and does not fit in the standard syslog buffer (1024 characters). this
    is the reason why this field must always remain the last one.

8.2.4\. custom log format
------------------------

the directive log-format allows you to customize the logs in http mode and tcp
mode. it takes a string as argument.

haproxy understands some log format variables. % precedes log format variables.
variables can take arguments using braces ('{}'), and multiple arguments are
separated by commas within the braces. flags may be added or removed by
prefixing them with a '+' or '-' sign.

special variable "%o" may be used to propagate its flags to all other
variables on the same format string. this is particularly handy with quoted
string formats ("q").

if a variable is named between square brackets ('[' .. ']') then it is used
as a sample expression rule (see section 7.3). this it useful to add some
less common information such as the client's ssl certificate's dn, or to log
the key that would be used to store an entry into a stick table.

note: spaces must be escaped. a space character is considered as a separator.
in order to emit a verbatim '%', it must be preceded by another '%' resulting
in '%%'. haproxy will automatically merge consecutive separators.

flags are :
  * q: quote a string
  * x: hexadecimal representation (ips, ports, %ts, %rt, %pid)

  example:

    log-format %t\ %t\ some\ text
    log-format %{+q}o\ %t\ %s\ %{-q}r

at the moment, the default http format is defined this way :

    log-format %ci:%cp\ [%t]\ %ft\ %b/%s\ %tq/%tw/%tc/%tr/%tt\ %st\ %b\ %cc\ \
               %cs\ %tsc\ %ac/%fc/%bc/%sc/%rc\ %sq/%bq\ %hr\ %hs\ %{+q}r

the default clf format is defined this way :

    log-format %{+q}o\ %{-q}ci\ -\ -\ [%t]\ %r\ %st\ %b\ \"\"\ \"\"\ %cp\ \
               %ms\ %ft\ %b\ %s\ \%tq\ %tw\ %tc\ %tr\ %tt\ %tsc\ %ac\ %fc\ \
               %bc\ %sc\ %rc\ %sq\ %bq\ %cc\ %cs\ \%hrl\ %hsl

and the default tcp format is defined this way :

    log-format %ci:%cp\ [%t]\ %ft\ %b/%s\ %tw/%tc/%tt\ %b\ %ts\ \
               %ac/%fc/%bc/%sc/%rc\ %sq/%bq

please refer to the table below for currently defined variables :

  +---+------+-----------------------------------------------+-------------+
  | r | var  | field name (8.2.2 and 8.2.3 for description)  | type        |
  +---+------+-----------------------------------------------+-------------+
  |   | %o   | special variable, apply flags on all next var |             |
  +---+------+-----------------------------------------------+-------------+
  |   | %b   | bytes_read           (from server to client)  | numeric     |
  | h | %cc  | captured_request_cookie                       | string      |
  | h | %cs  | captured_response_cookie                      | string      |
  |   | %h   | hostname                                      | string      |
  | h | %hm  | http method (ex: post)                        | string      |
  | h | %hp  | http request uri without query string (path)  | string      |
  | h | %hu  | http request uri (ex: /foo?bar=baz)           | string      |
  | h | %hv  | http version (ex: http/1.0)                   | string      |
  |   | %id  | unique-id                                     | string      |
  |   | %st  | status_code                                   | numeric     |
  |   | %t   | gmt_date_time                                 | date        |
  |   | %tc  | tc                                            | numeric     |
  |   | %tl  | local_date_time                               | date        |
  | h | %tq  | tq                                            | numeric     |
  | h | %tr  | tr                                            | numeric     |
  |   | %ts  | timestamp                                     | numeric     |
  |   | %tt  | tt                                            | numeric     |
  |   | %tw  | tw                                            | numeric     |
  |   | %u   | bytes_uploaded       (from client to server)  | numeric     |
  |   | %ac  | actconn                                       | numeric     |
  |   | %b   | backend_name                                  | string      |
  |   | %bc  | beconn      (backend concurrent connections)  | numeric     |
  |   | %bi  | backend_source_ip       (connecting address)  | ip          |
  |   | %bp  | backend_source_port     (connecting address)  | numeric     |
  |   | %bq  | backend_queue                                 | numeric     |
  |   | %ci  | client_ip                 (accepted address)  | ip          |
  |   | %cp  | client_port               (accepted address)  | numeric     |
  |   | %f   | frontend_name                                 | string      |
  |   | %fc  | feconn     (frontend concurrent connections)  | numeric     |
  |   | %fi  | frontend_ip              (accepting address)  | ip          |
  |   | %fp  | frontend_port            (accepting address)  | numeric     |
  |   | %ft  | frontend_name_transport ('~' suffix for ssl)  | string      |
  |   | %lc  | frontend_log_counter                          | numeric     |
  |   | %hr  | captured_request_headers default style        | string      |
  |   | %hrl | captured_request_headers clf style            | string list |
  |   | %hs  | captured_response_headers default style       | string      |
  |   | %hsl | captured_response_headers clf style           | string list |
  |   | %ms  | accept date milliseconds                      | numeric     |
  |   | %pid | pid                                           | numeric     |
  | h | %r   | http_request                                  | string      |
  |   | %rc  | retries                                       | numeric     |
  |   | %rt  | request_counter (http req or tcp session)     | numeric     |
  |   | %s   | server_name                                   | string      |
  |   | %sc  | srv_conn     (server concurrent connections)  | numeric     |
  |   | %si  | server_ip                   (target address)  | ip          |
  |   | %sp  | server_port                 (target address)  | numeric     |
  |   | %sq  | srv_queue                                     | numeric     |
  | s | %sslc| ssl_ciphers (ex: aes-sha)                     | string      |
  | s | %sslv| ssl_version (ex: tlsv1)                       | string      |
  |   | %t   | date_time      (with millisecond resolution)  | date        |
  |   | %ts  | termination_state                             | string      |
  | h | %tsc | termination_state with cookie status          | string      |
  +---+------+-----------------------------------------------+-------------+

    r = restrictions : h = mode http only ; s = ssl only

8.2.5\. error log format
-----------------------

when an incoming connection fails due to an ssl handshake or an invalid proxy
protocol header, haproxy will log the event using a shorter, fixed line format.
by default, logs are emitted at the log_info level, unless the option
"log-separate-errors" is set in the backend, in which case the log_err level
will be used. connections on which no data are exchanged (eg: probes) are not
logged if the "dontlognull" option is set.

the format looks like this :

    >>> dec  3 18:27:14 localhost \
          haproxy[6103]: 127.0.0.1:56059 [03/dec/2012:17:35:10.380] frt/f1: \
          connection error during ssl handshake

  field   format                                extract from the example above
      1   process_name '[' pid ']:'                             haproxy[6103]:
      2   client_ip ':' client_port                            127.0.0.1:56059
      3   '[' accept_date ']'                       [03/dec/2012:17:35:10.380]
      4   frontend_name "/" bind_name ":"                              frt/f1:
      5   message                        connection error during ssl handshake

these fields just provide minimal information to help debugging connection
failures.

8.3\. advanced logging options
-----------------------------

some advanced logging options are often looked for but are not easy to find out
just by looking at the various options. here is an entry point for the few
options which can enable better logging. please refer to the keywords reference
for more information about their usage.

8.3.1\. disabling logging of external tests
------------------------------------------

it is quite common to have some monitoring tools perform health checks on
haproxy. sometimes it will be a layer 3 load-balancer such as lvs or any
commercial load-balancer, and sometimes it will simply be a more complete
monitoring system such as nagios. when the tests are very frequent, users often
ask how to disable logging for those checks. there are three possibilities :

  - if connections come from everywhere and are just tcp probes, it is often
    desired to simply disable logging of connections without data exchange, by
    setting "option dontlognull" in the frontend. it also disables logging of
    port scans, which may or may not be desired.

  - if the connection come from a known source network, use "monitor-net" to
    declare this network as monitoring only. any host in this network will then
    only be able to perform health checks, and their requests will not be
    logged. this is generally appropriate to designate a list of equipment
    such as other load-balancers.

  - if the tests are performed on a known uri, use "monitor-uri" to declare
    this uri as dedicated to monitoring. any host sending this request will
    only get the result of a health-check, and the request will not be logged.

8.3.2\. logging before waiting for the session to terminate
----------------------------------------------------------

the problem with logging at end of connection is that you have no clue about
what is happening during very long sessions, such as remote terminal sessions
or large file downloads. this problem can be worked around by specifying
"option logasap" in the frontend. haproxy will then log as soon as possible,
just before data transfer begins. this means that in case of tcp, it will still
log the connection status to the server, and in case of http, it will log just
after processing the server headers. in this case, the number of bytes reported
is the number of header bytes sent to the client. in order to avoid confusion
with normal logs, the total time field and the number of bytes are prefixed
with a '+' sign which means that real numbers are certainly larger.

8.3.3\. raising log level upon errors
------------------------------------

sometimes it is more convenient to separate normal traffic from errors logs,
for instance in order to ease error monitoring from log files. when the option
"log-separate-errors" is used, connections which experience errors, timeouts,
retries, redispatches or http status codes 5xx will see their syslog level
raised from "info" to "err". this will help a syslog daemon store the log in
a separate file. it is very important to keep the errors in the normal traffic
file too, so that log ordering is not altered. you should also be careful if
you already have configured your syslog daemon to store all logs higher than
"notice" in an "admin" file, because the "err" level is higher than "notice".

8.3.4\. disabling logging of successful connections
--------------------------------------------------

although this may sound strange at first, some large sites have to deal with
multiple thousands of logs per second and are experiencing difficulties keeping
them intact for a long time or detecting errors within them. if the option
"dontlog-normal" is set on the frontend, all normal connections will not be
logged. in this regard, a normal connection is defined as one without any
error, timeout, retry nor redispatch. in http, the status code is checked too,
and a response with a status 5xx is not considered normal and will be logged
too. of course, doing is is really discouraged as it will remove most of the
useful information from the logs. do this only if you have no other
alternative.

8.4\. timing events
------------------

timers provide a great help in troubleshooting network problems. all values are
reported in milliseconds (ms). these timers should be used in conjunction with
the session termination flags. in tcp mode with "option tcplog" set on the
frontend, 3 control points are reported under the form "tw/tc/tt", and in http
mode, 5 control points are reported under the form "tq/tw/tc/tr/tt" :

  - tq: total time to get the client request (http mode only). it's the time
    elapsed between the moment the client connection was accepted and the
    moment the proxy received the last http header. the value "-1" indicates
    that the end of headers (empty line) has never been seen. this happens when
    the client closes prematurely or times out.

  - tw: total time spent in the queues waiting for a connection slot. it
    accounts for backend queue as well as the server queues, and depends on the
    queue size, and the time needed for the server to complete previous
    requests. the value "-1" means that the request was killed before reaching
    the queue, which is generally what happens with invalid or denied requests.

  - tc: total time to establish the tcp connection to the server. it's the time
    elapsed between the moment the proxy sent the connection request, and the
    moment it was acknowledged by the server, or between the tcp syn packet and
    the matching syn/ack packet in return. the value "-1" means that the
    connection never established.

  - tr: server response time (http mode only). it's the time elapsed between
    the moment the tcp connection was established to the server and the moment
    the server sent its complete response headers. it purely shows its request
    processing time, without the network overhead due to the data transmission.
    it is worth noting that when the client has data to send to the server, for
    instance during a post request, the time already runs, and this can distort
    apparent response time. for this reason, it's generally wise not to trust
    too much this field for post requests initiated from clients behind an
    untrusted network. a value of "-1" here means that the last the response
    header (empty line) was never seen, most likely because the server timeout
    stroke before the server managed to process the request.

  - tt: total session duration time, between the moment the proxy accepted it
    and the moment both ends were closed. the exception is when the "logasap"
    option is specified. in this case, it only equals (tq+tw+tc+tr), and is
    prefixed with a '+' sign. from this field, we can deduce "td", the data
    transmission time, by subtracting other timers when valid :

        td = tt - (tq + tw + tc + tr)

    timers with "-1" values have to be excluded from this equation. in tcp
    mode, "tq" and "tr" have to be excluded too. note that "tt" can never be
    negative.

these timers provide precious indications on trouble causes. since the tcp
protocol defines retransmit delays of 3, 6, 12... seconds, we know for sure
that timers close to multiples of 3s are nearly always related to lost packets
due to network problems (wires, negotiation, congestion). moreover, if "tt" is
close to a timeout value specified in the configuration, it often means that a
session has been aborted on timeout.

most common cases :

  - if "tq" is close to 3000, a packet has probably been lost between the
    client and the proxy. this is very rare on local networks but might happen
    when clients are on far remote networks and send large requests. it may
    happen that values larger than usual appear here without any network cause.
    sometimes, during an attack or just after a resource starvation has ended,
    haproxy may accept thousands of connections in a few milliseconds. the time
    spent accepting these connections will inevitably slightly delay processing
    of other connections, and it can happen that request times in the order of
    a few tens of milliseconds are measured after a few thousands of new
    connections have been accepted at once. setting "option http-server-close"
    may display larger request times since "tq" also measures the time spent
    waiting for additional requests.

  - if "tc" is close to 3000, a packet has probably been lost between the
    server and the proxy during the server connection phase. this value should
    always be very low, such as 1 ms on local networks and less than a few tens
    of ms on remote networks.

  - if "tr" is nearly always lower than 3000 except some rare values which seem
    to be the average majored by 3000, there are probably some packets lost
    between the proxy and the server.

  - if "tt" is large even for small byte counts, it generally is because
    neither the client nor the server decides to close the connection, for
    instance because both have agreed on a keep-alive connection mode. in order
    to solve this issue, it will be needed to specify "option httpclose" on
    either the frontend or the backend. if the problem persists, it means that
    the server ignores the "close" connection mode and expects the client to
    close. then it will be required to use "option forceclose". having the
    smallest possible 'tt' is important when connection regulation is used with
    the "maxconn" option on the servers, since no new connection will be sent
    to the server until another one is released.

other noticeable http log cases ('xx' means any value to be ignored) :

  tq/tw/tc/tr/+tt  the "option logasap" is present on the frontend and the log
                   was emitted before the data phase. all the timers are valid
                   except "tt" which is shorter than reality.

  -1/xx/xx/xx/tt   the client was not able to send a complete request in time
                   or it aborted too early. check the session termination flags
                   then "timeout http-request" and "timeout client" settings.

  tq/-1/xx/xx/tt   it was not possible to process the request, maybe because
                   servers were out of order, because the request was invalid
                   or forbidden by acl rules. check the session termination
                   flags.

  tq/tw/-1/xx/tt   the connection could not establish on the server. either it
                   actively refused it or it timed out after tt-(tq+tw) ms.
                   check the session termination flags, then check the
                   "timeout connect" setting. note that the tarpit action might
                   return similar-looking patterns, with "tw" equal to the time
                   the client connection was maintained open.

  tq/tw/tc/-1/tt   the server has accepted the connection but did not return
                   a complete response in time, or it closed its connection
                   unexpectedly after tt-(tq+tw+tc) ms. check the session
                   termination flags, then check the "timeout server" setting.

8.5\. session state at disconnection
-----------------------------------

tcp and http logs provide a session termination indicator in the
"termination_state" field, just before the number of active connections. it is
2-characters long in tcp mode, and is extended to 4 characters in http mode,
each of which has a special meaning :

  - on the first character, a code reporting the first event which caused the
    session to terminate :

        c : the tcp session was unexpectedly aborted by the client.

        s : the tcp session was unexpectedly aborted by the server, or the
            server explicitly refused it.

        p : the session was prematurely aborted by the proxy, because of a
            connection limit enforcement, because a deny filter was matched,
            because of a security check which detected and blocked a dangerous
            error in server response which might have caused information leak
            (eg: cacheable cookie).

        l : the session was locally processed by haproxy and was not passed to
            a server. this is what happens for stats and redirects.

        r : a resource on the proxy has been exhausted (memory, sockets, source
            ports, ...). usually, this appears during the connection phase, and
            system logs should contain a copy of the precise error. if this
            happens, it must be considered as a very serious anomaly which
            should be fixed as soon as possible by any means.

        i : an internal error was identified by the proxy during a self-check.
            this should never happen, and you are encouraged to report any log
            containing this, because this would almost certainly be a bug. it
            would be wise to preventively restart the process after such an
            event too, in case it would be caused by memory corruption.

        d : the session was killed by haproxy because the server was detected
            as down and was configured to kill all connections when going down.

        u : the session was killed by haproxy on this backup server because an
            active server was detected as up and was configured to kill all
            backup connections when going up.

        k : the session was actively killed by an admin operating on haproxy.

        c : the client-side timeout expired while waiting for the client to
            send or receive data.

        s : the server-side timeout expired while waiting for the server to
            send or receive data.

        - : normal session completion, both the client and the server closed
            with nothing left in the buffers.

  - on the second character, the tcp or http session state when it was closed :

        r : the proxy was waiting for a complete, valid request from the client
            (http mode only). nothing was sent to any server.

        q : the proxy was waiting in the queue for a connection slot. this can
            only happen when servers have a 'maxconn' parameter set. it can
            also happen in the global queue after a redispatch consecutive to
            a failed attempt to connect to a dying server. if no redispatch is
            reported, then no connection attempt was made to any server.

        c : the proxy was waiting for the connection to establish on the
            server. the server might at most have noticed a connection attempt.

        h : the proxy was waiting for complete, valid response headers from the
            server (http only).

        d : the session was in the data phase.

        l : the proxy was still transmitting last data to the client while the
            server had already finished. this one is very rare as it can only
            happen when the client dies while receiving the last packets.

        t : the request was tarpitted. it has been held open with the client
            during the whole "timeout tarpit" duration or until the client
            closed, both of which will be reported in the "tw" timer.

        - : normal session completion after end of data transfer.

  - the third character tells whether the persistence cookie was provided by
    the client (only in http mode) :

        n : the client provided no cookie. this is usually the case for new
            visitors, so counting the number of occurrences of this flag in the
            logs generally indicate a valid trend for the site frequentation.

        i : the client provided an invalid cookie matching no known server.
            this might be caused by a recent configuration change, mixed
            cookies between http/https sites, persistence conditionally
            ignored, or an attack.

        d : the client provided a cookie designating a server which was down,
            so either "option persist" was used and the client was sent to
            this server, or it was not set and the client was redispatched to
            another server.

        v : the client provided a valid cookie, and was sent to the associated
            server.

        e : the client provided a valid cookie, but with a last date which was
            older than what is allowed by the "maxidle" cookie parameter, so
            the cookie is consider expired and is ignored. the request will be
            redispatched just as if there was no cookie.

        o : the client provided a valid cookie, but with a first date which was
            older than what is allowed by the "maxlife" cookie parameter, so
            the cookie is consider too old and is ignored. the request will be
            redispatched just as if there was no cookie.

        u : a cookie was present but was not used to select the server because
            some other server selection mechanism was used instead (typically a
            "use-server" rule).

        - : does not apply (no cookie set in configuration).

  - the last character reports what operations were performed on the persistence
    cookie returned by the server (only in http mode) :

        n : no cookie was provided by the server, and none was inserted either.

        i : no cookie was provided by the server, and the proxy inserted one.
            note that in "cookie insert" mode, if the server provides a cookie,
            it will still be overwritten and reported as "i" here.

        u : the proxy updated the last date in the cookie that was presented by
            the client. this can only happen in insert mode with "maxidle". it
            happens every time there is activity at a different date than the
            date indicated in the cookie. if any other change happens, such as
            a redispatch, then the cookie will be marked as inserted instead.

        p : a cookie was provided by the server and transmitted as-is.

        r : the cookie provided by the server was rewritten by the proxy, which
            happens in "cookie rewrite" or "cookie prefix" modes.

        d : the cookie provided by the server was deleted by the proxy.

        - : does not apply (no cookie set in configuration).

the combination of the two first flags gives a lot of information about what
was happening when the session terminated, and why it did terminate. it can be
helpful to detect server saturation, network troubles, local system resource
starvation, attacks, etc...

the most common termination flags combinations are indicated below. they are
alphabetically sorted, with the lowercase set just after the upper case for
easier finding and understanding.

  flags   reason

     --   normal termination.

     cc   the client aborted before the connection could be established to the
          server. this can happen when haproxy tries to connect to a recently
          dead (or unchecked) server, and the client aborts while haproxy is
          waiting for the server to respond or for "timeout connect" to expire.

     cd   the client unexpectedly aborted during data transfer. this can be
          caused by a browser crash, by an intermediate equipment between the
          client and haproxy which decided to actively break the connection,
          by network routing issues between the client and haproxy, or by a
          keep-alive session between the server and the client terminated first
          by the client.

     cd   the client did not send nor acknowledge any data for as long as the
          "timeout client" delay. this is often caused by network failures on
          the client side, or the client simply leaving the net uncleanly.

     ch   the client aborted while waiting for the server to start responding.
          it might be the server taking too long to respond or the client
          clicking the 'stop' button too fast.

     ch   the "timeout client" stroke while waiting for client data during a
          post request. this is sometimes caused by too large tcp mss values
          for pppoe networks which cannot transport full-sized packets. it can
          also happen when client timeout is smaller than server timeout and
          the server takes too long to respond.

     cq   the client aborted while its session was queued, waiting for a server
          with enough empty slots to accept it. it might be that either all the
          servers were saturated or that the assigned server was taking too
          long a time to respond.

     cr   the client aborted before sending a full http request. most likely
          the request was typed by hand using a telnet client, and aborted
          too early. the http status code is likely a 400 here. sometimes this
          might also be caused by an ids killing the connection between haproxy
          and the client. "option http-ignore-probes" can be used to ignore
          connections without any data transfer.

     cr   the "timeout http-request" stroke before the client sent a full http
          request. this is sometimes caused by too large tcp mss values on the
          client side for pppoe networks which cannot transport full-sized
          packets, or by clients sending requests by hand and not typing fast
          enough, or forgetting to enter the empty line at the end of the
          request. the http status code is likely a 408 here. note: recently,
          some browsers started to implement a "pre-connect" feature consisting
          in speculatively connecting to some recently visited web sites just
          in case the user would like to visit them. this results in many
          connections being established to web sites, which end up in 408
          request timeout if the timeout strikes first, or 400 bad request when
          the browser decides to close them first. these ones pollute the log
          and feed the error counters. some versions of some browsers have even
          been reported to display the error code. it is possible to work
          around the undesirable effects of this behaviour by adding "option
          http-ignore-probes" in the frontend, resulting in connections with
          zero data transfer to be totally ignored. this will definitely hide
          the errors of people experiencing connectivity issues though.

     ct   the client aborted while its session was tarpitted. it is important to
          check if this happens on valid requests, in order to be sure that no
          wrong tarpit rules have been written. if a lot of them happen, it
          might make sense to lower the "timeout tarpit" value to something
          closer to the average reported "tw" timer, in order not to consume
          resources for just a few attackers.

     lr   the request was intercepted and locally handled by haproxy. generally
          it means that this was a redirect or a stats request.

     sc   the server or an equipment between it and haproxy explicitly refused
          the tcp connection (the proxy received a tcp rst or an icmp message
          in return). under some circumstances, it can also be the network
          stack telling the proxy that the server is unreachable (eg: no route,
          or no arp response on local network). when this happens in http mode,
          the status code is likely a 502 or 503 here.

     sc   the "timeout connect" stroke before a connection to the server could
          complete. when this happens in http mode, the status code is likely a
          503 or 504 here.

     sd   the connection to the server died with an error during the data
          transfer. this usually means that haproxy has received an rst from
          the server or an icmp message from an intermediate equipment while
          exchanging data with the server. this can be caused by a server crash
          or by a network issue on an intermediate equipment.

     sd   the server did not send nor acknowledge any data for as long as the
          "timeout server" setting during the data phase. this is often caused
          by too short timeouts on l4 equipments before the server (firewalls,
          load-balancers, ...), as well as keep-alive sessions maintained
          between the client and the server expiring first on haproxy.

     sh   the server aborted before sending its full http response headers, or
          it crashed while processing the request. since a server aborting at
          this moment is very rare, it would be wise to inspect its logs to
          control whether it crashed and why. the logged request may indicate a
          small set of faulty requests, demonstrating bugs in the application.
          sometimes this might also be caused by an ids killing the connection
          between haproxy and the server.

     sh   the "timeout server" stroke before the server could return its
          response headers. this is the most common anomaly, indicating too
          long transactions, probably caused by server or database saturation.
          the immediate workaround consists in increasing the "timeout server"
          setting, but it is important to keep in mind that the user experience
          will suffer from these long response times. the only long term
          solution is to fix the application.

     sq   the session spent too much time in queue and has been expired. see
          the "timeout queue" and "timeout connect" settings to find out how to
          fix this if it happens too often. if it often happens massively in
          short periods, it may indicate general problems on the affected
          servers due to i/o or database congestion, or saturation caused by
          external attacks.

     pc   the proxy refused to establish a connection to the server because the
          process' socket limit has been reached while attempting to connect.
          the global "maxconn" parameter may be increased in the configuration
          so that it does not happen anymore. this status is very rare and
          might happen when the global "ulimit-n" parameter is forced by hand.

     pd   the proxy blocked an incorrectly formatted chunked encoded message in
          a request or a response, after the server has emitted its headers. in
          most cases, this will indicate an invalid message from the server to
          the client. haproxy supports chunk sizes of up to 2gb - 1 (2147483647
          bytes). any larger size will be considered as an error.

     ph   the proxy blocked the server's response, because it was invalid,
          incomplete, dangerous (cache control), or matched a security filter.
          in any case, an http 502 error is sent to the client. one possible
          cause for this error is an invalid syntax in an http header name
          containing unauthorized characters. it is also possible but quite
          rare, that the proxy blocked a chunked-encoding request from the
          client due to an invalid syntax, before the server responded. in this
          case, an http 400 error is sent to the client and reported in the
          logs.

     pr   the proxy blocked the client's http request, either because of an
          invalid http syntax, in which case it returned an http 400 error to
          the client, or because a deny filter matched, in which case it
          returned an http 403 error.

     pt   the proxy blocked the client's request and has tarpitted its
          connection before returning it a 500 server error. nothing was sent
          to the server. the connection was maintained open for as long as
          reported by the "tw" timer field.

     rc   a local resource has been exhausted (memory, sockets, source ports)
          preventing the connection to the server from establishing. the error
          logs will tell precisely what was missing. this is very rare and can
          only be solved by proper system tuning.

the combination of the two last flags gives a lot of information about how
persistence was handled by the client, the server and by haproxy. this is very
important to troubleshoot disconnections, when users complain they have to
re-authenticate. the commonly encountered flags are :

     --   persistence cookie is not enabled.

     nn   no cookie was provided by the client, none was inserted in the
          response. for instance, this can be in insert mode with "postonly"
          set on a get request.

     ii   a cookie designating an invalid server was provided by the client,
          a valid one was inserted in the response. this typically happens when
          a "server" entry is removed from the configuration, since its cookie
          value can be presented by a client when no other server knows it.

     ni   no cookie was provided by the client, one was inserted in the
          response. this typically happens for first requests from every user
          in "insert" mode, which makes it an easy way to count real users.

     vn   a cookie was provided by the client, none was inserted in the
          response. this happens for most responses for which the client has
          already got a cookie.

     vu   a cookie was provided by the client, with a last visit date which is
          not completely up-to-date, so an updated cookie was provided in
          response. this can also happen if there was no date at all, or if
          there was a date but the "maxidle" parameter was not set, so that the
          cookie can be switched to unlimited time.

     ei   a cookie was provided by the client, with a last visit date which is
          too old for the "maxidle" parameter, so the cookie was ignored and a
          new cookie was inserted in the response.

     oi   a cookie was provided by the client, with a first visit date which is
          too old for the "maxlife" parameter, so the cookie was ignored and a
          new cookie was inserted in the response.

     di   the server designated by the cookie was down, a new server was
          selected and a new cookie was emitted in the response.

     vi   the server designated by the cookie was not marked dead but could not
          be reached. a redispatch happened and selected another one, which was
          then advertised in the response.

8.6\. non-printable characters
-----------------------------

in order not to cause trouble to log analysis tools or terminals during log
consulting, non-printable characters are not sent as-is into log files, but are
converted to the two-digits hexadecimal representation of their ascii code,
prefixed by the character '#'. the only characters that can be logged without
being escaped are comprised between 32 and 126 (inclusive). obviously, the
escape character '#' itself is also encoded to avoid any ambiguity ("#23"). it
is the same for the character '"' which becomes "#22", as well as '{', '|' and
'}' when logging headers.

note that the space character (' ') is not encoded in headers, which can cause
issues for tools relying on space count to locate fields. a typical header
containing spaces is "user-agent".

last, it has been observed that some syslog daemons such as syslog-ng escape
the quote ('"') with a backslash ('\'). the reverse operation can safely be
performed since no quote may appear anywhere else in the logs.

8.7\. capturing http cookies
---------------------------

cookie capture simplifies the tracking a complete user session. this can be
achieved using the "capture cookie" statement in the frontend. please refer to
section 4.2 for more details. only one cookie can be captured, and the same
cookie will simultaneously be checked in the request ("cookie:" header) and in
the response ("set-cookie:" header). the respective values will be reported in
the http logs at the "captured_request_cookie" and "captured_response_cookie"
locations (see section 8.2.3 about http log format). when either cookie is
not seen, a dash ('-') replaces the value. this way, it's easy to detect when a
user switches to a new session for example, because the server will reassign it
a new cookie. it is also possible to detect if a server unexpectedly sets a
wrong cookie to a client, leading to session crossing.

  examples :
        # capture the first cookie whose name starts with "aspsession"
        capture cookie aspsession len 32

        # capture the first cookie whose name is exactly "vgnvisitor"
        capture cookie vgnvisitor= len 32

8.8\. capturing http headers
---------------------------

header captures are useful to track unique request identifiers set by an upper
proxy, virtual host names, user-agents, post content-length, referrers, etc. in
the response, one can search for information about the response length, how the
server asked the cache to behave, or an object location during a redirection.

header captures are performed using the "capture request header" and "capture
response header" statements in the frontend. please consult their definition in
section 4.2 for more details.

it is possible to include both request headers and response headers at the same
time. non-existent headers are logged as empty strings, and if one header
appears more than once, only its last occurrence will be logged. request headers
are grouped within braces '{' and '}' in the same order as they were declared,
and delimited with a vertical bar '|' without any space. response headers
follow the same representation, but are displayed after a space following the
request headers block. these blocks are displayed just before the http request
in the logs.

as a special case, it is possible to specify an http header capture in a tcp
frontend. the purpose is to enable logging of headers which will be parsed in
an http backend if the request is then switched to this http backend.

  example :
        # this instance chains to the outgoing proxy
        listen proxy-out
            mode http
            option httplog
            option logasap
            log global
            server cache1 192.168.1.1:3128

            # log the name of the virtual server
            capture request  header host len 20

            # log the amount of data uploaded during a post
            capture request  header content-length len 10

            # log the beginning of the referrer
            capture request  header referer len 20

            # server name (useful for outgoing proxies only)
            capture response header server len 20

            # logging the content-length is useful with "option logasap"
            capture response header content-length len 10

            # log the expected cache behaviour on the response
            capture response header cache-control len 8

            # the via header will report the next proxy's name
            capture response header via len 20

            # log the url location during a redirection
            capture response header location len 20

    >>> aug  9 20:26:09 localhost \
          haproxy[2022]: 127.0.0.1:34014 [09/aug/2004:20:26:09] proxy-out \
          proxy-out/cache1 0/0/0/162/+162 200 +350 - - ---- 0/0/0/0/0 0/0 \
          {fr.adserver.yahoo.co||http://fr.f416.mail.} {|864|private||} \
          "get http://fr.adserver.yahoo.com/"

    >>> aug  9 20:30:46 localhost \
          haproxy[2022]: 127.0.0.1:34020 [09/aug/2004:20:30:46] proxy-out \
          proxy-out/cache1 0/0/0/182/+182 200 +279 - - ---- 0/0/0/0/0 0/0 \
          {w.ods.org||} {formilux/0.1.8|3495|||} \
          "get http://trafic.1wt.eu/ http/1.1"

    >>> aug  9 20:30:46 localhost \
          haproxy[2022]: 127.0.0.1:34028 [09/aug/2004:20:30:46] proxy-out \
          proxy-out/cache1 0/0/2/126/+128 301 +223 - - ---- 0/0/0/0/0 0/0 \
          {www.sytadin.equipement.gouv.fr||http://trafic.1wt.eu/} \
          {apache|230|||http://www.sytadin.} \
          "get http://www.sytadin.equipement.gouv.fr/ http/1.1"

8.9\. examples of logs
---------------------

these are real-world examples of logs accompanied with an explanation. some of
them have been made up by hand. the syslog part has been removed for better
reading. their sole purpose is to explain how to decipher them.

    >>> haproxy[674]: 127.0.0.1:33318 [15/oct/2003:08:31:57.130] px-http \
          px-http/srv1 6559/0/7/147/6723 200 243 - - ---- 5/3/3/1/0 0/0 \
          "head / http/1.0"

    => long request (6.5s) entered by hand through 'telnet'. the server replied
       in 147 ms, and the session ended normally ('----')

    >>> haproxy[674]: 127.0.0.1:33319 [15/oct/2003:08:31:57.149] px-http \
          px-http/srv1 6559/1230/7/147/6870 200 243 - - ---- 324/239/239/99/0 \
          0/9 "head / http/1.0"

    => idem, but the request was queued in the global queue behind 9 other
       requests, and waited there for 1230 ms.

    >>> haproxy[674]: 127.0.0.1:33320 [15/oct/2003:08:32:17.654] px-http \
          px-http/srv1 9/0/7/14/+30 200 +243 - - ---- 3/3/3/1/0 0/0 \
          "get /image.iso http/1.0"

    => request for a long data transfer. the "logasap" option was specified, so
       the log was produced just before transferring data. the server replied in
       14 ms, 243 bytes of headers were sent to the client, and total time from
       accept to first data byte is 30 ms.

    >>> haproxy[674]: 127.0.0.1:33320 [15/oct/2003:08:32:17.925] px-http \
          px-http/srv1 9/0/7/14/30 502 243 - - ph-- 3/2/2/0/0 0/0 \
          "get /cgi-bin/bug.cgi? http/1.0"

    => the proxy blocked a server response either because of an "rspdeny" or
       "rspideny" filter, or because the response was improperly formatted and
       not http-compliant, or because it blocked sensitive information which
       risked being cached. in this case, the response is replaced with a "502
       bad gateway". the flags ("ph--") tell us that it was haproxy who decided
       to return the 502 and not the server.

    >>> haproxy[18113]: 127.0.0.1:34548 [15/oct/2003:15:18:55.798] px-http \
          px-http/<nosrv> -1/-1/-1/-1/8490 -1 0 - - cr-- 2/2/2/0/0 0/0 ""

    => the client never completed its request and aborted itself ("c---") after
       8.5s, while the proxy was waiting for the request headers ("-r--").
       nothing was sent to any server.

    >>> haproxy[18113]: 127.0.0.1:34549 [15/oct/2003:15:19:06.103] px-http \
         px-http/<nosrv> -1/-1/-1/-1/50001 408 0 - - cr-- 2/2/2/0/0 0/0 ""

    => the client never completed its request, which was aborted by the
       time-out ("c---") after 50s, while the proxy was waiting for the request
       headers ("-r--").  nothing was sent to any server, but the proxy could
       send a 408 return code to the client.

    >>> haproxy[18989]: 127.0.0.1:34550 [15/oct/2003:15:24:28.312] px-tcp \
          px-tcp/srv1 0/0/5007 0 cd 0/0/0/0/0 0/0

    => this log was produced with "option tcplog". the client timed out after
       5 seconds ("c----").

    >>> haproxy[18989]: 10.0.0.1:34552 [15/oct/2003:15:26:31.462] px-http \
          px-http/srv1 3183/-1/-1/-1/11215 503 0 - - sc-- 205/202/202/115/3 \
          0/0 "head / http/1.0"

    => the request took 3s to complete (probably a network problem), and the
       connection to the server failed ('sc--') after 4 attempts of 2 seconds
       (config says 'retries 3'), and no redispatch (otherwise we would have
       seen "/+3"). status code 503 was returned to the client. there were 115
       connections on this server, 202 connections on this proxy, and 205 on
       the global process. it is possible that the server refused the
       connection because of too many already established.

9\. statistics and monitoring
----------------------------

it is possible to query haproxy about its status. the most commonly used
mechanism is the http statistics page. this page also exposes an alternative
csv output format for monitoring tools. the same format is provided on the
unix socket.

9.1\. csv format
---------------

the statistics may be consulted either from the unix socket or from the http
page. both means provide a csv format whose fields follow. the first line
begins with a sharp ('#') and has one word per comma-delimited field which
represents the title of the column. all other lines starting at the second one
use a classical csv format using a comma as the delimiter, and the double quote
('"') as an optional text delimiter, but only if the enclosed text is ambiguous
(if it contains a quote or a comma). the double-quote character ('"') in the
text is doubled ('""'), which is the format that most tools recognize. please
do not insert any column before these ones in order not to break tools which
use hard-coded column positions.

in brackets after each field name are the types which may have a value for
that field. the types are l (listeners), f (frontends), b (backends), and
s (servers).

  0\. pxname [lfbs]: proxy name
  1\. svname [lfbs]: service name (frontend for frontend, backend for backend,
     any name for server/listener)
  2\. qcur [..bs]: current queued requests. for the backend this reports the
     number queued without a server assigned.
  3\. qmax [..bs]: max value of qcur
  4\. scur [lfbs]: current sessions
  5\. smax [lfbs]: max sessions
  6\. slim [lfbs]: configured session limit
  7\. stot [lfbs]: cumulative number of connections
  8\. bin [lfbs]: bytes in
  9\. bout [lfbs]: bytes out
 10\. dreq [lfb.]: requests denied because of security concerns.
     - for tcp this is because of a matched tcp-request content rule.
     - for http this is because of a matched http-request or tarpit rule.
 11\. dresp [lfbs]: responses denied because of security concerns.
     - for http this is because of a matched http-request rule, or
       "option checkcache".
 12\. ereq [lf..]: request errors. some of the possible causes are:
     - early termination from the client, before the request has been sent.
     - read error from the client
     - client timeout
     - client closed connection
     - various bad requests from the client.
     - request was tarpitted.
 13\. econ [..bs]: number of requests that encountered an error trying to
     connect to a backend server. the backend stat is the sum of the stat
     for all servers of that backend, plus any connection errors not
     associated with a particular server (such as the backend having no
     active servers).
 14\. eresp [..bs]: response errors. srv_abrt will be counted here also.
     some other errors are:
     - write error on the client socket (won't be counted for the server stat)
     - failure applying filters to the response.
 15\. wretr [..bs]: number of times a connection to a server was retried.
 16\. wredis [..bs]: number of times a request was redispatched to another
     server. the server value counts the number of times that server was
     switched away from.
 17\. status [lfbs]: status (up/down/nolb/maint/maint(via)...)
 18\. weight [..bs]: total weight (backend), server weight (server)
 19\. act [..bs]: number of active servers (backend), server is active (server)
 20\. bck [..bs]: number of backup servers (backend), server is backup (server)
 21\. chkfail [...s]: number of failed checks. (only counts checks failed when
     the server is up.)
 22\. chkdown [..bs]: number of up->down transitions. the backend counter counts
     transitions to the whole backend being down, rather than the sum of the
     counters for each server.
 23\. lastchg [..bs]: number of seconds since the last up<->down transition
 24\. downtime [..bs]: total downtime (in seconds). the value for the backend
     is the downtime for the whole backend, not the sum of the server downtime.
 25\. qlimit [...s]: configured maxqueue for the server, or nothing in the
     value is 0 (default, meaning no limit)
 26\. pid [lfbs]: process id (0 for first instance, 1 for second, ...)
 27\. iid [lfbs]: unique proxy id
 28\. sid [l..s]: server id (unique inside a proxy)
 29\. throttle [...s]: current throttle percentage for the server, when
     slowstart is active, or no value if not in slowstart.
 30\. lbtot [..bs]: total number of times a server was selected, either for new
     sessions, or when re-dispatching. the server counter is the number
     of times that server was selected.
 31\. tracked [...s]: id of proxy/server if tracking is enabled.
 32\. type [lfbs]: (0=frontend, 1=backend, 2=server, 3=socket/listener)
 33\. rate [.fbs]: number of sessions per second over last elapsed second
 34\. rate_lim [.f..]: configured limit on new sessions per second
 35\. rate_max [.fbs]: max number of new sessions per second
 36\. check_status [...s]: status of last health check, one of:
        unk     -> unknown
        ini     -> initializing
        sockerr -> socket error
        l4ok    -> check passed on layer 4, no upper layers testing enabled
        l4tout  -> layer 1-4 timeout
        l4con   -> layer 1-4 connection problem, for example
                   "connection refused" (tcp rst) or "no route to host" (icmp)
        l6ok    -> check passed on layer 6
        l6tout  -> layer 6 (ssl) timeout
        l6rsp   -> layer 6 invalid response - protocol error
        l7ok    -> check passed on layer 7
        l7okc   -> check conditionally passed on layer 7, for example 404 with
                   disable-on-404
        l7tout  -> layer 7 (http/smtp) timeout
        l7rsp   -> layer 7 invalid response - protocol error
        l7sts   -> layer 7 response error, for example http 5xx
 37\. check_code [...s]: layer5-7 code, if available
 38\. check_duration [...s]: time in ms took to finish last health check
 39\. hrsp_1xx [.fbs]: http responses with 1xx code
 40\. hrsp_2xx [.fbs]: http responses with 2xx code
 41\. hrsp_3xx [.fbs]: http responses with 3xx code
 42\. hrsp_4xx [.fbs]: http responses with 4xx code
 43\. hrsp_5xx [.fbs]: http responses with 5xx code
 44\. hrsp_other [.fbs]: http responses with other codes (protocol error)
 45\. hanafail [...s]: failed health checks details
 46\. req_rate [.f..]: http requests per second over last elapsed second
 47\. req_rate_max [.f..]: max number of http requests per second observed
 48\. req_tot [.f..]: total number of http requests received
 49\. cli_abrt [..bs]: number of data transfers aborted by the client
 50\. srv_abrt [..bs]: number of data transfers aborted by the server
     (inc. in eresp)
 51\. comp_in [.fb.]: number of http response bytes fed to the compressor
 52\. comp_out [.fb.]: number of http response bytes emitted by the compressor
 53\. comp_byp [.fb.]: number of bytes that bypassed the http compressor
     (cpu/bw limit)
 54\. comp_rsp [.fb.]: number of http responses that were compressed
 55\. lastsess [..bs]: number of seconds since last session assigned to
     server/backend
 56\. last_chk [...s]: last health check contents or textual error
 57\. last_agt [...s]: last agent check contents or textual error
 58\. qtime [..bs]: the average queue time in ms over the 1024 last requests
 59\. ctime [..bs]: the average connect time in ms over the 1024 last requests
 60\. rtime [..bs]: the average response time in ms over the 1024 last requests
     (0 for tcp)
 61\. ttime [..bs]: the average total session time in ms over the 1024 last
     requests

9.2\. unix socket commands
-------------------------

the stats socket is not enabled by default. in order to enable it, it is
necessary to add one line in the global section of the haproxy configuration.
a second line is recommended to set a larger timeout, always appreciated when
issuing commands by hand :

    global
        stats socket /var/run/haproxy.sock mode 600 level admin
        stats timeout 2m

it is also possible to add multiple instances of the stats socket by repeating
the line, and make them listen to a tcp port instead of a unix socket. this is
never done by default because this is dangerous, but can be handy in some
situations :

    global
        stats socket /var/run/haproxy.sock mode 600 level admin
        stats socket ipv4@192.168.0.1:9999 level admin
        stats timeout 2m

to access the socket, an external utility such as "socat" is required. socat is a
swiss-army knife to connect anything to anything. we use it to connect terminals
to the socket, or a couple of stdin/stdout pipes to it for scripts. the two main
syntaxes we'll use are the following :

    # socat /var/run/haproxy.sock stdio
    # socat /var/run/haproxy.sock readline

the first one is used with scripts. it is possible to send the output of a
script to haproxy, and pass haproxy's output to another script. that's useful
for retrieving counters or attack traces for example.

the second one is only useful for issuing commands by hand. it has the benefit
that the terminal is handled by the readline library which supports line
editing and history, which is very convenient when issuing repeated commands
(eg: watch a counter).

the socket supports two operation modes :
  - interactive
  - non-interactive

the non-interactive mode is the default when socat connects to the socket. in
this mode, a single line may be sent. it is processed as a whole, responses are
sent back, and the connection closes after the end of the response. this is the
mode that scripts and monitoring tools use. it is possible to send multiple
commands in this mode, they need to be delimited by a semi-colon (';'). for
example :

    # echo "show info;show stat;show table" | socat /var/run/haproxy stdio

the interactive mode displays a prompt ('>') and waits for commands to be
entered on the line, then processes them, and displays the prompt again to wait
for a new command. this mode is entered via the "prompt" command which must be
sent on the first line in non-interactive mode. the mode is a flip switch, if
"prompt" is sent in interactive mode, it is disabled and the connection closes
after processing the last command of the same line.

for this reason, when debugging by hand, it's quite common to start with the
"prompt" command :

   # socat /var/run/haproxy readline
   prompt
   > show info
   ...
   >

since multiple commands may be issued at once, haproxy uses the empty line as a
delimiter to mark an end of output for each command, and takes care of ensuring
that no command can emit an empty line on output. a script can thus easily
parse the output even when multiple commands were pipelined on a single line.

it is important to understand that when multiple haproxy processes are started
on the same sockets, any process may pick up the request and will output its
own stats.

the list of commands currently supported on the stats socket is provided below.
if an unknown command is sent, haproxy displays the usage message which reminds
all supported commands. some commands support a more complex syntax, generally
it will explain what part of the command is invalid when this happens.

add acl <acl> <pattern>
  add an entry into the acl <acl>. <acl> is the #<id> or the <file> returned by
  "show acl". this command does not verify if the entry already exists. this
  command cannot be used if the reference <acl> is a file also used with a map.
  in this case, you must use the command "add map" in place of "add acl".

add map <map> <key> <value>
  add an entry into the map <map> to associate the value <value> to the key
  <key>. this command does not verify if the entry already exists. it is
  mainly used to fill a map after a clear operation. note that if the reference
  <map> is a file and is shared with a map, this map will contain also a new
  pattern entry.

clear counters
  clear the max values of the statistics counters in each proxy (frontend &
  backend) and in each server. the cumulated counters are not affected. this
  can be used to get clean counters after an incident, without having to
  restart nor to clear traffic counters. this command is restricted and can
  only be issued on sockets configured for levels "operator" or "admin".

clear counters all
  clear all statistics counters in each proxy (frontend & backend) and in each
  server. this has the same effect as restarting. this command is restricted
  and can only be issued on sockets configured for level "admin".

clear acl <acl>
  remove all entries from the acl <acl>. <acl> is the #<id> or the <file>
  returned by "show acl". note that if the reference <acl> is a file and is
  shared with a map, this map will be also cleared.

clear map <map>
  remove all entries from the map <map>. <map> is the #<id> or the <file>
  returned by "show map". note that if the reference <map> is a file and is
  shared with a acl, this acl will be also cleared.

clear table <table> [ data.<type> <operator> <value> ] | [ key <key> ]
  remove entries from the stick-table <table>.

  this is typically used to unblock some users complaining they have been
  abusively denied access to a service, but this can also be used to clear some
  stickiness entries matching a server that is going to be replaced (see "show
  table" below for details).  note that sometimes, removal of an entry will be
  refused because it is currently tracked by a session. retrying a few seconds
  later after the session ends is usual enough.

  in the case where no options arguments are given all entries will be removed.

  when the "data." form is used entries matching a filter applied using the
  stored data (see "stick-table" in section 4.2) are removed.  a stored data
  type must be specified in <type>, and this data type must be stored in the
  table otherwise an error is reported. the data is compared according to
  <operator> with the 64-bit integer <value>.  operators are the same as with
  the acls :

    - eq : match entries whose data is equal to this value
    - ne : match entries whose data is not equal to this value
    - le : match entries whose data is less than or equal to this value
    - ge : match entries whose data is greater than or equal to this value
    - lt : match entries whose data is less than this value
    - gt : match entries whose data is greater than this value

  when the key form is used the entry <key> is removed.  the key must be of the
  same type as the table, which currently is limited to ipv4, ipv6, integer and
  string.

  example :
        $ echo "show table http_proxy" | socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:2
    >>> 0x80e6a4c: key=127.0.0.1 use=0 exp=3594729 gpc0=0 conn_rate(30000)=1 \
          bytes_out_rate(60000)=187
    >>> 0x80e6a80: key=127.0.0.2 use=0 exp=3594740 gpc0=1 conn_rate(30000)=10 \
          bytes_out_rate(60000)=191

        $ echo "clear table http_proxy key 127.0.0.1" | socat stdio /tmp/sock1

        $ echo "show table http_proxy" | socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:1
    >>> 0x80e6a80: key=127.0.0.2 use=0 exp=3594740 gpc0=1 conn_rate(30000)=10 \
          bytes_out_rate(60000)=191
        $ echo "clear table http_proxy data.gpc0 eq 1" | socat stdio /tmp/sock1
        $ echo "show table http_proxy" | socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:1

del acl <acl> [<key>|#<ref>]
  delete all the acl entries from the acl <acl> corresponding to the key <key>.
  <acl> is the #<id> or the <file> returned by "show acl". if the <ref> is used,
  this command delete only the listed reference. the reference can be found with
  listing the content of the acl. note that if the reference <acl> is a file and
  is shared with a map, the entry will be also deleted in the map.

del map <map> [<key>|#<ref>]
  delete all the map entries from the map <map> corresponding to the key <key>.
  <map> is the #<id> or the <file> returned by "show map". if the <ref> is used,
  this command delete only the listed reference. the reference can be found with
  listing the content of the map. note that if the reference <map> is a file and
  is shared with a acl, the entry will be also deleted in the map.

disable agent <backend>/<server>
  mark the auxiliary agent check as temporarily stopped.

  in the case where an agent check is being run as a auxiliary check, due
  to the agent-check parameter of a server directive, new checks are only
  initialised when the agent is in the enabled. thus, disable agent will
  prevent any new agent checks from begin initiated until the agent
  re-enabled using enable agent.

  when an agent is disabled the processing of an auxiliary agent check that
  was initiated while the agent was set as enabled is as follows: all
  results that would alter the weight, specifically "drain" or a weight
  returned by the agent, are ignored. the processing of agent check is
  otherwise unchanged.

  the motivation for this feature is to allow the weight changing effects
  of the agent checks to be paused to allow the weight of a server to be
  configured using set weight without being overridden by the agent.

  this command is restricted and can only be issued on sockets configured for
  level "admin".

disable frontend <frontend>
  mark the frontend as temporarily stopped. this corresponds to the mode which
  is used during a soft restart : the frontend releases the port but can be
  enabled again if needed. this should be used with care as some non-linux oses
  are unable to enable it back. this is intended to be used in environments
  where stopping a proxy is not even imaginable but a misconfigured proxy must
  be fixed. that way it's possible to release the port and bind it into another
  process to restore operations. the frontend will appear with status "stop"
  on the stats page.

  the frontend may be specified either by its name or by its numeric id,
  prefixed with a sharp ('#').

  this command is restricted and can only be issued on sockets configured for
  level "admin".

disable health <backend>/<server>
  mark the primary health check as temporarily stopped. this will disable
  sending of health checks, and the last health check result will be ignored.
  the server will be in unchecked state and considered up unless an auxiliary
  agent check forces it down.

  this command is restricted and can only be issued on sockets configured for
  level "admin".

disable server <backend>/<server>
  mark the server down for maintenance. in this mode, no more checks will be
  performed on the server until it leaves maintenance.
  if the server is tracked by other servers, those servers will be set to down
  during the maintenance.

  in the statistics page, a server down for maintenance will appear with a
  "maint" status, its tracking servers with the "maint(via)" one.

  both the backend and the server may be specified either by their name or by
  their numeric id, prefixed with a sharp ('#').

  this command is restricted and can only be issued on sockets configured for
  level "admin".

enable agent <backend>/<server>
  resume auxiliary agent check that was temporarily stopped.

  see "disable agent" for details of the effect of temporarily starting
  and stopping an auxiliary agent.

  this command is restricted and can only be issued on sockets configured for
  level "admin".

enable frontend <frontend>
  resume a frontend which was temporarily stopped. it is possible that some of
  the listening ports won't be able to bind anymore (eg: if another process
  took them since the 'disable frontend' operation). if this happens, an error
  is displayed. some operating systems might not be able to resume a frontend
  which was disabled.

  the frontend may be specified either by its name or by its numeric id,
  prefixed with a sharp ('#').

  this command is restricted and can only be issued on sockets configured for
  level "admin".

enable health <backend>/<server>
  resume a primary health check that was temporarily stopped. this will enable
  sending of health checks again. please see "disable health" for details.

  this command is restricted and can only be issued on sockets configured for
  level "admin".

enable server <backend>/<server>
  if the server was previously marked as down for maintenance, this marks the
  server up and checks are re-enabled.

  both the backend and the server may be specified either by their name or by
  their numeric id, prefixed with a sharp ('#').

  this command is restricted and can only be issued on sockets configured for
  level "admin".

get map <map> <value>
get acl <acl> <value>
  lookup the value <value> in the map <map> or in the acl <acl>. <map> or <acl>
  are the #<id> or the <file> returned by "show map" or "show acl". this command
  returns all the matching patterns associated with this map. this is useful for
  debugging maps and acls. the output format is composed by one line par
  matching type. each line is composed by space-delimited series of words.

  the first two words are:

     <match method>:   the match method applied. it can be "found", "bool",
                       "int", "ip", "bin", "len", "str", "beg", "sub", "dir",
                       "dom", "end" or "reg".

     <match result>:   the result. can be "match" or "no-match".

  the following words are returned only if the pattern matches an entry.

     <index type>:     "tree" or "list". the internal lookup algorithm.

     <case>:           "case-insensitive" or "case-sensitive". the
                       interpretation of the case.

     <entry matched>:  match="<entry>". return the matched pattern. it is
                       useful with regular expressions.

  the two last word are used to show the returned value and its type. with the
  "acl" case, the pattern doesn't exist.

     return=nothing:        no return because there are no "map".
     return="<value>":      the value returned in the string format.
     return=cannot-display: the value cannot be converted as string.

     type="<type>":         the type of the returned sample.

get weight <backend>/<server>
  report the current weight and the initial weight of server <server> in
  backend <backend> or an error if either doesn't exist. the initial weight is
  the one that appears in the configuration file. both are normally equal
  unless the current weight has been changed. both the backend and the server
  may be specified either by their name or by their numeric id, prefixed with a
  sharp ('#').

help
  print the list of known keywords and their basic usage. the same help screen
  is also displayed for unknown commands.

prompt
  toggle the prompt at the beginning of the line and enter or leave interactive
  mode. in interactive mode, the connection is not closed after a command
  completes. instead, the prompt will appear again, indicating the user that
  the interpreter is waiting for a new command. the prompt consists in a right
  angle bracket followed by a space "> ". this mode is particularly convenient
  when one wants to periodically check information such as stats or errors.
  it is also a good idea to enter interactive mode before issuing a "help"
  command.

quit
  close the connection when in interactive mode.

set map <map> [<key>|#<ref>] <value>
  modify the value corresponding to each key <key> in a map <map>. <map> is the
  #<id> or <file> returned by "show map". if the <ref> is used in place of
  <key>, only the entry pointed by <ref> is changed. the new value is <value>.

set maxconn frontend <frontend> <value>
  dynamically change the specified frontend's maxconn setting. any positive
  value is allowed including zero, but setting values larger than the global
  maxconn does not make much sense. if the limit is increased and connections
  were pending, they will immediately be accepted. if it is lowered to a value
  below the current number of connections, new connections acceptation will be
  delayed until the threshold is reached. the frontend might be specified by
  either its name or its numeric id prefixed with a sharp ('#').

set maxconn global <maxconn>
  dynamically change the global maxconn setting within the range defined by the
  initial global maxconn setting. if it is increased and connections were
  pending, they will immediately be accepted. if it is lowered to a value below
  the current number of connections, new connections acceptation will be
  delayed until the threshold is reached. a value of zero restores the initial
  setting.

set rate-limit connections global <value>
  change the process-wide connection rate limit, which is set by the global
  'maxconnrate' setting. a value of zero disables the limitation. this limit
  applies to all frontends and the change has an immediate effect. the value
  is passed in number of connections per second.

set rate-limit http-compression global <value>
  change the maximum input compression rate, which is set by the global
  'maxcomprate' setting. a value of zero disables the limitation. the value is
  passed in number of kilobytes per second. the value is available in the "show
  info" on the line "compressbpsratelim" in bytes.

set rate-limit sessions global <value>
  change the process-wide session rate limit, which is set by the global
  'maxsessrate' setting. a value of zero disables the limitation. this limit
  applies to all frontends and the change has an immediate effect. the value
  is passed in number of sessions per second.

set rate-limit ssl-sessions global <value>
  change the process-wide ssl session rate limit, which is set by the global
  'maxsslrate' setting. a value of zero disables the limitation. this limit
  applies to all frontends and the change has an immediate effect. the value
  is passed in number of sessions per second sent to the ssl stack. it applies
  before the handshake in order to protect the stack against handshake abuses.

set server <backend>/<server> addr <ip4 or ip6 address>
  replace the current ip address of a server by the one provided.

set server <backend>/<server> agent [ up | down ]
  force a server's agent to a new state. this can be useful to immediately
  switch a server's state regardless of some slow agent checks for example.
  note that the change is propagated to tracking servers if any.

set server <backend>/<server> health [ up | stopping | down ]
  force a server's health to a new state. this can be useful to immediately
  switch a server's state regardless of some slow health checks for example.
  note that the change is propagated to tracking servers if any.

set server <backend>/<server> state [ ready | drain | maint ]
  force a server's administrative state to a new state. this can be useful to
  disable load balancing and/or any traffic to a server. setting the state to
  "ready" puts the server in normal mode, and the command is the equivalent of
  the "enable server" command. setting the state to "maint" disables any traffic
  to the server as well as any health checks. this is the equivalent of the
  "disable server" command. setting the mode to "drain" only removes the server
  from load balancing but still allows it to be checked and to accept new
  persistent connections. changes are propagated to tracking servers if any.

set server <backend>/<server> weight <weight>[%]
  change a server's weight to the value passed in argument. this is the exact
  equivalent of the "set weight" command below.

set ssl ocsp-response <response>
  this command is used to update an ocsp response for a certificate (see "crt"
  on "bind" lines). same controls are performed as during the initial loading of
  the response. the <response> must be passed as a base64 encoded string of the
  der encoded response from the ocsp server.

  example:
    openssl ocsp -issuer issuer.pem -cert server.pem \
                 -host ocsp.issuer.com:80 -respout resp.der
    echo "set ssl ocsp-response $(base64 -w 10000 resp.der)" | \
                 socat stdio /var/run/haproxy.stat

set ssl tls-key <id> <tlskey>
  set the next tls key for the <id> listener to <tlskey>. this key becomes the
  ultimate key, while the penultimate one is used for encryption (others just
  decrypt). the oldest tls key present is overwritten. <id> is either a numeric
  #<id> or <file> returned by "show tls-keys". <tlskey> is a base64 encoded 48
  bit tls ticket key (ex. openssl rand -base64 48).

set table <table> key <key> [data.<data_type> <value>]*
  create or update a stick-table entry in the table. if the key is not present,
  an entry is inserted. see stick-table in section 4.2 to find all possible
  values for <data_type>. the most likely use consists in dynamically entering
  entries for source ip addresses, with a flag in gpc0 to dynamically block an
  ip address or affect its quality of service. it is possible to pass multiple
  data_types in a single call.

set timeout cli <delay>
  change the cli interface timeout for current connection. this can be useful
  during long debugging sessions where the user needs to constantly inspect
  some indicators without being disconnected. the delay is passed in seconds.

set weight <backend>/<server> <weight>[%]
  change a server's weight to the value passed in argument. if the value ends
  with the '%' sign, then the new weight will be relative to the initially
  configured weight.  absolute weights are permitted between 0 and 256.
  relative weights must be positive with the resulting absolute weight is
  capped at 256\.  servers which are part of a farm running a static
  load-balancing algorithm have stricter limitations because the weight
  cannot change once set. thus for these servers, the only accepted values
  are 0 and 100% (or 0 and the initial weight). changes take effect
  immediately, though certain lb algorithms require a certain amount of
  requests to consider changes. a typical usage of this command is to
  disable a server during an update by setting its weight to zero, then to
  enable it again after the update by setting it back to 100%. this command
  is restricted and can only be issued on sockets configured for level
  "admin". both the backend and the server may be specified either by their
  name or by their numeric id, prefixed with a sharp ('#').

show errors [<iid>]
  dump last known request and response errors collected by frontends and
  backends. if <iid> is specified, the limit the dump to errors concerning
  either frontend or backend whose id is <iid>. this command is restricted
  and can only be issued on sockets configured for levels "operator" or
  "admin".

  the errors which may be collected are the last request and response errors
  caused by protocol violations, often due to invalid characters in header
  names. the report precisely indicates what exact character violated the
  protocol. other important information such as the exact date the error was
  detected, frontend and backend names, the server name (when known), the
  internal session id and the source address which has initiated the session
  are reported too.

  all characters are returned, and non-printable characters are encoded. the
  most common ones (\t = 9, \n = 10, \r = 13 and \e = 27) are encoded as one
  letter following a backslash. the backslash itself is encoded as '\\' to
  avoid confusion. other non-printable characters are encoded '\xnn' where
  nn is the two-digits hexadecimal representation of the character's ascii
  code.

  lines are prefixed with the position of their first character, starting at 0
  for the beginning of the buffer. at most one input line is printed per line,
  and large lines will be broken into multiple consecutive output lines so that
  the output never goes beyond 79 characters wide. it is easy to detect if a
  line was broken, because it will not end with '\n' and the next line's offset
  will be followed by a '+' sign, indicating it is a continuation of previous
  line.

  example :
        $ echo "show errors" | socat stdio /tmp/sock1
    >>> [04/mar/2009:15:46:56.081] backend http-in (#2) : invalid response
          src 127.0.0.1, session #54, frontend fe-eth0 (#1), server s2 (#1)
          response length 213 bytes, error at position 23:

          00000  http/1.0 200 ok\r\n
          00017  header/bizarre:blah\r\n
          00038  location: blah\r\n
          00054  long-line: this is a very long line which should b
          00104+ e broken into multiple lines on the output buffer,
          00154+  otherwise it would be too large to print in a ter
          00204+ minal\r\n
          00211  \r\n

    in the example above, we see that the backend "http-in" which has internal
    id 2 has blocked an invalid response from its server s2 which has internal
    id 1\. the request was on session 54 initiated by source 127.0.0.1 and
    received by frontend fe-eth0 whose id is 1\. the total response length was
    213 bytes when the error was detected, and the error was at byte 23\. this
    is the slash ('/') in header name "header/bizarre", which is not a valid
    http character for a header name.

show info
  dump info about haproxy status on current process.

show map [<map>]
  dump info about map converters. without argument, the list of all available
  maps is returned. if a <map> is specified, its contents are dumped. <map> is
  the #<id> or <file>. the first column is a unique identifier. it can be used
  as reference for the operation "del map" and "set map". the second column is
  the pattern and the third column is the sample if available. the data returned
  are not directly a list of available maps, but are the list of all patterns
  composing any map. many of these patterns can be shared with acl.

show acl [<acl>]
  dump info about acl converters. without argument, the list of all available
  acls is returned. if a <acl> is specified, its contents are dumped. <acl> if
  the #<id> or <file>. the dump format is the same than the map even for the
  sample value. the data returned are not a list of available acl, but are the
  list of all patterns composing any acl. many of these patterns can be shared
  with maps.

show pools
  dump the status of internal memory pools. this is useful to track memory
  usage when suspecting a memory leak for example. it does exactly the same
  as the sigquit when running in foreground except that it does not flush
  the pools.

show sess
  dump all known sessions. avoid doing this on slow connections as this can
  be huge. this command is restricted and can only be issued on sockets
  configured for levels "operator" or "admin".

show sess <id>
  display a lot of internal information about the specified session identifier.
  this identifier is the first field at the beginning of the lines in the dumps
  of "show sess" (it corresponds to the session pointer). those information are
  useless to most users but may be used by haproxy developers to troubleshoot a
  complex bug. the output format is intentionally not documented so that it can
  freely evolve depending on demands. you may find a description of all fields
  returned in src/dumpstats.c

  the special id "all" dumps the states of all sessions, which must be avoided
  as much as possible as it is highly cpu intensive and can take a lot of time.

show stat [<iid> <type> <sid>]
  dump statistics in the csv format. by passing <id>, <type> and <sid>, it is
  possible to dump only selected items :
    - <iid> is a proxy id, -1 to dump everything
    - <type> selects the type of dumpable objects : 1 for frontends, 2 for
       backends, 4 for servers, -1 for everything. these values can be ored,
       for example:
          1 + 2     = 3   -> frontend + backend.
          1 + 2 + 4 = 7   -> frontend + backend + server.
    - <sid> is a server id, -1 to dump everything from the selected proxy.

  example :
        $ echo "show info;show stat" | socat stdio unix-connect:/tmp/sock1
    >>> name: haproxy
        version: 1.4-dev2-49
        release_date: 2009/09/23
        nbproc: 1
        process_num: 1
        (...)

        # pxname,svname,qcur,qmax,scur,smax,slim,stot,bin,bout,dreq,  (...)
        stats,frontend,,,0,0,1000,0,0,0,0,0,0,,,,,open,,,,,,,,,1,1,0, (...)
        stats,backend,0,0,0,0,1000,0,0,0,0,0,,0,0,0,0,up,0,0,0,,0,250,(...)
        (...)
        www1,backend,0,0,0,0,1000,0,0,0,0,0,,0,0,0,0,up,1,1,0,,0,250, (...)

        $

    here, two commands have been issued at once. that way it's easy to find
    which process the stats apply to in multi-process mode. notice the empty
    line after the information output which marks the end of the first block.
    a similar empty line appears at the end of the second block (stats) so that
    the reader knows the output has not been truncated.

show stat resolvers <resolvers section id>
  dump statistics for the given resolvers section.
  for each name server, the following counters are reported:
    sent: number of dns requests sent to this server
    valid: number of dns valid responses received from this server
    update: number of dns responses used to update the server's ip address
    cname: number of cname responses
    cname_error: cname errors encountered with this server
    any_err: number of empty response (ie: server does not support any type)
    nx: non existent domain response received from this server
    timeout: how many time this server did not answer in time
    refused: number of requests refused by this server
    other: any other dns errors
    invalid: invalid dns response (from a protocol point of view)
    too_big: too big response
    outdated: number of response arrived too late (after an other name server)
show table
  dump general information on all known stick-tables. their name is returned
  (the name of the proxy which holds them), their type (currently zero, always
  ip), their size in maximum possible number of entries, and the number of
  entries currently in use.

  example :
        $ echo "show table" | socat stdio /tmp/sock1
    >>> # table: front_pub, type: ip, size:204800, used:171454
    >>> # table: back_rdp, type: ip, size:204800, used:0

show table <name> [ data.<type> <operator> <value> ] | [ key <key> ]
  dump contents of stick-table <name>. in this mode, a first line of generic
  information about the table is reported as with "show table", then all
  entries are dumped. since this can be quite heavy, it is possible to specify
  a filter in order to specify what entries to display.

  when the "data." form is used the filter applies to the stored data (see
  "stick-table" in section 4.2).  a stored data type must be specified
  in <type>, and this data type must be stored in the table otherwise an
  error is reported. the data is compared according to <operator> with the
  64-bit integer <value>.  operators are the same as with the acls :

    - eq : match entries whose data is equal to this value
    - ne : match entries whose data is not equal to this value
    - le : match entries whose data is less than or equal to this value
    - ge : match entries whose data is greater than or equal to this value
    - lt : match entries whose data is less than this value
    - gt : match entries whose data is greater than this value

  when the key form is used the entry <key> is shown.  the key must be of the
  same type as the table, which currently is limited to ipv4, ipv6, integer,
  and string.

  example :
        $ echo "show table http_proxy" | socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:2
    >>> 0x80e6a4c: key=127.0.0.1 use=0 exp=3594729 gpc0=0 conn_rate(30000)=1  \
          bytes_out_rate(60000)=187
    >>> 0x80e6a80: key=127.0.0.2 use=0 exp=3594740 gpc0=1 conn_rate(30000)=10 \
          bytes_out_rate(60000)=191

        $ echo "show table http_proxy data.gpc0 gt 0" | socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:2
    >>> 0x80e6a80: key=127.0.0.2 use=0 exp=3594740 gpc0=1 conn_rate(30000)=10 \
          bytes_out_rate(60000)=191

        $ echo "show table http_proxy data.conn_rate gt 5" | \
            socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:2
    >>> 0x80e6a80: key=127.0.0.2 use=0 exp=3594740 gpc0=1 conn_rate(30000)=10 \
          bytes_out_rate(60000)=191

        $ echo "show table http_proxy key 127.0.0.2" | \
            socat stdio /tmp/sock1
    >>> # table: http_proxy, type: ip, size:204800, used:2
    >>> 0x80e6a80: key=127.0.0.2 use=0 exp=3594740 gpc0=1 conn_rate(30000)=10 \
          bytes_out_rate(60000)=191

  when the data criterion applies to a dynamic value dependent on time such as
  a bytes rate, the value is dynamically computed during the evaluation of the
  entry in order to decide whether it has to be dumped or not. this means that
  such a filter could match for some time then not match anymore because as
  time goes, the average event rate drops.

  it is possible to use this to extract lists of ip addresses abusing the
  service, in order to monitor them or even blacklist them in a firewall.
  example :
        $ echo "show table http_proxy data.gpc0 gt 0" \
          | socat stdio /tmp/sock1 \
          | fgrep 'key=' | cut -d' ' -f2 | cut -d= -f2 > abusers-ip.txt
          ( or | awk '/key/{ print a[split($2,a,"=")]; }' )

show tls-keys
  dump all loaded tls ticket keys. the tls ticket key reference id and the
  file from which the keys have been loaded is shown. both of those can be
  used to update the tls keys using "set ssl tls-key".

shutdown frontend <frontend>
  completely delete the specified frontend. all the ports it was bound to will
  be released. it will not be possible to enable the frontend anymore after
  this operation. this is intended to be used in environments where stopping a
  proxy is not even imaginable but a misconfigured proxy must be fixed. that
  way it's possible to release the port and bind it into another process to
  restore operations. the frontend will not appear at all on the stats page
  once it is terminated.

  the frontend may be specified either by its name or by its numeric id,
  prefixed with a sharp ('#').

  this command is restricted and can only be issued on sockets configured for
  level "admin".

shutdown session <id>
  immediately terminate the session matching the specified session identifier.
  this identifier is the first field at the beginning of the lines in the dumps
  of "show sess" (it corresponds to the session pointer). this can be used to
  terminate a long-running session without waiting for a timeout or when an
  endless transfer is ongoing. such terminated sessions are reported with a 'k'
  flag in the logs.

shutdown sessions server <backend>/<server>
  immediately terminate all the sessions attached to the specified server. this
  can be used to terminate long-running sessions after a server is put into
  maintenance mode, for instance. such terminated sessions are reported with a
  'k' flag in the logs.

/*
 * local variables:
 *  fill-column: 79
 * end:
 */