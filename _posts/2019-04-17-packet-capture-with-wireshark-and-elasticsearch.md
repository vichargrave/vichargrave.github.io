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

The *Wireshark* command line application *tshark* works much like *tcpdump* with the added capabilities to recognize a wide range of protocols as well as output captured packets in JSON format. This output capability makes it a natural to use with Elasticsearch. We developed a Python application **Espcap** to ciphon JSON packets from *tshark* then send them to Elasticsearch to be indexed. This article discusses the design and usage of **Espcap**. 

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

If *espcap.yml* cannot be found, the application exits. When in doubt, just put *espcap.yml* in the same location and the **Espcap** program files. More on that later in the **Installation** section.

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

Both methods use `subprocess.Popen()` to execute a *tshark* command where the output is read line by line from `proc.stdout`. The only difference between the two methods is the *tshark* command that is constructed by the *_make_command()* method call in lines 4.  In the file capture case the only command option that matters is the path to the PCAP file.  All other arguments are set to *`None`*.  This creates a *tshark* command of the following form:
```
tshark -T json -r <PCAP file>
```

For live capture, the PCAP file argument is set to *`None`* and the other arguments are assigned values.  This creates a *tshark* command that looks like this:
```
tshark -i <interface> -T json  [-c <count>] [packet filter]
```

Note that if *count* is 0 the `-c <count>` part will be omitted from the command. This makes *tshark* run indefinitely. If the *bpf* argument is set to `None`, the `[packet filter]` part of the command is left out and *tshark* captures all packets. 

As it turns out, *tshark* outputs JSON by placing each element on a separate line. Each packet is separated by a single comma on a line by itself, as shown the example below. Notice this mode also includes Elasticsearch metadata fields, which are undesirable and must be deleted.

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

A new packet begins at line 16.  Ultimately what we want to index in Elasticsearch is the `layers` JSON object. The Elasticsearch metafields produced by *tshark* in lines 5 - 6 must be filtered out since they will be set by Elasticsearch when packets are indexed.  The leading bracket and opening curly brace in lines 1 and 2 must go as well as the trailing curly brace and comma in lines 14 and 15. 

The *_filter_line()* method does part of the job by getting rid of any line that contains an open bracket `[`, a single comment with no other characters on the line, and any blank lines.   

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

This method drops objectionable lines returning `None` to indicate the line was dropped and `False` to indicate packet that the end of the packet has not been reached. If a given line does not have to be dropped, it is returned with the end of packet flag set to `False`. When a close curly brace by itself on a given line is encountered we have a full packet packet.  The line is then returned along with the end of packet flag set to `True`. 

#### Packet Construction

Getting back to *file_capture()* and *live_capture()*, in lines 8 - 9 for any JSON line that is dropped the code skips to get the next line.  Otherwise, if the line is OK and we are still building the packet, the line is added to the the raw packet string. When the end of packet is reached, add the closed curly brace `}` and add it to the raw packet.  Then call *_format_packet()* to filter the unnecessary Elasticsearch metafields and format the raw packet string into a JSON object.  This method will be discussed in the next section.  Since these methods are generators, *yield* is called to return each packet so the flow of control can return to get more packets.  

If the packet capture is interrupted somehow, *_exit_gracefully()* function is called to set the global *closing* flag to `True`.  This flag is checked after the current packet construction is done at which time the application is exited.

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

#### List Network Interfaces

The last *tshark* modality is listing all the network interfaces.  This can be done with with the command:
```
tshark --list
```  

The `list_interfaces()` method handles this in fashion similar to the previous methods:

{% highlight python linenos%}
    def list_interfaces(self):
        command = self._make_command(nic=None, count=None, bpf=None, pcap_file=None, interfaces=True)
        with subprocess.Popen(command, stdout=subprocess.PIPE, bufsize=1) as proc:
            for interface in proc.stdout:
                print(interface.decode().rstrip('\n'))
{% endhighlight %}

###  JSON Conversion

