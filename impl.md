# {{ page.title }}

So far, we have talked about what switches and routers must do without
discussing how to do it. There is a very simple way to build a switch or
router: Buy a general-purpose processor and equip it with a number of
network interfaces. Such a device, running suitable software, can
receive packets on one of its interfaces, perform any of the switching
or forwarding functions described above, and send packets out another of
its interfaces. This is, in fact, a popular way to build experimental
routers and switches when you want to be able to do things like develop
new routing protocols because it offers extreme flexibility and a
familiar programming environment. It is also not too far removed from
the architecture of many commercial mid- to low-end routers.

## Switch Basics

Switches and routers use similar implementation techniques, so we'll
start this section by looking at those common techniques, then move on
to look at the specific issues affecting router implementation in the
next section. For most of this section, we'll use the word *switch* to
cover both types of devices, since their internal designs are so similar
(and it's tedious to say "switch or router" all the time).

<figure class="line">
	<a id="softswitch"></a>
	<img src="figures/f03-37-9780123850591.png" width="500px"/>
	<figcaption>A general-purpose processor used as a packet
	switch.</figcaption>
</figure>
	
[Figure 1](#softswitch) shows a processor with three network
interfaces used as a switch. The figure shows a path that a packet
might take from the time it arrives on interface 1 until it is output
on interface 2. We have assumed here that the processor has a
mechanism to move data directly from an interface to its main memory
without having to be directly copied by the CPU, a technique called
*direct memory access* (DMA). Once the packet is in memory, the CPU
examines its header to determine which interface the packet should be
sent out on. It then uses DMA to move the packet out to the
appropriate interface. Note that [Figure 1](#softswitch) does
not show the packet going to the CPU because the CPU inspects only the
header of the packet; it does not have to read every byte of data in
the packet.

The main problem with using a general-purpose processor as a switch is
that its performance is limited by the fact that all packets must pass
through a single point of contention: In the example shown, each packet
crosses the I/O bus twice and is written to and read from main memory
once. The upper bound on aggregate throughput of such a device (the
total sustainable data rate summed over all inputs) is, thus, either
half the main memory bandwidth or half the I/O bus bandwidth, whichever
is less. (Usually, it's the I/O bus bandwidth.) For example, a machine
with a 133-MHz, 64-bit-wide I/O bus can transmit data at a peak rate of
a little over 8 Gbps. Since forwarding a packet involves crossing the
bus twice, the actual limit is 4 Gbps—enough to build a switch with a
fair number of 100-Mbps Ethernet ports, for example, but hardly enough
for a high-end router in the core of the Internet.

Moreover, this upper bound also assumes that moving data is the only
problem—a fair approximation for long packets but a bad one when
packets are short. In the latter case, the cost of processing each
packet—parsing its header and deciding which output link to transmit
it on—is likely to dominate. Suppose, for example, that a processor
can perform all the necessary processing to switch 2 million packets
each second. This is sometimes called the packet per second (pps) rate.
(This number is representative of what is achievable on an inexpensive
PC.) If the average packet is short, say, 64 bytes, this would imply

{% center %} Throughput = pps x BitsPerPacket {% endcenter %}

$$
= 2 \times 10^6 \times 64 \times 8
$$

$$
= 1024 \times 10^6
$$

that is, a throughput of about 1 Gbps—substantially below the range
that users are demanding from their networks today. Bear in mind that
this 1 Gbps would be shared by all users connected to the switch, just
as the bandwidth of a single (unswitched) Ethernet segment is shared
among all users connected to the shared medium. Thus, for example, a
20-port switch with this aggregate throughput would only be able to cope
with an average data rate of about 50 Mbps on each port.

To address this problem, hardware designers have come up with a large
array of switch designs that reduce the amount of contention and provide
high aggregate throughput. Note that some contention is unavoidable: If
every input has data to send to a single output, then they cannot all
send it at once. However, if data destined for different outputs is
arriving at different inputs, then a well-designed switch will be able to
move data from inputs to outputs in parallel, thus increasing the
aggregate throughput.

<figure class="line">
	<a id="portfab"></a>
	<img src="figures/f03-38-9780123850591.png" width="500px"/>
	<figcaption>A 4 $$\times$$ 4 switch.</figcaption>
</figure>

## Ports

Most switches look conceptually similar to the one shown in
[Figure 2](#portfab). They consist of a number of *input* and *output
ports* and a *fabric*. There is usually at least one control processor
in charge of the whole switch that communicates with the ports either
directly or, as shown here, via the switch fabric. The ports communicate
with the outside world. They may contain fiber optic receivers and
lasers, buffers to hold packets that are waiting to be switched or
transmitted, and often a significant amount of other circuitry that
enables the switch to function. The fabric has a very simple and
well-defined job: When presented with a packet, deliver it to the right
output port.

One of the jobs of the ports, then, is to deal with the complexity of
the real world in such a way that the fabric can do its relatively
simple job. For example, suppose that this switch is supporting a
virtual circuit model of communication. In general, the virtual circuit
mapping tables are located in the ports. The ports maintain lists of
virtual circuit identifiers that are currently in use, with information
about what output a packet should be sent out on for each VCI and how
the VCI needs to be remapped to ensure uniqueness on the outgoing link.
Similarly, the ports of an Ethernet switch store tables that map between
Ethernet addresses and output ports (bridge forwarding tables). In
general, when a packet is handed from an input port to the fabric, the
port has figured out where the packet needs to go, and either the port
sets up the fabric accordingly by communicating some control information
to it, or it attaches enough information to the packet itself (e.g., an
output port number) to allow the fabric to do its job automatically.
Fabrics that switch packets by looking only at the information in the
packet are referred to as *self-routing*, since they require no external
control to route packets. An example of a self-routing fabric is
discussed below.

The input port is the first place to look for performance bottlenecks.
The input port has to receive a steady stream of packets, analyze
information in the header of each one to determine which output port (or
ports) the packet must be sent to, and pass the packet on to the fabric.
The type of header analysis that it performs can range from a simple
table lookup on a VCI to complex matching algorithms that examine many
fields in the header. This is the type of operation that sometimes
becomes a problem when the average packet size is very small. Consider,
for example, 64-byte packets arriving on a port connected to an OC-48
(2.48 Gbps) link. Such a port needs to process packets at a rate of

$$
2.48 \times 10^9 / (64 \times 8) = 4.83 \times 10^6\ pps
$$

In other words, when small packets are arriving as fast as possible on
this link (the worst-case scenario that most ports are engineered to
handle), the input port has approximately 200 nanoseconds to process
each packet.

Another key function of ports is buffering. Observe that buffering can
happen in either the input or the output port; it can also happen within
the fabric (sometimes called *internal buffering*). Simple input
buffering has some serious limitations. Consider an input buffer
implemented as a FIFO. As packets arrive at the switch, they are placed
in the input buffer. The switch then tries to forward the packets at the
front of each FIFO to their appropriate output port. However, if the
packets at the front of several different input ports are destined for
the same output port at the same time, then only one of them can be
forwarded; the rest must stay in their input buffers.

> For a simple input-buffered switch, exactly one packet at a time 
> can be sent to a given output port. It is possible to design 
> switches that can forward more than one packet to the same output at 
> once, at a cost of higher switch complexity, but there is always 
> some upper limit on the number. 

<figure class="line">
	<a id="hol"></a>
	<img src="figures/f03-39-9780123850591.png" width="400px"/>
	<figcaption>Simple illustration of head-of-line blocking.</figcaption>
</figure>
	
 The drawback of this feature is that those packets left at the front of
the input buffer prevent other packets further back in the buffer from
getting a chance to go to their chosen outputs, even though there may be
no contention for those outputs. This phenomenon is called *head-of-line
blocking*. A simple example of head-of-line blocking is given in
[Figure 3](#hol), where we see a packet destined for port 1 blocked
behind a packet contending for port 2. It can be shown that when traffic
is uniformly distributed among outputs, head-of-line blocking limits the
throughput of an input-buffered switch to 59% of the theoretical maximum
(which is the sum of the link bandwidths for the switch). Thus, the
majority of switches use either pure output buffering or a mixture of
internal and output buffering. Those that do rely on input buffers use
more advanced buffer management schemes to avoid head-of-line blocking.

Buffers actually perform a more complex task than just holding onto
packets that are waiting to be transmitted. Buffers are the main source
of delay in a switch, and also the place where packets are most likely
to get dropped due to lack of space to store them. The buffers therefore
are the main place where the quality of service characteristics of a
switch are determined. For example, if a certain packet has been sent
along a VC that has a guaranteed delay, it cannot afford to sit in a
buffer for very long. This means that the buffers, in general, must be
managed using packet scheduling and discard algorithms that meet a wide
range of QoS requirements.

## Fabrics

While there has been an abundance of impressive research conducted on
the design of efficient and scalable fabrics, it is sufficient for our
purposes here to understand only the high-level properties of a switch
fabric. A switch fabric should be able to move packets from input ports
to output ports with minimal delay and in a way that meets the
throughput goals of the switch. That usually means that fabrics display
some degree of parallelism. A high-performance fabric with n ports can
often move one packet from each of its n ports to one of the output
ports at the same time. A sample of fabric types includes the following:

<figure class="line">
	<a id="xbar"></a>
	<img src="figures/f03-40-9780123850591.png" width="400px"/>
	<figcaption>A 4 $$\times$$ 4 crossbar switch.</figcaption>
</figure>
	
- *Shared Bus*—This is the type of "fabric" found in a conventional
    processor used as a switch, as described above. Because the bus
    bandwidth determines the throughput of the switch, high-performance
    switches usually have specially designed busses rather than the
    standard busses found in PCs.

- *Shared Memory*—In a shared memory switch, packets are written
    into a memory location by an input port and then read from memory by
    the output ports. Here it is the memory bandwidth that determines
    switch throughput, so wide and fast memory is typically used in this
    sort of design. A shared memory switch is similar in principle to
    the shared bus switch, except it usually uses a specially designed,
    high-speed memory bus rather than an I/O bus.

- *Crossbar*—A crossbar switch is a matrix of pathways that can be
    configured to connect any input port to any output port.
    [Figure 4](#xbar) shows a 4 $$\times$$ 4 crossbar switch. The main
    problem with crossbars is that, in their simplest form, they require
    each output port to be able to accept packets from all inputs at
    once, implying that each port would have a memory bandwidth equal to
    the total switch throughput. In reality, more complex designs are
    typically used to address this issue (see, for example, the Knockout
    switch and McKeown's virtual output-buffered approach in the Further
    Reading section.)

- *Self-routing*—As noted above, self-routing fabrics rely on some
    information in the packet header to direct each packet to its
    correct output. Usually a special "self-routing header" is appended
    to the packet by the input port after it has determined which output
    the packet needs to go to, as illustrated in
    [Figure 5](#self-route); this extra header is removed before the
    packet leaves the switch. Self-routing fabrics are often built from
    large numbers of very simple 2 $$\times$$ 2 switching elements
    interconnected in regular patterns, such as the *banyan* switching
    fabric shown in[Figure 6](#banyaneg).

<figure class="line">
	<a id="self-route"></a>
	<img src="figures/f03-41-9780123850591.png" width="500px"/>
	<figcaption>A self-routing header is applied to a packet at input
	to enable the fabric to send the packet to the correct output,
	where it is removed: (a) Packet arrives at input port; (b) input
	port attaches self-routing header to direct packet to correct
	output; (c) self-routing header is removed at output port before
	packet leaves switch.</figcaption>	
</figure>
	
<figure class="line">
	<a id="banyaneg"></a>
	<img src="figures/f03-42-9780123850591.png" width="400px"/>
	<figcaption>Routing packets through a banyan network. The 3-bit
	numbers represent values in the self-routing headers of four
	arriving packets.</figcaption>
</figure>
	
 Self-routing fabrics are among the most scalable approaches to fabric
design, and there has been a wealth of research on the topic, some of
which is listed in the Further Reading section. Many self-routing
fabrics resemble the one shown in [Figure 6](#banyaneg), consisting
of regularly interconnected 2 $$\times$$ 2 switching elements. For example,
the 2 $$\times$$ 2 switches in the banyan network perform a simple task:
They look at 1 bit in each self-routing header and route packets toward
the upper output if it is zero or toward the lower output if it is one.
Obviously, if two packets arrive at a banyan element at the same time
and both have the bit set to the same value, then they want to be routed
to the same output and a collision will occur. Either preventing or
dealing with these collisions is a main challenge for self-routing
switch design. The banyan network is a clever arrangement of
2 $$\times$$ 2 switching elements that routes all packets to the correct
output without collisions if the packets are presented in ascending
order.

We can see how this works in an example, as shown in
[Figure 6](#banyaneg), where the self-routing header contains the
output port number encoded in binary. The switch elements in the first
column look at the most significant bit of the output port number and
route packets to the top if that bit is a 0 or the bottom if it is a 1.
Switch elements in the second column look at the second bit in the
header, and those in the last column look at the least significant bit.
You can see from this example that the packets are routed to the correct
destination port without collisions. Notice how the top outputs from the
first column of switches all lead to the top half of the network, thus
getting packets with port numbers 0 to 3 into the right half of the
network. The next column gets packets to the right quarter of the
network, and the final column gets them to the right output port. The
clever part is the way switches are arranged to avoid collisions. Part
of the arrangement includes the "perfect shuffle" wiring pattern at the
start of the network. To build a complete switch fabric around a banyan
network would require additional components to sort packets before they
are presented to the banyan. The Batcher-banyan switch design is a
notable example of such an approach. The Batcher network, which is also
built from a regular interconnection of 2 $$\times$$ 2 switching elements,
sorts packets into descending order. On leaving the Batcher network, the
packets are then ready to be directed to the correct output, with no
risk of collisions, by the banyan network.

One of the interesting things about switch design is the wide range of
different types of switches that can be built using the same basic
technology. For example, Ethernet switches, ATM switches, and Internet
routers, discussed below, have all been built using designs such as
those outlined in this section.

## Router Implementation

We have now seen a variety of ways to build a switch, ranging from a
general-purpose processor with a suitable number of network interfaces
to some sophisticated hardware designs. In general, the same range of
options is available for building routers, many of which look something
like [Figure 7](#router-imp). The control processor is
responsible for running the routing protocols discussed above, among
other things, and generally acts as the central point of control of
the router. The switching fabric transfers packets from one port to
another, just as in a switch; and the ports provide a range of
functionality to allow the router to interface to links of various
types (e.g., Ethernet, SONET).

<figure class="line">
	<a id="router-imp"></a>
	<img src="figures/f03-43-9780123850591.png" width="500px"/>
	<figcaption>Block diagram of a router.</figcaption>
</figure>
	
 A few points are worth noting about router design and how it differs
from switch design. First, routers must be designed to handle
variable-length packets, a constraint that does not apply to ATM
switches but is certainly applicable to Ethernet or Frame Relay
switches. It turns out that many high-performance routers are designed
using a switching fabric that is cell based. In such cases, the ports
must be able to convert variable-length packets into cells and back
again. This is known as *segmentation and re-assembly* (SAR), a problem
also faced by network adaptors for ATM networks.

Another consequence of the variable length of IP datagrams is that it
can be harder to characterize the performance of a router than a switch
that forwards only cells. Routers can usually forward a certain number
of packets per second, and this implies that the total throughput in
*bits* per second depends on packet size. Router designers generally
have to make a choice as to what packet length they will support at
*line rate*. That is, if 'pps' (packets per second) is the rate at which
packets arriving on a particular port can be forwarded, and 'linerate'
is the physical speed of the port in bits per second, then there will be
some 'packetsize' in bits such that:

{% center %} packetsize x pps = linerate {% endcenter %}

This is the packet size at which the router can forward at line rate; it
is likely to be able to sustain line rate for longer packets but not for
shorter packets. Sometimes a designer might decide that the right packet
size to support is 40 bytes, since that is the minimum size of an IP
packet that has a TCP header attached. Another choice might be the
expected *average* packet size, which can be determined by studying
traces of network traffic. For example, measurements of the Internet
backbone suggest that the average IP packet is around 300 bytes long.
However, such a router would fall behind and perhaps start dropping
packets when faced with a long sequence of short packets, which is
statistically likely from time to time and also very possible if the
router is subject to an active attack. Design decisions of this type
depend heavily on cost considerations and the intended application of
the router.

When it comes to the task of forwarding IP packets, routers can be
broadly characterized as having either a *centralized* or *distributed*
forwarding model. In the centralized model, the IP forwarding algorithm,
outlined earlier in this chapter, is done in a single processing engine
that handles the traffic from all ports. In the distributed model, there
are several processing engines, perhaps one per port, or more often one
per line card, where a line card may serve one or more physical ports.
Each model has advantages and disadvantages. All things being equal, a
distributed forwarding model should be able to forward more packets per
second through the router as a whole, because there is more processing
power in total. But a distributed model also complicates the software
architecture, because each forwarding engine typically needs its own
copy of the forwarding table, and thus it is necessary for the control
processor to ensure that the forwarding tables are updated consistently
and in a timely manner.

Another aspect of router implementation that is significantly different
from that of switches is the IP forwarding algorithm itself. In bridges
and most ATM switches, the forwarding algorithm simply involves looking
up a fixed-length identifier (MAC address or VCI) in a table, finding
the correct output port in the table, and sending the packet to that
port. We have already seen that the IP forwarding algorithm is a little
more complicated than that, in part because the relevant number of bits
that need to be examined when forwarding a packet is not fixed but
variable, typically ranging from 8 bits to 32 bits.

Because of the relatively high complexity of the IP forwarding
algorithm, there have been periods of time when it seemed IP routers
might be running up against fundamental upper limits of performance.
However, as we discuss in the Further Reading section of this chapter,
there have been many innovative approaches to IP forwarding developed
over the years, and at the time of writing there are commercial routers
that can forward 40 Gbps of IP traffic *per interface*. By combining
many such high-performance IP forwarding engines with the sort of very
scalable switch fabrics, it has now become possible to build routers
with many terabits of total throughput. That is more than enough to see
us through the next few years of growth in Internet traffic.

Another technology of interest in the field of router implementation is
the *network processor*. A network processor is intended to be a device
that is just about as programmable as a standard PC processor, but that
is more highly optimized for networking tasks. For example, a network
processor might have instructions that are particularly well suited to
performing lookups on IP addresses, or calculating checksums on IP
datagrams. Such devices could be used in routers and other networking
devices (e.g., firewalls).

One of the interesting and ongoing debates about network processors is
whether they can do a better job than the alternatives. For example,
given the continuous and remarkable improvements in performance of
conventional processors, and the huge industry that drives those
improvements, can network processors keep up? And can a device that
strives for generality do as good a job as a custom-designed
application-specific integrated circuit (ASIC) that does nothing except,
say, IP forwarding? Part of the answer to questions like these depends
on what you mean by "do a better job." For example, there will always be
tradeoffs to be made between cost of hardware, time to market,
performance, power consumption, and flexibility—the ability to change
the features supported by a router after it is built. We will see in
later chapters just how diverse the requirements for router
functionality can be. It is safe to assume that a wide range of router
designs will exist for the foreseeable future and that network
processors will have some role to play.

