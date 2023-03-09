# Nginx Reverse Proxy Analysis 

A small research project aimed at understanding the behaviour of a simple nginx reverse proxy given various upstream server conditions. Specifically the relationship with a "standard" Nginx proxy and the [Socket backlog queue](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/023/2333/2333s2.html) also known as the "Accept Queue".

 - nginx/1.22.1
 - Linux 6.1.12
 - Linux 6.2.1
 - Linux 6.2.2

# Findings

Observing lines in `/proc/net/unix` with state `03` `SS_CONNECTING` is an accurate indicator of connections currently in the backlog queue between the proxy server and the upstream server.

The **Active Connections** metric from the [nginx stub status module](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html) is an accurate indicator of which client connections are currently in the accept queue plus the number of accepted requests plus 1.


The [worker_connections](http://nginx.org/en/docs/ngx_core_module.html#worker_connections)  nginx parameter will accurately limit the **Active Connections** metric captured by the nginx stub status module.


---

Q = _Inbound proxy requests in the accept queue ≤ SOMAXCONN_

A = _Inbound proxy requests which have been accepted by an nginx worker_

**Active Connections** = Q (Accept Queue) + A (Accepted) + 1

```
                   ┌──────────────────────────┐
        ┌──────────┤ Accept Queue ≤ SOMAXCONN │ ◄─────────────────────┐
        │          └──────────────────────────┘                       │
        │                       Q                                     │
        │                                                             │
        ▼           ┌────────────────────────┐              ┌─────────┴──────────┐
                    │                        │              │ Active Connections │
    listen(); ────► │  Nginx Master Process  │              └─────────┬──────────┘
                    │                        │                        │
                    └─┬──────────────────────┘                        │
                      │                                               │
                      │   ┌────────────────────────┐                  │
                      │   │                        │                  │
    accept(); ────►   ├──►│ Nginx Worker Thread 1  │  ◄────────┐      │
                      │   │                        │           │      ▼
                      │   └────────────────────────┘    ┌──────┴─────────────┐
                      │                                 │Accepted Connections│
                      │   ┌────────────────────────┐    └──────┬─────────────┘
                      │   │                        │           │     A
    accept(); ─────►  └──►│ Nginx Worker Thread 2  │  ◄────────┘
                          │                        │
                          └────────────────────────┘
```

Nginx memory usage remains constant when the accept queue grows, or when `SOMAXCONN` is approached or exceeded. Nginx does not allocate memory for new requests if at all possible.

Demonstrating memory consumption and stability with nginx is trivial, despite volatile conditions. See [ngx_palloc.c](https://github.com/nginx/nginx/blob/master/src/core/ngx_palloc.c) for implementation detail.

### Observed Nginx Proxy Behavior

Nginx will call `listen()` with an internal `backlog` counter for every instance of `proxy_pass` defined in a configuration. See [how nginx calls listen() in ngx_open_listening_sockets](https://github.com/nginx/nginx/blob/master/src/core/ngx_connection.c#L406-L408).

For every client request, Nginx can be expected to perform the following:

- Call `accept()` on the proxy socket connection for each client
- Call `connect()` against the upstream server (asynchronous)
- Call `write()` against the upstream server (if corresponding upstream calls `read()`)
- Call `epoll_wait()` against the file descriptor (wait for upstream)
- Call `close()` against the upstream socket as soon as the upstream returns, or the request times out
    - In the event a 5XX is returned due to a faulty upstream, nginx will call `write()` against the client socket to return the 504 Gateway Timeout

# Test Cases and Observations

The following lab can be performed using the code in this repository. Note that running this code is relatively dangerous and should not be executed by anyone without understanding the impact on the local system.

#### Dependencies

- [GoAccess](https://goaccess.io/) installed locally to serve access dashboard.
- [Nginx](https://www.nginx.com/) installed locally to serve as reverse proxy.

```bash 
# Arch Linux
yay -Sy goaccess nginx
```

The `run-cases` script can be used to execute the following test cases, each with their own corresponding server binary found in the `/src` directory after running `make upstream`.

### Case 1: Upstream always returns HTTP 200 

In the case where the upstream server works as expected (happy path) we call `accept()`, and `read()` and `write()` and `close(newsockfd)` and spoof a successful HTTP 200 for every request here are the findings:

- Single lines in the `/proc/net/unix` file indicating that a single Unix socket operates behind the socket file. This is to be expected as we call `close()` for each connection.
- The socket state saturation from the `/proc/net/unix` file is the vast majority (90% or higher) in state `01` **SS_UNCONNECTED**.
- Nginx will work as expected and proxy requests to/from the upstream server.

### Case 2: Upstream not accepting connections

In the case where the upstream server does not call the C function `accept()` after a socket has been established here are the findings:

- Multiple lines in the `/proc/net/unix` file indicating multiple Unix sockets opened on the same socket file.
- The socket state saturation from the `/proc/net/unix` file is the vast majority (90% or higher) in state `02` **SS_CONNECTING** with the minority in state `01` **SS_UNCONNECTED**.
- Nginx will "hang" and `curl` clients wait for a connection to be established, `curl` clients are synchronously hanging until timeout.

### Case 3: Upstream accepts new connections, but does not read

In the case where the upstream server does call the C function `accept()` but never calls `read()` or  `write()` on the connection, nor `close()` on the file descriptor here are the findings:

- Multiple lines in the `/proc/net/unix` file indicating multiple Unix sockets opened on the same socket file.
- The socket state saturation from the `/proc/net/unix` file is the vast majority (90% or higher) in state `03` **SS_CONNECTED** with the minority in state `01` **SS_UNCONNECTED**.
- Nginx will "hang" and `curl` clients wait for a response from the server, `curl` clients are synchronously hanging until timeout.

### Case 4: Upstream accepts new connections, reads, but does not write

In the case where the upstream server calls the C functions `accept()` and `read()` but never calls `write()` on the connection, nor `close()` on the file descriptor here are the findings:

- Multiple lines in the `/proc/net/unix` file indicating multiple Unix sockets opened on the same socket file.
- The socket state saturation from the `/proc/net/unix` file is the vast majority (90% or higher) in state `03` **SS_CONNECTED** with the minority in state `01` **SS_UNCONNECTED**.
- Nginx will "hang" and `curl` clients wait for a response from the server, `curl` clients are synchronously hanging until timeout.

Note: The output of the test indicates there is a small latency between when the socket state switches from `03` **SS_CONNECTED** to `01` **SS_UNCONNECTED** that increases in this test case when compared to case 3. During the duration between socket states, there is a sample where there is only a single line in the `/proc/net/unix` file. Presumably this occurs when the `curl` client timeouts.

### Case 5: Upstream accepts new connections, reads, writes but does not close

In the case where the upstream server calls the C functions `accept()`, `read()`, `write()` on the connection but never calls `close()` on the file descriptor here are the findings:

- Multiple lines in the `/proc/net/unix` file indicating multiple Unix sockets opened on the same socket file.
- The socket state saturation from the `/proc/net/unix` file is the vast majority (90% or higher) in state `03` **SS_CONNECTED** with the minority in state `01` **SS_UNCONNECTED**.
- Nginx will "hang" and `curl` clients wait for a response from the server, `curl` clients are synchronously hanging until timeout.

Note: The output of the test indicates there is an even larger latency between when the socket state switches from `03` **SS_CONNECTED** to `01` **SS_UNCONNECTED** that increases again in this test case when compared to cases 3 and 4. During the duration between socket states, there is a sample where there is only a single line in the `/proc/net/unix` file. Presumably this occurs when the `curl` client timeouts.

```c 
// Kernel Linux 6.2
// Taken from https://github.com/torvalds/linux/blob/v6.2/include/uapi/linux/net.h
typedef enum {
	SS_FREE = 0,			/* not allocated		*/
	SS_UNCONNECTED,			/* unconnected to any socket	*/
	SS_CONNECTING,			/* in process of connecting	*/
	SS_CONNECTED,			/* connected to socket		*/
	SS_DISCONNECTING		/* in process of disconnecting	*/
} socket_state;
```

| Enum             | Value | Sample (/proc/net/unix) |
|------------------|-------|-------------------------|
| SS_FREE          | 0     | 00                      |
| SS_UNCONNECTED   | 1     | 01                      |
| SS_CONNECTING    | 2     | 02                      |
| SS_CONNECTED     | 3     | 03                      |
| SS_DISCONNECTING | 4     | 04                      |

# Research

Below is basic reference material which has been compiled from the results and observations of this research project, as well as references in code and other online resources.

### Nginx Workers

Nginx doesn't spawn a process or thread for every connection. Instead, worker processes accept new requests from a shared "listen" socket and execute a highly efficient run-loop inside each worker to process thousands of connections per worker. [Source](http://www.aosabook.org/en/nginx.html).

### The "Accept Queue"

Note: _The "accept queue" is sometimes referred to as "the backlog queue" as found in this [Tuning NGINX for Performance](https://www.nginx.com/blog/tuning-nginx/) guide. The term "backlog queue" corresponds to the internal counter nginx uses to limit connections in addition to the system `SOMAXCONN` kernel parameter. See [how nginx calls listen() in ngx_open_listening_sockets](https://github.com/nginx/nginx/blob/master/src/core/ngx_connection.c#L406-L408).

Nginx, like any server running on top of a Linux kernel, is bound by the constraints of the kernel. 

Every socket server that calls `listen()` as defined in `<sys/socket.h>` is bound by the `SOMAXCONN` value configured with [sysctl(8)](https://man7.org/linux/man-pages/man8/sysctl.8.html). [Source](https://github.com/torvalds/linux/blob/master/net/socket.c#L1824-L1843). The server will refuse new connections if the amount of incoming connections equals the value of `SOMAXCONN` before a corresponding `accept()` can process the accept queue.

```c 
/* Prepare to accept connections on socket FD.
   N connection requests will be queued before further requests are refused.
   Returns 0 on success, -1 for errors.  */
extern int listen (int __fd, int __n) __THROW;
```

This  queue of inbound and unaccepted requests is often referred to as "The Accept Queue".

### Nginx Memory

For each connection, the necessary memory buffers are dynamically allocated, linked, used for storing and manipulating the header and body of the request and the response, and then freed upon connection release. [Source](http://www.aosabook.org/en/nginx.html)

### Nginx Proxy Behavior

For every proxy request, Nginx can be expected to perform the following:

- The nginx master calls `listen()` for each `proxy_pass` defined.
- An Nginx worker calls `accept()` on the proxy socket.
- An Nginx worker calls `connect()` against the upstream socket (synchronous).
- An Nginx worker calls `write()` against the upstream server if applicable.
- An Nginx worker calls `epoll_wait()` against the file descriptor (wait for upstream to respond).
- An Nginx worker will `close()` the upstream socket as soon as the upstream returns, or the request times out.
    - In the event a 5XX is returned due to a faulty upstream, nginx will call `write()` against the client socket to return the 504 Gateway Timeout.

![img_1.png](img_1.png)

# Additional Labs

Below are one-off labs which can be performed with the tools in this repository. These labs can be used to demonstrate the claims made above.

### Lab: Demonstrate Active Connections > SOMAXCONN is possible.

First set `net.core.somaxconn` to a small number.

```bash 
sudo -E sysctl -n net.core.somaxconn # Note the existing value for later
sudo -E sysctl -w net.core.somaxconn=8
```
Next start the proxy and a bad upstream server.

```bash 
sudo -E ./proxy
sudo -E ./src/upstream-server-not-accept
```

Next send a number of large requests to the proxy server greater than the value above and quickly check the result

```bash
for i in {1..32}
do
  curl -X POST -d /tmp/payload.dat localhost:80 &
done
curl localhost:80/stats
```

Find the value is greater than the kernel limit set by `SOMAXCONN` at runtime.

```
Active connections: 27 
server accepts handled requests
 114 114 114 
Reading: 0 Writing: 27 Waiting: 0
```

Return `somaxconn` to the value noted above.

```bash 
sudo -E sysctl -w net.core.somaxconn=2048
```

### Lab: Demonstrate a single request in the accept queue

In order to demonstrate a single request in the accept queue perform the following steps:

1. Start an nginx reverse proxy `./proxy # Terminal 1`
2. Start a bad upstream server `./src/upstream-server-not-accept # Terminal 2`
3. Send requests to the proxy server `curl localhost:80 & # Terminal 3`
4. Read the output of the status module `curl localhost:80/stats # Terminal 3`

Notice that the client will eventually time out with a 504 Gateway Timeout from nginx.
Even though nginx was able to `listen()` for new connections, and a worker was able to `accept()` the client connection to proxy, the **Active Connections** metric is still increased by 1 for each request as the proxy waits for the upstream to call `accept()`.

```bash 
# strace -f -p <nginx-master-pid>
[{events=EPOLLIN, data={u32=145702928, u64=139848575827984}}], 512, 60000) = 1
[pid 301051] accept4(5, {sa_family=AF_INET, sin_port=htons(49960), sin_addr=inet_addr("127.0.0.1")}, [112 => 16], SOCK_NONBLOCK) = 12
[pid 301051] epoll_ctl(8, EPOLL_CTL_ADD, 12, {events=EPOLLIN|EPOLLRDHUP|EPOLLET, data={u32=145703888, u64=139848575828944}}) = 0
[pid 301051] epoll_wait(8, [{events=EPOLLIN, data={u32=145703888, u64=139848575828944}}], 512, 55619) = 1
[pid 301051] recvfrom(12, "POST / HTTP/1.1\r\nHost: localhost"..., 1024, 0, NULL, NULL) = 163
[pid 301051] epoll_ctl(8, EPOLL_CTL_MOD, 12, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, data={u32=145703888, u64=139848575828944}}) = 0
[pid 301051] socket(AF_UNIX, SOCK_STREAM, 0) = 13
[pid 301051] ioctl(13, FIONBIO, [1])    = 0
[pid 301051] epoll_ctl(8, EPOLL_CTL_ADD, 13, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, data={u32=145704128, u64=139848575829184}}) = 0
[pid 301051] connect(13, {sa_family=AF_UNIX, sun_path="/var/run/nginx-proxy-analysis.sock"}, 110) = 0
[pid 301051] getsockopt(13, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
[pid 301051] writev(13, [{iov_base="POST / HTTP/1.0\r\nHost: case-upst"..., iov_len=170}, {iov_base="/tmp/payload.dat", iov_len=16}], 2) = 186
[pid 301051] epoll_wait(8, [{events=EPOLLOUT, data={u32=145703888, u64=139848575828944}}, {events=EPOLLOUT, data={u32=145704128, u64=139848575829184}}], 512, 55618) = 2
[pid 301051] epoll_wait(8,
```

### Lab: Demonstrate nginx memory stability 

Initial research shows that nginx will not allocate significant memory in the event that a request is queued in the `listen()` queue while waiting on an upstream server.

During 100 requests each with a 1Mb payload against a known upstream that will never `accept()` a connection the only nginx worker grow from 3Mb memory consumption to 5Mb memory consumption. Where memory was read from the pid's corresponding `/proc/<pid>/status.VmSize`.

First create a large file to serve as a large payload

```bash 
# Create a ~1 Mb file
dd if=/dev/urandom of=/tmp/payload.dat  bs=1M  count=1
```

Next start the nginx proxy, and a known faulty upstream server.

```bash 
./proxy
./src/upstream-server-not-accept
```

Next send large amounts of POST requests to the server with the large payload.

```bash
for i in {1..1024}
do
  curl -X POST -d @/tmp/payload.dat localhost:80 &
done
curl localhost:80/stats
```


Memory consumption of the main nginx process, and subsequent nested worker processes remains relatively stable.

![img_1.png](img_1.png)

Additionally `strace` the proxy server.

```bash 
nginx \
  -c $(pwd)/etc/nginx.conf &
strace -f -p $!
```

See strace logs of the above memory test which can be reasoned about as a small denial of service attack. The upstream would never `accept()` a connection simulating a busy upstream server. During the flood nginx was surprisingly efficient in managing memory consumption. A moderate amount of `brk(2) system calls were deteced indicating memory allocation, however the amount of memory allocation was seemingly not impacted by proxy requests with large (1Mb) and larger (10Mb+) payloads.

Note: This test should not be used as a way to validate that nginx will never consume memory during a flood with a busy upstream server. There are many conditions at larger scales which presumably could create this affect.

<details>
	
[pid 332278] writev(286, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(287)                 = 0
[pid 332278] setsockopt(286, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970279536, u64=139748321189488}}], 512, 667) = 1
[pid 332278] recvfrom(286, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(286)                 = 0
[pid 332278] epoll_wait(8, [], 512, 667) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:08 [error] 3322"..., 275) = 275
[pid 332278] close(291)                 = 0
[pid 332278] writev(289, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(290)                 = 0
[pid 332278] setsockopt(289, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970280016, u64=139748321189968}}], 512, 213) = 1
[pid 332278] recvfrom(289, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(289)                 = 0
[pid 332278] epoll_wait(8, [], 512, 212) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:08 [error] 3322"..., 275) = 275
[pid 332278] close(294)                 = 0
[pid 332278] writev(292, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(293)                 = 0
[pid 332278] setsockopt(292, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970280496, u64=139748321190448}}], 512, 191) = 1
[pid 332278] recvfrom(292, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(292)                 = 0
[pid 332278] epoll_wait(8, [], 512, 190) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:08 [error] 3322"..., 275) = 275
[pid 332278] close(297)                 = 0
[pid 332278] writev(295, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(296)                 = 0
[pid 332278] setsockopt(295, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970280976, u64=139748321190928}}], 512, 230) = 1
[pid 332278] recvfrom(295, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(295)                 = 0
[pid 332278] epoll_wait(8, [], 512, 229) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:08 [error] 3322"..., 275) = 275
[pid 332278] close(300)                 = 0
[pid 332278] writev(298, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(299)                 = 0
[pid 332278] setsockopt(298, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970281456, u64=139748321191408}}], 512, 227) = 1
[pid 332278] recvfrom(298, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(298)                 = 0
[pid 332278] epoll_wait(8, [], 512, 226) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:09 [error] 3322"..., 275) = 275
[pid 332278] close(303)                 = 0
[pid 332278] writev(301, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(302)                 = 0
[pid 332278] setsockopt(301, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970281936, u64=139748321191888}}], 512, 213) = 1
[pid 332278] recvfrom(301, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(301)                 = 0
[pid 332278] epoll_wait(8, [], 512, 212) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:09 [error] 3322"..., 275) = 275
[pid 332278] close(306)                 = 0
[pid 332278] writev(304, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(305)                 = 0
[pid 332278] setsockopt(304, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970282416, u64=139748321192368}}], 512, 538) = 1
[pid 332278] recvfrom(304, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(304)                 = 0
[pid 332278] epoll_wait(8, [], 512, 536) = 0
[pid 332278] gettid()                   = 332278
[pid 332278] write(3, "2023/03/06 07:53:09 [error] 3322"..., 275) = 275
[pid 332278] close(309)                 = 0
[pid 332278] writev(307, [{iov_base="HTTP/1.1 504 Gateway Time-out\r\nS"..., iov_len=162}, {iov_base="<html>\r\n<head><title>504 Gateway"..., iov_len=114}, {iov_base="<hr><center>nginx/1.22.1</center"..., iov_len=53}], 3) = 329
[pid 332278] write(4, "127.0.0.1 - - [06/Mar/2023:07:53"..., 91) = 91
[pid 332278] close(308)                 = 0
[pid 332278] setsockopt(307, SOL_TCP, TCP_NODELAY, [1], 4) = 0
[pid 332278] epoll_wait(8, [{events=EPOLLIN|EPOLLOUT|EPOLLRDHUP, data={u32=2970282896, u64=139748321192848}}], 512, 65000) = 1
[pid 332278] recvfrom(307, "", 1024, 0, NULL, NULL) = 0
[pid 332278] close(307)                 = 0
	
</details>

- See [ngx_palloc.c](https://github.com/nginx/nginx/blob/master/src/core/ngx_palloc.c) for implementation detail on how nginx manages an internal memory pool.

![img_1.png](img_1.png)

### Lab: Demonstrate worker_connections limits active connections

First set the `worker_connections` value to `2` in the `nginx.conf` file.

Next run the test suite as normal and observe that **Active Connections** never exceeds the limit set.

### References

- [Manual for /proc/net/unix](https://man7.org/linux/man-pages/man5/proc.5.html)
- [Linux 6.2 Create Unix Socket](https://github.com/torvalds/linux/blob/v6.2/net/unix/af_unix.c#L995)
- [A popular tuning guide](https://www.nginx.com/blog/tuning-nginx/)
- [Nginx buffering](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)
- [HTTP Server Starting Source © 2022 J.P.H. Bruins Slot](https://bruinsslot.jp/post/simple-http-webserver-in-c)
