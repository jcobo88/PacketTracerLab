# Cisco Packet Tracer Networking Lab

## Overview

I built this lab to practice configuring and troubleshooting a network that grew beyond a basic router-and-switch exercise.

I started with two VLANs on a single switch and router. As I worked through the lab, I added more networks, a second router, centralized DHCP and DNS, OSPF, NAT/PAT, an ISP connection, SSH management, ACLs, port security, an internal web server, and redundant switch links with STP.

I also intentionally broke working configurations along the way. I wanted to get used to looking at the symptoms first and figuring out whether a problem was at Layer 2, Layer 3, routing, DNS, DHCP, NAT, or access control instead of immediately changing configuration.

### Main areas covered

- VLANs
- 802.1Q trunking
- Router-on-a-stick
- Inter-VLAN routing
- DHCP
- DHCP relay
- DNS
- Static routing
- Default routes
- OSPF
- NAT and PAT
- Static PAT / port forwarding
- Standard and extended ACLs
- SSH management
- Port security
- HTTP services
- STP
- Layer 2 and Layer 3 troubleshooting

---

## Lab Files

The repository includes the completed Packet Tracer topology and the running configurations from the Cisco devices.

- [Packet Tracer Lab](packet_tracer_networking_lab.pkt)
- [ISP01 Configuration](Configs/ISP01.txt)
- [ROUTER01 Configuration](Configs/ROUTER01.txt)
- [ROUTER02 Configuration](Configs/ROUTER02.txt)
- [SWITCH01 Configuration](Configs/SWITCH01.txt)
- [SWITCH02 Configuration](Configs/SWITCH02.txt)
- [SWITCH03 Configuration](Configs/SWITCH03.txt)

---

## Final Network

![Final network topology](screenshots/01-network-topology.png)

The finished network contains four internal VLANs, two internal routers, a simulated ISP, two remote networks, centralized DHCP and DNS services, internal and external web servers, and redundant Layer 2 links.

```text
                          INTERNET-SERVER
                           198.51.100.10
                                  |
                               ISP01
                           203.0.113.1
                                  |
                           203.0.113.0/30
                                  |
                           203.0.113.2
                             ROUTER01
                         /             \
                802.1Q trunk          10.0.0.1
                    |                   |
                 SWITCH01          10.0.0.0/30
              /     |     |             |
           VLAN10 VLAN20 VLAN50      10.0.0.2
               IT   SALES   HR        ROUTER02
                                     /       \
                             192.168.30.0    192.168.40.0
                                   |
                            DHCP / DNS Server
                              192.168.30.20

                   SWITCH01
                      |
            two redundant trunks
                      |
                   SWITCH03
                      |
                  PC-STP01
```

---

## Addressing

| Network | Purpose | Gateway |
|---|---|---|
| `192.168.10.0/24` | IT | `192.168.10.1` |
| `192.168.20.0/24` | Sales | `192.168.20.1` |
| `192.168.50.0/24` | HR | `192.168.50.1` |
| `192.168.99.0/24` | Management | `192.168.99.1` |
| `10.0.0.0/30` | ROUTER01 ↔ ROUTER02 | Point-to-point |
| `192.168.30.0/24` | Remote LAN / Services | `192.168.30.1` |
| `192.168.40.0/24` | Second Remote LAN | `192.168.40.1` |
| `203.0.113.0/30` | ROUTER01 ↔ ISP01 | Point-to-point |
| `198.51.100.0/24` | Simulated Internet | `198.51.100.1` |

Important infrastructure addresses:

```text
SWITCH01 management:  192.168.99.2
DHCP/DNS server:      192.168.30.20
INTERNAL-WEB:         192.168.10.20
INTERNET-SERVER:      198.51.100.10
```

---

# 1. VLANs and Inter-VLAN Routing

I started by separating IT and Sales into different VLANs and later added HR and Management.

```text
VLAN 10 - IT
VLAN 20 - SALES
VLAN 50 - HR
VLAN 99 - MANAGEMENT
```

