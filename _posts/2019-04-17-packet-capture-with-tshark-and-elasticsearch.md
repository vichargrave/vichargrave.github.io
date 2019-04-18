---
title:  "Packet Capture with Wireshark and Elasticsearch"
date:   2019-04-17 12:40:37
classes: wide
author_profile: false
toc: true
toc_sticky: true
toc_label: <a href="#site-nav"><font color="#eadcb5">On This Page</font></a>
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


## Espcap Application Structure

### Application Modules

Espcap is organized into these four modules representing the functional areas of the application:

- `tshark.py`    – Wrapper API for tshark functionality
- `jsonifier.py` – Filters elements from tshark JSON output not required for indexing
- `indexer.py`   – Indexes packets in Elasticsearch or prints the packets 
- `espcap.py`    – Main program that accepts program arguments and initiates packet capture

Each module contains a class that supports the given functionality. For sake of brevity, we'll discuss the core methods in each class, leaving out the details of the support methods.

###  Tshark Wrapper

The Tshark wrapper class is designed to invoke *tshark* in a separate process, then pipe its output to the caller.  We will define the path to *tshark* in the *espcap.yml* file that is located in a specific directory. Taking a slight detour here and jumping ahead a bit, the [Espcap](https://github.com/vichargrave/espcap){:target="_blank"} project is maintained in a Github repo that includes a *config* directory which contains *espcap.py*.  

#### Class Initialization

The Tshark class includes a member list that contains all the possible paths to *espcap.yml*.  There are additional members that contain the *tshark* command to run and the *Jsonifier* object which we will discuss later.  The first method in Tshark finds *espcap.yml* and sets a command member variable with the *tshark* path.  

{% highlight python linenos %}
class Tshark(object):
    _jsonifier = Jsonifier()
    _command = list()
    _config_paths = ['espcap.yml','../config/espcap.yml','/etc/espcap/espcap.yml']

    def __init__(self):
        config = None
        for config_path in self._config_paths:
            if os.path.isfile(config_path):
                with open(config_path, 'r') as ymlconfig:
                    config = yaml.load(ymlconfig, Loader=yaml.FullLoader)
                self._command.append(config['tshark_path'])
                ymlconfig.close()
                return
        print("Could not find configuration file")
        sys.exit(1)
{% endhighlight %}

If *espcap.yml* cannot be found, the application exits. When in doubt, just put *espcap.yml* in the same location and the **Espcap** program files


#### Packet Capture Generators

Next we define two methods to capture packets live from a network interface or from a set of PCAP files.  These methods are written as Python [generators](https://nvie.com/posts/iterators-vs-generators/){:target="blank"} that capture one packet at a time then *yield* each JSON formatted packet to the caller. 

{% highlight python linenos %}
    def file_capture(self, pcap_file):
        global closing
        command = self._make_command(nic=None, count=None, bpf=None, pcap_file=pcap_file, interfaces=None)
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            packet = ''
            for line in proc.stdout:
                line, done = self._filter_line(line)
                if done is False and line is None:
                    continue
                elif done is False and line is not None:
                    packet += line
                else:
                    packet += line
                    packet = self._format_packet(packet)
                    yield packet
                    packet = ''

            if closing is True:
                print('Capture interrupted')
                sys.exit()

    def live_capture(self, nic, count, bpf):
        global closing
        command = self._make_command(nic=nic, count=count, bpf=bpf, pcap_file=None, interfaces=False)
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            packet = ''
            for line in proc.stdout:
                line, done = self._filter_line(line)
                if done is False and line is None:
                    continue
                elif done is False and line is not None:
                    packet += line
                else:
                    packet += line
                    packet = self._format_packet(packet)
                    yield packet
                    packet = ''

            if closing is True:
                print('Capture interrupted')
                sys.exit()
{% endhighlight %}

Note that these methods are identical except for the *tshark* command used that is constructed by the *_make_command()* method call in lines 4 and 24.  In the file capture case the only command option that matters is the path to the PCAP file.  All other arguments are set to *None*.  This creates a *tshark* command of the following form:
```
tshark -T json -r <PCAP file>
```

For live capture, the PCAP file argument is set to *None* and the other arguments are assigned values.  This creates a *tshark* command that looks like this:
```
tshark -i <interface> -T json  [-c <count>] [packet filter]
```

Note that if *count* is 0 the `-c <count>` part will be omitted from the command. This makes *tshark* run indefinitely. If the *bpf* argument is set to None, the `[packet filter]` part of the command is left out and *tshark* captures all packets. 

#### Packet Formatting


###  JSON Conversion


### Packet Indexing


### Main Application

Espcap supports these command line options:

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

