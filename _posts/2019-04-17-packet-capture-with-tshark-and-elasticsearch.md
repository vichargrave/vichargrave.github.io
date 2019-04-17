---
title:  "Packet Capture with Wireshark and Elasticsearch"
date:   2019-04-17 12:40:37
classes: wide
author_profile: false
toc: true
toc_sticky: true
toc_label: <a href="#site-nav"><font color="#cca300">On This Page</font></a>
header:
  image: /assets/images/Packet_Capture_with_Pyshark_and_Elasticsearch.png
  teaser: /assets/images/Packet_Capture_with_Pyshark_and_Elasticsearch.png
categories:
  - Programming
tags: 
  - Elasticsearch 
  - Python
  - Network
---

Network packet capture and analysis are commonly done with tools like *tcpdump*, *snort*, and *Wireshark*. These tools provide the capability to capture packets live from networks and store the captures in PCAP files for later analysis. A much better way to store packets is to index them in Elasticsearch where you can easily search for packets based on any combination of packet fields.


Place all your code in the *escpap* directory.

## Espcap Structure

### Application Modules

- `espcap.py` – 
- `tshark.py` – 
- `jsonifier.py` – 
- `indexer.py` – 

### Main Application - espcap.py

escpacp.py supports these command line options:

- `--node` – The IP address and port of the Elasticsearch instance where the packets will be indexed.
- `--nic` – The network interface that will be used for live capture
- `--file` – The PCAP file that containing the packets that will be loaded
- `--dir`  – 
- `--list` – Lists the network interfaces that can be used for live capture
- `--help` – Displays a help message summarizing the application usage

### Tshark  Wrapper API - tshark.py

### Packet JSON Conversion - jsonifier.py

### Packet Indexing in Elasticsearch - indexer.py

## Running Espcap

### Create Packet Index Template

### Command Modes
You can download from Github at [https://github.com/vichargrave/espcap](https://github.com/vichargrave/espcap){:target="_blank"}. Just clone the repo and cd into the *espcap/src* directory, then you can test the script with one of the test packet captures.

{% highlight bash %}
{% endhighlight %}

The first few packets should look like this:

{% highlight bash %}
{% endhighlight %}

Next try to index the same packets in Elasticsearch. Assuming you have an Elasticsearch instance running on  your local system you can run **espcap_lite.py** like this:

{% highlight bash %}

{% endhighlight %}

You can check to see if the packets were indexed by running this `curl` command:

{% highlight bash %}

{% endhighlight %}

The first few packets in Elasticsearch should look like this:

{% highlight bash %}

{% endhighlight %}

### Packet Search