![VLAN configuration](screenshots/02-vlan-configuration.png)

Before routing was configured, hosts in different VLANs could not communicate.

![VLAN connectivity test](screenshots/03-vlan-connectivity-test.png)

I configured `Gi0/1` on SWITCH01 as a trunk to ROUTER01 and created router subinterfaces.

```text
G0/0.10 -> 192.168.10.1
G0/0.20 -> 192.168.20.1
G0/0.50 -> 192.168.50.1
G0/0.99 -> 192.168.99.1
```

Each subinterface was tagged for its VLAN using 802.1Q.

After that, the VLANs could route through ROUTER01.

![Inter-VLAN routing verified](screenshots/04-inter-vlan-routing-verified.png)

---

# 2. DHCP

ROUTER01 initially provided DHCP for the IT and Sales VLANs.

The pools supplied the correct:

```text
IP address
Subnet mask
Default gateway
DNS server
```

I checked the client configuration after requesting an address.

![DHCP client configuration](screenshots/05-dhcp-client-configuration.png)

I also checked the router's DHCP bindings instead of relying only on the workstation.

![DHCP bindings](screenshots/06-dhcp-bindings.png)

---

# 3. ACL Between Sales and IT

I wanted Sales and IT to remain routed but have some traffic restrictions.

I created an extended ACL that allowed ICMP replies from Sales while preventing Sales devices from starting pings toward IT.

```text
permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
deny   icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo
permit ip any any
```

Sales could not initiate the ping:

![Sales to IT blocked](screenshots/07-acl-sales-to-it-blocked.png)

IT could still initiate communication toward Sales:

![IT to Sales allowed](screenshots/08-acl-it-to-sales-allowed.png)

I checked the ACL counters afterward to make sure the traffic was actually matching the entries I expected.

![ACL match counters](screenshots/09-acl-match-counters.png)

That was more useful than treating a successful or failed ping by itself as proof that the ACL was correct.

---

# 4. Early Troubleshooting

Before expanding the network, I intentionally broke several basic configurations.

## Wrong VLAN

I placed a Sales port in the wrong VLAN.

The Sales workstation lost the connectivity I expected.

![Sales network failure](screenshots/10-troubleshooting-sales-network-failure.png)

I checked the switch VLAN assignments, corrected the access VLAN, and tested again.

![Sales network restored](screenshots/11-troubleshooting-sales-network-restored.png)

## Broken trunk

I then broke the trunk between SWITCH01 and ROUTER01.

Local switching could still work, but traffic that depended on the router stopped working.

![Trunk failure](screenshots/12-troubleshooting-trunk-failure.png)

I used the switch trunk information to identify the problem and restored trunk operation.

![Trunk restored](screenshots/13-troubleshooting-trunk-restored.png)

## Wrong default gateway

I gave a workstation an incorrect gateway.

The workstation could communicate locally but could not reach remote networks.

![Default gateway failure](screenshots/14-troubleshooting-default-gateway-failure.png)

After correcting the gateway from the incorrect `.254` address to the router's `.1` address, remote connectivity returned.

![Default gateway restored](screenshots/15-troubleshooting-default-gateway-restored.png)

These three tests helped me separate:

```text
Same VLAN failure     -> check Layer 2

Local works,
remote fails          -> check gateway / routing

Multiple VLANs fail
through one uplink    -> check trunking
```

---

# 5. MAC Tables, Routing, and Switch Management

I checked the SWITCH01 MAC address table to see which MAC addresses had been learned on each interface.

![Switch MAC address table](screenshots/16-switch-mac-address-table.png)

I also checked ROUTER01's routing table.

![Router routing table](screenshots/17-router-routing-table.png)

Seeing both helped connect what the switch does at Layer 2 with what the router does at Layer 3.

## Management VLAN

I created VLAN 99 for switch management.

```text
ROUTER01:  192.168.99.1
SWITCH01:  192.168.99.2
```

