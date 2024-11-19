## Overview and Concepts

### Introduction

Traditionally, Data Centers used lots of Layer 2 links that spanned entire racks, rows, cages, floors, for as far as the eye could see. These large L2 domains were not ideal for a data center, due to the slow convergence, unnecessary broadcasts, and difficulty in administering. To optimize the data center network, we needed to reduce the use of and reliance on layer 2 protocols such as Spanning Tree. The challenge, however, is the fact that Data Centers *need* Layer 2 stretching from rack to rack, row to row, sometimes from data center to data center, not only for application requirements but also for fault tolerance and workload mobility. Numerous technologies have come forth to battle this limitation, such as TRILL, FabricPath, and VXLAN. Of these three, it is **Virtual Extensible LAN (VXLAN)** that has seen rapid adoption in modern data centers.

### Quick overview of VXLAN

VXLAN is a tunneling mechanism which can take a Layer 2 frame or a Layer 3 packet, encapsulate it with an IP header and route it to some other **VXLAN Tunnel Endpoint (VTEP)** for decapsulation. VXLAN is often referred to as MAC-in-IP because we fundamentally do just that – put a MAC address inside of an IP address. The VXLAN header includes a 24-bit field called the **VXLAN Network Identifier (VNI)**, which allows us to have up to 16 million layer 2 domain – much more than the 4096 limit with classic VLANs!

Take the diagram below for example, where we have two hosts on the same VLAN, but the links between the racks are Layer 3 routed. The hosts can be on the same VLAN and reach each other (ping) at Layer 2 by encapsulating the traffic in VXLAN and delivering it to the appropriate switches.![vxlan1](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/vxlan1.png?resize=700%2C534&ssl=1)

1. Host 10.0.0.1 initiates ping to 10.0.0.2
2. ICMP (ping) is encapsulated IP, with the source address set to 10.0.0.1 and destination set to 10.0.0.2
3. IP is placed inside of an Ethernet header, with source MAC of f8be and destination MAC of a271
4. Device running VXLAN receives the packet, looks up the MAC and see’s the destination interface as the VTEP, then encapsulates the data, adding a VXLAN header
5. A UDP header followed by an IP header is attached to the VXLAN packet, with the source IP being the local VXLAN Endpoint (VTEP) IP of 192.168.56.11 with the destination as the remote VXLAN Endpoint (VTEP) of 192.168.56.12
6. The packet is tunneled across an IP core, which knows how to get to the IP address of each VTEP
7. Upon receiving the VXLAN packet, the VTEP strips off the VXLAN header and looks up the destination MAC address
8. The packet is forwarded to the host

We aren’t limited to just encapsulating L2 MACs – we can also encapsulate L3 IPs in VXLAN and deliver it to some remote VTEP. This flexibility is synonymous with MPLS L2VPNs and MPLS L3VPNs. More on this later.

### Leaf-Spine Topology

VXLAN in a data center is most often coupled with a hierarchical 2-tier Leaf and Spine architecture, known as a Clos topology, where end-hosts connect to Leafs, and Leafs connect to Spines. It is a modular design that can scale horizontally as the end-hosts increase in volume, similar to the diagram below, where L3 point-to-point links form the underlying network in which an overlay protocol can ride, such as VXLAN. BGP is typically the protocol performing routing between all of these point-to-point links.![2-l3ls](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/2-l3ls.png?resize=524%2C391&ssl=1)

## VXLAN Data Planes and Control Planes

Traditionally, VXLAN tunnels are created between each Leaf switch (or a pair of leaf switches) using the default multicast control plane. Not a big deal if we had some small data center with just a couple of Leaf switches and a couple of Spines; we’d likely be just fine to use VXLAN natively without much concern for scale. However, once the data center starts expanding, we can quickly see how difficult it becomes to manage VXLAN using the flood-and-learn behavior of the multicast control plane.

![3-l3ls-multicast](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/3-l3ls-multicast.png?resize=700%2C220&ssl=1)

