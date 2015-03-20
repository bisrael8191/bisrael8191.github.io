---
layout: post
title: Go Packet Sniffer
excerpt: "Packet sniffing and decoding in Go using gopacket"
tags: [golang, gopacket, ip, tcp, sniffer]
modified: 2015-03-17
comments: true
share: false
---

One of my coworkers wrote an exellent post a while back about writing a 
[Reasonably Fast Python IP Sniffer](https://askldjd.wordpress.com/2014/01/15/a-reasonably-fast-python-ip-sniffer/)
in Python. We were able to refactor it and use it in a very successful project
at work, but we ended up implementing a very small, focused subset of [scapy](http://www.secdev.org/projects/scapy/)
in order to make it work. This also included a very basic [packet decoder](https://github.com/bisrael8191/tcp-fragmentation)
that only supported the layers we cared about and handled fragmentation and checksumming.
I relearned a lot about the networking stack during this project, but always felt that
there was a better way. At the end of my friend's post, he mentions that if we wanted
to make the sniffer faster, we could always write it in C. I'm sure that would work ...
but it wouldn't give me a good reason to learn [Go](http://golang.org/) (which has been at
the top of my 'languages I should get around to looking into' list).

## Learning Go
Go turned out to be fairly easy to get a handle on the basics. The main [Go](http://golang.org/)
site has great documentation and a nice REPL playground to test out small examples live.
I also found the [How I Start: Go](https://howistart.org/posts/go/1) and 
[Go: Best Practices for Production Environments](http://peter.bourgon.org/go-in-production/)
posts (written by the same person) to be useful and include some nice tips and tricks.
Overall, based on the short experience I've had with Go so far, it gave me the 'Hey, this is kind of fun'
feeling that I got when first learning Python or NodeJS and less like the 'OMG, my code never just works'
feeling of Java/C/C++/Assembly. Time will tell, but I do plan on trying to use Go more at home and possibly work.

## Code
The code is pretty simple and essentially a wrapper around the [gopacket](https://github.com/google/gopacket) library.
This made it closer to my friend's [first example](http://pastebin.com/SJwWikZz) that is a one-liner in scapy. The
important section of the code looks something like:

{% highlight go %}
// Capture settings
const (
    // Ethernet interface
    iface string = "eth0"
    // Max packet length
    snaplen int32 = 65536
    // Set the interface in promiscuous mode
    promisc bool = true
    // Timeout duration 
    flushAfter string = "10s"
    // BPF filter when capturing packets
    filter string = "ip"
)
flushDuration, _ := time.ParseDuration(flushAfter)
handle, _ := pcap.OpenLive(iface, snaplen, promisc, flushDuration/2)
{% endhighlight %}

I use the BPF filter to ensure that I only get IP-based packets and then use gopacket's 
[DecodingLayerParser](https://godoc.org/github.com/google/gopacket#hdr-Fast_Decoding_With_DecodingLayerParser)
to only decode the layers that I'm interested in. It is also much faster because you preallocate the layer objects
ahead of time. If you look at the code, I also make use of the 
[ZeroCopyPacketDataSource](https://godoc.org/github.com/google/gopacket#ZeroCopyPacketDataSource)
function in my read loop to minimize the number of copies and copies of slices that are made of each packet.

{% highlight go %}
// Layers that we care about decoding
var (
    eth layers.Ethernet
    ip layers.IPv4
    tcp layers.TCP
    udp layers.UDP
    icmp layers.ICMPv4
    dns layers.DNS
    payload gopacket.Payload
)
	
// Array to store which layers were decoded
decoded := []gopacket.LayerType{}

// Faster, predefined layer parser that doesn't make copies of the layer slices
parser := gopacket.NewDecodingLayerParser(
    layers.LayerTypeEthernet,
    &eth,
    &ip,
    &tcp,
    &udp,
    &icmp,
    &dns,
    &payload)
{% endhighlight %}

I also added a slightly different version of the sniffer based on the [AFPacket](https://godoc.org/github.com/google/gopacket/afpacket)
library that uses a MMAP ring buffer to more efficiently pass packets out of kernel space. 
[PacketBeat](http://packetbeat.com/blog/sniffing-performance-and-ipv6.html) has a good article showing
the performance difference between their libpcap and afpacket versions. They also have a 
[fork of gopacket](https://github.com/packetbeat/gopacket) with many improvements (BPF) to the afpacket 
part of the library that will hopefully be merged back into the original library.

Check out the Go Packet Sniffer [code on Github](https://github.com/bisrael8191/sniffer)!

## Performance
*All test were done in an Linux Mint 17.1 VM on a Win8.1 host [Intel Core i7-4500U (1.80GHz 1600MHz 4MB), 8GB DDR3L memory]
while downloading the Linux Mint ISO.* The http://reflection.oss.ou.edu mirror gave me a very stable 4-5 MB/s download speed
during each of the test runs.

Up first is the original [Scapy implementation](http://pastebin.com/SJwWikZz):
<figure>
	<a href="/images/2015-03-17-Go-Packet_Sniffer_scapyperf.png"><img src="/images/2015-03-17-Go-Packet_Sniffer_scapyperf.png"></a>
	<center><figcaption>Scapy Performance</figcaption></center>
</figure>

Then the [Faster Python implementation](http://pastebin.com/0PKj0aSB):
<figure>
	<a href="/images/2015-03-17-Go-Packet_Sniffer_pythonperf.png"><img src="/images/2015-03-17-Go-Packet_Sniffer_pythonperf.png"></a>
	<center><figcaption>Faster Python Performance</figcaption></center>
</figure>

Now for the new versions, [Go with libpcap](https://github.com/bisrael8191/sniffer/blob/master/pcap.go)
using the command `./sniffer`:
<figure>
	<a href="/images/2015-03-17-Go-Packet_Sniffer_gopcapperf.png"><img src="/images/2015-03-17-Go-Packet_Sniffer_gopcapperf.png"></a>
	<center><figcaption>Go Pcap Performance</figcaption></center>
</figure>

And finally, [Go with AFPacket](https://github.com/bisrael8191/sniffer/blob/master/afpacket.go)
using the command `./sniffer -enableAf`:
<figure>
	<a href="/images/2015-03-17-Go-Packet_Sniffer_goafperf.png"><img src="/images/2015-03-17-Go-Packet_Sniffer_goafperf.png"></a>
	<center><figcaption>Go AFPacket Performance</figcaption></center>
</figure>

## Finale
The Scapy version is still clearly not going to handle traffic in a production environment. The implementation only takes advantage
of one CPU core and the graph shows that it uses 100% of it. The faster python version is a great improvement, only using ~15% of
CPU but it also only decodes the IP layer to get the source and destination IP addresses. Any further decoding has to be done later
in the pipeline. The Go versions use a little more memory (0.7% vs 0.4%) but only use between 5-10% CPU. The cool part is that both
versions do full packet decoding at the same time, so there is no need to write extra custom decoders. Gopacket even does proper
checksumming and looks like it can do fragmentation fairly easily. There are still a couple oddities that I found during testing.
First, the libpcap version seems to be a little more efficient, which goes against the PacketBeat research. The second is that
there were some pretty big CPU spikes in both Go versions. I haven't learned enough about Go's garbage collection yet but it could
be related to when it runs. I also need to learn more about profiling Go, using [pprof](https://golang.org/pkg/net/http/pprof/)
or something else, in order to really track down the root cause.

Hope you found this as interesting as I did and that it helps someone out!