![Management VLAN verified](screenshots/18-switch-management-vlan-verified.png)

I then configured SSH version 2 on SWITCH01 and tested remote CLI access from an IT workstation.

![SSH management verified](screenshots/19-ssh-switch-management-verified.png)

I restricted the VTY lines so management access was allowed from the IT subnet but not from user networks such as Sales.

![SSH management access control](screenshots/20-ssh-management-access-control.png)

---

# 6. Port Security

I configured sticky port security on user-facing switchports.

The ports were limited to one learned MAC address and used shutdown behavior for violations.

![Port security configured](screenshots/21-port-security-configured.png)

To test it, I disconnected the authorized workstation and connected another device.

The interface went into a security shutdown state.

![Port security violation](screenshots/22-troubleshooting-port-security-violation.png)

I reconnected the correct workstation and recovered the port with:

```text
shutdown
no shutdown
```

![Port security restored](screenshots/23-troubleshooting-port-security-restored.png)

I then checked the protected access ports together.

![Port security summary](screenshots/24-access-port-security-summary.png)

---

# 7. Adding ROUTER02 and Static Routing

I added ROUTER02 and a remote network:

```text
192.168.30.0/24
```

The router-to-router link used:

```text
ROUTER01: 10.0.0.1/30
ROUTER02: 10.0.0.2/30
```

At first ROUTER01 had no route to the remote LAN.

![Remote network unreachable](screenshots/25-static-routing-remote-network-unreachable.png)

I added:

```text
ip route 192.168.30.0 255.255.255.0 10.0.0.2
```

The error changed, but communication still failed.

![Missing return route](screenshots/26-static-routing-missing-return-route.png)

That change in symptoms was useful. The packet could now move in the forward direction, but ROUTER02 did not yet know how to get the response back to the IT network.

After adding the required return routing, communication worked in both directions.

![Static routing verified](screenshots/27-static-routing-bidirectional-verified.png)

I used traceroute to verify the path.

![Traceroute multi-router path](screenshots/28-traceroute-multi-router-path.png)

The path was:

```text
PC-IT01
   ↓
ROUTER01
   ↓
ROUTER02
   ↓
PC-REMOTE01
```

I also tested a default route on ROUTER02 instead of maintaining an individual return route for every network behind ROUTER01.

![ROUTER02 default route](screenshots/29-router02-default-route.png)

---

# 8. OSPF

After working through static routing manually, I replaced the internal static routes with OSPF.

```text
ROUTER01 Router ID: 1.1.1.1
ROUTER02 Router ID: 2.2.2.2
Area: 0
```

The routers formed a FULL adjacency.

![OSPF neighbor adjacency](screenshots/30-ospf-neighbor-adjacency.png)

I checked the routing tables and confirmed that routes were being learned dynamically.

![OSPF dynamic routes](screenshots/31-ospf-dynamic-routes.png)

ROUTER01 learned the remote network behind ROUTER02.

![OSPF remote route learned](screenshots/32-ospf-remote-route-learned.png)

I later added:

```text
192.168.40.0/24
```

behind ROUTER02.

Instead of adding another static route to ROUTER01, I advertised the new network through OSPF and confirmed that ROUTER01 learned it.

![New OSPF network learned](screenshots/33-ospf-new-network-learned.png)

## Breaking OSPF

I deliberately created an area mismatch on the link between the two routers.

IP connectivity over the transit network still existed, but the OSPF neighbor disappeared and dynamic routes were removed.

![OSPF area mismatch](screenshots/34-troubleshooting-ospf-area-mismatch.png)

That helped narrow the fault down to OSPF rather than cabling or interface addressing.

I restored both sides to Area 0. The routers rebuilt the adjacency automatically and the learned routes returned.

![OSPF restored](screenshots/35-troubleshooting-ospf-area-restored.png)

---

# 9. Central DHCP and DHCP Relay

I added a centralized DHCP and DNS server at:

```text
192.168.30.20
```

