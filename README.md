# Nginx Proxy Analysis

Small research project aimed at understanding failure modes of nginx and reverse proxy buffering. 

### Thesis

By understanding various socket conditions on an upstream proxy we can predict the state and resource consumption of an nginx process. 

### Prompt

What is the default failure mode of nginx reverse proxy when a Unix domain socket is in various states? What other socket conditions have an impact on nginx failures?

At what point does the resource consumption of nginx becoming significant due to upstream socket conditions?

### Control

```bash
nginx/1.22.1
Linux 6.1.12-arch1-1
```

### Setting up

Ensure the following dependencies are on the current filesystem:

 - `goaccess`
 - `nginx` (complete with stub_status and http_status modules)

Run the dashboard

```bash
./dashboard
```

and open the `report.html` file dashboard in your local browser.

### Findings 

|                 | Runtime Condition                                                                   | Nginx Impact                                                                                         | Upstream State                                                             |
|-----------------|-------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| No socket file  | No socket file                                                                      | 502 Bad Gateway                                                                                      | No upstream server exists. No socket on the host filesystem.               |
| server-not-accept-unix | A unix domain socket server is running, but no longer will accept() new connections | Clients hang on server response until timeout, ` accept4() failed (24: Too many open files)` in logs | Unix domain socket server is established, but no connections are accepted. |


### Case: Upstream server not accepting connections

In the case where the upstream server does not call the C function `accept()` after a socket has been established here are the findings:

 - Multiple lines in the `/proc/net/unix` file indicating multiple Unix sockets opened on the same socket file.
 - The socket state saturation from the `/proc/net/unix` file is the vast majority (90% or higher) in state `02` **SS_CONNECTING** with the minorty in state `01` **SS_UNCONNECTED**.
 - Nginx will "hang" and clients wait for a connection to be established, clients are synchronously hanging.

### Resources

 - [Manual for /proc/net/unix](https://man7.org/linux/man-pages/man5/proc.5.html)
 - [Linux 6.2 Create Unix Socket](https://github.com/torvalds/linux/blob/v6.2/net/unix/af_unix.c#L995)

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

