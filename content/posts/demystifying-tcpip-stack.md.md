---
title: "Demystifying TCP/IP Stack"
date: 2024-11-01T23:19:58+05:30
description: "TCP/IP stands for Transmission Control Protocol/Internet Protocol, and it is a group of communication protocols that allow devices to connect and communicate over the internet."
tags: [TCP/IP, network, golang]
---

TCP/IP stands for Transmission Control Protocol/Internet Protocol, and it is a 
group of communication protocols that allow devices to connect and communicate 
over the internet.

Among the protocols in this suite, TCP and IP are the most essential, though 
several others also play a role. The TCP/IP suite acts as a middle layer that 
bridges internet applications with the network’s underlying infrastructure, 
which handles routing and switching.

TCP/IP defines how data is sent across the internet by enabling end-to-end 
communication. It outlines how data should be divided into packets, assigned 
addresses, sent, directed through the network, and reassembled at the receiving 
end. It operates with minimal centralized control, ensuring networks remain 
reliable and can recover automatically if a device in the network fails.

Currently, the internet primarily uses Internet Protocol Version 4 (IPv4). 
However, because IPv4 has a limited number of unique addresses, a newer 
protocol, IPv6. IPv6 greatly increases the number of available addresses and is 
being gradually adopted to meet growing demands.

## How does TCP/IP work?

he TCP/IP stack operates based on a layered architecture, facilitating 
communication between devices over a network.

### Client-Server Communication

TCP/IP follows a client-server model, where a client (a user or device) requests 
a service, such as accessing a webpage, from a server (a computer in the network). 
The server processes the request and sends back the desired data. For example, 
when you open a browser and visit a website, your device acts as the client, 
and the website's hosting server responds to your request.

### Stateless Protocol Suite

The TCP/IP suite is considered stateless, meaning that each client request is 
treated as independent and not linked to previous interactions. This stateless 
nature makes the network more efficient because it doesn't need to store 
information about prior requests, freeing up resources and network paths for 
continuous use.

### Stateful Transport Layer

While the TCP/IP protocol suite is stateless, the transport layer, specifically 
the Transmission Control Protocol (TCP), is stateful. When a message is 
transmitted, TCP ensures that all packets belonging to the message are sent, 
received, and reassembled in the correct order. The connection between the 
sender and receiver remains active until the entire message has been 
successfully delivered.


## example

```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello world")
}
```