The HR network was on:

```text
192.168.50.0/24
```

Because DHCP discovery starts as a local broadcast, the HR client could not directly reach a DHCP server on another routed network.

I first tested the failure.

![DHCP relay failure](screenshots/36-dhcp-relay-before-helper-failure.png)

On ROUTER01's HR subinterface, I configured:

```text
ip helper-address 192.168.30.20
```

After correcting the relay and DHCP pool configuration, PC-HR01 received the correct lease.

![DHCP relay verified](screenshots/37-dhcp-relay-lease-verified.png)

The client received:

```text
Address:     192.168.50.10
Gateway:     192.168.50.1
DHCP Server: 192.168.30.20
DNS Server:  192.168.30.20
```

---

# 10. Simulated Internet and PAT

I added ISP01 and INTERNET-SERVER to simulate an outside network.

```text
ROUTER01 G0/2:   203.0.113.2
ISP01 G0/0:      203.0.113.1

INTERNET-SERVER: 198.51.100.10
```

ROUTER01 used:

```text
0.0.0.0/0 -> 203.0.113.1
```

as its default route.

Before NAT was configured, the private internal hosts could not successfully communicate with the simulated Internet host.

![Before NAT](screenshots/38-nat-before-translation-failure.png)

I configured the internal interfaces as NAT inside and the ISP-facing interface as NAT outside, then enabled PAT overload.

Internal clients could then reach INTERNET-SERVER.

![PAT Internet access verified](screenshots/39-nat-pat-internet-access-verified.png)

I checked the translation table while traffic was active.

![NAT translation table](screenshots/40-nat-translation-table.png)

I then generated traffic from more than one inside host.

![Multiple PAT clients](screenshots/41-pat-multiple-inside-hosts.png)

The hosts shared:

```text
203.0.113.2
```

while PAT kept their sessions separate.

---

# 11. DNS and HTTP

The centralized server also provided DNS to internal clients.

I created:

```text
www.cobo.test
```

for the simulated external website.

Before the record existed, the server was reachable by IP but not by hostname.

![DNS before record](screenshots/42-dns-before-record-failure.png)

I also intentionally created the record with the wrong IP address.

![Incorrect DNS A record](screenshots/43-troubleshooting-dns-wrong-a-record.png)

That produced a different type of failure: DNS itself was answering, but it was giving the client bad information.

I corrected the A record to:

```text
198.51.100.10
```

![DNS record restored](screenshots/44-troubleshooting-dns-record-restored.png)

The internal workstation could then browse to:

```text
http://www.cobo.test
```

![Website accessed through DNS](screenshots/45-http-website-via-dns-verified.png)

I checked the NAT table and could see the TCP translation created by the HTTP session.

![HTTP PAT translation](screenshots/46-pat-http-tcp-translation.png)

---

# 12. HR DHCP and DNS Troubleshooting

The HR network gave me another chance to separate IP connectivity from service problems.

At one point PC-HR01 could communicate by IP while DNS resolution failed.

![HR DNS failure](screenshots/47-troubleshooting-hr-dns-failure.png)

I also encountered a failed DHCP renewal while working with the centralized DHCP server.

![HR DHCP renewal failure](screenshots/48-troubleshooting-hr-dhcp-renewal-failure.png)

The DHCP problem was traced back to the centralized pool configuration.

After correcting it, HR received the expected network settings and DNS resolution worked again.

![HR DHCP and DNS restored](screenshots/49-troubleshooting-hr-dhcp-dns-restored.png)

I later generated HTTP sessions from multiple internal clients and checked how PAT handled them.

![Multiple HTTP clients through PAT](screenshots/50-pat-multiple-http-clients.png)

---

# 13. Static PAT and Port Forwarding

PAT handled connections started from the inside, but I also wanted an outside device to reach one specific internal service.

I added:

```text
INTERNAL-WEB
192.168.10.20
```

Before publishing the service, INTERNET-SERVER could not connect to it through ROUTER01's public-facing address.