Another approach is to use ingress Head End Replication (HER), which doesn’t require multicast but is still a flood-and-learn data plane procedure. Controller-based solutions eventually came about as a way to intelligently manage the learning and distribution of MACs in the environment to the necessary VTEPs. This controller-based approach still means that we’re using multicast or HER in the data plane.

Due to this scaling issue, the **Ethernet VPN (EVPN)** control plane was created, utilizing a shiny new address family in **Multi-Protocol BGP (MP-BGP)**. Some have called this a controller-less approach to VXLAN since every node in the fabric is a member of the same EVPN overlay.

![4-l3ls-evpn](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/4-l3ls-evpn.png?resize=700%2C219&ssl=1)

EVPN can be especially helpful for multi-pod designs, maybe with Leaf-Spine pods per data center row interconnected via some Super Spines.

![5-evpn-super](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/5-evpn-super.png?resize=700%2C315&ssl=1)

# MP-BGP EVPN Overview

EVPN is a standards-based (RFC 7432) extension for BGP that provides a control plane for VXLAN (amongst other things) to delivery Layer 2 and Layer 3 VPN services. The beauty with BGP as the control plane for VXLAN is that we can use a single routing protocol with familiar concepts to manage new capabilities such as MAC address learning and VRF multi-tenancy while providing optimized **equal-cost multi-pathing (ECMP)** across data centers and within the enterprise. In the context of a VXLAN control plane, we use EVPN as a **Network Virtualization Overlay (NVO)**.  
A new address family in BGP was created to exchange **Network Layer Reachability Information (NLRI)** via a series of route types. Of these route types, the two most applicable for our discussion are:

- **Type 2** – Host MAC and IP addresses (MAC-VRF)
- **Type 5 –** IP Prefix information (IP-VRF)

You can think of Type-2 routes as VLAN-based, advertising an end host’s MAC and IP address within the VLAN over an IP network. A VXLAN Network Identifier (VNI) is mapped to a VLAN (e.g. VLAN 100 may equate to VNI 100100). Any Leaf in any pod configured with the VNI will be able to share end-host MAC addresses to provide Layer 2 reachability. As a Leaf switch learns a locally attached MAC address, it will advertise it to EVPN. Other Leaf VTEPs with this VNI configured will install the MAC in the CAM.

![6-evpn-l2vxlan](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/6-evpn-l2vxlan.png?resize=700%2C303&ssl=1)

Type-5 routes are IP Prefix-based, advertising a route prefix as opposed to a MAC Address. A VXLAN Network Identifier (VNI) is mapped to a **Virtual Routing & Forwarding (VRF)** context that identifies the customer/tenant/segment uniquely within the fabric, allowing for multitenancy and route tables to coexist (e.g. VRF TenantA may map to VNI 100002, while VRF TenantB may map to VNI 100003).

The advertisement of the type 5 EVPN attribute will provide the NLRI between subnets and routing contexts, allowing for learning of prefixes (not MACs) that are advertised across different VRFs in the fabric. This means the fabric can provide end-to-end segmentation without being aware of the segmentation itself. For example, a VRF context can be created on a pair of Leafs and be extended to some other pair of Leafs without the devices in between aware of the VRFs. With EVPN, only the leaf switches need to possess the VRFs that endpoints are attached to, allowing the Spine switches to simply provide transit between Leafs.

![7-evpn-l3vxlan](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/7-evpn-l3vxlan.png?resize=700%2C360&ssl=1)

## EVPN Functionality for Layer 2 VXLAN – Type 2 Routes

A few essential EVPN fundamentals to understand concerning L2 VXLAN are:

- MAC Learning and Mobility
- ARP Suppression
- Handling Broadcast, Multicast, and Unknown Unicast (BUM) traffic

### MAC Learning & Mobility

MAC Addresses are advertised as Type 2 routes in EVPN. Take the example below where we have and EVPN overlay established with 4 sets of Leaf-Pairs (VTEPs).

![8-vxlan-mac](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/8-vxlan-mac.png?resize=547%2C314&ssl=1)

