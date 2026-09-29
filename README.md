# SR Linux EVPN with Symmetric IRB (Interface-Ful with SBD)

> [!NOTE]
> **EVPN Symmetric IRB Architectural Variants in this Series:**
> - **Method 1 (Interface-Less with Dual-Label Type-2, RFC 9135):** [srl-evpn-ifl-irb](https://github.com/andywhitaker/srl-evpn-ifl-irb)
> - **Method 2 (Interface-Less with Type-5 Host Routes, RFC 9136 §4.3):** [srl-evpn-type5-irb](https://github.com/andywhitaker/srl-evpn-type5-irb)
> - **Method 3 (This Lab - Interface-Ful with SBD, RFC 9136 §4.4):** [srl-evpn-iff-irb](https://github.com/andywhitaker/srl-evpn-iff-irb)

## Topology
![topology](lab-topology.png)

## Lab Description
This lab demonstrates **Nokia SR Linux EVPN using Symmetric IRB with Interface-Ful (IFF) Routing**, utilizing a **Supplementary Broadcast Domain (SBD)** standardized under **RFC 9136 Section 4.4**.

In this architecture:
- **Supplementary Broadcast Domain (SBD):** The L3 VNI (10000) terminates inside a dedicated transit MAC-VRF (`sbd`) with `vxlan0.100 (type bridged)` and `supplementary-broadcast-domain` enabled.
- **Unnumbered SBD IRB Binding:** An unnumbered IRB interface (`irb0.100` configured with `evpn-interface-ful-unnumbered`) binds the transit SBD MAC-VRF to the tenant IP-VRF (`tenant1`), borrowing the IP-VRF router MAC.
- **EVPN Type-5 Route Propagation:** The SBD originates EVPN Type-5 IP Prefix routes with `advertise-interface-ful true`, carrying the SBD VNI (10000) and the Router's MAC extended community.
- **Two-Stage Next-Hop Resolution:** Ingress packets route from client subnets to `irb0.100`, which bridges across the SBD VNI encapsulated within a full Ethernet frame destined to the remote leaf's Router MAC (`bgp-evpn-iff`).
- **Standard for Multicast & Legacy ASICs:** This model is required for EVPN Optimized Inter-Subnet Multicast (OISM, RFC 9251) and platforms requiring L2 transit framing.

---

### Architectural Comparison: The Three Symmetric IRB Models

| Architectural Dimension | Method 1: IFL Dual-Label (RFC 9135) | Method 2: IFL Type-5 Host Routes (RFC 9136 §4.3) | Method 3: IFF with SBD (RFC 9136 §4.4) |
| :--- | :--- | :--- | :--- |
| **Standard / Reference** | RFC 9135 | RFC 9136 Section 4.3 | RFC 9136 Section 4.4 |
| **L3 VNI Network Instance** | `tenant1 (type ip-vrf)` | `tenant1 (type ip-vrf)` | `sbd (type mac-vrf)` |
| **VXLAN Interface Type** | `vxlan0.100 (type routed)` | `vxlan0.100 (type routed)` | `vxlan0.100 (type bridged)` |
| **Client MAC-VRF Type-2 Labels** | **Dual Labels:** `10010 + 10000` | **Single Label:** `10010` only | **Single Label:** `10010` only |
| **Host Route Carrier** | EVPN Type-2 MAC-IP | **EVPN Type-5 IP Prefix (/32)** | **EVPN Type-5 IP Prefix (/32)** via SBD |
| **Tenant Routing Table Type** | `bgp-evpn-ifl-host` | `bgp-evpn` | `bgp-evpn-iff` |
| **Next-Hop Resolution** | Direct to Remote VTEP Tunnel | Direct to Remote VTEP Tunnel | Two-Stage via SBD Bridge Table |
| **Inner Wire Payload** | Raw IPv4 / Direct L3 Payload | Raw IPv4 / Direct L3 Payload | Full Ethernet Frame (DMAC=Router MAC) |
| **Multicast Support (OISM)** | Unsupported | Unsupported | Mandatory for RFC 9251 OISM |
| **Target Use-Case** | Pure Nokia / Lowest BGP Prefix Count | Multi-Vendor / Hyperscale IP-VRFs | OISM Multicast & Legacy ASICs |

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
64 bytes from 192.168.20.2: icmp_seq=1 ttl=62 time=1.37 ms
64 bytes from 192.168.20.2: icmp_seq=2 ttl=62 time=2.13 ms
64 bytes from 192.168.20.2: icmp_seq=3 ttl=62 time=2.20 ms
64 bytes from 192.168.20.2: icmp_seq=4 ttl=62 time=1.45 ms
64 bytes from 192.168.20.2: icmp_seq=5 ttl=62 time=2.04 ms

--- 192.168.20.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4004ms
rtt min/avg/max/mdev = 1.373/1.837/2.203/0.354 ms
```
*(Notice `ttl=62`: the packet is decremented twice—once by leaf1 routing into SBD VNI 10000, and once by leaf2 routing from SBD VNI 10000 to destination subnet `web`.)*

---

### BGP Neighbors

```text
A:admin@leaf1# show network-instance protocols bgp neighbor
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
|       Net-Inst       |              Peer              |        Group         | Flags  |  Peer-AS   |      State       |      Uptime      |    AFI/SAFI    |         [Rx/Active/Tx]         |
+======================+================================+======================+========+============+==================+==================+================+================================+
| default              | 10.1.10.10                     | ebgp-evpn            | S      | 100        | established      | 0d:0h:2m:20s     | evpn           | [39/39/13]                     |
|                      |                                |                      |        |            |                  |                  | ipv4-unicast   | [4/4/2]                        |
| default              | 10.1.20.20                     | ebgp-evpn            | S      | 100        | established      | 0d:0h:2m:20s     | evpn           | [39/0/52]                      |
|                      |                                |                      |        |            |                  |                  | ipv4-unicast   | [4/4/5]                        |
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
2 configured neighbors, 2 configured sessions are established, 0 disabled peers
0 dynamic peers
```

---

### MAC Address Table
In IFF, `network-instance sbd` maintains a bridge table populated with the router MAC addresses of remote leaves learned across VNI 10000:

```text
A:admin@leaf1# show network-instance bridge-table mac-table all
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance app
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
|     Address     |                   Destination                    |   Dest   |     Type      | Active | Aging |     Not-      |   GBP Tags    |                   Last Update                    |
|                 |                                                  |  Index   |               |        |       |  Programmed   |               |                                                  |
|                 |                                                  |          |               |        |       |    Reason     |               |                                                  |
+=================+==================================================+==========+===============+========+=======+===============+===============+==================================================+
| 00:00:5E:00:01: | irb-interface                                    | 0        | irb-          | true   | N/A   | none          | N/A           | 2026-09-29T22:51:26.000Z                         |
| 01              |                                                  |          | interface-    |        |       |               |               |                                                  |
|                 |                                                  |          | anycast       |        |       |               |               |                                                  |
| 1A:17:06:FF:00: | vxlan-interface:vxlan0.101 vtep:3.3.3.3          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10010                                        | 7436     |               |        |       |               |               |                                                  |
| 1A:22:05:FF:00: | vxlan-interface:vxlan0.101 vtep:2.2.2.2          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10010                                        | 7431     |               |        |       |               |               |                                                  |
| 1A:76:04:FF:00: | irb-interface                                    | 0        | irb-interface | true   | N/A   | none          | N/A           | 2026-09-29T22:51:26.000Z                         |
| 41              |                                                  |          |               |        |       |               |               |                                                  |
| 1A:FD:07:FF:00: | vxlan-interface:vxlan0.101 vtep:4.4.4.4          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10010                                        | 7441     |               |        |       |               |               |                                                  |
| AA:C1:AB:18:BC: | vxlan-interface:vxlan0.101 vtep:3.3.3.3          | 28730778 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T22:51:55.000Z                         |
| FE              | vni:10010                                        | 7436     |               |        |       |               |               |                                                  |
| AA:C1:AB:5C:E4: | vxlan-interface:vxlan0.101 vtep:2.2.2.2          | 28730778 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T22:51:53.000Z                         |
| 54              | vni:10010                                        | 7431     |               |        |       |               |               |                                                  |
| AA:C1:AB:97:4D: | ethernet-1/3.0                                   | 7        | learnt        | true   | 225   | none          | 0             | 2026-09-29T22:51:53.000Z                         |
| 90              |                                                  |          |               |        |       |               |               |                                                  |
| AA:C1:AB:9C:09: | vxlan-interface:vxlan0.101 vtep:4.4.4.4          | 28730778 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T22:51:57.000Z                         |
| DC              | vni:10010                                        | 7441     |               |        |       |               |               |                                                  |
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance sbd
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
|     Address     |                   Destination                    |   Dest   |     Type      | Active | Aging |     Not-      |   GBP Tags    |                   Last Update                    |
|                 |                                                  |  Index   |               |        |       |  Programmed   |               |                                                  |
|                 |                                                  |          |               |        |       |    Reason     |               |                                                  |
+=================+==================================================+==========+===============+========+=======+===============+===============+==================================================+
| 1A:17:06:FF:00: | vxlan-interface:vxlan0.100 vtep:3.3.3.3          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10000                                        | 7438     |               |        |       |               |               |                                                  |
| 1A:22:05:FF:00: | vxlan-interface:vxlan0.100 vtep:2.2.2.2          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10000                                        | 7433     |               |        |       |               |               |                                                  |
| 1A:76:04:FF:00: | irb-interface                                    | 0        | irb-interface | true   | N/A   | none          | N/A           | 2026-09-29T22:51:26.000Z                         |
| 41              |                                                  |          |               |        |       |               |               |                                                  |
| 1A:FD:07:FF:00: | vxlan-interface:vxlan0.100 vtep:4.4.4.4          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10000                                        | 7442     |               |        |       |               |               |                                                  |
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance web
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
|     Address     |                   Destination                    |   Dest   |     Type      | Active | Aging |     Not-      |   GBP Tags    |                   Last Update                    |
|                 |                                                  |  Index   |               |        |       |  Programmed   |               |                                                  |
|                 |                                                  |          |               |        |       |    Reason     |               |                                                  |
+=================+==================================================+==========+===============+========+=======+===============+===============+==================================================+
| 00:00:5E:00:01: | irb-interface                                    | 0        | irb-          | true   | N/A   | none          | N/A           | 2026-09-29T22:51:26.000Z                         |
| 01              |                                                  |          | interface-    |        |       |               |               |                                                  |
|                 |                                                  |          | anycast       |        |       |               |               |                                                  |
| 1A:17:06:FF:00: | vxlan-interface:vxlan0.102 vtep:3.3.3.3          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10020                                        | 7437     |               |        |       |               |               |                                                  |
| 1A:22:05:FF:00: | vxlan-interface:vxlan0.102 vtep:2.2.2.2          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10020                                        | 7432     |               |        |       |               |               |                                                  |
| 1A:76:04:FF:00: | irb-interface                                    | 0        | irb-interface | true   | N/A   | none          | N/A           | 2026-09-29T22:51:26.000Z                         |
| 41              |                                                  |          |               |        |       |               |               |                                                  |
| 1A:FD:07:FF:00: | vxlan-interface:vxlan0.102 vtep:4.4.4.4          | 28730778 | evpn-static   | true   | N/A   | none          | 0             | 2026-09-29T22:51:38.000Z                         |
| 41              | vni:10020                                        | 7440     |               |        |       |               |               |                                                  |
| AA:C1:AB:4C:C3: | ethernet-1/4.0                                   | 8        | learnt        | true   | 279   | none          | 0             | 2026-09-29T22:51:59.000Z                         |
| 3D              |                                                  |          |               |        |       |               |               |                                                  |
| AA:C1:AB:61:F9: | vxlan-interface:vxlan0.102 vtep:4.4.4.4          | 28730778 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T22:52:09.000Z                         |
| 4C              | vni:10020                                        | 7440     |               |        |       |               |               |                                                  |
| AA:C1:AB:BC:49: | vxlan-interface:vxlan0.102 vtep:2.2.2.2          | 28730778 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T22:52:01.000Z                         |
| 21              | vni:10020                                        | 7432     |               |        |       |               |               |                                                  |
| AA:C1:AB:ED:20: | vxlan-interface:vxlan0.102 vtep:3.3.3.3          | 28730778 | evpn          | true   | N/A   | none          | 0             | 2026-09-29T22:52:06.000Z                         |
| 3F              | vni:10020                                        | 7437     |               |        |       |               |               |                                                  |
+-----------------+--------------------------------------------------+----------+---------------+--------+-------+---------------+---------------+--------------------------------------------------+
Total Irb Macs                 :    3 Total    3 Active
Total Static Macs              :    0 Total    0 Active
Total Duplicate Macs           :    0 Total    0 Active
Total Learnt Macs              :    2 Total    2 Active
Total Evpn Macs                :    6 Total    6 Active
Total Evpn static Macs         :    9 Total    9 Active
Total Irb anycast Macs         :    2 Total    2 Active
Total Proxy Antispoof Macs     :    0 Total    0 Active
Total Reserved Macs            :    0 Total    0 Active
Total Eth-cfm Macs             :    0 Total    0 Active
Total Irb Vrrps                :    0 Total    0 Active
```

---

### Underlay Routing Table

```text
A:admin@leaf1# show network-instance default ipv4 route
========================================================================================================================================================================================================
IPv4-unicast route table for default network-instance
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Flags: > (best), * (unviable), ! (failed)
     : L (leaked route from another network-instance)
     : B (backup NHG active and displayed)
     : S (statistics supported)
     : D (dynamic LB), R (resilient LB)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Prefix               Route Type   Metric   Pref    Flags    Next-Hop(s)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
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
========================================================================================================================================================================================================
IPv4-unicast route table for ip-vrf network-instance: tenant1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Flags: > (best), * (unviable), ! (failed)
     : L (leaked route from another network-instance)
     : B (backup NHG active and displayed)
     : S (statistics supported)
     : D (dynamic LB), R (resilient LB)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Prefix               Route Type   Metric   Pref    Flags    Next-Hop(s)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
192.168.10.0/24      local        0        0       >        192.168.10.254(irb0.1)
192.168.10.1/32      arp-nd       0        1       >        192.168.10.1(irb0.1)
192.168.10.2/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:22:05:FF:00:41)                                                                                                         
                     iff
192.168.10.3/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:17:06:FF:00:41)                                                                                                         
                     iff
192.168.10.4/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:FD:07:FF:00:41)                                                                                                         
                     iff
