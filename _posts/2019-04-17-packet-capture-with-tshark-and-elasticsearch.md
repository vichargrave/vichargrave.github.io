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

The *Wireshark* command line application *tshark* works much like *tcpdump* with tha added capability to output captured packets in JSON format. We developed a Python application **Espcap** to ciphon JSON packets from *tshark* then send them to Elasticsearch to be indexed. This article discusses the design and usage of **Espcap**. 

## Packet Capture with Tshark

*tshark* supports numerous options that control how it captures and handles packets. The options we are interested in using for **Espcap** include:

- `-i <interface>` – The network interface from where packets will be captured.
- `-T json`        – Output packet contents in JSON format.
- `-c <count>`     – The number of packets to capture, if omitted capture packets indefinitely. 

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
        config = `None`
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
        command = self._make_command(nic=`None`, count=`None`, bpf=`None`, pcap_file=pcap_file, interfaces=`None`)
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            packet = ''
            for line in proc.stdout:
                line, done = self._filter_line(line)
                if done is `False` and line is `None`:
                    continue
                elif done is `False` and line is not `None`:
                    packet += line
                else:
                    packet += line
                    packet = self._format_packet(packet)
                    yield packet
                    packet = ''

            if closing is `True`:
                print('Capture interrupted')
                sys.exit()
{% endhighlight %}

{% highlight python linenos %}
    def live_capture(self, nic, count, bpf):
        global closing
        command = self._make_command(nic=nic, count=count, bpf=bpf, pcap_file=`None`, interfaces=`False`)
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            packet = ''
            for line in proc.stdout:
                line, done = self._filter_line(line)
                if done is `False` and line is `None`:
                    continue
                elif done is `False` and line is not `None`:
                    packet += line
                else:
                    packet += line
                    packet = self._format_packet(packet)
                    yield packet
                    packet = ''

            if closing is `True`:
                print('Capture interrupted')
                sys.exit()
{% endhighlight %}

Both methods us `subprocess.Popen()` to execute a *tshark* command where the output is read line by line from `proc.stdout`. The only difference between the two methods is the *tshark* command that is constructed by the *_make_command()* method call in lines 4.  In the file capture case the only command option that matters is the path to the PCAP file.  All other arguments are set to *`None`*.  This creates a *tshark* command of the following form:
```
tshark -T json -r <PCAP file>
```

For live capture, the PCAP file argument is set to *`None`* and the other arguments are assigned values.  This creates a *tshark* command that looks like this:
```
tshark -i <interface> -T json  [-c <count>] [packet filter]
```

Note that if *count* is 0 the `-c <count>` part will be omitted from the command. This makes *tshark* run indefinitely. If the *bpf* argument is set to `None`, the `[packet filter]` part of the command is left out and *tshark* captures all packets. 

As it turns out, *tshark* outputs JSON by placing each element on a separate line. Each packet is separated by a single comma on a line by itself.  This mode also includes Elasticsearch metadata fields that have to stripped out before indexing packets.  Here is an example of the beginning and ending of a JSON packet:

{% highlight json linenos%}
[
  {
    "_index": "packets-2019-04-18",
    "_type": "pcap_file",
    "_score": null,
    "_source": {
      "layers": {
        "frame": {
          "frame.interface_id": "0",
          "frame.interface_id_tree": {
            "frame.interface_name": "en0"
          },
...
  }
  ,
...
{% endhighlight %}

A new packet begins at line 16.  Ultimately what we want to index in Elasticsearch is the `layers` JSON object and all of is elements, stripping out lines like 1 - 6 and 14 - 15 from each packet. The *_filter_line()* method does par of the job by getting rid of any line that contains an open bracket `[`, a single comment with no other characters on the line, and any blank lines.   

{% highlight python linenos %}
    def _filter_line(self, line):
        decoded_line = line.decode().rstrip('\n')
        if decoded_line.startswith('[') is `True`:
            return `None`, `False`
        elif decoded_line.isspace() is `True` or \
                len(decoded_line) == 0 or \
                decoded_line.find(' ,') >= 0:
            return `None`, `False`
        elif decoded_line.startswith('  }') is `True`:
            return decoded_line, `True`
        else:
            return decoded_line, `False`
{% endhighlight %}

This method drops objectionable lines returning `None` to indicate the line was dropped and `False` to indicate packet that the end of the packet has not been reached. If a given line does not have to be dropped, it is returned with the end of packet flag set to `False`. When an close curly brace by itself on a given line is encountered, this marks the end of the packet to the line is returned along with the end of packet flag set to `True`.

#### Packet Construction

Getting back to *file_capture()* and *live_capture()*, in lines 8 - 9 for any JSON line that is dropped the code skips to get the next line.  Otherwise, if the line is OK and we are still building the packet, the lines is added to the the raw packet string. When the end of packet is reached, add the closed curly brace `}` and add it to the raw packet.  Then call *_format_packet()* to filter the unecessary Elasticsearch metafields and format the raw packet string into a JSON object.  Since these methods are generators, *yield* is called to return each packet so the flow of control can return to get more packets.  

If the packet capture is interrupted somehow, *_exit_gracefully()* function is called to set the global *closing* flag to `True`.  This flag is checked after the current packet construction is done at which time the application is exited. ****

{% highlight python linenos %}
closing = False

def _exit_gracefully(signum, frame):
    global closing
    closing = True
{% endhighlight %}

The *set_interrupt_handlers()* sets this function as the interrupt handler.

{% highlight python linenos %}
    def set_interrupt_handler(self):
        signal.signal(signal.SIGTERM, _exit_gracefully)
        signal.signal(signal.SIGINT, _exit_gracefully)
{% endhighlight %}

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

### Download and Installation

You can download from Github at [https://github.com/vichargrave/espcap](https://github.com/vichargrave/espcap){:target="_blank"}. Just clone the repo and cd into the *espcap/src* directory, then you can test the script with one of the test packet captures.

### Test File Capture


### Test Live Capture