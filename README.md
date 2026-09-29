# SR Linux EVPN with Interface-Ful (IFF) Symmetric IRB

> [!NOTE]
> **EVPN Symmetric IRB Architectural Variants in this Series:**
> - **Method 1 (Interface-Less with Dual-Label Type-2, RFC 9135):** [srl-evpn-ifl-irb](https://github.com/andywhitaker/srl-evpn-ifl-irb)
> - **Method 2 (Interface-Less with Type-5 Host Routes, RFC 9136 §4.3):** [srl-evpn-type5-irb](https://github.com/andywhitaker/srl-evpn-type5-irb)
> - **Method 3 (This Lab - Interface-Ful with SBD, RFC 9136 §4.4):** [srl-evpn-iff-irb](https://github.com/andywhitaker/srl-evpn-iff-irb)

## Topology
![topology](lab-topology.png)

## Lab Description
This lab demonstrates **SR Linux EVPN using Symmetric IRB with Interface-Ful (IFF) Inter-Subnet Routing**, utilizing a **Supplementary Broadcast Domain (SBD)** as defined in **RFC 9135** and **RFC 9136 Section 4.4**.

### Key Differences Between IFL and IFF Symmetric IRB

| Architectural Dimension | Interface-Less (IFL) IRB | Interface-Ful (IFF) IRB with SBD |
| :--- | :--- | :--- |
| **L3 VNI Termination** | Terminated directly in the IP-VRF (`type routed` on `vxlan0.100`). No transit MAC-VRF exists. | Terminated in a dedicated **Supplementary Broadcast Domain (SBD)** MAC-VRF (`type mac-vrf`, `vxlan0.100` is `type bridged`). |
| **IP-VRF to Transit VNI Binding** | Direct `vxlan-interface` association inside the `ip-vrf`. | Via an **unnumbered SBD IRB interface** (`irb0.100` configured with `evpn-interface-ful-unnumbered`) binding the SBD MAC-VRF to the IP-VRF. |
| **Host Route Signaling** | EVPN **Route Type 2 (MAC-IP)** carrying **dual VNIs/labels** (Label 1 = MAC-VRF VNI, Label 2 = IP-VRF VNI) and dual Route Targets. | EVPN **Route Type 5 (IP Prefix)** advertised by the SBD MAC-VRF with VNI 10000 and the router MAC carried in the Gateway MAC / Router's MAC extended community. |
| **Inter-Subnet Data Plane** | VXLAN header carries the tenant IP-VRF VNI. The inner payload is directly an IP packet (or Ethernet frame addressed to router MAC). | VXLAN header carries the **SBD VNI (10000)**. The inner Ethernet frame is addressed from the ingress PE router MAC to the egress PE router MAC (`irb0.100` MAC). |
| **Tenant IP-VRF Complexity** | `tenant1` requires `vxlan-interface`, `protocols bgp-evpn`, and `protocols bgp-vpn` with export/import RTs. | `tenant1` is a clean L3 IP-VRF containing only IRB subinterfaces (`irb0.1`, `irb0.2`, `irb0.100`). EVPN and BGP-VPN are isolated to the SBD MAC-VRF. |
| **Multi-Vendor Interoperability** | Supported on modern platforms that implement pure EVPN IFL. | Essential for interoperability with platforms that require an SBD or implement RFC 9136 IFF unnumbered, as well as EVPN OISM (RFC 9251). |

---

## Containerlab Deployment

```
╭────────┬───────────────────────────────────────────┬─────────┬───────────────────╮
│  Name  │                 Kind/Image                │  State  │   IPv4/6 Address  │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ app1   │ linux                                     │ running │ 172.20.20.2       │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::2 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ app2   │ linux                                     │ running │ 172.20.20.7       │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::7 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ app3   │ linux                                     │ running │ 172.20.20.4       │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::4 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ app4   │ linux                                     │ running │ 172.20.20.6       │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::6 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ leaf1  │ nokia_srlinux                             │ running │ 172.20.20.13      │
│        │ ghcr.io/nokia/srlinux:26.3.1              │         │ 3fff:172:20:20::d │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ leaf2  │ nokia_srlinux                             │ running │ 172.20.20.11      │
│        │ ghcr.io/nokia/srlinux:26.3.1              │         │ 3fff:172:20:20::b │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ leaf3  │ nokia_srlinux                             │ running │ 172.20.20.3       │
│        │ ghcr.io/nokia/srlinux:26.3.1              │         │ 3fff:172:20:20::3 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ leaf4  │ nokia_srlinux                             │ running │ 172.20.20.8       │
│        │ ghcr.io/nokia/srlinux:26.3.1              │         │ 3fff:172:20:20::8 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ spine1 │ nokia_srlinux                             │ running │ 172.20.20.15      │
│        │ ghcr.io/nokia/srlinux:26.3.1              │         │ 3fff:172:20:20::f │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ spine2 │ nokia_srlinux                             │ running │ 172.20.20.9       │
│        │ ghcr.io/nokia/srlinux:26.3.1              │         │ 3fff:172:20:20::9 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ web1   │ linux                                     │ running │ 172.20.20.10      │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::a │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ web2   │ linux                                     │ running │ 172.20.20.12      │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::c │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ web3   │ linux                                     │ running │ 172.20.20.5       │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::5 │
├────────┼───────────────────────────────────────────┼─────────┼───────────────────┤
│ web4   │ linux                                     │ running │ 172.20.20.14      │
│        │ ghcr.io/srl-labs/network-multitool:latest │         │ 3fff:172:20:20::e │
╰────────┴───────────────────────────────────────────┴─────────┴───────────────────╯
```

---

## Configuration Architecture

### 1. Supplementary Broadcast Domain (SBD) MAC-VRF
```text
set / tunnel-interface vxlan0 vxlan-interface 100 type bridged
set / tunnel-interface vxlan0 vxlan-interface 100 ingress vni 10000
set / tunnel-interface vxlan0 vxlan-interface 100 egress source-ip use-system-ipv4-address

set / network-instance sbd type mac-vrf
set / network-instance sbd admin-state enable
set / network-instance sbd description "SBD for tenant1"
set / network-instance sbd interface irb0.100
set / network-instance sbd vxlan-interface vxlan0.100
set / network-instance sbd protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan0.100
set / network-instance sbd protocols bgp-evpn bgp-instance 1 evi 10000
set / network-instance sbd protocols bgp-evpn bgp-instance 1 ecmp 8
set / network-instance sbd protocols bgp-evpn bgp-instance 1 supplementary-broadcast-domain
set / network-instance sbd protocols bgp-evpn bgp-instance 1 routes route-table ip-prefix advertise-interface-ful true
set / network-instance sbd protocols bgp-vpn bgp-instance 1 route-target export-rt target:10000:10000
set / network-instance sbd protocols bgp-vpn bgp-instance 1 route-target import-rt target:10000:10000
```

### 2. Unnumbered SBD IRB Subinterface
```text
set / interface irb0 subinterface 100 description "SBD IRB for tenant1"
set / interface irb0 subinterface 100 evpn-interface-ful-unnumbered
set / interface irb0 subinterface 100 ipv4 admin-state enable

set / network-instance tenant1 interface irb0.100
set / network-instance sbd interface irb0.100
```

### 3. Host Route Population in Tenant IP-VRF
```text
set / interface irb0 subinterface 1 ipv4 arp host-route populate dynamic
set / interface irb0 subinterface 2 ipv4 arp host-route populate dynamic
```

---

## Validation

### Client Pings
All client devices can ping both intra-subnet and inter-subnet across the fabric:

| Client | Bridge Domain | Connected To | IP Address   |
| :--- | :--- | :--- | :--- |
| app1   | app           | leaf1        | 192.168.10.1 |
| app2   | app           | leaf2        | 192.168.10.2 |
| app3   | app           | leaf3        | 192.168.10.3 |
| app4   | app           | leaf4        | 192.168.10.4 |
| web1   | web           | leaf1        | 192.168.20.1 |
| web2   | web           | leaf2        | 192.168.20.2 |
| web3   | web           | leaf3        | 192.168.20.3 |
| web4   | web           | leaf4        | 192.168.20.4 |

#### Inter-Subnet Routed Ping (app1 -> web2 across SBD VNI 10000):
```bash
/ # ping -c 5 192.168.20.2
PING 192.168.20.2 (192.168.20.2) 56(84) bytes of data.
64 bytes from 192.168.20.2: icmp_seq=1 ttl=62 time=1.31 ms
64 bytes from 192.168.20.2: icmp_seq=2 ttl=62 time=1.60 ms
64 bytes from 192.168.20.2: icmp_seq=3 ttl=62 time=2.16 ms
64 bytes from 192.168.20.2: icmp_seq=4 ttl=62 time=2.41 ms
64 bytes from 192.168.20.2: icmp_seq=5 ttl=62 time=1.81 ms

--- 192.168.20.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 1.307/1.859/2.410/0.392 ms
```
*(Notice `ttl=62`: the packet is decremented twice—once by leaf1 routing into SBD VNI 10000, and once by leaf2 routing from SBD VNI 10000 to destination subnet `web`.)*

---

### BGP Neighbors

```text
A:admin@leaf1# show network-instance protocols bgp neighbor
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
|       Net-Inst       |              Peer              |        Group         | Flags  |  Peer-AS   |      State       |      Uptime      |    AFI/SAFI    |         [Rx/Active/Tx]         |
+======================+================================+======================+========+============+==================+==================+================+================================+
| default              | 10.1.10.10                     | ebgp-evpn            | S      | 100        | established      | 0d:1h:8m:31s     | evpn           | [39/39/13]                     |
|                      |                                |                      |        |            |                  |                  | ipv4-unicast   | [4/4/2]                        |
| default              | 10.1.20.20                     | ebgp-evpn            | S      | 100        | established      | 0d:1h:8m:32s     | evpn           | [39/0/52]                      |
|                      |                                |                      |        |            |                  |                  | ipv4-unicast   | [4/4/5]                        |
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
2 configured neighbors, 2 configured sessions are established, 0 disabled peers
0 dynamic peers
```

---

### MAC Address Table
In IFF, `network-instance sbd` maintains a bridge table populated with the router MAC addresses of remote leaves learned across VNI 10000:

```text
A:admin@leaf1# show network-instance bridge-table mac-table all
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance app
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
|     Address     |                   Destination                    |   Dest   |     Type      | Active | Aging |     Not-      |   GBP Tags    |                   Last Update                    |
|                 |                                                  |  Index   |               |        |       |  Programmed   |               |                                                  |
|                 |                                                  |          |               |        |       |    Reason     |               |                                                  |
+=================+==================================================+==========+===============+========+=======+===============+===============+==================================================+
| 00:00:5E:00:01: | irb-interface                                    | 0        | irb-          | true   | N/A   | none          | N/A           | 2026-09-29T16:44:52.000Z                         |
| 01              |                                                  |          | interface-    |        |       |               |               |                                                  |
|                 |                                                  |          | anycast       |        |       |               |               |                                                  |
| 1A:21:05:FF:00: | vxlan-interface:vxlan0.101 vtep:2.2.2.2          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T16:45:06.000Z                         |
| 41              | vni:10010                                        | 3467     |               |        |       |               |               |                                                  |
| 1A:76:04:FF:00: | irb-interface                                    | 0        | irb-interface | true   | N/A   | none          | N/A           | 2026-09-29T16:44:52.000Z                         |
| 41              |                                                  |          |               |        |       |               |               |                                                  |
| 1A:7D:06:FF:00: | vxlan-interface:vxlan0.101 vtep:3.3.3.3          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T16:45:06.000Z                         |
| 41              | vni:10010                                        | 3473     |               |        |       |               |               |                                                  |
| 1A:F0:07:FF:00: | vxlan-interface:vxlan0.101 vtep:4.4.4.4          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T16:45:06.000Z                         |
| 41              | vni:10010                                        | 3477     |               |        |       |               |               |                                                  |
| AA:C1:AB:12:DF: | vxlan-interface:vxlan0.101 vtep:3.3.3.3          | 28510834 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T17:51:56.000Z                         |
| 09              | vni:10010                                        | 3473     |               |        |       |               |               |                                                  |
| AA:C1:AB:3E:6A: | ethernet-1/3.0                                   | 6        | learnt        | true   | 298   | none          | 0             | 2026-09-29T17:48:40.000Z                         |
| 2B              |                                                  |          |               |        |       |               |               |                                                  |
| AA:C1:AB:C8:EA: | vxlan-interface:vxlan0.101 vtep:4.4.4.4          | 28510834 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T17:51:56.000Z                         |
| 80              | vni:10010                                        | 3477     |               |        |       |               |               |                                                  |
| AA:C1:AB:E6:AA: | vxlan-interface:vxlan0.101 vtep:2.2.2.2          | 28510834 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T17:48:43.000Z                         |
| 59              | vni:10010                                        | 3467     |               |        |       |               |               |                                                  |
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance sbd
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
|     Address     |                   Destination                    |   Dest   |     Type      | Active | Aging |     Not-      |   GBP Tags    |                   Last Update                    |
|                 |                                                  |  Index   |               |        |       |  Programmed   |               |                                                  |
|                 |                                                  |          |               |        |       |    Reason     |               |                                                  |
+=================+==================================================+==========+===============+========+=======+===============+===============+==================================================+
| 1A:21:05:FF:00: | vxlan-interface:vxlan0.100 vtep:2.2.2.2          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T17:47:49.000Z                         |
| 41              | vni:10000                                        | 3566     |               |        |       |               |               |                                                  |
| 1A:76:04:FF:00: | irb-interface                                    | 0        | irb-interface | true   | N/A   | none          | N/A           | 2026-09-29T17:40:21.000Z                         |
| 41              |                                                  |          |               |        |       |               |               |                                                  |
| 1A:7D:06:FF:00: | vxlan-interface:vxlan0.100 vtep:3.3.3.3          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T17:47:50.000Z                         |
| 41              | vni:10000                                        | 3567     |               |        |       |               |               |                                                  |
| 1A:F0:07:FF:00: | vxlan-interface:vxlan0.100 vtep:4.4.4.4          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T17:47:51.000Z                         |
| 41              | vni:10000                                        | 3568     |               |        |       |               |               |                                                  |
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance web
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
|     Address     |                   Destination                    |   Dest   |     Type      | Active | Aging |     Not-      |   GBP Tags    |                   Last Update                    |
|                 |                                                  |  Index   |               |        |       |  Programmed   |               |                                                  |
|                 |                                                  |          |               |        |       |    Reason     |               |                                                  |
+=================+==================================================+==========+===============+========+=======+===============+===============+==================================================+
| 00:00:5E:00:01: | irb-interface                                    | 0        | irb-          | true   | N/A   | none          | N/A           | 2026-09-29T16:44:52.000Z                         |
| 01              |                                                  |          | interface-    |        |       |               |               |                                                  |
|                 |                                                  |          | anycast       |        |       |               |               |                                                  |
| 1A:21:05:FF:00: | vxlan-interface:vxlan0.102 vtep:2.2.2.2          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T16:45:06.000Z                         |
| 41              | vni:10020                                        | 3468     |               |        |       |               |               |                                                  |
| 1A:76:04:FF:00: | irb-interface                                    | 0        | irb-interface | true   | N/A   | none          | N/A           | 2026-09-29T16:44:52.000Z                         |
| 41              |                                                  |          |               |        |       |               |               |                                                  |
| 1A:7D:06:FF:00: | vxlan-interface:vxlan0.102 vtep:3.3.3.3          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T16:45:06.000Z                         |
| 41              | vni:10020                                        | 3474     |               |        |       |               |               |                                                  |
| 1A:F0:07:FF:00: | vxlan-interface:vxlan0.102 vtep:4.4.4.4          | 28510834 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T16:45:06.000Z                         |
| 41              | vni:10020                                        | 3478     |               |        |       |               |               |                                                  |
| AA:C1:AB:59:4D: | vxlan-interface:vxlan0.102 vtep:4.4.4.4          | 28510834 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T17:51:57.000Z                         |
| 21              | vni:10020                                        | 3478     |               |        |       |               |               |                                                  |
| AA:C1:AB:6E:89: | vxlan-interface:vxlan0.102 vtep:3.3.3.3          | 28510834 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T17:51:57.000Z                         |
| 29              | vni:10020                                        | 3474     |               |        |       |               |               |                                                  |
| AA:C1:AB:D8:F4: | ethernet-1/4.0                                   | 7        | learnt        | true   | 286   | none          | 0             | 2026-09-29T17:48:45.000Z                         |
| 82              |                                                  |          |               |        |       |               |               |                                                  |
| AA:C1:AB:E3:EA: | vxlan-interface:vxlan0.102 vtep:2.2.2.2          | 28510834 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T17:48:50.000Z                         |
| F2              | vni:10020                                        | 3468     |               |        |       |               |               |                                                  |
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
```

---

### Underlay Routing Table

```text
A:admin@leaf1# show network-instance default ipv4 route
===============================================================================================================================================================================================
IPv4-unicast route table for default network-instance
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Flags: > (best), * (unviable), ! (failed)
     : L (leaked route from another network-instance)
     : B (backup NHG active and displayed)
     : S (statistics supported)
     : D (dynamic LB), R (resilient LB)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Prefix               Route Type   Metric   Pref    Flags    Next-Hop(s)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
2.2.2.2/32           bgp          0        170     >        10.1.10.10(route:local)                                                                                                                 
                                                            10.1.20.20(route:local)
3.3.3.3/32           bgp          0        170     >        10.1.10.10(route:local)                                                                                                                 
                                                            10.1.20.20(route:local)
4.4.4.4/32           bgp          0        170     >        10.1.10.10(route:local)                                                                                                                 
                                                            10.1.20.20(route:local)
10.1.10.0/24         local        0        0       >        10.1.10.1(ethernet-1/1.0)
10.1.20.0/24         local        0        0       >        10.1.20.1(ethernet-1/2.0)
10.10.10.10/32       bgp          0        170     >        10.1.10.10(route:local)
20.20.20.20/32       bgp          0        170     >        10.1.20.20(route:local)
```

---

### Tenant Routing Table
In IFF, remote host routes are installed with route type **`bgp-evpn-iff`**, resolving their next-hop via the unnumbered SBD IRB interface **`irb0.100`** with the remote router's Gateway MAC:

```text
A:admin@leaf1# show network-instance tenant1 ipv4 route
===============================================================================================================================================================================================
IPv4-unicast route table for ip-vrf network-instance: tenant1
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Flags: > (best), * (unviable), ! (failed)
     : L (leaked route from another network-instance)
     : B (backup NHG active and displayed)
     : S (statistics supported)
     : D (dynamic LB), R (resilient LB)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Prefix               Route Type   Metric   Pref    Flags    Next-Hop(s)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
192.168.10.0/24      local        0        0       >        192.168.10.254(irb0.1)
192.168.10.1/32      arp-nd       0        1       >        192.168.10.1(irb0.1)
192.168.10.2/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:21:05:FF:00:41)                                                                                                         
                     iff
192.168.10.3/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:7D:06:FF:00:41)                                                                                                         
                     iff
192.168.10.4/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:F0:07:FF:00:41)                                                                                                         
                     iff
192.168.20.0/24      local        0        0       >        192.168.20.254(irb0.2)
192.168.20.1/32      arp-nd       0        1       >        192.168.20.1(irb0.2)
192.168.20.2/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:21:05:FF:00:41)                                                                                                         
                     iff
192.168.20.3/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:7D:06:FF:00:41)                                                                                                         
                     iff
192.168.20.4/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:F0:07:FF:00:41)                                                                                                         
                     iff
```

---

### VXLAN Tunnel Table
In IFF, `vxlan0.100` is **`bridged`** (belonging to MAC-VRF `sbd`) rather than `routed`:

```text
A:admin@leaf1# show tunnel-interface vxlan-interface brief
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for vxlan-tunnels 
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+------------------+-----------------+---------+-------------+------------------+
| Tunnel Interface | VxLAN Interface |  Type   | Ingress VNI | Egress source-ip |
+==================+=================+=========+=============+==================+
| vxlan0           | vxlan0.100      | bridged | 10000       | 1.1.1.1/32       |
| vxlan0           | vxlan0.101      | bridged | 10010       | 1.1.1.1/32       |
| vxlan0           | vxlan0.102      | bridged | 10020       | 1.1.1.1/32       |
+------------------+-----------------+---------+-------------+------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary
  1 tunnel-interfaces, 3 vxlan interfaces
  15 vxlan-destinations, 9 unicast, 0 es, 6 multicast, 0 ip
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

---

### EVPN Routes

#### IMET Type-3 Routes
Type-3 (IMET) routes establish broadcast, unknown-unicast, and multicast (BUM) flood lists for the bridge domains:

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 3 summary
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "*"
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Type 3 Inclusive Multicast Ethernet Tag Routes
+--------+-----------------------------------------------+------------+---------------------+-----------------------------------------------+--------+-----------------------------------------------+
| Status |              Route-distinguisher              |   Tag-ID   |    Originator-IP    |                   neighbor                    | Path-  |                   Next-Hop                    |
|        |                                               |            |                     |                                               |   id   |                                               |
+========+===============================================+============+=====================+===============================================+========+===============================================+
| u*>    | 2.2.2.2:10010                                 | 0          | 2.2.2.2             | 10.1.10.10                                    | 0      | 2.2.2.2                                       |
| *      | 2.2.2.2:10010                                 | 0          | 2.2.2.2             | 10.1.20.20                                    | 0      | 2.2.2.2                                       |
| u*>    | 2.2.2.2:10020                                 | 0          | 2.2.2.2             | 10.1.10.10                                    | 0      | 2.2.2.2                                       |
| *      | 2.2.2.2:10020                                 | 0          | 2.2.2.2             | 10.1.20.20                                    | 0      | 2.2.2.2                                       |
| u*>    | 3.3.3.3:10010                                 | 0          | 3.3.3.3             | 10.1.10.10                                    | 0      | 3.3.3.3                                       |
| *      | 3.3.3.3:10010                                 | 0          | 3.3.3.3             | 10.1.20.20                                    | 0      | 3.3.3.3                                       |
| u*>    | 3.3.3.3:10020                                 | 0          | 3.3.3.3             | 10.1.10.10                                    | 0      | 3.3.3.3                                       |
| *      | 3.3.3.3:10020                                 | 0          | 3.3.3.3             | 10.1.20.20                                    | 0      | 3.3.3.3                                       |
| u*>    | 4.4.4.4:10010                                 | 0          | 4.4.4.4             | 10.1.10.10                                    | 0      | 4.4.4.4                                       |
| *      | 4.4.4.4:10010                                 | 0          | 4.4.4.4             | 10.1.20.20                                    | 0      | 4.4.4.4                                       |
| u*>    | 4.4.4.4:10020                                 | 0          | 4.4.4.4             | 10.1.10.10                                    | 0      | 4.4.4.4                                       |
| *      | 4.4.4.4:10020                                 | 0          | 4.4.4.4             | 10.1.20.20                                    | 0      | 4.4.4.4                                       |
+--------+-----------------------------------------------+------------+---------------------+-----------------------------------------------+--------+-----------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
12 Inclusive Multicast Ethernet Tag routes 6 used, 12 valid, 0 stale
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

#### MAC and MAC-IP Type 2 Routes
In IFF, Type-2 routes carry a single VNI (either VNI 10010 for `app`, 10020 for `web`, or 10000 for the SBD Router MAC). Notice that the dual-label encapsulation (`10010 + 10000`) found in IFL is absent:

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 2 summary
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "*"
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Type 2 MAC-IP Advertisement Routes
+-------+------------------+-----------+------------------+------------------+------------------+-------+------------------+------------------+-------------------------------+------------------+
| Statu |      Route-      |  Tag-ID   |   MAC-address    |    IP-address    |     neighbor     | Path- |     Next-Hop     |      Label       |              ESI              |   MAC Mobility   |
|   s   |  distinguisher   |           |                  |                  |                  |  id   |                  |                  |                               |                  |
+=======+==================+===========+==================+==================+==================+=======+==================+==================+===============================+==================+
| u*>   | 2.2.2.2:10000    | 0         | 1A:21:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10000    | 0         | 1A:21:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | 1A:21:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | 1A:21:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | AA:C1:AB:E6:AA:5 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 9                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | AA:C1:AB:E6:AA:5 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 9                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | 1A:21:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | 1A:21:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | AA:C1:AB:E3:EA:F | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 2                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | AA:C1:AB:E3:EA:F | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 2                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10000    | 0         | 1A:7D:06:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
+-------+------------------+-----------+------------------+------------------+------------------+-------+------------------+------------------+-------------------------------+------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
42 MAC-IP Advertisement routes 21 used, 42 valid, 0 stale
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

#### IP Prefix Type 5 Routes
In IFF, both subnet prefixes and `/32` host routes are advertised via **Type 5 IP Prefix** routes across the SBD MAC-VRF (VNI 10000):

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 5 summary
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "*"
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Type 5 IP Prefix Routes
+--------+----------------------------+------------+---------------------+----------------------------+--------+----------------------------+----------------------------+----------------------------+
| Status |    Route-distinguisher     |   Tag-ID   |     IP-address      |          neighbor          | Path-  |          Next-Hop          |           Label            |          Gateway           |
|        |                            |            |                     |                            |   id   |                            |                            |                            |
+========+============================+============+=====================+============================+========+============================+============================+============================+
| u*>    | 2.2.2.2:10000              | 0          | 192.168.10.0/24     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.10.0/24     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 2.2.2.2:10000              | 0          | 192.168.20.0/24     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.20.0/24     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 2.2.2.2:10000              | 0          | 192.168.10.2/32     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.10.2/32     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 2.2.2.2:10000              | 0          | 192.168.20.2/32     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.20.2/32     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.10.0/24     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.10.0/24     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.20.0/24     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.20.0/24     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.10.3/32     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.10.3/32     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.20.3/32     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.20.3/32     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.10.0/24     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.10.0/24     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.20.0/24     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.20.0/24     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.10.4/32     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.10.4/32     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.20.4/32     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.20.4/32     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
+--------+----------------------------+------------+---------------------+----------------------------+--------+----------------------------+----------------------------+----------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
24 IP Prefix routes 12 used, 24 valid, 0 stale
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```