*Tshark._format_packet()* relies on the *Jsonifier* class to remove extraneous Elasticsearch metafields and assemble an Elasticsearch compatible JSON object.  *cleanse()* takes the raw packet string, converts it to a JSON dictionary object, and calls *_filter_es_meta()*.

{% highlight python linenos %}
class Jsonifier(object):

    def cleanse(self, raw_packet):
        try:
            unfiltered_json_packet = json.loads(raw_packet)
            final_json_packet = self._filter_es_meta(unfiltered_json_packet)
            return final_json_packet

        except Exception as e:
            print("Error processing the input: ", e)
            return None
{% endhighlight %}

*_filter_es_meta()* removes the keys marked "_index", "_type", and "_score" with calls to *pop()* list function. Next we want to extract the true packet contents within the `_spurce` item in the JSON packet object with a call to *_dot_to_underscore()*

{% highlight python linenos %}
    def _filter_es_meta(self, unfiltered_json_packet):
        filter_keys = ["_index", "_type", "_score"]

        for item in filter_keys:
            unfiltered_json_packet.pop(item, None)

        if "_source" in unfiltered_json_packet:
            final_json_packet = self._dot_to_underscore(unfiltered_json_packet["_source"])

        return final_json_packet
{% endhighlight %}

The *tshark* JSON output mode has the habit of  duplicating `ip.addr`, `ip.host`, `udp.port`, and `tcp.port` keys in the body of the packet. This unfortunate tendency is a bug that renders the JSON invalid, not to mentions that the keys aren't necessary since *tshark* separately outputs the source and destination IP addresses and hosts as well as the source and destination TCP and UDP ports. *_dots_to_underscores()* filters out the duplicate keys.

{% highlight python linenos %}
    def _dot_to_underscore(self, _source_json_packet):
        filter_keys = ["ip.addr", "ip.host", "udp.port", "tcp.port"]
        final_json_packet = dict()

        for item in filter_keys:
            _source_json_packet.pop(item, None)

        for key, value in _source_json_packet.items():
            updated_key = key.replace(".", "_")
            final_json_packet[updated_key] = value

            if isinstance(value, dict):
                new_value = self._dot_to_underscore(value)
                if not len(new_value):
                    final_json_packet.pop(updated_key, None)
                else:
                    final_json_packet[updated_key] = new_value

        return final_json_packet
{% endhighlight %}

*tshark* JSON output mode also uses periods in the keys, which is not allowed by Elasticsearch, so the dots are replaced by underscores. When done, *_dot_to_underscores()*  returns an Elasticsearch compatible JSON packet dictionary that will yield JSON looking like this:

{% highlight json%}
{
  "layers": {
    "frame": {
      "frame.interface_id": "0",
       "frame.interface_id_tree": {
         "frame.interface_name": "en0"
       },
...
 }
{% endhighlight %}

### Packet Indexing and Display

Elasticsearch compatible JSON packet dictionaries are handled by the *Indexer* methods, *dump_packets()* to print packets to `stdout` and *index_packet()* to index them in Elasticsearch.  Both methods process packets returned by either the live or file capture generators defined in the *Tshark* class.

{% highlight python linenos%}
    def dump_packets(self, capture):
        pkt_no = 1
        for packet in capture:
            packet_timestamp = self._get_timestamp(packet)
            print('Packet no.', pkt_no)
            print('* packet date UTC  -', datetime.utcfromtimestamp(packet_timestamp).strftime('%Y-%m-%dT%H:%M:%S+0000'))
            print('* payload - ', packet)
            pkt_no += 1
{% endhighlight %}

*index_packets()* is another generator function that builds and returns `action` JSON objects to the Elasticsearch Python client helper function to bulk index packets in Elasticsearch.  See the ** Main Appliation ** section for more details. Action objects consist of these fields:

- `_opt_type` – Elasticsearch operation to perform, `index` in this case.
- `_index`    – Name of the index. The naming convention is `packets-yyyy-mm-dd`.
- `_type`     – The index type, `espcap` for both live and file capture.
- `_source`   – The JSON body of the packet.

To the packet `_source` we add the type of capture, `live` or `file`, the packet creation date in UTC provided by *_get_timestamp()*, and the packet content itself.  File capture packets have same action format as live capture, except with an additional field that records the UTC date that the PCAP file was created.

{% highlight python linenos%}
class Indexer(object):
    def index_packets(self, capture, pcap_file):
        for packet in capture:
            packet_timestamp = self._get_timestamp(packet)  # use this field for ordering the packets in ES
            if pcap_file is None:
                action = {
                    '_op_type': 'index',
                    '_index': 'packets-' + datetime.utcfromtimestamp(packet_timestamp).strftime('%Y-%m-%d'),
                    '_type': 'espcap',
                    '_source': {
                        'capture': 'live',
                        'packet_date_utc': datetime.utcfromtimestamp(packet_timestamp).strftime('%Y-%m-%dT%H:%M:%S+0000'),
                        'layers': packet['layers']
                    }
                }
            else:
                action = {
                    '_op_type': 'index',
                    '_index': 'packets-' + datetime.utcfromtimestamp(packet_timestamp).strftime('%Y-%m-%d'),
                    '_type': 'espcap',
                    '_source': {
                        'capture': 'file',
                        'file_name': pcap_file,
                        'packet_date_utc': datetime.utcfromtimestamp(packet_timestamp).strftime('%Y-%m-%dT%H:%M:%S+0000'),
                        'layers': packet['layers']
                    }
                }
            yield action
{% endhighlight %}

Both capture methods call *_get_timesamp()* to extract the packet creation time in milliseconds from the packet `frame`. 

{% highlight python linenos%}
    def _get_timestamp(self, packet):
        timestamp = (packet['layers']['frame']['frame_time_epoch']).split('.')
        return int(timestamp[0])
{% endhighlight %}

### Main Application

#### Display and Process Line Options

The main application is responsible for getting the command line options then initiating live or file packet capture. We use the `click` module to display and gather the command line options:

{% highlight python %}
@click.command()
@click.option('--node', default=None, help='Elasticsearch IP and port (default=None, dump packets to stdout)')
@click.option('--nic', default=None, help='Network interface for live capture (default=None, if file or dir specified)')
@click.option('--file', default=None, help='PCAP file for file capture (default=None, if nic specified)')
@click.option('--dir', default=None, help='PCAP directory for multiple file capture (default=None, if nic specified)')
@click.option('--bpf', default=None, help='Packet filter for live capture (default=all packets)')
@click.option('--chunk', default=1000, help='Number of packets to bulk index (default=1000)')
@click.option('--count', default=0, help='Number of packets to capture during live capture (default=0, capture indefinitely)')
@click.option('--list', is_flag=True, help='Lists the network interfaces')
{% endhighlight %}

The program is initialized by creating the `Tshark` and `Indexer` objects followed by connecting to Elasticsearch and setting the interrupt handler provided by `Tshark`.  If the `node` argument is not present, we set the Elasticsearch handle to `None`, which indicates to the downstream code that we are going to dump the packets to `stdout`

{% highlight python linenos%}
    indexer = Indexer()
    tshark = Tshark()
    es = None
    if node is not None:
        es = Elasticsearch(node)
    tshark.set_interrupt_handler()
{% endhighlight %}

We'll skip sorting through through the arguments. You can do that when you download the code.  Instead, let's finish up the discussion of the code with the functions that initialize packet capture.

#### Initialize Packet Capture

We have all the building blocks we need to do packet capture and indexing, all that's left is to get the packet capture going. For that we define two functions, *init_live_capture()* and *init_file_capture()*. The first function creates a live capture session with a call to *Tshark.live_capture()* given a network interface, packet filter, and packet count. To print out the packets, we just pass the `capture` object to *Indexer.dump_packets()*. 
 
{% highlight python linenos %}
def init_live_capture(es, tshark, indexer, nic, bpf, chunk, count):
    try:
        capture = tshark.live_capture(nic=nic, bpf=bpf, count=count)
        if es is None:
            indexer.dump_packets(capture=capture)
        else:
            helpers.bulk(client=es, actions=indexer.index_packets(capture=capture, pcap_file=None), chunk_size=chunk, raise_on_error=True)

    except Exception as e:
        print('[ERROR] ', e)
        syslog.syslog(syslog.LOG_ERR, e)
{% endhighlight %}

For live capture we use *helpers.bulk()* Elasticearch Python client function, which does all the heavy lifting to bulk index the packets in Elasticsearch. The method accepts the handle to the Elasticsearch cluster we want to use for indexing, the actions produced by the *Indexer.index_packets()* generator, the number of packets (chunk) to bulk index to Elasticsearch at a time, and whether or not exceptions will be raised for failures during bulk indexing. As we saw earlier, *Indexer.index_packets()* accepts a `capture` object and `None` for the PCAP file argument since this is a live capture.

*init_file_capture()* works pretty much the same way, except it can process one or more PCAP files. For each file a `capture` object is created then passed to *Indexer.dump_packets()* or *Indexer.index_packets()*. In the latter case, the PCAP file path is passed in so it can be recorded in the packets index. 

{% highlight python linenos %}
def init_file_capture(es, tshark, indexer, pcap_files, chunk):
    try:
        print('Loading packet capture file(s)')
        for pcap_file in pcap_files:
            print(pcap_file)
            capture = tshark.file_capture(pcap_file)
            if es is None:
                indexer.dump_packets(capture=capture)
            else:
                helpers.bulk(client=es, actions=indexer.index_packets(capture=capture, pcap_file=pcap_file), chunk_size=chunk, raise_on_error=True)

    except Exception as e:
        print('[ERROR] ', e)
        syslog.syslog(syslog.LOG_ERR, e)
{% endhighlight %}

It's worth noting that the Elasticsearch Python client is made available with these imports:

{% highlight python linenos %}
from elasticsearch import Elasticsearch
from elasticsearch import helpers
{% endhighlight %}

## Running Espcap with Elasticsearch

### Installation

You can download from Github at [https://github.com/vichargrave/espcap](https://github.com/vichargrave/espcap){:target="_blank"}. Just clone the repo and *cd* into the *espcap/src* directory, then run `python -r requirements.txt` to install the `elasticsearch` and `click` modules.  You may choose to this in a virtual environment, if you don't plan on doing anything else with these modules.

Open *config/espap.yml* file then set the `tshark_path` field to the localion of your *tshark* instance. If you move the **Espcap** source files to a different location, you might want to just place *espcap.yml* in the same location as the application files. If you want to put the file in amn entirely new location, add that path to the `_config_paths` list. 

### Test File Capture

Now you ready to capture some packets. Let's start with file capture mode and print the output to `stdout`.  *cd* into the *src* directory, then run *espcap.py* like this:
```
./espcap.py --file=../test_pcaps/test_dns.pcap
``` 

or like this to read all the packet capture files:
```
./espcap.py --dir=../test_pcaps
```

To test packet capture with indexing, start up a local instance of Elasticsearch or use a remote instance if you have one.  Although it's not absolutely necessary, it's a good idea to set up a packets index template to set the date formats to be consistent with the format used in *Tshark*.  If you are using a local Elasticsearch instance, run the *template.sh* script for Elasticsearch 7:
```
../scripts/template.sh localhost:9200
```

If you are running Elasticsearch 6.x, use the *template-6.x.sh* script instead.  Then run *espcap.puy as follows:
```
./epcap.py --node:localhost:9200 --file-../test_pcaps/test_http.pcap
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

To run this packet capture indefinitely, omit the `--count` argument. Note the rules and formatting of the packet filter expression `--bpf` are determined by *tshark* since this argument string is passed directly to the command.