VTEPs 1 and 2 are configured for VNI 100050 which maps to VLAN 50. A **Route Distinguisher (RD)** is assigned to identify the routes uniquely in the EVPN fabric. **Route Targets (RT)** are tags that are configured to import/export routes (MAC addresses in this case) that are advertised between VTEPs.

![9-vxlan-mac2](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/9-vxlan-mac2.png?resize=548%2C335&ssl=1)

A host connects to the Leafs that represent VTEP 1. At this point, the local VTEP advertises the learned MAC address to the EVPN fabric via the Spines.

![10-vxlan-mac3](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/10-vxlan-mac3.png?resize=570%2C393&ssl=1)

A packet capture reveals the BGP update message sent from VTEP 1 (10.0.250.11) that contains the NLRI for the host MAC address ending in 9561:

![11-vxlan-l2-pcap](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/11-vxlan-l2-pcap.png?resize=700%2C1031&ssl=1)

Upon receiving the BGP Update message, VTEP 2 will install this MAC address in it’s CAM table since it is configured for this VNI and is importing the appropriate Route-Targets. We can see the MAC is learned from interface Vxlan1:

```bash
leaf-3#**sh mac add vlan 50**
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  50    **262e.ea16.9561**    DYNAMIC     **Vx1**        1       0:00:04 ago
Total Mac Addresses for this criterion: 1

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----


leaf-3#sh bgp evpn route-type mac-ip
…
        Network             Next Hop         Metric  LocPref Weight Path
 * >     **RD: 65001:100050 mac-ip 262e.ea16.9561**
                            10.0.255.11      -       100     0      65000 65001 i
```

At this point, you may be asking about VTEP 3 and 4. What happens to them? Well, both will receive BGP update, but since neither are configured for that VNI nor that VLAN, the MAC address will not get installed.

If we were to bring up a host in the same VLAN on VTEP 4, we’d go through the same motions and the hosts would be able to reach each other at Layer 2 over the IP core.

![12-evpn-arp-suppression](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/12-evpn-arp-suppression.png?resize=574%2C390&ssl=1)

In a virtualized environment, we can expect MACs to migrate to various VTEPs. When this happens, a sequence number attached to the Type 2 BGP EVPN Update message is incremented by 1 and all VTEPs in the environment will update their tables accordingly with the new next-hop information.

### ARP Suppression

If you didn’t notice above in the packet capture, there is a field in the Type 2 route for advertising the host’s IP address.

![13-evpn-ip-not-included](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/13-evpn-ip-not-included.png?resize=400%2C120&ssl=1)

Data was not included in this field because the MAC learning example above was purely Layer 2. There were no Gateway SVIs configured; therefore no ARP entries existed. In many cases, you’ll want your hosts to have an IP gateway. In the case of Arista, typically each Leaf Pair is configured in a **Multi-chassis Link Aggregation Group (MLAG)**, with all hosts dual-connected to each Leaf. The Leaf Pair acts as the gateway for VLANs using an Anycast VARP SVI. For example, I may configure the following on all Leafs:

int vlan 50
ip address virtual 10.50.0.1/24

Now our hosts can reach their local gateways.

![14-evpn-local-gw](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/14-evpn-local-gw.png?resize=638%2C392&ssl=1)

Now, imagine these two hosts had never communicated with each other. If the host on the left wanted to ping the host on the right, the host would first ARP for its location. Typically you’d expect this ARP to be sent from VTEP 1 to VTEP 2 where the ARP can be resolved. However, with the ARP Suppression capabilities in Arista’s implementation of EVPN, each VTEP possesses an ARP entry even for hosts not directly connected.

For example, a Leaf member of VTEP 1 has an ARP entry for its locally connected host, and an ARP entry for the remote host learned via VXLAN:

leaf-1#show ip arp | in Vlan50
10.50.0.10 N/A 262e.ea16.9561 Vlan50, Port-Channel1
10.50.0.20 - d6f7.f3ee.0e0b Vlan50, Vxlan1

