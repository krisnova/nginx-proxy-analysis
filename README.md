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

 - `goacess`
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
| bad-server-unix | A unix domain socket server is running, but no longer will accept() new connections | Clients hang on server response until timeout, ` accept4() failed (24: Too many open files)` in logs | Unix domain socket server is established, but no connections are accepted. |
