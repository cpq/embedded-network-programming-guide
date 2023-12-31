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

<img alt="Network Frame" src="media/frame.svg" />

The purpose of the headers is as follows:

**MAC (Media Access Control) header** contains destination and source MAC
addresses. MAC addresses are 6-byte unique addresses of the network cards.  For
example, when your workstation sends a request to Github to view this page, the
destination (DST) MAC address would be MAC address of your router, most likely
WiFi router in your home or your office. The source (SRC) address would be the
address of your workstation's network card. The other important field in the
MAC header is protocol, which is usually 0x800 - IP protocol.  So, MAC header
handles frame addressing within your local network (LAN), and tells what the
higher level protocol is.

**IP (Inernet Protocol) header** handles global addressing. The most important
fields in the IP header are source and destination IP address. Those are 4-byte
numbers that identify two machines that communicate with each other.  In our
example, the source would be the the IP address of your workstation, and
destination - Github's machine IP address. The IP header also contains a higher
level protocol number: it is usually TCP (6) or UDP (16).  IP addresses are
similar to phone numbers, they serve the same purpose: to identify two devices
globally. So, IP header handles global addressing; in other words, IP address
tells which device should receive this frame.

**TCP or UDP header** have source and destination port numbers. They identify
which application on this device should receive this frame. For example, Github
machine can run a web server on port 443, and can run SSH server on port 22.
When your browser sends a request to Github to view this page, the destination
port number would be 443, and source port number - some random number.  Thus,
TCP/UDP header handles addressing on a given device. Overall, source and
destination IP addresses and source and desination TCP/UDP port numbers are 4
numbers that identify two communicating applications: `source_ip:source_port`
<-> `destination_ip:destination_port`.

**Application protocol** defines the format of the following data. For example,
before your browser can make a connection to Github server, it must figure out
its IP address. For that, it sends request to so-called DNS server, asking
"Hey, tell me the IP address of a machine with name `github.com`". The DNS
server responds: "Sure, `github.com` has address A.B.C.D". This is an example
of DNS communication, using DNS protocol, which is a complex binary - encoded
protocol.
When your browser figures out the Github's IP address, then it can send
frames to the Github server on Web port, using another protocol - HTTP,
saying "Hey, show me the guide on URL such and such."
There are many application protocols - like MQTT, HTTP, DNS, SSH, and so on.
Some of them are complex, like DNS or SSH, and some are quite simple, like
HTTP.

This structure of a frame makes it possible to accurately deliver
a frame to the correct device and application over the Internet. When a frame
arrives to a device, a software that handles that frame, is organised 
in four layers:


<img alt="Network Stack" src="media/stack.svg" style="margin-bottom: 2em;" />

1. Layer 1: **Driver layer**, only reads and writes frames from/to network hardware
2. Layer 2: **TCP/IP stack**, parses protocol headers and handles IP and TCP/UDP
3. Layer 3: **Network Library**, parses application protocols like MQTT or HTTP
4. Layer 4: **Application**, is made by you, firmware developer


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


