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

The *Wireshark* command line application *tshark* works much like *tcpdump* with the added capabilities to recognize a wide range of protocols as well as output captured packets in JSON format. This output capability makes it a natural to use with Elasticsearch. I developed the Python application **Espcap** to take advantage of this capability to ciphon off JSON packets from *tshark* then send them to Elasticsearch to be indexed. This article discusses the design and usage of **Espcap**. 

## Packet Capture with Tshark

*tshark* supports numerous options that control how it captures and handles packets. The options we are interested in using for **Espcap** include:

- `-i <interface>` – The network interface from where packets will be captured.
- `-T ek`          – Output packet contents in Elasticsearch compatible JSON format.
- `-c <count>`     – The number of packets to capture, if omitted capture packets indefinitely. 

Following one or more of these options on the command line, you can add a packet filter expression to capture specific packets. For example, to get all TCP packets use the expression `tcp` or to capture DNS packets the expression is `udp port 53`. 


Here are some examples of how to use *tshark* to do various captures. All of the examples assume formatting output packets as Elasticsearch compatible JSON and running on MacOS with network interface `en0`. If running on Linux, you would use `eth0` instead.

- All outbound and inbound HTTPs packets.

   ```
   tshark -i en0 -T ek tcp port 443
   ```

- All outbound  HTTPs packets.

   ```
   tshark -i en0 -T ek tcp dst port 443
   ```

- All outbound and inbound ICMP packets:

  ```
  tshark -i eth0 -T ek icmp
  ```

- All inbound DNS packets:

  ```
  tshark -i eth0 -T ek udp src port 53
  ```


## Espcap Application Structure

### Application Modules

Espcap is organized into these four modules representing the functional areas of the application:

- `tshark.py`    – Wrapper API for tshark functionality
- `indexer.py`   – Indexes packets in Elasticsearch or prints the packets 
- `espcap.py`    – Main program that accepts program arguments and initiates packet capture

###  Tshark API Wrapper