This means when host 1 tries to ARP for host 2, the local VTEP can proxy the request, effectively suppressing the ARP messages. Cool stuff! For example, I added a 10.50.0.30 host to the network to show the advertisement of this IP address in the Type 2 route:

![15-evpn-pcap-mac-ip](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/15-evpn-pcap-mac-ip.png?resize=530%2C177&ssl=1)

### Handling Broadcast, Unknown Unicast, and Multicast (BUM) Traffic

BUM traffic is handled by EVPN Type-3 route advertisements. A VTEP configured for a VNI will advertise itself as a member of that **EVPN Instance (EVI)** to all other VTEPs that are part of that EVI. Each VTEP maintains a flood list of other VTEPs in that EVI, and perform headend replication whenever a BUM packet is received.

![16-evpn-bum](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/16-evpn-bum.png?resize=589%2C371&ssl=1)

## EVPN Functionality for L3 VXLAN – Type 5 Routes

Similar to the way we extend Layer 2 with VXLAN but affixing a VNI to a VLAN, instead we will map a VNI to a VRF. Any VTEP configured with this VNI>VRF will learn routes and install them in the corresponding routing context. This could be used for multiple use cases, such as peering with external networks, peering with Border gateways, L3 DCI, multi-tenancy, and segmentation.

For me, it was most straightforward to understand Type 5 routes by building a pure EVPN Type 5 environment, removing any requirements for L2 VXLAN extension. I wanted an environment that could segment tenants into VRFs and use Type 5 EVPN routes to handle Layer 3 VPN services between Leaf switches.

Imagine you have 3 tenants – these could be customers in a data center, security zones within an enterprise, and so forth. Each tenant is placed in a routing context (VRF) per the table below:

Tenant 1 – VRF green  
Tenant 2 – VRF blue  
Tenant 3 – VRF yellow

![17-evpn-l3vxlan-1](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/17-evpn-l3vxlan-1.png?resize=504%2C428&ssl=1)

With Type 5 routes, VRF-compartmentalized prefixes are transported over the EVPN fabric to any VTEP Leaf-Pair configured for that VRF. With the example above, we will map each VRF to a VNI.

- VRF green = VNI 10007
- VRF blue = VNI 10008
- VRF yellow= VNI 10009

Just like with L2 VXLAN, EVPN Type 5 routes require a Route Distinguisher (RD) and Route Targets (RT). The Route Distinguisher uniquely identifies the route in the EVPN RIB by appending the RD value to the prefix. The Route Targets specify which routes get imported into and exported from any VRF. We could follow a schema similar to the following:

In this scenario, each Leaf has a unique Loopback0 IP address, which we’re advertising in BGP to form the Overlays. We are going to use this attribute to help us uniquely identify routes in the routing table:

- RD = Loopback0:VNI
- RT = VRF_ID:VNI

So, taking the Leafs that represent VTEP 1, our schema would look something like this:

Leaf 1:

router bgp 65001
vrf green
rd 10.0.250.11:10007
route-target 7:10007 both
redistribute connected
vrf blue
rd 10.0.250.11:10008
route-target 8:10008 both
redistribute connected
vrf yellow
rd 10.0.250.11:10009
route-target 9:10009 both
redistribute connected

Leaf 2:

router bgp 65001
vrf green
rd 10.0.250.12:10007
route-target 7:10007 both
redistribute connected
vrf blue
rd 10.0.250.12:10008
route-target 8:10008 both
redistribute connected
vrf yellow
rd 10.0.250.12:10009
route-target 9:10009 both
redistribute connected

Back to our diagram, we could expect to see something like below. Take note that these VRFs only exist on the VTEPs. The Spine layer is completely unaware of this information – it is simply passing routes. It is up the Leafs to derive the RDs and import routes tagged with the appropriate route-targets.

![18-evpn-l3vxlan-2](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/18-evpn-l3vxlan-2.png?resize=700%2C399&ssl=1)

Taking VRF green, for example, we can look at the EVPN routing table and see the advertisements of our networks between the VTEPs.