![Port forwarding failure](screenshots/51-static-pat-before-port-forward-failure.png)

I configured a static TCP mapping:

```text
203.0.113.2:80
        ↕
192.168.10.20:80
```

I checked the translation table first.

![Static PAT translation table](screenshots/52-static-pat-translation-table.png)

Then I tested from outside:

```text
http://203.0.113.2
```

The request reached the internal web server.

![Static PAT verified](screenshots/53-static-pat-port-forward-verified.png)

---

# 14. Internet-Edge ACL

Publishing port 80 did not mean I wanted all outside traffic accepted.

Before applying the edge ACL, the external host could ping ROUTER01's public-facing interface.

![Outside ICMP allowed before ACL](screenshots/54-outside-icmp-before-acl-allowed.png)

I applied an inbound extended ACL to the ISP-facing interface.

The rules were intended to allow the traffic needed for the published HTTP service and established return traffic while rejecting other unsolicited traffic.

Afterward, outside-initiated ICMP was blocked.

![Outside ICMP blocked](screenshots/55-outside-acl-icmp-blocked.png)

The published website still worked.

![Outside HTTP still allowed](screenshots/56-outside-acl-http-allowed.png)

I checked the ACL counters to confirm that traffic was hitting the expected entries.

![Outside ACL counters](screenshots/57-outside-acl-match-counters.png)

This was a good example of why I did not want to use a broad "allow outside traffic" rule just to make port forwarding work.

---

# 15. STP and Redundant Switch Links

For the last major addition, I connected SWITCH03 to SWITCH01 using two trunk links.

```text
SWITCH01 Fa0/23 <-> SWITCH03 Fa0/23
SWITCH01 Fa0/24 <-> SWITCH03 Fa0/24
```

Without Spanning Tree Protocol, two active Layer 2 paths between the same switches could form a switching loop.

I configured SWITCH01 as the STP root for the lab VLANs.

On SWITCH03, one link forwarded and the second stayed blocked as the backup path.

![STP redundant link blocked](screenshots/58-stp-redundant-link-blocked.png)

I disconnected the active trunk.

STP moved the backup interface into the forwarding role.

![STP backup link takeover](screenshots/59-stp-backup-link-takeover.png)

PC-STP01 remained able to reach its gateway through the alternate path.

![STP failover connectivity](screenshots/60-stp-failover-connectivity-verified.png)

After reconnecting the original trunk, STP reconverged and returned the network to one forwarding link and one backup link.

![STP redundancy restored](screenshots/61-stp-redundancy-restored.png)

This was more useful than only looking at the STP table because I actually removed the active path and confirmed that client traffic survived.

---

# 16. Final Validation

After all of the later changes, I went back through the network to make sure earlier parts of the lab still worked.

I verified HR addressing and centralized services.

![HR final validation](screenshots/62-final-validation-hr-services.png)

On ROUTER01, I checked OSPF and routing again.

![OSPF and routing final validation](screenshots/63-final-validation-ospf-routing.png)

I also opened an SSH session from the IT workstation to SWITCH01's management address:

```text
192.168.99.2
```

![SSH final validation](screenshots/64-final-validation-ssh-management.png)

That gave me a final check of switching, routing, network services, and management connectivity after the topology had been expanded several times.

---

# Problems I Worked Through

Some of the most useful parts of this lab were the configurations that did not work the first time.

