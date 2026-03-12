## Table of Contents

1. The Layered Model — Overview
2. Layer 1 — Physical
3. Layer 2 — Data Link
4. Layer 3 — Network (IP, ICMP, Routing)
5. Layer 4 — Transport (TCP, UDP, QUIC)
6. Layer 5/6/7 — Session, Presentation, Application
7. Network Services (DHCP, NAT, DNS)
8. Network Security
9. System Design Networking
10. Debugging & Tools

---

## 1. The Layered Model — Overview

Networking is organized into layers, each solving one piece of the communication problem. Each layer only communicates with its peer on the other side — lower layers are just the delivery mechanism.

### OSI Model (7 layers)

```
 Layer   Name            Responsibility                        Protocols            PDU
 -----   --------------  ------------------------------------  ------------------   --------
   7     Application     App-level semantics                   HTTP, DNS, SSH       Message
   6     Presentation    Data formatting, encryption           TLS, SSL, JPEG
   5     Session         Connection management                 Sockets, sessions
   4     Transport       End-to-end delivery between processes TCP, UDP, QUIC       Segment
   3     Network         Host-to-host delivery across networks IP, ICMP             Packet
   2     Data Link       Hop-to-hop delivery on a single link  Ethernet, WiFi, ARP  Frame
   1     Physical        Bits on the wire (or air)             Cables, radio        Bits
```

### TCP/IP Model (what the Internet actually uses)

The TCP/IP model collapses the top three OSI layers into a single Application layer, and the bottom two into a Network Access layer. Know OSI for interviews, use TCP/IP for real thinking.

```
  OSI                TCP/IP
  -----------        --------------
  Application  --+
  Presentation   +-- Application
  Session      --+
  Transport    ---- Transport
  Network      ---- Internet
  Data Link    --+
  Physical     --+-- Network Access
```

### Encapsulation and PDUs

As data moves down the stack, each layer wraps the payload from above with its own control information, forming a new **PDU (Protocol Data Unit)**. This process is called **encapsulation**. On the receiving side, each layer strips its header and passes the payload up — **decapsulation**.

#### PDU terminology

Each layer has its own name for the data unit it works with:

|Layer|PDU name|What gets added|
|---|---|---|
|Application|**Data / Message**|Application headers (e.g., HTTP headers)|
|Transport|**Segment** (TCP) / **Datagram** (UDP)|Transport header (ports, seq numbers, flags)|
|Network|**Packet**|IP header (source/destination IP, TTL)|
|Data Link|**Frame**|Frame header (MAC addresses) + trailer (CRC)|
|Physical|**Bits**|Encoded as electrical/optical/radio signals|

#### How wrapping works

The formal model uses three terms for what happens at each layer:

- **SDU (Service Data Unit):** the payload received from the layer above — what needs to be delivered.
- **PCI (Protocol Control Information):** the header (and sometimes trailer) added by the current layer — addresses, sequence numbers, error checks, flags.
- **PDU (Protocol Data Unit):** the complete unit at this layer = PCI + SDU.

The PDU of one layer becomes the SDU of the layer below:

```
  Layer N+1 produces a PDU
         |
         v
  Layer N receives it as an SDU
         |
         v  adds its own PCI (header / trailer)
         |
         v
  Layer N produces a new PDU  -->  becomes SDU for Layer N-1
```

The complete set of protocols used across all layers is called the **protocol stack**.

#### Encapsulation in action

An HTTP GET request traveling down the stack. Each layer treats everything above it as an opaque payload:

```
  Application    +------------------------------------------+
                 |            HTTP GET /api/data            |  <-- Data (L7 PDU)
                 +------------------------------------------+
                                    |
                         Transport receives this as its SDU
                         and adds a TCP header (PCI)
                                    |
                                    v
  Transport      +----------+-------------------------------+
                 | TCP HDR  |         HTTP payload          |  <-- Segment (L4 PDU)
                 | src:4921 |                               |
                 | dst:443  |                               |
                 +----------+-------------------------------+
                                    |
                         Network receives this as its SDU
                         and adds an IP header (PCI)
                                    |
                                    v
  Network        +----------+-------------------------------+
                 |  IP HDR  |        TCP segment            |  <-- Packet (L3 PDU)
                 | src:10.0 |                               |
                 | dst:93.x |                               |
                 +----------+-------------------------------+
                                    |
                         Data Link receives this as its SDU
                         and adds Ethernet header + CRC trailer (PCI)
                                    |
                                    v
  Data Link      +----------+-------------------------------+------+
                 | ETH HDR  |        IP packet              | CRC  | <-- Frame (L2 PDU)
                 | src:aa:bb|                               |      |
                 | dst:cc:dd|                               |      |
                 +----------+-------------------------------+------+
                                    |
                                    v
  Physical         101010001110010101001...                          <-- Bits (L1)
```

**Key points:**

- Each layer only reads its own header. A router (L3 device) strips the frame header, reads the IP header, makes a routing decision, and re-encapsulates in a new frame for the next hop.
- Headers are added at the front (**headers**), trailers at the end (**trailers**). The CRC (Cyclic Redundancy Check) in Ethernet is a trailer. Most layers only add headers.
- This is exactly what you see in Wireshark — nested layers that you can expand one by one.
- The total overhead of all headers is why your 1500-byte Ethernet MTU only carries ~1460 bytes of TCP payload: 20 bytes eaten by IP, 20 by TCP.

### What happens when you type a URL

The most common networking interview question. Here's the full story, layer by layer:

```
  Browser                                              Server
    |                                                    |
    |--- 1. DNS lookup: example.com -------------------> | DNS Server
    |<-- IP: 93.184.216.34 ----------------------------  |
    |                                                    |
    |--- 2. TCP SYN ---------------------------------->  |
    |<-- SYN-ACK --------------------------------------  |  1 RTT
    |--- ACK ----------------------------------------->  |
    |                                                    |
    |--- 3. TLS ClientHello -------------------------->  |
    |<-- ServerHello + Cert + Key ---------------------  |  1 RTT (TLS 1.3)
    |--- Finished ------------------------------------>  |
    |                                                    |
    |--- 4. HTTP GET / --------------------------------> |
    |<-- HTTP 200 OK + HTML ---------------------------  |
    |                                                    |
    |--- 5. Parse HTML, request CSS/JS/images -------->  |
    |<-- Assets ---------------------------------------  |
    |                                                    |
    |    6. Render page                                  |
```

> **Interview tip:** structure your answer through the layers — DNS resolution, TCP handshake, TLS handshake, HTTP request, response parsing. Mentioning ARP, routing, and frame-level delivery earns extra points.

---