• On VTEP 1, we are advertising 10.7.0.0/24 into VRF green  
• On VTEP 2, we are advertising 10.170.0.0/24 into VRF green

VTEP1 Leaf-1:

vrf definition green
ip routing vrf green

router bgp 65001
vrf green
rd 10.0.250.11:10007
route-target both 7:10007
redistribute connected

interface Vxlan1
vxlan vrf green vni 10007

vlan 7
int vlan 7
vrf forwarding green
ip add virtual 10.7.0.1/24

VTEP1 Leaf-2:

vrf definition green
ip routing vrf green

router bgp 65001
vrf green
rd 10.0.250.12:10007
route-target both 7:10007
redistribute connected

interface Vxlan1
vxlan vrf green vni 10007

vlan 7
int vlan 7
vrf forwarding green
ip add virtual 10.7.0.1/24

VTEP2 Leaf-3:

vrf definition green
ip routing vrf green

router bgp 65002
vrf green
rd 10.0.250.13:10007
route-target both 7:10007
redistribute connected

interface Vxlan1
vxlan vrf green vni 10007

vlan 170
int vlan 170
vrf forwarding green
ip add virtual 10.170.0.1/24

VTEP2 Leaf-4:

vrf definition green
ip routing vrf green

router bgp 65002
vrf green
rd 10.0.250.14:10007
route-target both 7:10007
redistribute connected

interface Vxlan1
vxlan vrf green vni 10007

vlan 170
int vlan 170
vrf forwarding green
ip add virtual 10.170.0.1/24

And here from VTEP1 (Leaf-1) perspective, we can see the prefixes being learned from VTEP2. Notice that we have four routes to 10.170.0.0/24 because we have two Spines and two switches in the VTEP, allowing us to achieve ECMP:

leaf-1#**show bgp evpn route-type ip-prefix ipv4 vni 10007**
BGP routing table information for VRF default
Router identifier 10.0.250.11, local AS number 65001
…

        Network             Next Hop         Metric  LocPref Weight Path

- >     RD: 10.0.250.11:10007 ip-prefix 10.7.0.0/24
                           -                -       -       0       i
- > Ec RD: 10.0.250.13:10007 ip-prefix 10.170.0.0/24
                           10.0.255.12      -       100     0      65000 65002 i
- ec RD: 10.0.250.13:10007 ip-prefix 10.170.0.0/24
  10.0.255.12 - 100 0 65000 65002 i
- > Ec RD: 10.0.250.14:10007 ip-prefix 10.170.0.0/24
                           10.0.255.12      -       100     0      65000 65002 i
- ec RD: 10.0.250.14:10007 ip-prefix 10.170.0.0/24
  10.0.255.12 - 100 0 65000 65002 i

Similar view from VTEP2:

leaf-3#**show bgp evpn route-type ip-prefix ipv4 vni 10007**
BGP routing table information for VRF default
Router identifier 10.0.250.13, local AS number 65002
…
Network Next Hop Metric LocPref Weight Path

- > Ec RD: 10.0.250.11:10007 ip-prefix 10.7.0.0/24
                           10.0.255.11      -       100     0      65000 65001 i
- ec RD: 10.0.250.11:10007 ip-prefix 10.7.0.0/24
  10.0.255.11 - 100 0 65000 65001 i
- > Ec RD: 10.0.250.12:10007 ip-prefix 10.7.0.0/24
                           10.0.255.11      -       100     0      65000 65001 i
- ec RD: 10.0.250.12:10007 ip-prefix 10.7.0.0/24
  10.0.255.11 - 100 0 65000 65001 i
- >     RD: 10.0.250.13:10007 ip-prefix 10.170.0.0/24
                           -                -       -       0       i

A packet capture reveals VXLAN-encapsulated prefixes as type 5 EVPN routes:

![19-evpn-l3vxlan-pcap](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/19-evpn-l3vxlan-pcap.png?resize=700%2C873&ssl=1)

Again, the routes are installed on the Leafs which are importing the routes, but although the spines route the traffic, the VRFs do not exist on them and the routes are not installed in the RIB.