192.168.20.0/24      local        0        0       >        192.168.20.254(irb0.2)
192.168.20.1/32      arp-nd       0        1       >        192.168.20.1(irb0.2)
192.168.20.2/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:22:05:FF:00:41)                                                                                                         
                     iff
192.168.20.3/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:17:06:FF:00:41)                                                                                                         
                     iff
192.168.20.4/32      bgp-evpn-    0        169     >        irb0.100(mac:1A:FD:07:FF:00:41)                                                                                                         
                     iff
```

---

### VXLAN Tunnel Table
In IFF, `vxlan0.100` is **`bridged`** (belonging to MAC-VRF `sbd`) rather than `routed`:

```text
A:admin@leaf1# show tunnel-interface vxlan-interface brief
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for vxlan-tunnels 
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+------------------+-----------------+---------+-------------+------------------+
| Tunnel Interface | VxLAN Interface |  Type   | Ingress VNI | Egress source-ip |
+==================+=================+=========+=============+==================+
| vxlan0           | vxlan0.100      | bridged | 10000       | 1.1.1.1/32       |
| vxlan0           | vxlan0.101      | bridged | 10010       | 1.1.1.1/32       |
| vxlan0           | vxlan0.102      | bridged | 10020       | 1.1.1.1/32       |
+------------------+-----------------+---------+-------------+------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary
  1 tunnel-interfaces, 3 vxlan interfaces
  15 vxlan-destinations, 9 unicast, 0 es, 6 multicast, 0 ip
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

