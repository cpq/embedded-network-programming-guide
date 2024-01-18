# Embedded network programming guide

[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](https://opensource.org/licenses/MIT)

This guide is written for embedded developers who work on connected products.
That includes microcontroller based systems that operate in bare metal or
RTOS mode, as well as microprocessor based systems which run embedded Linux.

## Prerequisites

All fundamental concepts are explained in details, so the guide should be
understood by a reader with no prior networking knowledge.

A reader is expected to be familiar with microcontroller C programming - for
that matter, I recommend reading my [bare metal programming
guide](https://github.com/cpq/bare-metal-programming-guide).  I will be using
Ethernet-enabled Nucleo-H743ZI board throughout this guide.  Examples for other
architectures are summarized in a table below - this list will expand with
time. Regardless, for the best experience I recommend Nucleo-H743ZI to get the
most from this guide: [buy it on
Mouser](https://www.mouser.ie/ProductDetail/STMicroelectronics/NUCLEO-H743ZI2?qs=lYGu3FyN48cfUB5JhJTnlw%3D%3D).

## Network stack explained

### Network frame structure

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
Protocol is usually 0x800, which means that the next header is IP. MAC header
handles addressing in the local network (LAN).


**IP (Inernet Protocol) header** has many fields, but the most important are:
destination IP address, source IP address, and upper level protocol. IP
addresses are 4-bytes, e.g. `209.85.202.102`, and they identify a machine
on the Internet, so their purpose is similar to phone numbers. The upper level
protocol is usually 6 (TCP) or 17 (UDP). IP header handles global addressing.

**TCP or UDP header** has many fields, but the most important are destination
port and source ports. On one device, there can be many network applications,
for example, many open tabs in a browser. Port number identifies
an application. 

**Application protocol** depends on the target application. For example, there
are servers on the Internet that can tell an accurate current time. If you want
to send data to those servers, the application protocol must SNTP (Simple
Network Time Protocol). If you want to talk to a web server, the protocol must
be HTTP. There are other protocols, like DNS, MQTT, etc, each having their own
headers, followed by the application data.

Install [Wireshark](https://www.wireshark.org/) tool to observe network
frames. It helps to identify issues quickly, and looks like this:

![Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/images/ws-main.png)

The structure of a frame described above, makes it possible to accurately
deliver a frame to the correct device and application over the Internet. When a
frame arrives to a device, a software that handles that frame (a network
stack), is organised in four layers.

### Network stack architecture

![Network Stack](media/stack.svg)

Layer 1: **Driver layer**, only reads and writes frames from/to network hardware  
Layer 2: **TCP/IP stack**, parses protocol headers and handles IP and TCP/UDP  
Layer 3: **Network Library**, parses application protocols like DNS, MQTT, HTTP  
Layer 4: **Application** - Web dashboard, smart sensor, etc


### DNS request example

Let's provide an example. In order to show your this guide on the Github, your
browser first needs to find out the IP address of the Github's machine. For
that, it should make a DNS (Domain Name System) request to one of the DNS
servers. Here's how your browser and the network stack work for that case:

![DNS request](media/dns.svg)

**1.** Your browser (an application) asks from the lower layer (library), "what
IP address `github.com` has?". The lower layer (layer 3, a library layer - in
this case, it is a C library) provides an API function `gethostbyname()`
that returns an IP address for a given host name. So everything said below,
essentially describes how `gethostbyname()` works.

**2.** The library layer gets the name `github.com` and creates a properly
formatted, binary DNS request: `struct dns_request`. Then it calls an API
function `sendto()` provided by the TCP/IP stack layer (layer 2), to send that
request over UDP to the DNS server. The IP of the DNS server is known to the library
from the workstation settings. The UDP port is also known - port 53, a standard
port for DNS.

**3.** The TCP/IP stack's `sendto()` function receives a chunk of data to send.
it contains DNS request, but `sendto()` does not know that and does not care
about that. All it knows is that this is the piece of user data that needs to
be delievered over UDP to a certain IP address (IP address of a DNS server) on
port 53. Hence TCP/IP stack prepends UDP, IP, and MAC headers
to the user data to form a frame. Then it calls API function `send_frame()`
provided by the driver layer, layer 1.

**4.** A driver's `send_frame()` function transmits a frame over the wire or
over the air, the frame travels to the destination DNS server. A chain of
Internet routers pass that frame from one to another, until a frame finally hits
DNS server's network card.

**5.** A network card on the DNS server gets a frame and generates a hardware
interrupt, invoking interrupt handler. It is part of a driver - layer 1. It
calls a function `recv_frame()` that reads a frame from the card, and passes
it up by calling `ethernet_input()` function provided by the TCP/IP stack

**6.** TCP/IP stack parses the frame, and finds out that it is for the UDP port
53, which is a DNS port number. TCP/IP stack finds an application that listens
on UDP port 53, which is a DNS server application, and wakes up its `recv()`
call. So, DNS server application that is blocked on a `recv()` call, receives
a chunk of data - which is a DNS request. A library routine parses that request
by extracting a host name, and passes that parsed DNS request to the application.


**7.** A DNS server application receives DNS request: "someone wants an
IP address for `github.com`". Then the application layer looks at its
configuration, figures out "Oh, it's me who is responsible for the github.com
domain, and this is the IP address I should respond with". The application
extracts an IP address from the configuration, and calls a library function
"get this IP, wrap into a DNS response, and send back". And the response
travels all the way back in the reverse order.

### BSD socket API

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

### TCP echo server implemented with socket API

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
when RTOS queue delivers data. Note that using non-blocking sockets and
`select()/poll()` changes things that instead of many application tasks,
there is only one application task, but the mechanism stays the same.

Therefore this approach with socket API has
the following major characteristics:

1. It uses queues for exchanging data between TCP/IP stack and
   application tasks, which consumes both RAM and time
2. TCP/IP stack buffers received and sent data for each socket. Note that
   the app/library layer may also buffer data - for example, buffering a full
   HTTP request before it can be processed. So the same data goes through
   two buffering "zones" - TCP/IP stack, and library/app

That means, socket API implementation takes extra time for data to be processed,
and takes extra RAM for double-buffering in the TCP/IP stack.

### TCP echo server implemented with callback API

Now let's see how the same approach works without BSD socket API. Several
implementations, including lwIP and Mongoose Library, provide callback API to
the TCP/IP stack. Here is how TCP echo server would look like written using
Mongoose API:

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

### Development environment and tools

Now let's make our hands dirty and implement a working network stack on
a microcontroller board. I will be using
[Mongoose Library](https://github.com/cesanta/mongoose) for all examples
further on, for the following reasons:

- Mongoose is very easy to integrate: just by copying two files,
  [mongoose.c]() and [mongoose.h]()
- Mongoose has a built-in drivers, TCP/IP stack, HTTP/MQTT/Websocket library,
  and TLS 1.3 all in one, so it does not need any other software to create
  a network-enabled application
- Mongoose provides a simple, polished callback API designed specifically
  for embedded developers

The diagram below shows Mongoose architecture. As you can see, Mongoose can
use external TCP/IP stack and TLS libraries, as well as built-in ones. In the
following example, we are going to use only a built-in functionality, so we
won't need any other software.

![Mongoose architecture](media/mongoose.svg)

All source code in this guide is MIT licensed, however Mongoose
is licensed under a dual GPLv2/commercial license.
I will be using a Nucleo board from ST Microelectronics, and there are several choices for the
development environment:
- Use Cube IDE provided by ST: [install Cube](https://www.st.com/en/development-tools/stm32cubeide.html)
- Use Keil from ARM: [install Keil](https://www.keil.com/)
- Use make + GCC compiler, no IDE: follow [this guide](https://mongoose.ws/documentation/tutorials/tools/)

Here, I am going to use Cube IDE. In the templates, however, both Keil and
make examples are provided, too. So, in order to proceed, install Cube IDE
on your workstation, and plug in Nucleo board to your workstation.

### Skeleton firmware

The first step would be to create a minimal, skeleton firmware that does
nothing but logs messages to the serial console. Once we've done that, we'll
add networking functionality on top of it. The table below summarises the
UART peripheral settings for variours boards:


| Board         | UART, TX, RX    | Ethernet                              |    LED |
| ------------- | --------------- | ------------------------------------- | ------ |
| Nucleo-H743ZI | USART3, D8, D9  | A1, A2, A7, B13, C1, C4, C5, G11, G13 | B0, E1, B14 |
| Nucleo-H563ZI | USART3, D8, D9  | A1, A2, A7, B15, C1, C4, C5, G11, G13 | B0, F4, G4 |
| Nucleo-F746ZG | USART3, D8, D9  | A1, A2, A7, B13, C1, C4, C5, G11, G13 | B0, B7, B14 |
| Nucleo-F429ZI | USART3, D8, D9  | A1, A2, A7, B13, C1, C4, C5, G11, G13 | B0, B7, B14 |

**Step 1.** Start Cube IDE. Choose File / New / STM32 project  
**Step 2.** In the "part number" field, type "h743zi". That should narrow down
the MCU/MPU list selection in the bottom right corner to a single row.
Click on it, then click on the Next button  
**Step 3.** In the project name field, type any name, click Finish.
Answer "yes" if a popup dialog appears  
**Step 4.** A configuration window appears. Click on Clock configuration tab.
Find a field with a system clock value. Type the maximum value, hit enter,
answer "yes" on auto-configuration question, wait until configured  
**Step 5.** Switch to Pinout tab, Connectivity, then enable the UART controller
and pins (see table above), choose "Asynchronous mode"  
**Step 6.** Click on Connectivity / ETH, Choose Mode / RMII, verify that the
configured pins are like in the table above - if not, change pins  
**Step 7.** Click Ctrl+S to save the configuration. This generates the code
and opens main.c file  


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


