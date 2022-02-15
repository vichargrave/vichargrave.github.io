---
title:  "Develop a Packet Sniffer with Libpcap"
date:   2012-12-09 12:40:37
classes: wide
author_profile: false
toc: true
toc_sticky: true
toc_label: <a href="#site-nav"><font color="#eadcb5">On This Page</font></a>
header:
  teaser: /assets/images/Develop_a_Packet_Sniffer_with_Libpcap.png
  image: /assets/images/Develop_a_Packet_Sniffer_with_Libpcap.png
categories: 
  - Programming
tags: 
  - C/C++ 
  - Libpcap 
  - Network
---

Libpcap is an open source C library that provides an API for capturing packets directly from the datalink layer of Unix derived operating systems. It is used by popular packet capture applications such as [tcpdump](https://www.tcpdump.org){:target="_blank"} and [snort](https://www.snort.org){:target="_blank"} that enables them to run on just about any flavor of Unix.

Here’s an example of a simple packet sniffer application based on libpcap that displays packet information in a snort-like format.

## Libpcap Installation

Chances are if you use an open source UNIX derived operating system like Linux or FreeBSD libcpap was most likely included with your distribution along with tcpdump. If you do not have libpcap you can download it from [tcpdump](https://www.tcpdump.org){:target="_blank"}. To install follow these instructions:

1. Run `tar -zxvf libpcap.tar.gz` to unpack the libpcap tarball.
2. cd to the resulting local libpcap directory.
3. Run `./configure` to create the make environment.
4. Run make to build the libpcap library in the local directory.
5. Edit the resulting *Makefile* to set the prefix variable to the path where you want to install the libpcap files.
6. `su` to root.
7. Run `make install` to copy the libpcap library, header and man pages to the installation directory set in step 5

## Program Structure

### Header Files and Global Variables

The code for the packet sniffer will reside in a single file `sniffer.c` that starts off with the include files shown below. All libpcap programs require the pcap.h header file to gain access to library functions and constants. The netinet and arpa headers provide data structures that simplify the task of accessing protocol specific header fields. ANSI and UNIX standard headers are included so the program can display packet contents and handle program termination signals.

{% highlight c %}
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <pcap/pcap.h>
#include <netinet/tcp.h>
#include <netinet/udp.h>
#include <netinet/ip_icmp.h>

pcap_t* handle;
int linkhdrlen;
int packets;
{% endhighlight %}

There are three global variables used in the sniffer, the libpcap handle, the link header size, and the number of packets captured.  The `pcap` handle is a `pcap_t` pointer to a structure identifies the packet capture channel and is used in all the libpcap function calls.  The `linkhdrlen` will be used during packet capture and parsing to skip over the datalink layer header to get to the IP header of each packet.  Similarly the `packets` value will be incremented every time a packet is captured and processed.

### Main Function

The goal of the example packet sniffer application is to collect raw IP packets traversing a network and inspect their header and payload fields to determine protocol type, source address, destination address and so on. Let’s take a look at the `main()` function for the program:

{% highlight c linenos %}
int main(int argc, char *argv[])
{
    char device[256];
    char filter[256];
    int count = 0;
    int opt;
 
    *device = 0;
    *filter = 0;

    // Get the command line options, if any
    while ((opt = getopt(argc, argv, "hi:n:")) != -1)
    {
        switch (opt)
        {
        case 'h':
            printf("usage: %s [-h] [-i interface] [-n count] [BPF expression]\n", argv[0]);
            exit(0);
            break;
        case 'i':
            strcpy(device, optarg);
            break;
        case 'n':
            count = atoi(optarg);
            break;
        }
    }

    // Get the packet capture filter expression, if any.
    for (int i = optind; i < argc; i++)
    {
        strcat(filter, argv[i]);
        strcat(filter, " ");
    }

    signal(SIGINT, stop_capture);
    signal(SIGTERM, stop_capture);
    signal(SIGQUIT, stop_capture);
    
    // Create packet capture handle.
    handle = create_pcap_handle(device, filter);
    if (handle == NULL) {
        return -1;
    }

    // Get the type of link layer.
    get_link_header_len(handle);
    if (linkhdrlen == 0) {
        return -1;
    }

    // Start the packet capture with a set count or continually if the count is 0.
    if (pcap_loop(handle, count, packet_handler, (u_char*)NULL) < 0) {
        fprintf(stderr, "pcap_loop failed: %s\n", pcap_geterr(handle));
        return -1;
    }
    
    stop_capture(0);
}
{% endhighlight %}

### Top Level Functions

The `main()` function processes the command line arguments then relies on the following 4 functions to do the work:

- `create_pcap_handle()` – Created a packet capture endpoint to receive packets described by a packet capture filter.
- `get_link_header_len` – Gets the link header type and size that will be used during the packet capture and parsing.
- `packet_handler()` – Call back function that will parses and displays the contents of each captured packet.
- `stop_capture()` – Function called when the program is inteerrupted or ends to display the packet capture statistics.

The packet sniffer supports the following program options:

- `-i` specifies the network interface to use for packet capture, by default libpcap looks one up.
- `-n` specifies the total number of packets to capture, by default packets are captured indefinitely.
- `-h` causes the program to display a program usage reminder.

All other string arguments are presumed to be parts of a packet filter statement and are combined into a single string. If no packet filter is entered, then all IP packets are captured.

## Create a Packet Capture Endpoint

The libpcap calls to create a packet capture endpoint are encapsulated in the `create_pcap_handle()` function:

{% highlight c linenos %}
pcap_t* create_pcap_handle(char* device, char* filter)
{
    char errbuf[PCAP_ERRBUF_SIZE];
    pcap_t *handle = NULL;
    pcap_if_t* devices = NULL;
    struct bpf_program bpf;
    bpf_u_int32 netmask;
    bpf_u_int32 srcip;

    // If no network interface (device) is specfied, get the first one.
    if (!*device) {
    	if (pcap_findalldevs(&devices, errbuf)) {
            fprintf(stderr, "pcap_findalldevs(): %s\n", errbuf);
            return NULL;
        }
        strcpy(device, devices[0].name);
    }

    // Get network device source IP address and netmask.
    if (pcap_lookupnet(device, &srcip, &netmask, errbuf) == PCAP_ERROR) {
        fprintf(stderr, "pcap_lookupnet: %s\n", errbuf);
        return NULL;
    }

    // Open the device for live capture.
    handle = pcap_open_live(device, BUFSIZ, 1, 1000, errbuf);
    if (handle == NULL) {
        fprintf(stderr, "pcap_open_live(): %s\n", errbuf);
        return NULL;
    }

    // Convert the packet filter epxression into a packet filter binary.
    if (pcap_compile(handle, &bpf, filter, 0, netmask) == PCAP_ERROR) {
        fprintf(stderr, "pcap_compile(): %s\n", pcap_geterr(handle));
        return NULL;
    }

    // Bind the packet filter to the libpcap handle.
    if (pcap_setfilter(handle, &bpf) == PCAP_ERROR) {
        fprintf(stderr, "pcap_setfilter(): %s\n", pcap_geterr(handle));
        return NULL;
    }

    return handle;
}
{% endhighlight %}

**[Lines 11-17]** Network interfaces, or devices, are denoted by unique character strings referred to as network devices in the libpcap man page. For instance under Linux, Ethernet devices have the general form **ethN** where `N` == `0`, `1`, `2`, and so on depending on how many network interfaces a system contains. The first argument to `create_pcap_handle()` is the device string obtained from the program command line. If no device is specified `pcap_findalldevs()` is called to select a device. This function returnms a list of devices and the program just picks the first one.

**[Lines 20-23]** `pcap_open_live()` opens the selected network device for packet capture and returns a libpcap socket handle, if successful. The term **live** refers to the fact that packets will be read from an active network as opposed to a file containing packet data that were previously saved. The first argument to this function is the network device from which packets will be captured, the second sets the packet capture snap length, the third toggles promiscuous mode, the fourth sets a time out (if supported by the underlying OS), and the last is a pointer to the error message buffer. 

**[Lines 26-30]** `pcap_lookupnet()` returns the network address and subnet mask of the network where packets will be captured.The subnet mask is used to later in the call to compile the BPF filter. The last argument to this function is a pointer to the error message buffer.

Network traffic is analogous to radio broadcasts. Packets carrying a variety of protocol data are continually traversing busy networks just as radio waves are constantly transmitted into the atmosphere. To listen to a radio station you have to tune in to the transmission frequency of the desired station while ignoring all other frequencies. With libpcap you *tune in* to the packets you want to capture by describing the attributes of the desired packets in C-like statments called packet filters. Here are some filters examples and what packets they tell libpcap to grab:

- `tcp` – **TCP** packets
- `udp` - **UDP** packets
- `icmp` – **ICMP** packets
- `udp port 53` – **DNS** request and response packets
- `tcp port 80` - **HTTP** request and response packets

**[Lines 33-36]** `pcap_compile()` converts the packet filter string argument of `open_pcap_live()` to a filter program that libcap can interpret. The first argument to `pcap_compile()` is the libpcap socket handle, the second is a pointer to the packet filter string, the third is a pointer to an empty libpcap filter program structure, the fourth is a code optimization flag set to 1 and the last is a 32 bit pointer to the subnet mask obtained with `pcap_lookupnet()`. `pcap_geterr()` returns a message describing the most recent error.

**[Lines 39-42]** `pcap_setfilter()` associates the compiled packet filter program with the packet capture.

**[Line 44]** Return the intialized packet capture handle.

## Get Link Header Type and Size

{% highlight c linenos %}
void get_link_header_len(pcap_t* handle)
{
    int linktype;
 
    // Determine the datalink layer type.
    if ((linktype = pcap_datalink(handle)) == PCAP_ERROR) {
        fprintf(stderr, "pcap_datalink(): %s\n", pcap_geterr(handle));
        return;
    }
 
    // Set the datalink layer header size.
    switch (linktype)
    {
    case DLT_NULL:
        linkhdrlen = 4;
        break;
 
    case DLT_EN10MB:
        linkhdrlen = 14;
        break;
 
    case DLT_SLIP:
    case DLT_PPP:
        linkhdrlen = 24;
        break;
 
    default:
        printf("Unsupported datalink (%d)\n", linktype);
        linkhdrlen = 0;
    }
}
{% endhighlight %}

**[Lines 6-9]** Packets captured at the datalink layer are completely raw in the sense that they include the headers applied by all the network stack layers, including the datalink header. The packet sniffer is concerned only with IP packets, so when it comes time to parse each packet the code must skip over the datalink header contained in each packet to get to the start of the IP section. `pcap_datalink()` returns the number corresponding to the datalink type which corresponds to a specific datalink header length.

**[Lines 12-30]** The datalink header size in the `linkhdrlen` global variable. The datalink types supported include `loopback` (`DLT_NULL`), `Ethernet` (`DLT_EN10MB`), `SLIP` (`DLT_SLIP`) and `PPP` (`DLT_PPP`). If the datalink is none of these, set `linkhdrlen` to 0.

## Parse and Display Packet Fields

The general technique for parsing packets is to set a character pointer to the beginning of the packet buffer then advance this pointer to a particlular protocol header by the size in bytes of the headers that precede it in the packet. The header can then be mapped to a IP, TCP, UDP and ICMP header structure by casting the character pointer to a protocol specific structure pointer. From there any protocol header field can be referenced directly though the protocol structure pointer. This technique is used in the packet capture call back function:

{% highlight c linenos %}
void packet_handler(u_char *user, const struct pcap_pkthdr *packethdr, const u_char *packetptr)
{
    struct ip* iphdr;
    struct icmp* icmphdr;
    struct tcphdr* tcphdr;
    struct udphdr* udphdr;
    char iphdrInfo[256];
    char srcip[256];
    char dstip[256];
 
     // Skip the datalink layer header and get the IP header fields.
    packetptr += linkhdrlen;
    iphdr = (struct ip*)packetptr;
    strcpy(srcip, inet_ntoa(iphdr->ip_src));
    strcpy(dstip, inet_ntoa(iphdr->ip_dst));
    sprintf(iphdrInfo, "ID:%d TOS:0x%x, TTL:%d IpLen:%d DgLen:%d",
            ntohs(iphdr->ip_id), iphdr->ip_tos, iphdr->ip_ttl,
            4*iphdr->ip_hl, ntohs(iphdr->ip_len));
 
    // Advance to the transport layer header then parse and display
    // the fields based on the type of hearder: tcp, udp or icmp.
    packetptr += 4*iphdr->ip_hl;
    switch (iphdr->ip_p)
    {
    case IPPROTO_TCP:
        tcphdr = (struct tcphdr*)packetptr;
        printf("TCP  %s:%d -> %s:%d\n", srcip, ntohs(tcphdr->th_sport),
               dstip, ntohs(tcphdr->th_dport));
        printf("%s\n", iphdrInfo);
        printf("%c%c%c%c%c%c Seq: 0x%x Ack: 0x%x Win: 0x%x TcpLen: %d\n",
               (tcphdr->th_flags & TH_URG ? 'U' : '*'),
               (tcphdr->th_flags & TH_ACK ? 'A' : '*'),
               (tcphdr->th_flags & TH_PUSH ? 'P' : '*'),
               (tcphdr->th_flags & TH_RST ? 'R' : '*'),
               (tcphdr->th_flags & TH_SYN ? 'S' : '*'),
               (tcphdr->th_flags & TH_SYN ? 'F' : '*'),
               ntohl(tcphdr->th_seq), ntohl(tcphdr->th_ack),
               ntohs(tcphdr->th_win), 4*tcphdr->th_off);
        printf("+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+\n\n");
        packets += 1;
        break;
 
    case IPPROTO_UDP:
        udphdr = (struct udphdr*)packetptr;
        printf("UDP  %s:%d -> %s:%d\n", srcip, ntohs(udphdr->uh_sport),
               dstip, ntohs(udphdr->uh_dport));
        printf("%s\n", iphdrInfo);
        printf("+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+\n\n");
        packets += 1;
        break;
 
    case IPPROTO_ICMP:
        icmphdr = (struct icmp*)packetptr;
        printf("ICMP %s -> %s\n", srcip, dstip);
        printf("%s\n", iphdrInfo);
        printf("Type:%d Code:%d ID:%d Seq:%d\n", icmphdr->icmp_type, icmphdr->icmp_code,
               ntohs(icmphdr->icmp_hun.ih_idseq.icd_id), ntohs(icmphdr->icmp_hun.ih_idseq.icd_seq));
        printf("+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+\n\n");
        packets += 1;
        break;
    }
}
{% endhighlight %}

**[Lines 3-9]** `packet_handler()` starts off by defining pointers to IP, TCP, UDP and ICMP header structures. Character buffers are included for storing header fields that will be displayed to stdout. 

**[Lines 12-19]** Advance the packet pointer past the datalink header by the number of bytes corresponding to the datalink type determined in `capture_loop()`. The packet pointer contains the address of the first byte of the IP header where it is cast it to a `struct ip` pointer to extract the packet id, time to live, IP header length and total IP packet length (including header). These values are placed into a single character buffer for display later. Since 2 and 4 byte header fields for all Internet protocols are in big endian format, `ntohs()` and `ntohl()` are called to correct the byte ordering on little endian systems. Then the packet pointer is advanced past the IP header so that it points to the IP payload. The protocol of the payload is obtained from the `ip_p` field in the switch statement to jump to a section of code designed to handle the protocol. 

**[Lines 22]** Advance the packet pointer past the IP header to point to the first byte of the transport layer payload.

**[Lines 25-50]** Casting the packet pointer to `struct tcphdr` and `struct udphdr` pointers enables access to TCP and UDP header fields, respectively. In both cases the source IP address and port are displayed with an arrow pointing to the destination IP address and port, followed by the TCP segment flags, sequence and acknowledgment numbers, window advertisement, and TCP segment length. The `packets` variable is incremeted for both TCP and UDP.

**[Lines 52-60]** The `struct icmp` pointer enables us to display ICMP packet type and code along with the source and destination IP addresses. The `packets` variable is incremeted for ICMP.

## Initiate Packet Capture

Libpcap provides three functions to capture packets: `pcap_next()`, `pcap_dispatch()`, and `pcap_loop()`. The first function grabs 1 packet at a time so the programmer must call it in a loop to receive multiple packets. The other 2 loop automatically to receive multiple packets and call a user supplied call back function to process each one. The packet sniffer in this example uses `pcap_loop()`, included in lines 52 through 55 of `main()` intiate the packet capture: 

{% highlight c %}
    // Start the packet capture with a set count or continually if the count is 0.
    if (pcap_loop(handle, count, packet_handler, (u_char*)NULL) < 0) {
    	fprintf(stderr, "pcap_loop failed: %s\n", pcap_geterr(handle));
	    return -1;
    }
{% endhighlight %}

- `handle` - The handle to the packet capture endpoint.
- `count` - Contains the number of packets to capture specified on the command line with option `-n count`. If none is specified `count` is 0 which causes packets to be captured indefinitely.
- `packet_handler` - Call back functions which processes each packet.

## Packet Capture Termination

The `SIGINT`, `SIGTERM` and `SIGQUIT` interrupt signals are set to call the function `stop_capture()` which displays the packet count, closes the packet capture socket then exits the program. The call to `pcap_stats()` fills a `pcap_stats` structure that contains fields indicating how many incoming and outgoing packets were captures and how many incoming packets were dropped. The call to `pcap_close()` closed the packet capture socket.

{% highlight c %}
void stop_capture(int signo)
{
    struct pcap_stat stats;

    if (pcap_stats(pd, &stats) >= 0) {
        printf("%d packets received\n", stats.ps_recv);
        printf("%d packets dropped\n\n", stats.ps_drop);
    }
    pcap_close(pd);
    exit(0);
}
{% endhighlight %}

## Build and Run the Sniffer

You can get the source code for the project from Github – [https://github.com/vichargrave/sniffer](https://github.com/vichargrave/sniffer){:target="_blank"}. To build it just _cd_ into the project directory and type _make_.

To test the sniffer, capture some packets starting with each of the supported protocols, starting with ICMP. Run the command `ping 8.8.8.8` then run the sniffer to capture 4 ICMP packets. 

{% highlight bash %}
$ sudo ./sniffer -n 4 icmp
CMP 192.168.1.35 -> 8.8.8.8
ID:19160 TOS:0x0, TTL:64 IpLen:20 DgLen:84
Type:8 Code:0 ID:34843 Seq:0
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

ICMP 8.8.8.8 -> 192.168.1.35
ID:0 TOS:0x0, TTL:116 IpLen:20 DgLen:84
Type:0 Code:0 ID:34843 Seq:0
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

ICMP 192.168.1.35 -> 8.8.8.8
ID:60421 TOS:0x0, TTL:64 IpLen:20 DgLen:84
Type:8 Code:0 ID:34843 Seq:1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

ICMP 8.8.8.8 -> 192.168.1.35
ID:0 TOS:0x0, TTL:116 IpLen:20 DgLen:84
Type:0 Code:0 ID:34843 Seq:1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


4 packets captured
8 packets received by filter
0 packets dropped
{% endhighlight %}

To capture some UDP packets, send a DNS request with the command `nslookup httpforever.com`. Run the sniffer application to get UDP packets on port 53. After capturing a couple of packets, interrupt it by entering Ctrl-C.

{% highlight bash %}
$ sudo ./sniffer udp port 53
UDP  192.168.1.35:57670 -> 192.168.1.1:53
ID:35650 TOS:0x0, TTL:64 IpLen:20 DgLen:61
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

UDP  192.168.1.1:53 -> 192.168.1.35:57670
ID:11593 TOS:0x0, TTL:64 IpLen:20 DgLen:93
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

^C
2 packets captured
34 packets received by filter
0 packets dropped
{% endhighlight %}

Finally, capture some TCP packets by opening a browser to `http://httpforever.com`.  Run the sniffer to capture 4 TCP packets.

{% highlight bash %}
$ sudo sniffer -n 4 tcp and host httpforever.com
TCP  192.168.1.35:63560 -> 172.67.181.181:80
ID:0 TOS:0x0, TTL:64 IpLen:20 DgLen:64
****SF Seq: 0x29e705e9 Ack: 0x0 Win: 0xffff TcpLen: 44
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

TCP  192.168.1.35:63561 -> 172.67.181.181:80
ID:0 TOS:0x0, TTL:64 IpLen:20 DgLen:64
****SF Seq: 0x997ace8c Ack: 0x0 Win: 0xffff TcpLen: 44
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

TCP  172.67.181.181:80 -> 192.168.1.35:63560
ID:0 TOS:0x0, TTL:52 IpLen:20 DgLen:52
*A**SF Seq: 0xf903f657 Ack: 0x29e705ea Win: 0xffff TcpLen: 32
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

TCP  192.168.1.35:63560 -> 172.67.181.181:80
ID:0 TOS:0x0, TTL:64 IpLen:20 DgLen:40
*A**** Seq: 0x29e705ea Ack: 0xf903f658 Win: 0x1000 TcpLen: 20
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


4 packets captured
315 packets received by filter
0 packets dropped
{% endhighlight %}