| Problem | What helped narrow it down |
|---|---|
| Sales workstation placed in wrong VLAN | Layer 2 connectivity and VLAN membership did not match the design |
| Trunk disabled | Multiple VLANs lost access through the same router link |
| Wrong default gateway | Local communication worked but remote communication failed |
| Port security violation | Interface entered a security shutdown state after the MAC changed |
| Missing forward route | ROUTER01 returned destination unreachable |
| Missing return route | Forward routing improved, but replies could not get back |
| OSPF area mismatch | Transit IP connectivity worked while the OSPF neighbor disappeared |
| Missing DHCP relay | HR broadcasts could not reach the remote DHCP server |
| Incorrect DHCP pool | Relay was present, but the server could not provide the expected lease |
| Missing DNS record | Server worked by IP but not hostname |
| Wrong DNS record | DNS responded, but returned the wrong destination |
| Missing NAT | Internal private hosts could not reach the simulated Internet |
| Missing static PAT | Outside clients could not reach the internal web service |
| Edge ACL | Ping was blocked while the explicitly published HTTP service remained available |
| STP link failure | The backup trunk moved into forwarding state automatically |

The troubleshooting approach I ended up using most often was:

```text
Start with what still works
        ↓
Decide whether the problem is
Layer 2, Layer 3, or a service
        ↓
Check the relevant device state
        ↓
Change one thing
        ↓
Test again
```

---

# Commands I Used Frequently

## Switching

```text
show vlan brief
show interfaces trunk
show mac address-table
show port-security
show spanning-tree
```

## Routing

```text
show ip interface brief
show ip route
show ip ospf neighbor
show ip protocols
traceroute
```

## ACLs

```text
show access-lists
show ip interface
```

## NAT

```text
show ip nat translations
show ip nat statistics
```

## DHCP

```text
show ip dhcp binding
show ip dhcp pool
```

These commands were usually more useful to me than immediately reopening the configuration because they showed what the device was actually doing at that moment.

---

# What I Took Away From the Lab

The biggest improvement for me was getting more comfortable following a packet through the network instead of treating every failed ping as the same problem.

For example:

```text
Client
  ↓
Access port / VLAN
  ↓
Switching
  ↓
Default gateway
  ↓
Routing table
  ↓
ACL
  ↓
NAT
  ↓
Next router
  ↓
Destination
```

If the client could communicate within its own VLAN but not outside it, I knew not to start by troubleshooting basic switching.

If an IP address worked but a hostname did not, I checked DNS.

If a forward route existed but the ping still failed, I checked whether the other side had a return route.

If the two routers could ping each other but OSPF was not forming an adjacency, I focused on OSPF rather than the physical link.

The project also made the difference between configuration and verification clearer to me. A command being present in the running configuration does not necessarily mean traffic is behaving the way I intended. I used routing tables, neighbor states, ACL counters, NAT translations, DHCP leases, traceroute, and actual client traffic to verify the results.

---

# Skills Used

- Cisco IOS
- VLAN configuration
- 802.1Q trunking
- Router-on-a-stick
- Inter-VLAN routing
- IPv4 addressing and subnetting
- DHCP
- DHCP relay
- DNS
- Static routing
- Default routing
- OSPF
- Route troubleshooting
- NAT
- PAT
- Static PAT
- Port forwarding
- Standard ACLs
- Extended ACLs
- SSH management
- VTY restrictions
- Port security
- MAC address table analysis
- ARP and next-hop troubleshooting
- HTTP testing
- STP
- Layer 2 redundancy
- Layer 2 troubleshooting
- Layer 3 troubleshooting

---

# Repository Structure

```text
PacketTracerLab/
│
├── README.md
├── packet_tracer_networking_lab.pkt
│
├── Configs/
│   ├── ISP01.txt
│   ├── ROUTER01.txt
│   ├── ROUTER02.txt
│   ├── SWITCH01.txt
│   ├── SWITCH02.txt
│   ├── SWITCH03.txt
│   └── readme.md
│
└── screenshots/
    ├── 01-network-topology.png
    ├── 02-vlan-configuration.png
    ├── 03-vlan-connectivity-test.png
    ├── ...
    ├── 62-final-validation-hr-services.png
    ├── 63-final-validation-ospf-routing.png
    └── 64-final-validation-ssh-management.png
```

---

# Status

**Completed**

The finished lab contains the Packet Tracer topology, exported configurations for all six Cisco devices, and 64 screenshots covering the build, troubleshooting, failover tests, and final validation.