leaf-1#**show ip route vrf green | b Gateway**
Gateway of last resort is not set

C 10.7.0.0/24 is directly connected, Vlan7
B E 10.170.0.0/24 [200/0] via VTEP 10.0.255.12 VNI 10007 router-mac 0c:56:33:29:b2:0c
via VTEP 10.0.255.12 VNI 10007 router-mac 0c:56:33:bc:22:bd

spine2#sho bgp evpn route-type ip-prefix ipv4 vni 10007 | b Network
Network Next Hop Metric LocPref Weight Path

- >     RD: 10.0.250.11:10007 ip-prefix 10.7.0.0/24
                           10.0.255.11      -       100     0      65001 i
- >     RD: 10.0.250.12:10007 ip-prefix 10.7.0.0/24
                           10.0.255.11      -       100     0      65001 i
- >     RD: 10.0.250.13:10007 ip-prefix 10.170.0.0/24
                           10.0.255.12      -       100     0      65002 i
- >     RD: 10.0.250.14:10007 ip-prefix 10.170.0.0/24
                           10.0.255.12      -       100     0      65002 i

spine2#show ip route vrf green
% IP Routing table for VRF green does not exist.

Now, if a ping is sent between 10.7.0.10 (a host connected to VTEP1) to 10.170.0.10 (a host connected to VTEP2), we would see the encapsulation as ICMP in IP in Ethernet in VXLAN in UDP in IP in Ethernet:

![20-evpn-l3vxlan-ping-pcap](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/20-evpn-l3vxlan-ping-pcap.png?resize=700%2C239&ssl=1)

## Integrated Routing & Bridging (IRB)

When reading about EVPN, you’ll undoubtedly come across the two types of IRB – Asymmetric, and Symmetric. For the sake of brevity, I’m just going to give a high-level view of these concepts.

Imagine you have 2 hosts in separate VLANs that need to communicate. These hosts are connected to separate VTEPs. Each VTEP is configured as an Anycast Gateway for the VLANs.

![21-evpn-irb](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/21-evpn-irb.png?resize=574%2C399&ssl=1)

### Asymmetric IRB

With Asymmetric IRB, if the host on the left wants to reach the host on the right, the local VTEP will receive the packet, then route it to the local SVI, then VXLAN Bridge it to the remote VTEP, where the remote VTEP will finally switch the packet. Return traffic would hit the local VTEP, get routed to the local SVI, then VXLAN Bridged back to the remote VTEP where it is finally switched to the host. This flow is Asymmetric in that each VTEP is performing a local ingress routing function before forwarding the traffic. This type of asymmetric flow requires that each VTEP in the fabric is configured for an Anycast gateway for all VLANs in the environment and that each switch maintains ARP and CAM tables for each of the VLANs. This behavior is illustrated below:

![22-evpn-irb-asymmetric](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/22-evpn-irb-asymmetric.png?resize=700%2C378&ssl=1)

### Symmetric IRB

With Symmetric IRB, each VTEP performs ingress and egress routing. It’s easiest to explain this in a few scenarios:

In our first scenario:  
• Each VTEP is configured only with the VLAN SVIs that are needed for attached hosts  
○ VTEP1 is configured with an SVI for VLAN 7 in VRF green  
○ VTEP2 is configured with an SVI for VLAN 170 in VRF green  
• Traffic between these two segments is symmetrical in that traffic is routed to the VRF, over the VNI, and back in the same manner

![23-evpn-irb-symmetric](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/23-evpn-irb-symmetric.png?resize=597%2C431&ssl=1)

In our second scenario, let’s add a variable into the mix:

- Same as before, but we need to extend Layer 2 for a VLAN between VTEPs (i.e. L2 VXLAN)  
   ○ VTEP1 is configured with an SVI for VLAN 7  
   ○ VTEP2 is configured with an SVI for VLAN 7  
   ○ The L3 VNI still exists for VRF green  
   ○ An L2 VNI is implemented for VLAN 7