## 2. Layer 1 — Physical

The physical layer handles transmission of raw bits over a medium: copper (Ethernet cables), fiber optics, or radio (WiFi, cellular). As a software engineer, you rarely interact with this layer directly, but a few things matter:

**What to know:**

- **Ethernet cable categories:** Cat5e (1 Gbps), Cat6 (10 Gbps short range), Cat6a (10 Gbps longer range). Datacenter interconnects use fiber.
- **WiFi standards:** 802.11n (WiFi 4), 802.11ac (WiFi 5), 802.11ax (WiFi 6). Higher numbers = more bandwidth, but wireless is always less reliable than wired.
- **Bandwidth vs. latency:** bandwidth is the pipe diameter (data per second), latency is the pipe length (time for one bit to arrive). High bandwidth does not mean low latency.
- **Full-duplex vs. half-duplex:** modern Ethernet is full-duplex (send and receive simultaneously). WiFi is inherently half-duplex (shared medium).

---

## 3. Layer 2 — Data Link

The data link layer handles delivery across a single link (one hop). It deals with MAC addresses, frames, error detection, and media access control.

### MAC (Media Access Control) Addresses

48-bit hardware identifiers burned into NICs (Network Interface Cards), written in hex: `aa:bb:cc:dd:ee:ff`. Globally unique per interface. Used only for **local delivery** within a LAN segment.

We cannot change the physical MAC Address but we can change the virtual MAC Address.

**Key insight:** source and destination MAC addresses are **rewritten at every router hop**, while IP addresses stay the same end-to-end.

### Ethernet Frames

Ethernet is the dominant LAN technology. Modern Ethernet is point-to-point, full-duplex over switches — collisions are a thing of the past (CSMA/CD (Carrier Sense Multiple Access with Collision Detection) is only relevant for legacy shared-media segments).

```
  Ethernet Frame Structure

  +----------+-----+----------+----------+----------+--------------+------+
  | Preamble | SFD | Dst MAC  | Src MAC  | Type/Len |   Payload    | CRC  |
  |  7 bytes | 1B  |  6 bytes |  6 bytes |  2 bytes | 46-1500 B    | 4B   |
  +----------+-----+----------+----------+----------+--------------+------+
   (Start                                              ^
    Frame                                              |
    Delimiter)                              MTU: 1500 bytes (default)
```

**MTU (Maximum Transmission Unit):** Ethernet's default max payload is **1500 bytes**. This is where TCP's common MSS (Maximum Segment Size) of 1460 comes from: 1500 (MTU) - 20 (IP header) - 20 (TCP header). CRC (Cyclic Redundancy Check) is the error-detection trailer appended to each frame. Jumbo frames (9000B) exist in datacenter environments.

### ARP (Address Resolution Protocol)

Resolves IP addresses to MAC addresses within a LAN. Operates with broadcast requests and unicast replies.

```
  Host A (10.0.0.2)                                Host B (10.0.0.5)
    |                                                 |
    |-- ARP Request (broadcast) -------------------->>|
    |   "Who has 10.0.0.5? Tell 10.0.0.2"             |  All hosts on the
    |                                                 |  LAN receive this,
    |<<- ARP Reply (unicast) -------------------------|  only B replies
    |   "10.0.0.5 is at aa:bb:cc:dd:ee:ff"            |
    |                                                 |
    |  (A caches: 10.0.0.5 -> aa:bb:cc:dd:ee:ff)      |
```

For off-subnet destinations, the frame goes to the **default gateway's MAC**. The router then does ARP on the next segment. This is why `arp -a` on your machine shows your gateway's MAC but not the MAC of remote servers.

> **Security:** ARP has zero authentication. Any host can claim any IP, enabling man-in-the-middle attacks on the LAN (ARP spoofing). Mitigations: DAI (Dynamic ARP Inspection), 802.1X port authentication, static ARP entries.

### Switches

Switches operate at Layer 2: they learn MAC addresses by inspecting source addresses of incoming frames, build a forwarding table (`MAC -> port`), and send frames only to the correct port. Each port is a separate collision domain. Tables are built automatically — plug-and-play.

### VLANs (Virtual Local Area Networks)

A VLAN lets one physical switch act as multiple isolated L2 networks. Traffic between VLANs must pass through a router (or L3 switch). Essential for security segmentation and reducing broadcast domains.

```
             +--------------- Switch ---------------+
             |                                       |
        VLAN 10               VLAN 20              VLAN 30
      +-----------+       +-----------+        +-----------+
      | Employees |       |  Guests   |        |  Servers  |
      | 10.0.10.x |       | 10.0.20.x |        | 10.0.30.x |
      +-----------+       +-----------+        +-----------+
             |                                       |
             +----- traffic between VLANs -----------+
                   must go through a router
```

### STP (Spanning Tree Protocol)

When switches have redundant paths (for resilience), loops can form. STP prevents this by electing a root bridge and pruning redundant links. In production, **RSTP (Rapid Spanning Tree Protocol, 802.1w)** replaced classic STP for much faster convergence (~1-2 seconds vs. 30-50 seconds).

---

## 4. Layer 3 — Network (IP, ICMP, Routing)

The network layer handles host-to-host delivery across different networks. This is where IP addressing, routing, and fragmentation live.

### Unicast, Broadcast, and Multicast

Three fundamental modes of IP communication:

**Unicast** — one-to-one. A packet is sent from one source to one specific destination. This is the default for almost all traffic (HTTP requests, SSH sessions, database queries). Each packet has a single source IP and a single destination IP.

```
  Host A  ---------> Host B         One sender, one receiver
```

**Broadcast** — one-to-all. A packet is sent to every host on the local network. The destination address is the subnet's broadcast address (all host bits set to 1). For example, on the `192.168.1.0/24` network, the broadcast address is `192.168.1.255`.

```
  Host A  ---------> All hosts      One sender, all receivers on the subnet
                     on the LAN
```

Broadcasts are confined to the local subnet — routers do not forward them (this boundary is called the **broadcast domain**). Protocols that use broadcast: ARP, DHCP Discover/Request. Excessive broadcasts degrade network performance, which is one reason to use VLANs to keep broadcast domains small.

> **IPv6 has no broadcast.** It uses multicast and a new concept called **anycast** instead.

**Multicast** — one-to-many (subscribers only). A packet is sent to a **group address** and delivered only to hosts that have joined that group. Uses the IP range `224.0.0.0/4` (224.0.0.0 – 239.255.255.255) in IPv4.

```
  Host A  ---------> Group members  One sender, only subscribed receivers
                     (not everyone)
```