The Tshark API wrapper class is designed to invoke *tshark* in a separate process, then pipe its output to the caller.  The path to the *tshark* executable is set in the *espcap.yml* file. Taking a slight detour here and jumping ahead a bit, the [Espcap](https://github.com/vichargrave/espcap){:target="_blank"} project is maintained in a Github repo that includes a *config* directory which contains *espcap.py*.

#### Class Initialization

The Tshark class includes the`_config_paths` list which contains all the possible paths to *espcap.yml* and the `_command` list, the first member of which is set to the *tshark* path in the *__init__()* method.  

{% highlight python linenos %}
import signal
import subprocess
import sys
import os
import yaml
import json

class Tshark(object): 

    def __init__(self):
        _config_paths = ['espcap.yml','../config/espcap.yml','/etc/espcap/espcap.yml']
        _command = list()
        
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

If *espcap.yml* cannot be found, the application exits. When in doubt, just put *espcap.yml* in the same location and the **Espcap** program files. More on that later in the **Installation** section.

#### Command Construction

The commands supported by **Espcap** include live packet capture, capture from PCAP files, and listing all possible network interfaces. The following *tshark* command lines handle each of these use cases, respectively:

```
tshark -T ek  -i <interface> [-c <count>] [packet filter]
tshark -T ek -r <PCAP file>
tshark -D
```

When *count* is 0 the `-c <count>` part will be omitted from the command and *tshark* run indefinitely. Also, if the *bpf* argument absent, the `[packet filter]` part of the command is left out and *tshark* captures all packets. 

*tshark* commands are constructed with the *mark_command()* method: 

{% highlight python linenos %}
    def make_command(self, nic, count, bpf, pcap_file, interfaces):
        command = self._command

        if interfaces is True:
            command.append('-D')
            return command
    
        command.append('-T')
        command.append('ek')
        if nic is not None:
            command.append('-i')
            command.append(nic)
        if count != 0:
            command.append('-c')
            command.append(str(count))
        if bpf is not None:
            elements = bpf.split()
            for element in elements:
                command.append(element)
        if pcap_file is not None:
            command.append('-r')
            command.append(pcap_file)
    
        return command
{% endhighlight %}

If the *tshark* command line option `-D` is used, it is append to the *command* and the *make_commands()* returns the command string immediately.  Otherwise, all other arguments are appended to the *command* then it is returned.  *make_command()* does not check to see whether the mutually exclusive *nic* and *pcap_file* are both not equal to `None` since that situation is prevented by application earlir in the program flow.

#### Packet Capture Generators

The *capture()* method is written as a Python [generator](https://nvie.com/posts/iterators-vs-generators/){:target="blank"}, which captures one packet at a time then yields each JSON formatted packet back to the caller. The *command* argument contains the full *tshark* command which must be constructed by the caller with *make_command()*.

{% highlight python linenos %}
    def capture(self, command):
        global closing
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            for packet in proc.stdout:
                packet = self._drop_index_line(packet)
                if packet is None:
                    continue
                else:
                    yield json.loads(packet)

            if closing is True:
                print('Capture interrupted')
                sys.exit()
{% endhighlight %}

The *tshark* command is invoked in a separate process in line 3 with a call to *subprocess.Popen()* specifying that the *stdout* of the process will be piped back to the *capture()* method and each packet received by iterating over *proc.stdout*. 

Output from *tshark* with the *-T ek* option for each packet contains two lines, one that represents an Elasticsearch `index` command and the other containing the packet JSON. Here is an example of the packet structure with the packet fields omitted for sake of brevity: 

```
{"index":{"_index":"packets-2019-05-01","_type":"pcap_file"}}
{"timestamp":"1556728888362","layers":{ <packet fields> }}
```

The first line in each response can be dropped because the application will use the Elasticsearch Python Client API to create the bulk index commands.  *_drop_index_line()* is called in line 5 to take care of removing the index command lines.  It returns `None` if an index command was dropped or the packet contents if not. Each packet JSON string is converted to a JSON dictionary object then returned in lines 8 and it 9.

{% highlight python linenos %}
    def _drop_index_line(self, line):
        decoded_line = line.decode().rstrip('\n')
        if decoded_line.startswith('{\"index\":') is True:
            return None
        else:
            return decoded_line
{% endhighlight %}

#### List Network Interfaces

The *list_interfaces()* method simply runs the `tshark -D` command then prints each of the interfaces it gets back.

{% highlight python linenos%}
    def list_interfaces(self, command):
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            for interface in proc.stdout:
                print(interface.decode().rstrip('\n'))
{% endhighlight %}

#### Handling Interrupts

If the packet capture is interrupted somehow, the *_exit_gracefully()* function is called to set the global *closing* flag to `True`.  This flag is checked after the current packet construction is done.  If the flag is `True`, the application is exited.

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

### Packet Indexing and Display

When indexing packets in Elasticsearch, a new index is created every day. The index naming format is _packets-yyyy-mm-dd_. Note that Elasticsearch 7 and later only allows one *_type* field per index. Index IDs are automatically assigned by Elasticsearch.

Elasticsearch compatible JSON packet dictionaries are handled with two functions: *index_packet()* to index them in Elasticsearch and *dump_packets()* to print packets to *stdout*.  *index_packets()* is another generator function that builds and returns *action* JSON objects to the Elasticsearch Python Client API helper function to bulk index packets in Elasticsearch. See the **Main Application** section for more details. 

{% highlight python linenos %}
from datetime import datetime

def index_packets(self, capture, pcap_file):
    for packet in capture:
        timestamp = int(packet['timestamp'])/1000
        action = {
            '_op_type': 'index',
            '_index': 'packets-' + datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d'),
            '_source': packet
        }
        yield action
{% endhighlight %}

The time stamp for for the index name is obtained from the packet JSON dictionary in line 6. Action objects consist of these fields:

- `_opt_type` – Elasticsearch operation to perform, `index` in this case.
- `_index`    – Name of the index. The naming convention is `packets-yyyy-mm-dd`.
- `_type`     – The index type, `espcap` for both live and file capture.
- `_source`   – The JSON body of the packet.

*dump_packets()* works in a similar fashion as *index_packets()* by obtaining each packet from a capture object, adding the time stamp to the display, and enumerating each packet as it is printed out. 

{% highlight python linenos%}
     def dump_packets(self, capture):
       packet_no = 1
        for packet in capture:
            timestamp = int(packet['timestamp'])/1000
            print('Packet no.', packet_no)
            print('* packet time stamp -', datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d %H:%M:%S'))
            print('* payload - ', packet)
            packet_no += 1{% endhighlight %}

### Main Application

#### Process Command Line Options

The main application sorts through command line the arguments and sets up the *tshark* command modes.  The imports pulls all the required modules, including the *tshark* and *indexer* discussed earlier and the Elasticsearch modules, including the Python Client API. 

{% highlight python linenos %}
import syslog
import os
import sys
import click

from elasticsearch import Elasticsearch
from elasticsearch import helpers

from tshark import Tshark
from indexer import index_packets, dump_packets
{% endhighlight %}

The *click* module is a convenient module that handles the numerous command line options.

{% highlight python linenos %}
@click.option('--node', default=None, help='Elasticsearch IP and port (default=None, dump packets to stdout)')
@click.option('--nic', default=None, help='Network interface for live capture (default=None, if file or dir specified)')
@click.option('--file', default=None, help='PCAP file for file capture (default=None, if nic specified)')
@click.option('--dir', default=None, help='PCAP directory for multiple file capture (default=None, if nic specified)')
@click.option('--bpf', default=None, help='Packet filter for live capture (default=all packets)')
@click.option('--chunk', default=1000, help='Number of packets to bulk index (default=1000)')
@click.option('--count', default=0, help='Number of packets to capture during live capture (default=0, capture indefinitely)')
@click.option('--list', is_flag=True, help='Lists the network interfaces')
{% endhighlight %}

Now for the *main()* function, which accepts all the command line arguments from *click*:

{% highlight python linenos %}
def main(node, nic, file, dir, bpf, chunk, count, list):
    try:
        tshark = Tshark()
        tshark.set_interrupt_handler()

        es = None
        if node is not None:
            es = Elasticsearch(node)
    
        if list:
            command = tshark.make_command(nic=None, count=0, bpf=None, pcap_file=None, interfaces=True)
            tshark.list_interfaces(command)
            sys.exit(0)
    
        if nic is None and file is None and dir is None:
            print('You must specify either file or live capture')
            sys.exit(1)
    
        if nic is not None and (file is not None or dir is not None):
            print('You cannot specify file and live capture at the same time')
            sys.exit(1)
    
        syslog.syslog("espcap started")
    
        if nic is not None:
            init_live_capture(es=es, tshark=tshark, nic=nic, bpf=bpf, chunk=chunk, count=count)
    
        elif file is not None:
            pcap_files = []
            pcap_files.append(file)
            init_file_capture(es=es, tshark=tshark, pcap_files=pcap_files, chunk=chunk)
    
        elif dir is not None:
            pcap_files = []
            files = os.listdir(dir)
            files.sort()
            for file in files:
                pcap_files.append(dir+'/'+file)
            init_file_capture(es=es, tshark=tshark, pcap_files=pcap_files, chunk=chunk)
    
    except Exception as e:
        print('[ERROR] ', e)
        syslog.syslog(syslog.LOG_ERR, e)
        sys.exit(1)
{% endhighlight %}

**[Lines 3-8]** Create the *Tshark* object and set the interrupt handlers. If the `node` argument is set to an Elasticsearch IP and port, create an Elasticsearch client object.

**[Lines 10-13]** If the list of network interfaces has been requested, create the *tshark* command specifying all the arguments to *make_command()* tp be `None` except the *interfaces* argument which is set to `True`.  Call *list_interfaces()* with the command then exit the appliction when done.

**[Lines 15-21]** Check to the input arguments to make sure either a network interface, file capture, or a directory of files has been specified. If all of thee arguments are `None`, then print an error indicating that one of them must be set then exit the application.  

**[Lines 25-39]** Initiate live capture, single file capture, or multiple file capture depending on this is specified. In the case of multiple file capture, build a list containing names of all the PCAP files in a given directory, then pass it to *init_file_capture()*. When single file capture is specified, the list contains just the one file.

The last bit of code just calls the *main()* function and reports when the application is done.

{% highlight python linenos %}
if __name__ == '__main__':
    main()
    print('Done')
{% endhighlight %}

#### Initiate Packet Capture

Packet capture is initiated with two functions, *init_live_capture()* and *init_file_capture()*. The first function creates a live capture session with a call to *Tshark.live_capture()* given a network interface, packet filter, and packet count. To print out the packets, we just pass the `capture` object to *dump_packets()*. 

{% highlight python linenos %}
def init_live_capture(es, tshark, nic, bpf, chunk, count):
    try:
        command = tshark.make_command(nic=nic, count=count, bpf=bpf, pcap_file=None, interfaces=False)
        capture = tshark.capture(command)
        if es is None:
            dump_packets(capture)
        else:
            helpers.bulk(client=es, actions=index_packets(capture), chunk_size=chunk, raise_on_error=True)

    except Exception as e:
        print('[ERROR] ', e)
        syslog.syslog(syslog.LOG_ERR, e)
        sys.ext(1)
{% endhighlight %}

The *helpers.bulk()* Elasticearch Python client function does all the heavy lifting to bulk index the packets in Elasticsearch. It accepts a handle to the Elasticsearch cluster we want to use for indexing, the actions produced by the *index_packets()* generator, the number of packets (chunk) to bulk index to Elasticsearch at a time, and whether or not exceptions will be raised for failures during bulk indexing. As we saw earlier, *index_packets()* accepts a `capture` iterable.

*init_file_capture()* works pretty much the same way, except it processes one or more PCAP files. For each file a `capture` object is created then passed to either of the indexer functions. *init_file_capture()* handles one file at a time, creating a separate capture for each, from the list of files passed to it.

{% highlight python linenos %}
def init_file_capture(es, tshark, pcap_files, chunk):
    try:
        print('Loading packet capture file(s)')
        for pcap_file in pcap_files:
            command = tshark.make_command(nic=None, count=0, bpf=None, pcap_file=pcap_file, interfaces=None)
            print(pcap_file)
            capture = tshark.capture(command)
            if es is None:
                dump_packets(capture)
            else:
                helpers.bulk(client=es, actions=index_packets(capture), chunk_size=chunk, raise_on_error=True)

    except Exception as e:
        print('[ERROR] ', e)
        syslog.syslog(syslog.LOG_ERR, e)
        sys.ext(1)
{% endhighlight %}

## Running Espcap

### Installation

You can download from Github at [https://github.com/vichargrave/espcap](https://github.com/vichargrave/espcap){:target="_blank"}. Just clone the repo and *cd* into the *espcap/src* directory, then run `python -r requirements.txt` to install the *elasticsearch* and *click* modules.  You may choose to this in a virtual environment, if you don't plan on doing anything else with these modules.

Open *config/espap.yml* file then set the *tshark_path* field to the localion of your *tshark* instance. If you move the **Espcap** source files to a different location, you might want to just place *espcap.yml* in the same location as the application files. If you want to put the file in an entirely new location, add that path to the *_config_paths* list. 

### Test File Capture

Now you ready to capture some packets. Let's start with file capture mode and print the output to *stdout*.  *cd* into the *src* directory, then run *espcap.py* like this:
 ```
./espcap.py --file=../test_pcaps/test_dns.pcap
 ```

or like this to read all the packet capture files:
```
./espcap.py --dir=../test_pcaps
```

To test packet capture with indexing, start up a local instance of Elasticsearch or use a remote instance if you have one.  Although it's not absolutely necessary, it's a good idea to set up a packets index template to set the date formats to be consistent with the format used in *Tshark*.  If you are using a local Elasticsearch instance, run the *packet_template.sh* script for Elasticsearch 7:
```
../scripts/packet_template.sh localhost:9200
```

If you are running Elasticsearch 6.x, use the *packet_template-6.x.sh* script instead.  Next run *espcap.py* as follows:
```
./epcap.py --node=localhost:9200 --file=../test_pcaps/test_http.pcap
```

You can verify the packet were indexed by running a query to get all packets in the `packets-*` indexes or run the *packet_query.sh* script:
```
../scripts/packet_query.sh localhost:9200
```

### Test Live Capture

Last but certainly not least, you'll want to try live capture.  For example, if you want to capture 300 inbound and outbound TCP packets for a server using port 443 (https) from the `en0 network interface, run *espcap.py* like this:
```
./espcap.py --node=localhost:9200 --interface=en0 --count=3000 --bpf="tcp port 443"
```

To run this packet capture indefinitely, omit the `--count` argument or set it to 0, the default. The rules and formatting of the packet filter expression `--bpf` are determined by *tshark* since this argument string is passed directly to the command.

## Summing Up

**Escpap** illustrates how to use some interesting Python features, most notably iterator functions, and the Elasticsearch Python client API. The API makes good use of iterators in the bulk indexing helper functions. Iterators made it possible to delegate packet capture to our generator functions for live and file capture then continually pass back packets to the packet indexing function, which is itself a generator that conveys the packets to the Elasticsearch client bulk indexing function. This chain of functions runs concurrently to provide a stream of packets to Elasticsearch.

Another approach to capturing packets is discussed in the article [Analyzing Network Packets with Wireshark, Elasticsearch, and Kibana](https://www.elastic.co/blog/analyzing-network-packets-with-wireshark-elasticsearch-and-kibana){:target="_blank"}. The system described in this article uses gets packets dumped to files using Filebeats and Logstash that feed them to Elasticsearch, which is more scaleable if packet captures are sent from a large number of agent systems. It would be worthwhile experimenting with the use of Logstash to handle the direct indexing of packets in Elasticsearch. Espcap could run on each agent, then forward the captures to Logstash running on the Elasticsearch cluster.
