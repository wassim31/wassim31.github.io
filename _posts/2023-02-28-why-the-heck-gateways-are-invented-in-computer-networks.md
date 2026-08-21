---
layout: post
title: "Why the heck gateways are invented in computer networks ?"
date: 2023-02-28 01:24:34 +0000
description: "I've always wondered why the default gateway is needed in computer networks , I realized finally, so..."
canonical_devto: https://dev.to/wassim31/why-the-heck-gateways-are-invented-in-computer-networks--50lf
image: /images/posts/why-the-heck-gateways-are-invented-in-computer-networks/cover.jpeg
---

**I've always wondered why the default gateway is needed in computer networks , I realized finally, so i wanted to share that with you guys :) .**


let's take an example , an HTTPS request to Google.com , a DATA chunk is generated , which contains data like the destination google's IP address resolved from DNS , type of HTTP request ( GET OR POST ..) etc...
The encapsulation process will start in the TCP/IP Suite installed in your device , and it's like the following :

![Image description](/images/posts/why-the-heck-gateways-are-invented-in-computer-networks/img3.png)


each layer will perform some work on this DATA sent to Google's Server , the first inferior level is the Transport layer , SSL/HTTP mainly runs on top of TCP , so the connection oriented protocol will prepare the TCP tunnel by performing the TCP handshake with the server and allow TCP tunneling to exchange both data and metadata .

The TCP will also segment the DATA and does the sequencing to avoid incoherent DATA in the destination , also adds both source and destination operating system processes ports( TCP Does a lot of things , google to know more ).


![Image description](/images/posts/why-the-heck-gateways-are-invented-in-computer-networks/img2.png)


So we will have now a bunch of segmented , sequenced pdu's called Segments , with DATA and the TCP Header , it passes to the next layer .

The Internet Layer , for example IPv4 , the layer will add the IPv4 header that contains importantly the source and destination IP address, so we will have now a packet , an important header will be added , it's the TTL , time to live , how much time the packet will live in the Internet routers around the world , each time it passed a Router , the value of the TTL will get decremented until it reaches the zero and the packet get destroyed .

Now the most important thing , we have an IPv4 packet ready to be routed in the Internet , but it needs to pass through Data Link layer , the Ethernet ( even 802.3 or 802.11 ) , this layer will add to the packet both source and destination MAC address to make sure that the data is sent to the correct device , since the MAC address is the unique network identifier for each device, and ip addresses can change .

It's possible for the device that sent this HTTPS request to know the destination mac address if we're on the same local area network , that's thanks to the Address resolution protocol by sending a broadcast ipv4 packet to all devices in the same network and saving results on the ARP table which contains the mac address for each device connected to this specific router interface.

the ethernet frame need to be transformed into bits and sent via UTP or fiber optic cables to our router so it routes it to the google's server .


![Image description](/images/posts/why-the-heck-gateways-are-invented-in-computer-networks/img1.png)



In this situation , the device needs to know the mac address of the router , so the ethernet frame will be sent to the router , and routers communicates with each other with layer 3 addresses aka ip addresses .
so the device will just need to replace the destination address with the gateway aka the local ip address of the router , and get it's mac address via the ARP table , and put it in the ethernet frame .
and the packet is sent.