Hosts join/leave groups using IGMP (Internet Group Management Protocol). Multicast is efficient for streaming video to many viewers, service discovery (mDNS uses `224.0.0.251`), and cluster coordination. Multicast-capable routers and switches are required — on the public Internet it's rarely available, but within datacenters and enterprise LANs it's common.

**Anycast** — one-to-nearest. Multiple hosts share the same IP address, and the network routes packets to the **topologically closest** one based on routing metrics. Implemented via BGP. Used by CDNs (Content Delivery Networks), DNS root servers, and services like Cloudflare (`1.1.1.1`) and Google DNS (`8.8.8.8`).

```
  Client  ---------> Nearest server   Multiple servers share one IP,
                     with same IP     routing picks the closest
```

|Mode|Destination|Receivers|Scope|Example|
|---|---|---|---|---|
|Unicast|Single host|1|Global|HTTP request|
|Broadcast|All hosts on subnet|All on LAN|Local subnet|ARP, DHCP|
|Multicast|Group address|Subscribed hosts|LAN or routed|Video streaming, mDNS|
|Anycast|Nearest of many|1 (closest)|Global (via BGP)|CDN, DNS root servers|

### IPv4

32-bit address, written in dotted decimal. Every address has a **network prefix** and a **host portion**, separated by the subnet mask.

```
  IP address:    192  .  168  .   1   .  10
  Binary:     11000000.10101000.00000001.00001010
              +------ network (24 bits) ------++host+

  Subnet mask:   255  .  255  .  255  .   0
              11111111.11111111.11111111.00000000

  CIDR notation: 192.168.1.10/24
```

### IPv6

128-bit addresses written in hex: `2001:db8::1`. Solves IPv4 address exhaustion, simplifies the header (no checksum, no router fragmentation). ~45% of Google traffic is IPv6 as of 2025. Dual-stack (v4 + v6) is the standard transition strategy.

Key differences from IPv4: no broadcast (uses multicast instead), no NAT needed (enough addresses for every device), mandatory IPsec support, simplified header (fixed 40 bytes, no options field, no header checksum).

### CIDR (Classless Inter-Domain Routing) and Subnetting

CIDR replaced the old Class A/B/C system. Addresses are `IP/prefix_length`, allowing variable-length prefixes and route aggregation.

**Quick math:** usable hosts = 2^(32 - prefix) - 2. The minus 2 accounts for the network address (all-zeros host) and broadcast (all-ones host).

|CIDR|Mask|Total IPs|Usable hosts|Typical use|
|---|---|---|---|---|
|`/32`|255.255.255.255|1|Single host|Loopback, host route|
|`/30`|255.255.255.252|4|2|Router interconnect|
|`/28`|255.255.255.240|16|14|Small server subnet|
|`/26`|255.255.255.192|64|62|Department LAN|
|`/24`|255.255.255.0|256|254|Standard LAN|
|`/20`|255.255.240.0|4,096|4,094|VPC subnet (AWS default)|
|`/16`|255.255.0.0|65,536|65,534|Large VPC|
|`/8`|255.0.0.0|16.7M|16.7M|Private class A|

**VLSM (Variable Length Subnet Masking)** lets you use different prefix lengths within the same allocation — standard practice:

```
  10.0.0.0/16 (entire allocation)
  +-- 10.0.0.0/20   -- Production servers    (4094 hosts)
  +-- 10.0.16.0/20  -- Staging               (4094 hosts)
  +-- 10.0.32.0/24  -- Management / bastion   (254 hosts)
  +-- 10.0.33.0/28  -- Load balancers          (14 hosts)
  +-- 10.0.34.0/28  -- Database tier           (14 hosts)
```

### Private Addresses (RFC 1918)

|Range|CIDR|Common use|
|---|---|---|
|10.0.0.0 – 10.255.255.255|`10.0.0.0/8`|Cloud VPCs, enterprise|
|172.16.0.0 – 172.31.255.255|`172.16.0.0/12`|Docker default|
|192.168.0.0 – 192.168.255.255|`192.168.0.0/16`|Home routers|

**Special:** `127.0.0.0/8` (loopback), `169.254.0.0/16` (link-local / DHCP failure), `0.0.0.0` (all interfaces).

### IPv4 Packet Header

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |Version|  IHL  |    DSCP   |ECN|         Total Length          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |         Identification        |Flags|    Fragment Offset      |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  TTL  |    Protocol           |       Header Checksum         |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                       Source Address                          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                    Destination Address                        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                    Options (if IHL > 5)                       |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key fields explained:**

