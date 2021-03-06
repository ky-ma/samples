## Multi process solutions

Crystal could not use multiple cores. All calculations running on same core. To effectively use all cores for network application there are multiple solutions:
1. Create a master-workers approach, where one process listen for incoming requests and then send requests to specific worker to process it.
2. Allow listen for same port to different processes. There is feature calls (reuse port)[https://lwn.net/Articles/542629/].

## Example

Try to run multiple application, that bind to same port:

```shell
$ lsof -nP -iTCP:3000 | grep LISTEN
$ crystal experiments/2-tcp-server-port-reuse/server.cr # Run first process
Listen 3000
$ lsof -nP -iTCP:3000 | grep LISTEN
crystal-r 77323 miry   11u  IPv6 0x5ba3efab06a43411      0t0  TCP [::1]:3000 (LISTEN)
$ crystal experiments/2-tcp-server-port-reuse/server.cr # Run second process
Listen 3000
$ lsof -nP -iTCP:3000 | grep LISTEN
crystal-r 77323 miry   11u  IPv6 0x5ba3efab06a43411      0t0  TCP [::1]:3000 (LISTEN)
crystal-r 77665 miry   11u  IPv6 0x5ba3efab06a45c51      0t0  TCP [fe80:1::1]:3000 (LISTEN)
$ crystal experiments/2-tcp-server-port-reuse/server.cr # Run third process
Listen 3000
$ lsof -nP -iTCP:3000 | grep LISTEN
crystal-r 77323 miry   11u  IPv6 0x5ba3efab06a43411      0t0  TCP [::1]:3000 (LISTEN)
crystal-r 77665 miry   11u  IPv6 0x5ba3efab06a45c51      0t0  TCP [fe80:1::1]:3000 (LISTEN)
crystal-r 77911 miry   11u  IPv4 0x5ba3efab11f0a859      0t0  TCP 127.0.0.1:3000 (LISTEN)
$ crystal experiments/2-tcp-server-port-reuse/server.cr # Run forth process
Unhandled exception: bind: Address already in use (Errno)
  from /usr/local/Cellar/crystal/0.27.0/src/socket/tcp_server.cr:73:15 in 'initialize'
  from /usr/local/Cellar/crystal/0.27.0/src/socket/tcp_server.cr:32:3 in 'new'
  from experiments/2-tcp-server-port-reuse/server.cr:15:10 in '__crystal_main'
  from /usr/local/Cellar/crystal/0.27.0/src/crystal/main.cr:97:5 in 'main_user_code'
  from /usr/local/Cellar/crystal/0.27.0/src/crystal/main.cr:86:7 in 'main'
  from /usr/local/Cellar/crystal/0.27.0/src/crystal/main.cr:106:3 in 'main'
```

This behavior I have not expected.
TCP server looks for all interfaces,
that related to `localhost` and tries to bind to port.

```shell
$ nc localhost 3000
hello
hello
$ nc 127.0.0.1 3000
world
world
$ nc fe80:1::1 3000
!
!
```

----

cat <<EOF | crystal eval --no-debug
require "socket"
s = TCPServer.new("localhost", 3000)
puts "Listen on #{s.local_address}"
while c = s.accept?
   c << "busy\n"
   c.close()
end
EOF


How resolve is working?

http://man7.org/linux/man-pages/man3/getaddrinfo.3.html

Quote from man:

       The getaddrinfo() function allocates and initializes a linked list of
       addrinfo structures, one for each network address that matches node
       and service, subject to any restrictions imposed by hints, and
       returns a pointer to the start of the list in res.  The items in the
       linked list are linked by the ai_next field.

       There are several reasons why the linked list may have more than one
       addrinfo structure, including: the network host is multihomed, acces‐
       sible over multiple protocols (e.g., both AF_INET and AF_INET6); or
       the same service is available from multiple socket types (one
       SOCK_STREAM address and another SOCK_DGRAM address, for example).
       Normally, the application should try using the addresses in the order
       in which they are returned.  The sorting function used within getad‐
       drinfo() is defined in RFC 3484; the order can be tweaked for a par‐
       ticular system by editing /etc/gai.conf (available since glibc 2.5).

----

How to use **SO_REUSEPORT** with TCPServer.

```shell
$ crystal experiments/2-tcp-server-port-reuse/server_reuse_port.cr
Listen 3000
$ lsof -nP -iTCP:3000 | grep LISTEN
crystal-r 78722 miry   11u  IPv6 0x5ba3efab103579d1      0t0  TCP [::1]:3000 (LISTEN)
$ crystal experiments/2-tcp-server-port-reuse/server_reuse_port.cr
Listen 3000
$ lsof -nP -iTCP:3000 | grep LISTEN
crystal-r 78722 miry   11u  IPv6 0x5ba3efab103579d1      0t0  TCP [::1]:3000 (LISTEN)
crystal-r 78796 miry   11u  IPv6 0x5ba3efab10358551      0t0  TCP [::1]:3000 (LISTEN)
```

```shell
$ nc localhost 3000
hello
hello
$ nc localhost 3000
world
world
```


Experiment:

```
$ crystal experiments/2-tcp-server-port-reuse/server_reuse_port.cr
Listen 3000
[2018-11-28 20:18:58 +01:00] [server] Client is connected: #<TCPSocket:0x1058d4a50>
[2018-11-28 20:19:03 +01:00] [#<TCPSocket:0x1058d4a50>] Message: first client
[2018-11-28 20:20:18 +01:00] [#<TCPSocket:0x1058d4a50>] Message: third message

$ nc localhost 3000
first client
first client
third message
third message

$ crystal experiments/2-tcp-server-port-reuse/server_reuse_port.cr
Listen 3000
[2018-11-28 20:19:14 +01:00] [server] Client is connected: #<TCPSocket:0x105cdba50>
[2018-11-28 20:19:19 +01:00] [#<TCPSocket:0x105cdba50>] Message: second client
[2018-11-28 20:20:22 +01:00] [#<TCPSocket:0x105cdba50>] Message: forth message

$ nc localhost 3000
second client
second client
forth message
forth message

$ % lsof -nP -iTCP:3000
COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
crystal-r 84882 miry   11u  IPv6 0x5ba3efab06a43f91      0t0  TCP [::1]:3000 (LISTEN)
crystal-r 84882 miry   12u  IPv6 0x5ba3efab06a42e51      0t0  TCP [::1]:3000->[::1]:61993 (ESTABLISHED)
nc        84884 miry    5u  IPv6 0x5ba3efab06a44551      0t0  TCP [::1]:61993->[::1]:3000 (ESTABLISHED)
crystal-r 84906 miry   11u  IPv6 0x5ba3efab06a45691      0t0  TCP [::1]:3000 (LISTEN)
crystal-r 84906 miry   12u  IPv6 0x5ba3efab06a43411      0t0  TCP [::1]:3000->[::1]:61994 (ESTABLISHED)
nc        84908 miry    5u  IPv6 0x5ba3efab06a46211      0t0  TCP [::1]:61994->[::1]:3000 (ESTABLISHED)
```


Linux:

```
-> % docker run --rm -it -p 3000:3000 miry/experiment:2-tcp-server-port-reuse bash
root@60e98e9de71f:/app# ./server A &
[2018-11-29 21:00:21 UTC] [A] Listen 3000
root@60e98e9de71f:/app#
root@60e98e9de71f:/app# ./server B
[2018-11-29 21:00:30 UTC] [B] Listen 3000
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c59a50>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c59960>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c59870>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c59780>
[2018-11-29 21:00:48 UTC] [A] Client is connected: #<TCPSocket:0x7fe4574e2a50>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c59690>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c59a50>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c595a0>
[2018-11-29 21:00:48 UTC] [A] Closing client's connection: #<TCPSocket:0x7fe4574e2a50>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c59780>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c59870>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c59960>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c59690>
[2018-11-29 21:00:48 UTC] [A] Client is connected: #<TCPSocket:0x7fe4574e2960>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c594b0>
[2018-11-29 21:00:48 UTC] [B] Client is connected: #<TCPSocket:0x7fb762c593c0>
[2018-11-29 21:00:48 UTC] [A] Closing client's connection: #<TCPSocket:0x7fe4574e2960>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c593c0>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c595a0>
[2018-11-29 21:00:48 UTC] [B] Closing client's connection: #<TCPSocket:0x7fb762c594b0>
```

```
% crystal client.cr "127.0.0.1"
[2018-11-29 22:00:48 +01:00] [1] Welcome in B
[2018-11-29 22:00:48 +01:00] [5] Welcome in A
[2018-11-29 22:00:48 +01:00] [2] Welcome in B
[2018-11-29 22:00:48 +01:00] [4] Welcome in B
[2018-11-29 22:00:48 +01:00] [0] Welcome in B
[2018-11-29 22:00:48 +01:00] [3] Welcome in B
[2018-11-29 22:00:48 +01:00] [7] Welcome in A
[2018-11-29 22:00:48 +01:00] [6] Welcome in B
[2018-11-29 22:00:48 +01:00] [9] Welcome in B
[2018-11-29 22:00:48 +01:00] [8] Welcome in B
```
