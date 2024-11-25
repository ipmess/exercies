# GRE

In this exercise, we will configure a point-to-point GRE tunnel between two site. Before starting with the configuration, you can get an introduction to GRE from the [Generic routing encapsulation Wikipedia page](https://en.wikipedia.org/wiki/Generic_routing_encapsulation).

## Step 0: Setup

For our purposes, you must first setup the following network:

![Network Topology](./images/GRE.underlay.svg)

At the end, PC1 should not be able to ping PC2, but there should not be any routes to the 10.0.0.0/24 or 10.0.1.0/24 subnets in either the R3, nor the R4 routers.

The R3 and R4 routers represent Internet routers. Internet routers will not accept RFC 1918 subnets, and they will not route packets destined to any destination in the 10.0.0.0/8 network. To simulate this Internet behavior , do not advertise the internal subnets (10.0.0.0/24 and 10.0.1.0/24) to R3 or R4.  

You should also enable OSPF between R1, R3, R4, and R2, but do not advertise the 10.0.0.0/24 or 10.0.1.0/24 subnets into OSPF.

To verify that the initial setup is operational, you should be able to ping 15.7.8.1 (R1's IP address) from R2, sourced from the 8.4.2.2 (R2's IP address).

## Step 1: The Tunnel

Now you must setup a tunnel between R1 and R2. The tunnel will be part of the internal, private address space. Configure a GRE tunnel between R1 and R2, as shown below:

![GRE tunnel Topology](./images/GRE-topo.svg)

At the end, you should be able to ping R1's tunnel interface (10.1.12.1) from R2, and vice versa. PC1 should not be able to ping PC2, yet.

### Routing though the tunnel

To route through the tunnel, R2 should learn about 10.0.0.0/24 and R1 about 10.0.1.0/24. There are two ways for these two things to happen:

* Static routes
* Dynamic routing

## Step 2: Static routing through the tunnel

For now, configure static routes on R1 and R2, to point to each other's tunnel interface.

## Step 3: Dynamic routing through the tunnel.

In this exercise's last step, configure dynamic routing (OSPF) between R1 and R2, so they can advertise their respective local subnets to each other. Note that R3 and R4 should not learn about the three internal networks (10.0.0.0/24, 10.0.1.0/24, 10.1.12.0/30).

## Solution

For my simulation, I used GNS3. In my setup, the interface names are shown in the diagram below:

![Network Topology, including my interface names](./images/GRE.underlay.wi.svg)

### Setup

**R1** initial configuration:

~~~~~~~~
hostname R1
!
interface Ethernet0/0
 ip address 15.7.8.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
interface Ethernet0/1
 ip address 10.0.0.254 255.255.255.0
 no shutdown
~~~~~~~~

**R2** initial configuration:

~~~~~~~~
hostname R2
!
interface Ethernet0/0
 ip address 8.4.2.2 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
interface Ethernet0/1
 ip address 10.0.1.254 255.255.255.0
 no shutdown
~~~~~~~~

**R3** initial configuration:

~~~~~~~~
hostname R3
!
interface Ethernet0/0
 ip address 15.7.8.2 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
interface Ethernet0/1
 ip address 115.3.4.2 255.255.255.252
 ip ospf 1 area 0
 no shutdown
~~~~~~~~

**R4** initial configuration:

~~~~~~~~
hostname R4
!
interface Ethernet0/0
 ip address 8.4.2.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
interface Ethernet0/1
 ip address 115.3.4.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
~~~~~~~~

Let us now verify that we can ping R1 from R2:

~~~~~~~~
R1#ping 8.4.2.2 source Ethernet 0/1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.4.2.2, timeout is 2 seconds:
Packet sent with a source address of 10.0.0.254 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
~~~~~~~~

### GRE tunnel setup

To create a GRE tunnel in Cisco IOS, you need to:

1. create a Tunnel interface, and 
2. provide the source and destination IP addresses for the GRE tunnel,
3. give the GRE tunnel interface an IP address:

Create the GRE tunnel endpoint on **R1**:

~~~~~~~~
interface Tunnel1
 tunnel source 15.7.8.1
 tunnel destination 8.4.2.2
 ip address 10.1.12.1 255.255.255.0
~~~~~~~~

Create the GRE tunnel endpoint on **R2**:

~~~~~~~~
interface Tunnel1
 tunnel source 8.4.2.2
 tunnel destination 15.7.8.1
 ip address 10.1.12.2 255.255.255.0
~~~~~~~~

Now let us verify that R1 can ping R2:

~~~~~~~~
R1#ping 10.1.12.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.12.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
R1#
~~~~~~~~

Also, **PC1** can ping its gateway, but it cannot ping **PC2**:

~~~~~~~~
PC1> ping 10.0.0.254

84 bytes from 10.0.0.254 icmp_seq=1 ttl=255 time=0.381 ms
84 bytes from 10.0.0.254 icmp_seq=2 ttl=255 time=0.548 ms
^C
PC1> ping 10.0.1.1  

*10.0.0.254 icmp_seq=1 ttl=255 time=0.467 ms (ICMP type:3, code:1, Destination host unreachable)
*10.0.0.254 icmp_seq=2 ttl=255 time=0.566 ms (ICMP type:3, code:1, Destination host unreachable)
^C
PC1> 
~~~~~~~~

### Static routing though the tunnel

We can configure static routes on R1 and R2 to make them reach the 10.0.1.0/24 and 10.0.2.0/24 subnets, respectively.

Configure a static route on R1 to reach the 10.0.1.0/24 subnet, via 10.1.12.2:

~~~~~~~~
ip route 10.0.1.0 255.255.255.0 10.1.12.2
~~~~~~~~

And then configure a static route on R2 to reach the 10.0.0.0/24 subnet, via 10.1.12.1:

~~~~~~~~
ip route 10.0.0.0 255.255.255.0 10.1.12.1
~~~~~~~~

### Static routing though the tunnel

We can configure static routes on R1 and R2 to make them reach the 10.0.1.0/24 and 10.0.2.0/24 subnets, respectively.

Configure a static route on R1 to reach the 10.0.1.0/24 subnet, via 10.1.12.2:

~~~~~~~~
ip route 10.0.1.0 255.255.255.0 10.1.12.2
~~~~~~~~

And then configure a static route on R2 to reach the 10.0.0.0/24 subnet, via 10.1.12.1:

~~~~~~~~
ip route 10.0.0.0 255.255.255.0 10.1.12.1
~~~~~~~~

To see if the static route is working as expected, we should ping PC2 from PC1:

~~~~~~~~
PC1> ping 10.0.1.1 -c 1

84 bytes from 10.0.1.1 icmp_seq=1 ttl=62 time=2.936 ms

PC1>
~~~~~~~~

### Dynamic routing though the tunnel

Dynamic routing through the tunnel is not much more complicated than static routing, really. First, we should remove the static route from **R1**:

~~~~~~~~
no ip route 10.0.1.0 255.255.255.0 10.1.12.2
~~~~~~~~

And remove the respective static route from R2:

~~~~~~~~
ip route 10.0.0.0 255.255.255.0 10.1.12.1
~~~~~~~~

Lastly, we need to enable OSPF on the Tunnel1 interfaces on both **R1** and **R2**. So as not to leak the internal subnets (10.0.0.0/24, 10.1.12.0/30, and 10.0.1.0/24) into the OSPF process that is communicating with R3 and R4, we will create a new, separate, OSPF process for the internal network:

~~~~~~~~
interface Tunnel1
  ip ospf 2 area 0
~~~~~~~~

Now we must also advertise the internal subnets attached to **R1**'s and **R2**'s Ethernet0/1 interfaces. We do not want to send out OSPF hellos out to the internal subnets, so I will also make those interfaces _passive_:

~~~~~~~~
interface Ethernet0/1
 ip ospf 2 area 0
router ospf 2
 passive-interface Ethernet0/1
~~~~~~~~

The routing table entry for 10.0.0.0/24 on R2 should now look like the following:

~~~~~~~~
R2#show ip route 10.0.0.0 255.255.255.0
Routing entry for 10.0.0.0/24
  Known via "ospf 2", distance 110, metric 1010, type intra area
  Last update from 10.1.12.1 on Tunnel1, 00:01:01 ago
  Routing Descriptor Blocks:
  * 10.1.12.1, from 10.1.12.1, 00:01:01 ago, via Tunnel1
      Route metric is 1010, traffic share count is 1
R2#
~~~~~~~~

And the ping from PC2 to PC1 should also work:

~~~~~~~~
PC2> ping 10.0.0.1 -c 2

84 bytes from 10.0.0.1 icmp_seq=1 ttl=62 time=2.691 ms
84 bytes from 10.0.0.1 icmp_seq=2 ttl=62 time=1.050 ms

PC2>
~~~~~~~~