In an Asymmetrical IRB environment, we would expect traffic from host 10.7.0.10 to 10.7.0.20 to be ingress routed at the local VTEP and then take the L2 VXLAN path to the remote VTEP. However, since these SVIs are configured inside of a VRF that is shared across the EVPN fabric, we can route the traffic symmetrically via the VRF.

Traffic from 10.7.0.10 to 10.7.0.20 using L2 VXLAN:

![24-evpn-irb-symmetric2](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/24-evpn-irb-symmetric2.png?resize=604%2C467&ssl=1)

The MAC address of 10.7.0.10 is advertised into EVPN with both the L2 VNI and the L3 VNI:

BGP routing table entry for mac-ip 562e.665b.dc1b 10.7.0.10, Route Distinguisher: 65001:100070
Paths: 1 available
Local - from - (0.0.0.0)
Origin IGP, metric -, localpref -, weight 0, valid, local, best
Extended Community: Route-Target-AS:7:10007 Route-Target-AS:7:100070 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:0c:56:33:7c:d0:51
**VNI: 100070 L3 VNI: 10007** ESI: 0000:0000:0000:0000:0000

Traffic from 10.7.0.10 to 10.170.0.10 using L3 VXLAN:

![25-evpn-irb-symmetric3](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/25-evpn-irb-symmetric3.png?resize=700%2C492&ssl=1)

## Deployment Models

There are multiple ways to deploy the EVPN underlay and overlay. You can use an IGP like OSPF or IS-IS in the underlay and run either IBGP or EBGP in the overlay. You can run EBGP in the underlay and IBGP on the overlay. Or, my favorite option, run EBGP in both the underlay and the overlay, which I feel is best for multi-pod designs. Various design docs out there will demonstrate one of these deployment models, so keep in mind that there are indeed multiple options, but it’s up to your discretion which is best for your environment. If you have a single-pod design, maybe it’s best to do EBGP underlay and IBGP overlay, which provides more functionality with Arista Cloud Vision Portal (CVP) and allows the use or autogenerated Route Distinguishers and Route Targets – a handy benefit! Alternatively, get clever with Ansible or some other automation tool, and automatically generate your RDs and RTs that way.

As an example, with a pure EBGP deployment model, point-to-point links would be connected with /31 subnets. Over each of these links, BGP is established – EBGP between Leafs and Spines, and IBGP between within a Leaf-Pair.

![26-l3ls-underlay](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/26-l3ls-underlay.png?resize=516%2C398&ssl=1)

Next, a Loopback is advertised into BGP from each node. Another separate BGP adjacency is formed between Leaf-Spine Loopbacks.

![27-l3ls-overlay](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/27-l3ls-overlay.png?resize=561%2C346&ssl=1)

Finally, our Network Virtualization Overlay – EVPN – is ready for action. VXLAN tunnel end-points (VTEPs) can optimally exchange data across the fabric, completely abstracting the overlay network from the series of underlay /31 point-to-point links.

![28-l3ls-evpn](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/28-l3ls-evpn.png?resize=536%2C305&ssl=1)

## Routing into and out of the EVPN network

One might think that the best way out of the EVPN network would be through the Spines. However, this is not the case. Typically, a set of Border/Edge Leaf switches are deployed, which act as the gateway into and out of the EVPN network. These Border Leafs could peer with various systems, such as Firewalls, Internet Routers, WAN Routers, Cloud circuits, etc., serving as an exchange point where redistribution of routes into and out of the EVPN network can occur.

Some use cases:

• A Data Center may use the Border Leafs to perform redistribution into and out of the data center or to perform Data Center Interconnects (DCI) with some other remote data center  
• A Service Provider with multiple tenants may use Border Leafs to inject default routes into their customers VRF contexts to provide a pathway out to the Internet.  
• An Enterprise may peer Border Leafs with Firewalls to perform segmentation between various zones, such as Users, PCI, Security, Vendors, etc.

![29-outside](https://i0.wp.com/overlaid.net/wp-content/uploads/2018/08/29-outside.png?resize=504%2C473&ssl=1)
