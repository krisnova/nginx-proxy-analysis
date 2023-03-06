# Nginx Reverse Proxy Analysis 

A small research project aimed at understanding the behaviour of a simple nginx reverse proxy given various upstream server conditions.

 - nginx/1.22.1
 - Linux 6.1.12
 - Linux 6.2.1
 - Linux 6.2.2

# Findings

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

# Lab 

The following lab can be performed using the code in this repository. Note that running this code is relatively dangerous and should not be executed by anyone without understanding the impact on the local system.

The `run-cases` script can be used to execute the following test cases.

#### Dependencies 

 - [GoAccess](https://goaccess.io/) installed locally to serve access dashboard.
 - [Nginx](https://www.nginx.com/) installed locally to serve as reverse proxy.

```bash 
# Arch Linux
yay -Sy goaccess nginx
```

### Case 1: Upstream returns HTTP 200

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

Note: See https://github.com/krisnova/nginx-proxy-analysis/issues/2 for open question on Linux loopback devices.

First create a large file to use as a payload for the lab.

```bash 
# Create a ~17 Mb file
dd if=/dev/urandom of=/tmp/payload.dat  bs=1M  count=16
```

Next start the nginx proxy, and a bad upstream server.

```bash 
./proxy # Note: uncomment the strace line in ./proxy
./src/upstream-server-not-accept
```

Next send a number of large requests to the proxy server.

```bash
for i in {1..1024}
do
  curl -X POST -d /tmp/payload.dat localhost:80 &
done
curl localhost:80/stats
```

Memory consumption of nginx and worker threads remains constant. Strace shows no signs of memory allocation or similar system calls.

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