- **Version (4 bits):** 4 for IPv4, 6 for IPv6.
- **IHL (Internet Header Length, 4 bits):** header length in 32-bit words. Minimum 5 (= 20 bytes), maximum 15 (= 60 bytes with options).
- **DSCP (Differentiated Services Code Point) / ECN (Explicit Congestion Notification):** used for QoS (Quality of Service) marking and congestion signaling.
- **Total Length (16 bits):** entire packet size in bytes. Max 65,535 bytes.
- **Identification, Flags, Fragment Offset:** used for fragmentation/reassembly.
    - **DF (Don't Fragment) flag:** when set, routers drop oversized packets instead of fragmenting, replying with ICMP "fragmentation needed." Basis of **PMTUD (Path MTU Discovery)**.
    - **MF (More Fragments) flag:** 0 = last fragment, 1 = more fragments follow.
    - In practice, avoid fragmentation — it's inefficient, duplicates effort if any fragment is lost, and many firewalls drop fragments.
- **TTL (Time To Live, 8 bits):** decremented at every router. Packet dropped at 0, triggering an ICMP Time Exceeded message. Prevents infinite routing loops. Typical initial values: 64 (Linux), 128 (Windows), 255 (network gear). Basis of `traceroute`.
- **Protocol (8 bits):** identifies the upper-layer protocol. TCP=6, UDP=17, ICMP=1, OSPF=89.
- **Header Checksum (16 bits):** error check on the header only (not payload). Recalculated at every hop because TTL changes. Removed in IPv6 (redundant with L2 and L4 checksums).
- **Source/Destination Address (32 bits each):** unchanged end-to-end (except when NAT rewrites source).
- **Options:** rarely used. Can carry source routing, record route, timestamps. Most networks drop packets with options for security reasons.

### ICMP (Internet Control Message Protocol)

ICMP is the network layer's diagnostic and error-reporting protocol. ICMP messages are encapsulated inside IP packets (protocol field = 1) but are considered part of Layer 3, not Layer 4.

**ICMP is not used for data transfer** — it exists to communicate problems back to the source and to perform network diagnostics.

#### ICMP message structure

Every ICMP message has: **Type** (what kind of message), **Code** (specific condition within that type), **Checksum**, and type-specific **Data**.

#### Key ICMP message types

|Type|Name|Purpose|Used by|
|---|---|---|---|
|0|Echo Reply|Response to a ping|`ping`|
|3|Destination Unreachable|Host/network/port not reachable|Routers, firewalls|
|4|Source Quench|Congestion notification (deprecated)|—|
|5|Redirect|Tells source to use a better route|Routers|
|8|Echo Request|"Are you there?"|`ping`|
|11|Time Exceeded|TTL reached 0|`traceroute`|
|12|Parameter Problem|Malformed header|Routers|

#### ICMP Destination Unreachable codes

When a router or host can't deliver a packet, the code field explains why:

|Code|Meaning|Common cause|
|---|---|---|
|0|Network unreachable|No route to destination network|
|1|Host unreachable|Route exists but specific host down|
|3|Port unreachable|UDP packet to a port with no listener|
|4|Fragmentation needed, DF set|Packet too big, can't fragment (PMTUD)|
|13|Communication administratively filtered|Firewall blocking traffic|

#### How `ping` works

1. Source sends ICMP Echo Request (type 8) to destination.
2. Destination replies with ICMP Echo Reply (type 0).
3. Source measures the round-trip time.

If no reply comes back, the destination might be down, ICMP might be blocked by a firewall, or the route might be broken. No reply does not always mean the host is unreachable.

#### How `traceroute` works

1. Source sends packets with TTL=1. The first router decrements TTL to 0, drops the packet, and sends back ICMP Time Exceeded (type 11).
2. Source records the router's IP and RTT, then sends with TTL=2.
3. Repeats, incrementing TTL, until the destination is reached (responds with ICMP Echo Reply or ICMP Port Unreachable depending on the implementation).

Linux `traceroute` uses UDP by default (to high ports); Windows `tracert` uses ICMP Echo Request. `mtr` combines ping and traceroute in a continuous display — best for diagnosing intermittent issues.

> **Important rules:** an ICMP error about another ICMP error is never generated (prevents storms). ICMP errors are never sent for broadcast/multicast packets. ICMP errors always include the original packet's IP header + first 8 bytes of payload so the source can identify which flow caused the error.

### Routing

Every router has a **routing table** mapping destination prefixes to next-hop addresses and output interfaces. The core mechanism:

**Longest prefix match:** when multiple entries match a destination, the most specific one (longest mask) wins. This is how route aggregation and CIDR work.

```
  Packet journey: Host A (10.0.1.5) -> Host B (203.0.113.10)

  Host A           Router 1           Router 2           Host B
  10.0.1.5         10.0.1.1           203.0.113.1        203.0.113.10
    |                  |                  |                  |
    |-- frame ------>> |                  |                  |
    |  dst MAC: R1     |-- frame ------>> |                  |
    |  dst IP:  B      |  dst MAC: R2     |-- frame ------>> |
    |                  |  dst IP:  B      |  dst MAC: B      |
    |                  |                  |  dst IP:  B      |

  IP addresses stay the same end-to-end.
  MAC addresses change at every hop.
```

### Autonomous Systems and Routing Protocols

The Internet is divided into **AS (Autonomous Systems)**, each operated by one entity (ISP, cloud provider, enterprise).

- **IGP (Interior Gateway Protocol):** runs within an AS. Finds shortest paths.
    - **OSPF (Open Shortest Path First):** link-state (Dijkstra). Every router knows full topology. Fast convergence. Supports hierarchical areas. The standard for enterprise/ISP networks.
    - **RIP (Routing Information Protocol):** distance-vector (Bellman-Ford). Hop count metric, max 15. Simple but slow to converge. Mostly legacy.
- **EGP (Exterior Gateway Protocol):** runs between ASes.
    - **BGP (Border Gateway Protocol):** the protocol that holds the Internet together. Path-vector, policy-driven. Routing decisions consider business relationships, not just shortest path.

**Why BGP matters for software engineers:**

- **Anycast** (same IP from multiple locations) is built on BGP — used by CDNs, DNS (8.8.8.8), Cloudflare.
- BGP hijacks/misconfigurations can take your service offline or redirect traffic.
- Multi-region failover and CDN routing depend on BGP behavior.

---

## 5. Layer 4 — Transport (TCP, UDP, QUIC)

The transport layer provides end-to-end communication between processes on different hosts, identified by **port numbers**.

### Port Numbers

Assigned by IANA (Internet Assigned Numbers Authority). Well-known ports (0-1023) require root/admin privileges. Ephemeral ports (>1023, typically 49152-65535) are used by clients.

|Port|Service|Port|Service|
|---|---|---|---|
|22|SSH (Secure Shell)|443|HTTPS|
|25|SMTP (Simple Mail Transfer Protocol)|3306|MySQL|
|53|DNS|5432|PostgreSQL|
|80|HTTP|6379|Redis|
|110|POP3 (Post Office Protocol)|8080|HTTP (alternate)|
|143|IMAP (Internet Message Access Protocol)|27017|MongoDB|

### Sockets — Deep Dive

A **socket** is one endpoint of a network connection — an abstraction provided by the OS that lets applications send and receive data over the network. Think of it as a file descriptor for a network connection.

#### What identifies a socket

- **UDP socket:** identified by the tuple `(local IP, local port)`. Any host can send datagrams to it. There's no "connection" — each datagram is independent.
- **TCP socket:** a listening socket is identified by `(local IP, local port)`. Once a connection is accepted, the connected socket is identified by the full **5-tuple:** `(protocol, local IP, local port, remote IP, remote port)`.

This is how a web server handles thousands of connections on port 443 — each client creates a unique 5-tuple because they have different (source IP, source port) combinations.

#### Socket lifecycle (TCP)

```
  Server                                Client
    |                                      |
    |  socket()     -- create socket       |
    |  bind()       -- assign address      |
    |  listen()     -- mark as passive     |
    |                                      |  socket()
    |                                      |  connect()  --> SYN
    |  accept()     <-- SYN received       |
    |               --> SYN-ACK            |
    |               <-- ACK                |  connect() returns
    |  accept() returns new socket         |
    |                                      |
    |  read()/write()  <--- data --->  read()/write()
    |                                      |
    |  close()      --> FIN                |
    |               <-- ACK, FIN           |  close()
    |               --> ACK                |
```

#### Socket types in practice

|Type|C constant|Protocol|Behavior|
|---|---|---|---|
|Stream|`SOCK_STREAM`|TCP|Reliable, ordered byte stream|
|Datagram|`SOCK_DGRAM`|UDP|Unreliable, independent messages|
|Raw|`SOCK_RAW`|Any|Direct access to IP layer (requires root)|

#### Practical socket concepts

- **Backlog queue:** `listen(fd, backlog)` sets how many pending connections the OS will queue before refusing new ones. If your server is slow to `accept()`, clients get connection refused when the backlog fills.
- **`SO_REUSEADDR`:** allows binding to an address that's in TIME_WAIT. Essential for servers that restart frequently — without it, you get "address already in use" errors.
- **`SO_REUSEPORT`:** allows multiple sockets to bind to the same address/port. The kernel distributes incoming connections across them. Used for multi-process/multi-thread server architectures (Nginx, Go).
- **Non-blocking I/O and multiplexing:** production servers don't use one thread per connection. They use `epoll` (Linux), `kqueue` (macOS/BSD), or `io_uring` (modern Linux) to handle thousands of connections in one thread via event loops. This is what Node.js, Nginx, and Go's runtime use under the hood.
- **`TCP_NODELAY`:** disables Nagle's algorithm, which batches small writes into larger segments. For latency-sensitive applications (real-time, interactive), enable `TCP_NODELAY` to send data immediately.
- **Keep-alive:** `SO_KEEPALIVE` sends periodic probes on idle connections to detect dead peers. Configurable timers: how long before first probe, interval between probes, how many probes before giving up.

### TCP (Transmission Control Protocol)

Reliable, ordered, byte-stream delivery. Turns unreliable IP into a pipe applications can trust.

#### Three-Way Handshake

```
  Client                            Server
    |                                  |
    |--- SYN (seq=x) --------------->>|  Client picks ISN
    |                                  |  (Initial Sequence Number) x
    |<<- SYN-ACK (seq=y, ack=x+1) ----|  Server picks ISN y,
    |                                  |  acknowledges x+1
    |--- ACK (seq=x+1, ack=y+1) ---->>|  Connection open.
    |                                  |  Can carry data.
    |         <--- 1 RTT --->          |
```

Costs **1 RTT (Round-Trip Time)** before data flows. Add TLS 1.3 (another RTT) = **2 RTTs** before the first byte of application data on a new HTTPS connection.

#### Connection Teardown

```
  Side A                            Side B
    |--- FIN ---------------------->>|
    |<<- ACK ------------------------|
    |<<- FIN ------------------------|
    |--- ACK ---------------------->>|
    |                                |
    |  A enters TIME_WAIT (~60s)     |
```

**TIME_WAIT** holds the socket to handle delayed duplicates. On busy servers, thousands of TIME_WAIT sockets can exhaust ephemeral ports. Tunable via `net.ipv4.tcp_tw_reuse` on Linux.

#### Reliability

TCP retransmits lost segments, detected by **timeout** or **3 duplicate ACKs** (fast retransmit). The RTO (Retransmission Timeout) adapts dynamically to measured RTT.

**SACK (Selective Acknowledgment):** the receiver reports exactly which byte ranges it has, so only truly lost segments are retransmitted. Without SACK, TCP falls back to Go-Back-N (retransmit everything from the lost segment onward).

#### Flow Control

The receiver advertises available buffer space in the **window** field. The sender limits in-flight data to this value. Window = 0 means "stop sending" — the sender probes periodically until the window opens.

#### Congestion Control

TCP regulates sending rate to avoid collapsing the network. The sender maintains a **cwnd (congestion window)** — the actual window is the minimum of cwnd and the receiver's advertised window.

```
  cwnd
   ^
   |        /\      /\      /\
   |       /  \    /  \    /  \       <-- multiplicative decrease (halve)
   |      /    \  /    \  /    \
   |     /      \/      \/      \     <-- additive increase (linear)
   |    /
   |   / <-- slow start (exponential)
   |  /
   | /
   +--------------------------------> time
```

1. **Slow start:** cwnd starts at 1 MSS, doubles every RTT (exponential) until ssthresh or loss.
2. **Congestion avoidance:** cwnd grows by 1 MSS per RTT (linear).
3. **On timeout:** ssthresh = cwnd/2, cwnd = 1, back to slow start.
4. **On 3 dup ACKs (fast recovery):** ssthresh = cwnd/2, cwnd = ssthresh, stay linear.

**Modern algorithms:** **CUBIC** (Linux default, more aggressive recovery), **BBR** (Google, estimates bandwidth + RTT instead of relying on loss signals, much better on lossy/long-distance links).

#### Why TCP Matters for Application Code

- **New connections start slow.** Short-lived connections pay full handshake + slow start cost every time. This is why connection pooling and keep-alive exist.
- **Head-of-line blocking:** one lost packet stalls all data on that connection, even if it belongs to independent streams. This is HTTP/2's problem over TCP.
- **Loss = congestion assumption.** On lossy wireless links, this is wrong and throughput collapses.

#### Key TCP Options

|Option|Purpose|
|---|---|
|MSS (Maximum Segment Size)|Negotiated at handshake (typically 1460)|
|Window Scaling|Allows windows > 65535 bytes (essential for high-bandwidth links)|
|Timestamps|RTT measurement, duplicate detection via PAWS (Protection Against Wrapped Sequences)|
|SACK (Selective Acknowledgment)|Reports received byte ranges so only lost segments are retransmitted|

### UDP (User Datagram Protocol)

Minimal transport: adds port numbers and an optional checksum on top of IP. No connection, no reliability, no ordering, no congestion control. **8 bytes of header** vs. 20+ for TCP.

```
  UDP Header
   0               16              32
   +---------------+---------------+
   |  Source Port   |   Dest Port   |
   +---------------+---------------+
   |    Length      |   Checksum    |
   +---------------+---------------+
   |           Payload              |
   +-------------------------------+
```

**When to use UDP:** DNS queries, real-time media (VoIP, video), gaming, DHCP, NTP (Network Time Protocol), and as the substrate for QUIC.

**Pitfalls:** no congestion control (buggy senders can flood networks), NAT mappings expire fast (~30s, need keepalives), some firewalls block/rate-limit UDP.

### QUIC

Transport protocol designed by Google, running over UDP. Implements reliability, multiplexing, and congestion control in userspace. HTTP/3 runs over QUIC.

Key advantages over TCP+TLS:

- **Per-stream loss recovery:** eliminates head-of-line blocking.
- **Built-in TLS 1.3:** encryption is mandatory, not optional.
- **Faster setup:** 1 RTT (0-RTT on resumption) vs. 2-3 RTTs for TCP+TLS.
- **Connection migration:** survives IP changes (e.g., WiFi to cellular).

```
  TCP + TLS 1.3 (2 RTTs):           QUIC (1 RTT, 0-RTT on resumption):

  Client      Server                 Client      Server
    |-- SYN ---->|                     |-- Initial -->|  crypto + transport
    |<- SYNACK --|  1 RTT              |<- Handshk ---|  combined: 1 RTT
    |-- ACK ---->|                     |-- Data ----->|
    |-- TLS ---->|                     |              |
    |<- TLS -----|  1 RTT
    |-- Data --->|
```

---

## 6. Layer 5/6/7 — Session, Presentation, Application

In the TCP/IP model these are merged into the Application layer. This is where you spend most of your time as a software engineer.

### HTTP

The protocol of the web and the dominant API transport.

|Version|Transport|Key feature|Limitation|
|---|---|---|---|
|HTTP/1.1|TCP|Keep-alive, chunked encoding|1 request at a time per connection (HOL blocking)|
|HTTP/2|TCP|Binary framing, multiplexed streams, HPACK header compression|TCP-level HOL blocking on packet loss|
|HTTP/3|QUIC (UDP)|Per-stream loss recovery, 0-RTT, built-in TLS 1.3|UDP sometimes blocked by networks|

**HTTP request/response structure:**

```
  Request:                         Response:
  GET /api/users HTTP/1.1          HTTP/1.1 200 OK
  Host: example.com                Content-Type: application/json
  Authorization: Bearer xyz        Content-Length: 128

                                   {"users": [...]}
```

**HTTP methods:** GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE (remove), HEAD (metadata only), OPTIONS (CORS preflight).

**Status codes to know:**

- 2xx: success (200 OK, 201 Created, 204 No Content)
- 3xx: redirect (301 Moved Permanently, 302 Found, 304 Not Modified)
- 4xx: client error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- 5xx: server error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout)

