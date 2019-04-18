---
title:  "Packet Capture with Wireshark and Elasticsearch"
date:   2019-04-17 12:40:37
classes: wide
author_profile: false
toc: true
toc_sticky: true
toc_label: <a href="#site-nav"><font color="#cca300">On This Page</font></a>
header:
  image: /assets/images/Packet_Capture_with_Wireshark_and_Elasticsearch.png
  teaser: /assets/images/Packet_Capture_with_Wireshark_and_Elasticsearch.png
categories:
  - Programming
tags: 
  - Elasticsearch 
  - Python
  - Network
---

Network packet capture and analysis are commonly done with tools like *tcpdump*, *snort*, and *Wireshark*. These tools provide the capability to capture packets live from networks and store the captures in PCAP files for later analysis. A much better way to store packets is to index them in Elasticsearch where you can easily search for packets based on any combination of packet fields.

The *Wireshark* command line application *tshark* works much like *tcpdum* with tha added capability to output captured packets in JSON format. We developed a Python application **Espcap** to ciphon JSON packets from *tshark* then send them to Elasticsearch to be indexed. This article discusses the design and usage of **Espcap**. 

## Packet Capture with Tshark

*tshark* supports numerous options that control how it captures and handles packets. The options we are interested in using for **Espcap** include:

- `-i <interface>` – The network interface, e.g. `en0` on MacOS, from where packets will be captured.
- `-T json`        – Tells *tshark* to output JSON formatted packets
- `-c <count>`     – The number of packets to capture.  If this option is omitted, *tshark* captures for an indefinite period of time. 

After one or more of these options are applied on the command line, you can add a packet filter expression to capture specific packets. For example, to get all TCP packets use the expression `tcp` or to capture DNS packets the expression is `udp port 53`. If no filter expression is included, all packates will be captured.


Putting all of these options together, here are some examples of how to use *tshark* to do various captures. All of the examples assume formatting output packets as JSON and running on MacOS with network interface `en0`. If running on Linux, you would use `eth0` instead.

- All outbound and inbound HTTPs packets.

   ```
   tshark -i en0 -T json tcp port 443
   ```

- All outbound  HTTPs packets.

   ```
   tshark -i en0 -T json tcp dst port 443
   ```

- All outbound and inbound ICMP packets:

  ```
  tshark -i eth0 -T json icmp
  ```

- All inbound DNS packets:

  ```
  tshark -i eth0 -T json udp src port 53
  ```


## Espcap Appplication Structure

### Application Modules

Espcap is organized into these four modules representing the functional areas of the application:

- `tshark.py`    – Wrapper API for tshark functionality
- `jsonifier.py` – Filters elements from tshark JSON output not required for indexing
- `indexer.py`   – Indexes packets in Elasticsearch or prints the packets 
- `espcap.py`    – Main program that accepts program arguments and initiates packet capture

Each module contains a class that supports the given functionality. For sake of brevity, we'll discuss the core methods in each class, leaving out the details of the support methods.

###  Tshark Wrapper

Given the 

###  JSON Conversion


### Packet Indexing


### Main Application

espcap.py supports these command line options:

- `--node`  – The IP address and port of the Elasticsearch instance where the packets will be indexed.
- `--nic`   – The network interface that will be used for live capture
- `--file`  – The PCAP file that containing the packets that will be loaded
- `--dir`   – Directory containin several PCAP files that will be read anf processed
- `--bpf`   – Packet filter expression to select the packets you want to capture
- `--chunk` – Sets the number of packets to bulk index in Elasticsearch
- `--count` – Number of packets to capture during live capture
- `--list`  – Lists the network interfaces that can be used for live capture
- `--help`  – Lists these program options


## Running Espcap with Elasticsearch

You can download from Github at [https://github.com/vichargrave/espcap](https://github.com/vichargrave/espcap){:target="_blank"}. Just clone the repo and cd into the *espcap/src* directory, then you can test the script with one of the test packet captures.

