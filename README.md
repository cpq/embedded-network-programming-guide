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

1. Layer 1: **Driver layer**, only reads and writes frames from/to network hardware
2. Layer 2: **TCP/IP stack**, parses protocol headers and handles IP and TCP/UDP
3. Layer 3: **Network Library**, parses application protocols like MQTT or HTTP
4. Layer 4: **Application**, is made by you, firmware developer


Let's provide an example. In order to show your this guide on the Github, your
browser first needs to find out the IP address of the Github's machine. For
that, it should make a DNS (Domain Name System) request to one of the DNS
servers. Here's how your browser and the network stack work for that case:

![DNS request](media/dns.svg)

1. Your browser (an application) asks from the lower layer (library),
   "what IP address `github.com` has?". The lower layer provides an API
   function `gethostbyname()` for that.
2. The library layer receives the name `github.com` and creates a properly
   formatted, binary DNS request: `struct dns_request`. Then it calls an
   API function `sendto()` provided by the TCP/IP stack layer, to send that
   request over UDP to the DNS server. The IP of the DNS server is known
   to the library from the workstation settings.
3. The TCP/IP stack's `sendto()` function gets a buffer to send. It prepends
   UDP, IP, and MAC headers to the buffer, forming a frame. Then it calls
   an API function `send_frame()` provided by the driver layer
4. A driver's `send_frame()` function transmits a frame over the wire or over
   the air, the frame travels to the destination DNS server and finally hits
   DNS server's network card
5. A driver on the DNS server receives a frame, and passes it up by
   calling an API function `ethernet_input()` provided by the TCP/IP stack
6. TCP/IP stack parses the frame, and finds out that it is for the UDP port 53,
   which is a DNS port number. TCP/IP stack finds an application that
   listens on UDP port 53, which is a DNS server application, and wakes up
   its `read()` call, passing UDP payload to it
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

Embedded systems very often use TCP/IP stacks that provide the same BSD API
as mainstream OSes do. For example, a well-known lwIP (LightWeight IP) TCP/IP
stack, Keil's MDK TCP/IP stacks, Zephyr RTOS TCP/IP stack - all provide
BSD socket API. Thus let's review the most important BSD API stack functions:

- `socket(protocol)` - creates a connection descriptor and assigns an integer ID for it, a "socket"
- `bind(sock, addr)` - assigns a local IP:PORT for a listening socket. If
   protocol is TCP, and a handshake request arrives to that listening socket,
   it can call the following API:
- `accept(sock, addr)` - creates a new socket, and assigns both local IP:PORT
   and remote IP:PORT for it, forming a connection
- `connect(sock, addr)` - assigns both local IP:PORT and remote IP:PORT for
   an socket, forming an outgoing connection. If protocol is TCP, it
   initiates TCP handshake exchange
- `send(sock, buf, len)` - sends data
- `recv(sock, buf, len)` - receives data
- `close(sock)` - closes a socket



## Implementing layers 1,2,3 - making ping work

## Implementing layer 4 - a simple web server

## Socket API

Many network stack implementations do not provide high level library API:
they let user to implement it. Therefore, the API they provide is the API
to TCP or UDP functionality. Let's take for example of the popular network
stack implementations, lwIP (LightWeight IP).

First of, lwIP has 3 APIs: raw callback-based, netconn, and socket.
Depending on an API, the architecture of the whole stack is different.

The raw API handles frames in the same task: driver -> stack -> application, and data is passed via a direct function call.
This is also how Mongoose stack works. This is the fastest architecture in terms of performance. I won't cover LWIP netconn here: it is a "connection abstraction" built on top of raw lwip, very much like Mongoose, which also has a "struct mg_connection" abstraction.

The socket API organises frame handling differently. TCP/IP stack and user app run in different RTOS tasks, and data is passed not via direct function calls, but via a RTOS queue. This makes an implementation slower in comparison to the "raw callback" one, but on the other hand it allows blocking send()/recv() calls. A user app task can block whilst not blocking the driver or stack. Also, a typical lwip firmware runs a separate task to sense PHY status.

Socket layer organises it's own data buffering, to allow app to read data long time after it was received. Data gets buffered by each socket's send/recv buffers, and application has its own buffers. Mongoose built-in stack does not use sockets API, thus it does not have that intermediate buffering - as well as raw LWIP. That means that socket-based implementations are more memory hungry than non-socket. So, socket-based frame handling looks like this:
driver (queue)--> stack + socket buffers task (queue)--> application task

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