### TLS (Transport Layer Security) — Deep Dive

TLS encrypts the communication channel between two parties, providing three guarantees: **confidentiality** (nobody can read the data), **integrity** (nobody can modify the data undetected), and **authentication** (you're talking to who you think you are).

#### TLS versions

|Version|Status|Handshake RTTs|Key difference|
|---|---|---|---|
|TLS 1.0 / 1.1|Deprecated|2|Vulnerable, disabled in modern browsers|
|TLS 1.2|Supported|2|Still widely used, many cipher suites|
|TLS 1.3|Current standard|1 (0-RTT on resumption)|Faster, simpler, removed weak ciphers|

#### TLS 1.3 handshake in detail

```
  Client                                     Server
    |                                           |
    |-- ClientHello ------------------------->>|
    |   - supported cipher suites              |
    |   - key share (client's DH public key)   |
    |   - supported TLS versions               |
    |                                           |
    |<<- ServerHello + EncryptedExtensions -----|
    |   - chosen cipher suite                  |
    |   - key share (server's DH public key)   |
    |   - server certificate                   |
    |   - certificate verify (signature)       |
    |   - finished (MAC of handshake)          |
    |                                           |
    |   (both sides derive session keys         |
    |    from the DH key exchange)              |
    |                                           |
    |-- Finished ---------------------------->>|
    |   (MAC of handshake)                     |
    |                                           |
    |<<========= encrypted data ===========>>=|
```

**Key exchange:** TLS 1.3 exclusively uses ephemeral Diffie-Hellman (ECDHE — Elliptic Curve Diffie-Hellman Ephemeral). Every connection generates fresh keys, providing **forward secrecy** — if the server's private key is compromised later, past sessions can't be decrypted.

**0-RTT resumption:** if the client has connected before, it can send data in the first message using a PSK (Pre-Shared Key) from a previous session. Faster, but the 0-RTT data is not protected against replay attacks — only use it for idempotent requests.

#### Certificates and trust

- The server presents a **certificate** signed by a CA (Certificate Authority).
- The client verifies the chain: server cert -> intermediate CA -> root CA (trusted by the OS/browser).
- The certificate contains the server's public key and the domain(s) it's valid for (CN (Common Name) or SAN (Subject Alternative Name) field).
- **Let's Encrypt** provides free, automated certificates and has become the dominant CA.

#### TLS for engineers — practical knowledge

- **TLS termination:** where TLS gets decrypted. Could be at the load balancer (AWS ALB, Nginx), a reverse proxy, or the application itself. Traffic between the termination point and your backend may be unencrypted.
- **mTLS (mutual TLS):** both sides present certificates. The server verifies the client's certificate too. Standard in service mesh (Istio, Linkerd) and zero-trust architectures. Used to authenticate service-to-service communication.
- **Certificate pinning:** the client only trusts specific certificates (not any CA-signed cert). Used in mobile apps for extra security. Makes rotation harder — use with care.
- **Certificate transparency (CT):** public logs of all issued certificates. Lets you detect unauthorized certificates for your domain.
- **SNI (Server Name Indication):** TLS extension where the client specifies which hostname it's connecting to. Required for virtual hosting (multiple HTTPS sites on one IP). Sent in plaintext in TLS 1.2 — encrypted in TLS 1.3 via ECH (Encrypted Client Hello).
- **HSTS (HTTP Strict Transport Security):** response header telling browsers to always use HTTPS for the domain. Prevents SSL-stripping MITM attacks.

### WebSocket

Upgrades an HTTP/1.1 connection to persistent, full-duplex communication. After the initial HTTP handshake (`101 Switching Protocols`), both sides exchange frames freely. Used for chat, live updates, collaborative editing.

**Alternative:** SSE (Server-Sent Events) for simpler one-way push — it's plain HTTP and works with standard proxies/CDNs without special config.

### gRPC (Google Remote Procedure Call)

RPC (Remote Procedure Call) framework built on HTTP/2 + Protocol Buffers. Binary serialization, bidirectional streaming, code generation. Common in microservice architectures. Requires HTTP/2-aware infrastructure.

### REST vs. GraphQL vs. gRPC

|Aspect|REST|GraphQL|gRPC|
|---|---|---|---|
|Transport|HTTP/1.1 or HTTP/2|HTTP|HTTP/2|
|Data format|JSON (text)|JSON (text)|Protobuf (binary)|
|Schema|OpenAPI/Swagger (optional)|Required (SDL)|Required (.proto files)|
|Streaming|Limited (SSE, WebSocket)|Subscriptions (via WS)|Native bidirectional streaming|
|Overfetching|Common (fixed responses)|Solved (client picks fields)|N/A (defined per method)|
|Browser support|Native|Native|Needs grpc-web proxy|
|Best for|Public APIs, web apps|Flexible queries, BFF|Internal microservices, perf-critical|

---

## 7. Network Services (DHCP, NAT, DNS)

### DHCP (Dynamic Host Configuration Protocol)

DHCP automatically configures hosts to join a network. Without it, every device would need manual IP configuration. It assigns: IP address, subnet mask, default gateway, DNS servers, and a lease duration.

Runs over UDP: client uses port 68, server uses port 67. Works via broadcast because the client doesn't have an IP yet.

#### DORA Process

```
  Client (no IP yet)              DHCP Server (10.0.0.1)
    |                                |
    |-- DISCOVER (broadcast) ------>>|  "I need an IP address"
    |   src: 0.0.0.0:68             |   (may reach multiple DHCP servers)
    |   dst: 255.255.255.255:67     |
    |                                |
    |<<- OFFER ----------------------|  "Here, you can use 10.0.0.15"
    |   offered IP: 10.0.0.15       |   (server reserves this IP)
    |   lease: 24 hours             |
    |   gateway: 10.0.0.1           |
    |   DNS: 8.8.8.8               |
    |                                |
    |-- REQUEST (broadcast) ------->>|  "I'll take 10.0.0.15"
    |   (broadcast so other DHCP     |   (if multiple servers offered,
    |    servers know to release     |    this tells them which was chosen)
    |    their offers)               |
    |                                |
    |<<- ACK -----------------------|  "Confirmed. Lease starts now."
    |                                |
    |  Client configures interface   |
    |  with assigned parameters      |
```

#### DHCP lease lifecycle

- **Lease duration:** configurable (1 hour to 30 days typical). Short leases in guest WiFi, long leases for static infrastructure.
- **Renewal:** at 50% of lease time (T1), client sends REQUEST directly to the server (unicast). If no response by 87.5% (T2), client broadcasts REQUEST to any DHCP server.
- **Release:** client sends RELEASE when disconnecting (voluntary, not guaranteed).
- **Reservation:** DHCP server can assign a fixed IP to a specific MAC address. Combines the convenience of DHCP with the predictability of static assignment. Commonly used for printers, servers, and network gear.

#### DHCP relay

DHCP uses broadcast, which doesn't cross routers. In multi-subnet networks, a **DHCP relay agent** (usually configured on the router) forwards DHCP broadcasts from the local subnet to the DHCP server on another subnet as unicast.

> **Debugging tip:** if a host has `169.254.x.x` (link-local), DHCP failed. Check: is the DHCP server running? Is the relay agent configured? Is the scope exhausted (no IPs left)?

### NAT (Network Address Translation)

NAT allows devices with private IP addresses to communicate with the public Internet by rewriting IP addresses (and ports) in packet headers as they pass through the NAT device.

#### Types of NAT

|Type|How it works|Use case|
|---|---|---|
|**SNAT (Source NAT)**|Rewrites source IP on outbound packets|Most common: private hosts accessing the Internet|
|**DNAT (Destination NAT)**|Rewrites destination IP on inbound packets|Port forwarding, load balancing|
|**PAT (Port Address Translation)**|Maps many private IPs to one public IP using different ports|Home routers, cloud NAT gateways|
|**1:1 NAT**|Maps one private IP to one public IP|Servers that need a dedicated public IP|

Most consumer and cloud NAT is **PAT** (also called "NAT overload" or "NAPT").

#### How PAT works

```
  Private network               NAT Router                  Internet

  +------------+
  | 10.0.0.2   |- src:10.0.0.2:4921 ->+-----------+-- src:82.1.2.3:5001 --> Server
  | 10.0.0.3   |- src:10.0.0.3:8080 ->| NAT table |-- src:82.1.2.3:5002 --> 203.0.113.10
  +------------+                      | 5001 <-> 10.0.0.2:4921 |
                                      | 5002 <-> 10.0.0.3:8080 |
                                      +-----------+
                                      Public IP: 82.1.2.3

  Return traffic: server sends to 82.1.2.3:5001 -->
  NAT router looks up table, rewrites dst to 10.0.0.2:4921, forwards to private network.
```

#### NAT implications for software engineers

- **Inbound connections:** NAT only creates mappings for outbound traffic. Inbound connections to private hosts require port forwarding or NAT traversal:
    - **STUN (Session Traversal Utilities for NAT):** discovers the public IP/port mapping. Works when both sides are behind "simple" NATs.
    - **TURN (Traversal Using Relays around NAT):** relays traffic through a public server. Works always but adds latency and costs.
    - **ICE (Interactive Connectivity Establishment):** tries STUN first, falls back to TURN. Used by WebRTC.
- **UDP mapping timeout:** NAT entries for UDP expire fast (~30 seconds). Long-lived UDP flows (VoIP, gaming, VPN) need periodic keepalive packets.
- **TCP mapping timeout:** longer (~minutes), but idle connections can still be dropped. TCP keepalive helps.
- **Protocols with embedded IPs:** SIP, FTP active mode, and some gaming protocols embed IP addresses in the payload. NAT doesn't rewrite payloads, so these break without an ALG (Application Level Gateway).
- **Cloud NAT layers:** in AWS/GCP/Azure, you often have: instance private IP -> VPC subnet -> NAT gateway -> public IP. Each layer has its own mapping table and timeouts.
- **Port exhaustion:** 16-bit port field = ~65K mappings per public IP. High-traffic NAT gateways can exhaust this, causing connection failures. Solution: add more public IPs.
- **Hairpin NAT:** when a host behind NAT tries to access another host behind the same NAT using the public IP. Not all NAT implementations support this — can cause confusing connectivity issues.

### DNS (Domain Name System)

Translates names to IP addresses. Globally distributed, hierarchical, eventually-consistent database.

#### Resolution Process

```
  Your app              Recursive Resolver       Root        .com TLD      Authoritative NS
  (getaddrinfo)         (8.8.8.8)                Server      Server        for example.com
     |                       |                      |           |               |
     |-- example.com? ----->|                       |           |               |
     |                       |-- where is .com? --->|           |               |
     |                       |<- try 192.5.6.30 ----|           |               |
     |                       |-- where is example.com? ------->|               |
     |                       |<- try ns1.example.com ----------|               |
     |                       |-- A record? ---------------------------------------->|
     |                       |<- 93.184.216.34 (TTL 3600) -------------------------|
     |<- 93.184.216.34 -----|   (cached for 3600s)                                 |
```

#### Record Types

|Type|Purpose|Example|
|---|---|---|
|**A**|Name -> IPv4|`example.com -> 93.184.216.34`|
|**AAAA**|Name -> IPv6|`example.com -> 2606:2800:...`|
|**CNAME**|Alias to another name|`www.example.com -> example.com`|
|**MX**|Mail server for the domain|`example.com -> mail.example.com`|
|**NS**|Authoritative nameserver|`example.com -> ns1.example.com`|
|**TXT**|Arbitrary text (SPF, DKIM, verification)|`"v=spf1 include:..."`|
|**SRV**|Service discovery (host + port)|`_sip._tcp.example.com`|
|**PTR**|Reverse lookup (IP -> name)|`34.216.184.93 -> example.com`|

#### DNS in Practice

- **TTL (Time To Live):** low (60s) = fast failover, more queries. High (3600s) = fewer queries, slow changes. Typical production: 300-3600s.
- **DNS load balancing:** return different IPs per query (round-robin, geo-based, weighted). Every CDN does this.
- **Debugging:** `dig +short example.com` for quick lookup, `dig +trace` for full resolution chain.
- **DoH (DNS over HTTPS) / DoT (DNS over TLS):** encrypts queries, increasingly default in browsers.

> **Security threats:** DNS spoofing/cache poisoning, DNS exfiltration (encoding data in queries), DNS amplification (DDoS via open resolvers). DNSSEC (DNS Security Extensions) adds cryptographic signatures but adoption is slow.

---

## 8. Network Security

### Firewalls

|Type|Layer|What it does|
|---|---|---|
|Packet filter|L3/L4|Allow/deny by IP, port, protocol. Stateless.|
|Stateful firewall|L3/L4|Tracks connections. Only allows return traffic for established sessions.|
|WAF (Web Application Firewall)|L7|Inspects HTTP content. Blocks SQLi (SQL Injection), XSS (Cross-Site Scripting), etc.|

In cloud: **security groups** (stateful, per-instance) and **NACLs (Network Access Control Lists)** (stateless, per-subnet).

### Common Attack Types

|Attack|Layer|How it works|Mitigation|
|---|---|---|---|
|ARP spoofing|L2|Fake ARP replies redirect traffic|DAI (Dynamic ARP Inspection), 802.1X|
|MAC flooding|L2|Overflow switch table -> broadcast mode|Port security|
|DNS spoofing|L7|False DNS responses|DNSSEC, DoH/DoT|
|Man-in-the-Middle|Any|Intercept/modify traffic|TLS, cert pinning|
|DDoS (Distributed Denial of Service)|L3/L4/L7|Overwhelm with traffic|Rate limiting, CDN, scrubbing|
|SYN flood|L4|Exhaust server connection state|SYN cookies|
|BGP hijack|L3|Announce someone else's prefixes|RPKI (Resource Public Key Infrastructure), monitoring|

### Security Concepts

- **Zero Trust:** never trust based on network location. Authenticate and authorize every request, even inside the perimeter.
- **mTLS (mutual TLS):** both client and server present certificates. Standard in service mesh (Istio, Linkerd).
- **VPN (Virtual Private Network):** encrypted tunnel. IPsec (L3) or WireGuard (modern, simpler). Connects remote users or sites securely.
- **Network segmentation:** VLANs, subnets, security groups. Limit blast radius of a compromise.
- **IDS/IPS (Intrusion Detection System / Intrusion Prevention System):** IDS monitors and alerts, IPS monitors and blocks. Can be network-based or host-based.

---

## 9. System Design Networking

### Latency vs. Bandwidth

**Bandwidth** = data per second (pipe diameter). **Latency** = time for one bit to arrive (pipe length). High bandwidth doesn't help latency-sensitive workloads (many sequential small requests).

**BDP (Bandwidth-Delay Product)** = bandwidth x RTT. The amount of data "in flight" to fully utilize a link. TCP's window must be at least this large.

### Speed of Light — The Hard Limit

Light in fiber travels at ~200,000 km/s. NY to London (~5,500 km) = ~28ms one-way, ~56ms RTT minimum. No protocol beats physics. This is why CDNs (Content Delivery Networks), edge computing, and geo-distributed architectures exist.

### Connection Pooling

TCP+TLS = 2-3 RTTs per new connection. **Always reuse connections:** HTTP keep-alive, database connection pools, gRPC persistent channels. New connections also start in TCP slow start, so throughput ramps up gradually.

### Head-of-Line (HOL) Blocking

A pattern that repeats across protocols:

- **HTTP/1.1:** one request blocks the connection.
- **HTTP/2 over TCP:** one lost packet stalls all multiplexed streams.
- **HTTP/3 / QUIC:** per-stream loss recovery. Solved.

### Load Balancing

|Type|Layer|Routes by|Examples|
|---|---|---|---|
|L4|Transport|IP/port (no payload inspection)|AWS NLB, HAProxy TCP mode|
|L7|Application|HTTP headers, paths, cookies|AWS ALB, Nginx, Envoy|

### Timeouts

Every network boundary needs them: **connection timeout** (handshake), **read timeout** (waiting for data), **idle timeout** (unused connection). Missing or misconfigured timeouts are one of the most common causes of cascading failures in distributed systems.

### The Fallacies of Distributed Computing

Eight false assumptions developers make:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every one will bite you in production. Design for failure: retries with exponential backoff, circuit breakers, idempotent operations.

---

## 10. Debugging & Tools

### Essential Toolkit

|Tool|What it does|Quick example|
|---|---|---|
|`ping`|ICMP echo — reachability + RTT|`ping -c 4 example.com`|
|`traceroute` / `mtr`|Path and per-hop latency|`mtr example.com`|
|`dig`|DNS queries|`dig +short example.com`|
|`curl -v`|HTTP with headers + TLS info|`curl -v https://example.com`|
|`ss` / `netstat`|Socket state, listening ports|`ss -tlnp`|
|`tcpdump`|Packet capture|`tcpdump -i eth0 port 443 -w cap.pcap`|
|`openssl s_client`|TLS connection test|`openssl s_client -connect host:443`|
|`nmap`|Port scanning|`nmap -sT -p 80,443 host`|

### Debugging Checklist: "It Doesn't Connect"

```
  1. DNS      ->  does the name resolve? right IP?        dig example.com
  2. REACH    ->  can you reach the IP?                   ping / mtr
  3. PORT     ->  is the port open? service listening?    nc -zv host 443 / ss -tlnp
  4. TLS      ->  valid cert? right hostname? expired?    openssl s_client
  5. APP      ->  is the app returning errors?            curl -v
  6. FIREWALL ->  security groups / iptables / NACLs?     check cloud console
```

### Metrics to Monitor

- **RTT distribution** (p50, p95, p99) by destination.
- **Packet loss** — even 0.1% significantly impacts TCP throughput.
- **TCP retransmissions** — high rate = network or buffer problem.
- **Connection errors** — refused / timed out / reset.
- **DNS resolution time** — slow DNS = slow everything.
- **TIME_WAIT count** — excessive = ephemeral port exhaustion risk.