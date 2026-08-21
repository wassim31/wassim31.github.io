---
layout: post
title: "Why the heck gateways are invented in computer networks?"
date: 2023-02-28 01:24:34 +0000
description: "I've always wondered why the default gateway is needed in computer networks. I finally realized, so I wanted to share that with you guys :)"
canonical_devto: https://dev.to/wassim31/why-the-heck-gateways-are-invented-in-computer-networks--50lf
image: /images/posts/why-the-heck-gateways-are-invented-in-computer-networks/cover.jpeg
---

**I've always wondered why the default gateway is needed in computer networks. I finally realized, so I wanted to share that with you guys :)**

Let's take an example: an HTTPS request to Google.com. A DATA chunk is generated, which contains data like the destination Google IP address resolved from DNS, the type of HTTP request (GET or POST...), etc.

The encapsulation process will start in the TCP/IP suite installed on your device, and it goes like the following:

![The TCP/IP encapsulation process](/images/posts/why-the-heck-gateways-are-invented-in-computer-networks/img3.png)

Each layer will perform some work on this DATA sent to Google's server.

## The Transport layer

The first inferior level is the Transport layer. SSL/HTTP mainly runs on top of TCP, so the connection oriented protocol will prepare the TCP tunnel by performing the TCP handshake with the server, allowing TCP tunneling to exchange both data and metadata.

TCP will also segment the DATA and do the sequencing, to avoid incoherent DATA at the destination. It also adds both source and destination operating system process ports. (TCP does a lot of things, google it to know more.)

![TCP segmentation and sequencing](/images/posts/why-the-heck-gateways-are-invented-in-computer-networks/img2.png)

So we now have a bunch of segmented, sequenced PDUs called **segments**, with the DATA and the TCP header. It passes to the next layer.

## The Internet layer

The Internet layer, for example IPv4, will add the IPv4 header, which importantly contains the source and destination IP addresses. So we now have a **packet**.

Another important header will be added: the TTL, time to live. It defines how much time the packet will live in the Internet routers around the world. Each time it passes a router, the value of the TTL gets decremented, until it reaches zero and the packet gets destroyed.

## The Data Link layer

Now the most important thing. We have an IPv4 packet ready to be routed on the Internet, but it needs to pass through the Data Link layer, the Ethernet (either 802.3 or 802.11). This layer will add to the packet both source and destination MAC addresses, to make sure the data is sent to the correct device, since the MAC address is the unique network identifier for each device, while IP addresses can change.

It's possible for the device that sent this HTTPS request to know the destination MAC address if we're on the same local area network. That's thanks to the Address Resolution Protocol: it sends a broadcast IPv4 packet to all devices on the same network and saves the results in the ARP table, which contains the MAC address of each device connected to this specific router interface.

The Ethernet frame then needs to be transformed into bits and sent via UTP or fiber optic cables to our router, so it routes it to Google's server.

## So where does the gateway come in?

![The role of the default gateway](/images/posts/why-the-heck-gateways-are-invented-in-computer-networks/img1.png)

In this situation, the destination is outside our local network, so the device needs to know the MAC address of the router: the Ethernet frame will be sent to the router, and routers communicate with each other using layer 3 addresses, aka IP addresses.

So the device just needs to replace the destination address with the **gateway**, aka the local IP address of the router, get its MAC address via the ARP table, and put it in the Ethernet frame.

And the packet is sent.
