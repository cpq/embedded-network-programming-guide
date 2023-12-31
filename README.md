# Embedded network programming guide

[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](https://opensource.org/licenses/MIT)

This guide is written for embedded developers who work on connected products.
That includes microcontroller based systems that operate in bare metal or
RTOS mode, as well as microprocessor based systems which run embedded Linux.

All fundamental concepts are explained in details. A reader is expected to be
familiar with microcontroller C programming - for that matter, I recommend
reading my [bare metal programming
guide](https://github.com/cpq/bare-metal-programming-guide).  All source code
in this guide is MIT licensed, however examples use [Mongoose
Library](https://github.com/cesanta/mongoose), which is licensed under a dual
GPLv2/commercial license.

I will be using Ethernet-enabled Nucleo-H743ZI board throughout this guide.
Examples for other architectures are summarized in a table below - this list
will expand with time.

## Prerequisites

## Network stack explained

When any two devices communicate, they exchange discrete pieces of data called
frames. Frames can be sent over the wire (like Ethernet) or over the air (like
WiFi or Cellular).  Frames differ in size, and typically range from couple of
dozen bytes to a 1.5Kb.  Each frame cosists of a sequence of protocol headers
followed by user data:

![Network Frame](media/frame.svg)

The purpose of the headers is as follows:

**MAC (Media Access Control) header** is only 3 fields: destination MAC
address, source MAC addresses, and an upper level protocol. MAC addresses are
6-byte unique addresses of the network cards, e.g. `42:ef:15:c8:29:a1`.
Protocol is usually 0x800, which means that the next header is IP. This header
handles addressing in the local network, LAN.


**IP (Inernet Protocol) header** has many fields, but the most important are:
destination IP address, source IP address, and upper level protocol. IP
addresses are 4-bytes, e.g. `209.85.202.102`, and they identify a machine
on the Internet, so their purpose is similar to phone numbers. The upper level
protocol is usually 6 (TCP) or 17 (UDP). This header handles global addressing.

**TCP or UDP header** has many fields, but the most important are destination
port and source ports. Ports are 16-bit numbers and they identify an
application on a device, e.g. `80` means HTTP server.  have source and
destination port numbers. Together with IP addresses, they form a 4-element
tuple: source IP:PORT <-> destination IP:PORT, which uniquely identify data
channel for the two communicating applications.

**Application protocol** depends on the target application. For example, there
are servers on the Internet that can tell an accurate current time. If you want
to send data to those servers, the application protocol must SNTP (Simple
Network Time Protocol). If you want to talk to a web server, the protocol must
be HTTP. There are other protocols, like DNS, MQTT, etc, each having their own
headers, followed by the application data.

This structure of a frame makes it possible to accurately deliver
a frame to the correct device and application over the Internet. When a frame
arrives to a device, a software that handles that frame, is organised 
in four layers:


![Network Stack](media/stack.svg)

Layer 1: **Driver layer**, only reads and writes frames from/to network hardware  
Layer 2: **TCP/IP stack**, parses protocol headers and handles IP and TCP/UDP  
Layer 3: **Network Library**, parses application protocols like MQTT or HTTP  
Layer 4: **Application**, is made by you, firmware developer  


Let's provide an example. In order to show your this guide on the Github, your
browser first needs to find out the IP address of the Github's machine. For
that, it should make a DNS (Domain Name System) request to one of the DNS
servers. Here's how your browser and the network stack work for that case:

![DNS request](media/dns.svg)

**1.** Your browser (an application) asks from the lower layer (library), "what
IP address `github.com` has?". The lower layer provides an API function
`gethostbyname()` for that.

**2.** The library layer receives the name `github.com` and creates a properly
formatted, binary DNS request: `struct dns_request`. Then it calls an API
function `sendto()` provided by the TCP/IP stack layer, to send that request
over UDP to the DNS server. The IP of the DNS server is known to the library
from the workstation settings.

**3.** The TCP/IP stack's `sendto()` function gets a buffer to send. It
prepends UDP, IP, and MAC headers to the buffer, forming a frame. Then it calls
an API function `send_frame()` provided by the driver layer

**4.** A driver's `send_frame()` function transmits a frame over the wire or
over the air, the frame travels to the destination DNS server and finally hits
DNS server's network card

**5.** A driver on the DNS server receives a frame, and passes it up by calling
an API function `ethernet_input()` provided by the TCP/IP stack

**6.** TCP/IP stack parses the frame, and finds out that it is for the UDP port
53, which is a DNS port number. TCP/IP stack finds an application that listens
on UDP port 53, which is a DNS server application, and wakes up its `read()`
call, passing UDP payload to it

7. A DNS server application receives DNS request. A library routine parses
   the DNS request, and passes it on to the application layer:
   "someone wants an IP address for `github.com`". Then the application layer
   decides, what to respond, and the response travels all the way back in the
   reverse order.

The communication between layers are done via a function calls. So, each
layer has its own API, which uppper and lower level can call. They are not
standardized, so each implementation provides their own set of functions.
However, on OSes like Windows/Mac/Linux/UNIX, a driver and TCP/IP layers are
implemented in kernel, and TCP/IP layer provides a standard API to the
userland which is called a "BSD socket API":

![BSD socket API](media/bsd.svg)

This is done becase kernel code does not implement application level protocols
like MQTT, HTTP, etc, - so it let's user application to implement them in
userland. So, a library layer and an application layer reside in userland.
Some library level routines are provided in C standard library, like
DNS resolution function `gethostbyname()`, but that DNS library functions
are probably the only ones that are provided by OS. For other protocols,
many libraries exist that provide HTTP, MQTT, Websocket, SSH, API. Some
applications don't use any external libraries: they use BSD socket API
directly and implement library layer manually. Usually that is done when
application decides to use some custom protocol.

Embedded systems very often use TCP/IP stacks that provide the same BSD API as
mainstream OSes do. For example, lwIP (LightWeight IP) TCP/IP stack, Keil's MDK
TCP/IP stack, Zephyr RTOS TCP/IP stack - all provide BSD socket API. Thus let's
review the most important BSD API stack functions:

- `socket(protocol)` - creates a connection descriptor and assigns an integer ID for it, a "socket"
- `bind(sock, addr)` - assigns a local IP:PORT for a listening socket
- `accept(sock, addr)` - creates a new socket, assigns local IP:PORT
   and remote IP:PORT (incoming)
- `connect(sock, addr)` - assigns local IP:PORT and remote IP:PORT for
   a socket (outgoing)
- `send(sock, buf, len)` - sends data
- `recv(sock, buf, len)` - receives data
- `close(sock)` - closes a socket

Some implementations do not implement BSD socket API, and there are perfectly
good reasons for that. Examples for such implementation is lwIP raw API,
and Mongoose Library.

Let me demonstrate the two approaches on a simple
TCP echo server example. TCP echo server is a simple application that
listens on a TCP port, receives data from clients that connect to that port,
and writes (echoes) that data back to the client. That means, this application
does not use any application protocol on top of TCP, thus it does not need
a library layer. Let's see how this application would look like written
with a BSD socket API. First, a TCP listener should bind to a port, and
for every connected client, spawn a new thread that would handle it. A thread
function that sends/receives data, looks something like this:

```c
void handle_new_connection(int sock) {
  char buf[20];
  for (;;) {
    ssize_t len = recv(sock, buf, sizeof(buf), 0); // Receive data from remote
    if (len <= 0) break;   // Error! Exit the loop
    send(sock, buf, len);  // Success. Echo data back
  }
  close(sock);  // Close socket, stop thread
}
```

Note that `recv()` function blocks until it receives some data from the client.
Then, `send()` also blocks until is sends requested data back to the client.
That means that this code cannot run in a bare metal implementation, because
`recv()` would block the whole firmware. For this to work, an RTOS is required.
A TCP/IP stack should run in a separate RTOS task, and both `send()` and
`recv()` functions are implemented using an RTOS queue API, providing a
blocking way to pass data from one task to another. Overall, this is how an
embedded receive path looks like with socket API:

![BSD socket API](media/socket.svg)

The `send()` part would work in the reverse direction. Note that this approach
requires TCP/IP stack implement data buffering for each socket, because
an application consumes received data not immediately, but after some time,
when RTOS queue delivers data. Therefore this approach with socket API has
the following major characteristics:

1. It uses RTOS queues for exchanging data between tasks, which consumes
   both RAM and time
2. TCP/IP stack buffers received and sent data for each socket. Note that
   the app/library layer may also buffer data - for example, buffering a full
   HTTP request before it can be processed. So the same data goes through
   two buffering "zones" - TCP/IP stack, and library/app, running in two
   separate tasks

That means, socket API implementation takes extra time for data to be processed,
and takes extra RAM for double-buffering in the TCP/IP stack.

Now let's see how the same approach works without BSD socket API. lwIP raw API,
and Mongoose, provide a callback API to the TCP/IP stack. Here is how TCP
echo server would look like written using Mongoose API:

```c
// This callback function is called for various network events, MG_EV_*
void event_handler(struct mg_connection *c, int ev, void *ev_data, void *fn_data) {
  if (ev == MG_EV_READ) {
    // MG_EV_READ means that new data got buffered in the c->recv buffer
    mg_send(c, c->recv.buf, c->recv.len);  // Send back the data we received
    c->recv.len = 0;  // Discard received data
  }
}
```

In this case, all functions are non-blocking, that means that data exchange
between TCP/IP stack and an app can be implemented via direct function calls.
This is how receive path looks like:

![Raw callback API](media/raw.svg)

As you can see, in this case TCP/IP stack provides a callback API which
a library or application layer can use to receive data directly. No need
to send it over a queue. A library/app layer can buffer data, and that's
the only place where buffering takes place. This approach wins for
memory usage and performance. A firmware developer should use
a proprietory callback API instead of BSD socket API.

## Implementing layers 1,2,3 - making ping work

## Implementing layer 4 - a simple web server

## Baremetal, RTOS, and OS environments

## Implementing Web UI

## Implementing Device Dashboard

## Implementing MQTT client

## Implementing MQTT dashboard

## Enabling TLS

## Talking to AWS IoT and Microsoft Azure services

## About me

I am Sergey Lyubka, an engineer and entrepreneur. I hold a MSc in Physics from
Kyiv State University, Ukraine. I am a director and a co-founder at Cesanta - a
technology company based in Dublin, Ireland. Cesanta develops embedded
solutions:

- https://mongoose.ws - an open source HTTP/MQTT/Websocket network library
- https://vcon.io - a remote firmware update / serial monitor framework

You are welcome to register for
[my free webinar on embedded network programming](https://mongoose.ws/webinars/)