---

### EVPN Routes

#### IMET Type-3 Routes
Type-3 (IMET) routes establish broadcast, unknown-unicast, and multicast (BUM) flood lists for the bridge domains:

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 3 summary
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
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
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
12 Inclusive Multicast Ethernet Tag routes 6 used, 12 valid, 0 stale
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

#### MAC and MAC-IP Type 2 Routes
In IFF, Type-2 routes carry a single VNI (either VNI 10010 for `app`, 10020 for `web`, or 10000 for the SBD Router MAC). Notice that the dual-label encapsulation (`10010 + 10000`) found in IFL is absent:

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 2 summary
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Type 2 MAC-IP Advertisement Routes
+-------+------------------+-----------+------------------+------------------+------------------+-------+------------------+------------------+-------------------------------+------------------+
| Statu |      Route-      |  Tag-ID   |   MAC-address    |    IP-address    |     neighbor     | Path- |     Next-Hop     |      Label       |              ESI              |   MAC Mobility   |
|   s   |  distinguisher   |           |                  |                  |                  |  id   |                  |                  |                               |                  |
+=======+==================+===========+==================+==================+==================+=======+==================+==================+===============================+==================+
| u*>   | 2.2.2.2:10000    | 0         | 1A:22:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10000    | 0         | 1A:22:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | 1A:22:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | 1A:22:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | AA:C1:AB:5C:E4:5 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 4                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | AA:C1:AB:5C:E4:5 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 4                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | 1A:22:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | 1A:22:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | AA:C1:AB:BC:49:2 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | AA:C1:AB:BC:49:2 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10000    | 0         | 1A:17:06:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10000    | 0         | 1A:17:06:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10010    | 0         | 1A:17:06:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10010    | 0         | 1A:17:06:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10010    | 0         | AA:C1:AB:18:BC:F | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | E                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10010    | 0         | AA:C1:AB:18:BC:F | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | E                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10020    | 0         | 1A:17:06:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10020    | 0         | 1A:17:06:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10020    | 0         | AA:C1:AB:ED:20:3 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | F                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10020    | 0         | AA:C1:AB:ED:20:3 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | F                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10000    | 0         | 1A:FD:07:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10000    | 0         | 1A:FD:07:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10000            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10010    | 0         | 1A:FD:07:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10010    | 0         | 1A:FD:07:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10010    | 0         | AA:C1:AB:9C:09:D | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | C                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10010    | 0         | AA:C1:AB:9C:09:D | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | C                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10020    | 0         | 1A:FD:07:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10020    | 0         | 1A:FD:07:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10020    | 0         | AA:C1:AB:61:F9:4 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | C                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10020    | 0         | AA:C1:AB:61:F9:4 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | C                |                  |                  |       |                  |                  |                               |                  |
+-------+------------------+-----------+------------------+------------------+------------------+-------+------------------+------------------+-------------------------------+------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
42 MAC-IP Advertisement routes 21 used, 42 valid, 0 stale
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

#### IP Prefix Type 5 Routes
In IFF, both subnet prefixes and `/32` host routes are advertised via **Type 5 IP Prefix** routes across the SBD MAC-VRF (VNI 10000):

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 5 summary
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
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
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
24 IP Prefix routes 12 used, 24 valid, 0 stale
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```
