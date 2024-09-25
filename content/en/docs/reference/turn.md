---
title: The TURN protocol
description: Reference material about the TURN protocol.
weight: 20
status: draft
---

## Network Address Translation (NAT)

When IPv4 was introduced in 1981, the Internet was far smaller than it is today. The choice to use 32 bit numbers as IP addresses was short sighted, as it allows a mere 4.3 billion devices to be addressed. By the 90s it was clear that IPv4 addresses were going to limit the growth of the Internet. At the same time, a new system could not be easily adopted without losing compatibility with the already existing Internet.

Network address translation (NAT) was proposed as a solution to this problem. Under NAT, each individual network interface is assigned a private IPv4 address which uniquely identifies the interface within its local area network (LAN). These private IPv4 addresses are taken from one of three standard ranges, depending on the required size of the LAN:

- `10.0.0.0/8` for the largest networks
- `172.16.0.0/12` for medium-sized networks
- `192.168.0.0/16` for small networks (home networks are often configured to use a subrange of this).

The router uses NAT to act as a gateway to the Internet. When an interface in a LAN wishes to establish a connection to a public IPv4 address on the Internet, it sends its request to the router. The router selects a public IPv4 address from a small pool assigned to it, and then stores the following details in a NAT table:

- Private IPv4 address and port of the network interface that wishes to connect.
- Public IPv4 address and port of the destination.
- Public IPv4 address and port assigned by the router for this communication.

The router then uses the public IPv4 address it has selected to establish a connection with the destination, forwarding any data it receives from the destination to the private network interface using the information in the NAT table.

This is shown schematically in the following diagram:
![](/docs/reference/NAT-Concept.png)

Once the connection is terminated, the relevant entry in the NAT table is removed, and the public IPv4 address assigned by the router is free to use in a new connection.

In this way, a router can multiplex requests from many different private network interfaces using a small number of IPv4 addresses, reducing the demand for public IPv4 addresses. 

NAT has the disadvantage that private network interfaces cannot be addressed directly on the Internet. They must initiate any TCP or UDP connection over the Internet. This is not a fatal flaw, as the Internet is compromised mostly of client devices (smartphones, PCs, laptops, etc) which initiate connections with servers to use services over the Internet (e.g. email, web apps, etc).

However, certain applications such as real-time communication require a peer to peer (p2p) connection between two clients (e.g. two browsers). The Traversal Using Relays around NAT (TURN) protocol is one method which allows two peers contained behind NAT to communicate directly with one another despite the limitations of NAT discussed above.

## How does the TURN protocol work?

## Details