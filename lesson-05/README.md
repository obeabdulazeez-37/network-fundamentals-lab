# Lesson 5: NAT Overload (PAT)

## What is NAT/PAT?

**NAT (Network Address Translation)** lets devices with private addresses reach the internet. Private ranges such as 192.168.0.0/16 are not routable on the internet, so the router swaps the private source address for a public one before the packet leaves, and swaps it back when the reply returns.

**PAT (Port Address Translation)**, also called **NAT overload**, lets many private hosts share a single public address. The router keeps the conversations apart using port numbers (for ping, the number after the colon in the translation table). In this lab the shared public address is R1's outside interface, 203.0.113.1.

Four terms show up in the translation table:

| Term | Meaning | In this lab |
|---|---|---|
| Inside local | The host's private address | 192.168.71.129 |
| Inside global | The public address the host is translated to | 203.0.113.1 |
| Outside local | How the inside network sees the destination | 203.0.113.10 |
| Outside global | The destination's real address | 203.0.113.10 |

## Where the topology changes from lesson 4

Lesson 4 used the small office topology: R1 connected to SW1, SW2 trunked to SW1, and the departments sharing the 192.168.100.0/24 block, with DHCP handing out addresses.

Lesson 5 changes direction. The topology is rebuilt smaller and simpler so the focus stays on NAT:

- Two switches, SW-V2 and SW-V8, each connect to their own router interface (Gi0/0 and Gi0/1) instead of trunking SW2 into SW1 and sharing one router link.
- Each switch carries one subnet with two PCs: 192.168.50.0/24 on SW-V2 and 192.168.71.0/24 on SW-V8.
- Because each subnet sits on its own router interface, `ip nat inside` can be set on one interface and left off the other. That makes it simple to show NAT applying to one subnet only.
- A third interface, Gi0/2, is added as the outside link to the Internet-Server.

![Starting topology with no internet](images/01-starting-topology-no-internet.png)

*Starting topology, labelled with each device's address and port. R1 only has its two inside links. The internet side is not connected yet.*

## The lab

| Device | Address | Notes |
|---|---|---|
| V2-PC1 | 192.168.50.10 | Subnet 192.168.50.0/24, gateway R1 Gi0/0 |
| V2-PC2 | 192.168.50.20 | Subnet 192.168.50.0/24, gateway R1 Gi0/0 |
| V8-PC1 | 192.168.71.128 | Subnet 192.168.71.0/24, gateway R1 Gi0/1 |
| V8-PC2 | 192.168.71.129 | Subnet 192.168.71.0/24, gateway R1 Gi0/1 |
| R1 Gi0/2 | 203.0.113.1 | Outside interface toward the internet |
| Internet-Server | 203.0.113.10 | Static, gateway 203.0.113.1 |

**Goal:** only the 192.168.71.0/24 subnet can reach the internet, through NAT overload.

## 1. Connect the internet and configure

**Connect the server.** Add the Internet-Server and cable its Fa0 to R1 Gi0/2. The link shows red at first because the router interface is still shut down.

![Internet-Server cabled to R1 Gi0/2 with the link still down](images/02-server-cabled-to-r1-gi0-2-link-down.png)

**Give the server a static address.** IP 203.0.113.10, mask 255.255.255.0, gateway 203.0.113.1.

![Internet-Server static IP configuration](images/03-internet-server-static-ip-config.png)

**Configure R1.** Set up the outside interface, mark the inside interface, and add a default route.

```
enable
configure terminal
interface Gi0/2
 ip address 203.0.113.1 255.255.255.0
 ip nat outside
 no shutdown
 exit
interface Gi0/1
 ip nat inside
 exit
ip route 0.0.0.0 0.0.0.0 203.0.113.10
```

What each part does:
- `ip nat outside` marks Gi0/2 as the interface facing the internet.
- `ip nat inside` marks Gi0/1 as the interface facing the 192.168.71.0/24 subnet. Gi0/1 already has its IP address.
- Gi0/0 (the 192.168.50.0/24 side) is left unmarked on purpose.
- `ip route 0.0.0.0 0.0.0.0 203.0.113.10` is the default route. Anything R1 has no specific route for goes toward the Internet-Server.

## 2. Run tests and confirmations

Confirm the link between R1 and the server is up. The link lights go from red to green.

![All links up and green](images/04-all-links-up-green.png)

## 3. Write the access list and the source list

```
access-list 1 permit 192.168.71.0 0.0.0.255
ip nat inside source list 1 interface Gi0/2 overload
end
wr
```

What each part does:
- `access-list 1 permit 192.168.71.0 0.0.0.255` is a standard access list that names which addresses may be translated. The wildcard mask 0.0.0.255 matches the whole /24.
- `ip nat inside source list 1 interface Gi0/2 overload` ties that list to NAT. Traffic from addresses in list 1 is translated to the address on Gi0/2, and `overload` turns on PAT so many hosts share it.
- `end` leaves configuration mode and `wr` saves the configuration.

This access list does not block traffic. It only chooses which hosts get translated.

![R1 NAT overload CLI configuration](images/05-r1-nat-overload-cli-config.png)

## 4. Run tests and confirmations

**V8-PC1 (192.168.71.128) pings the Internet-Server.** All four replies come back, 0% loss, TTL 127 (the PC starts at 128 and loses one at R1).

![V8-PC1 ping to the internet succeeds](images/06-v8-pc1-ping-internet-success.png)

**V2-PC1 (192.168.50.10) pings the Internet-Server.** No replies, 100% loss. The 192.168.50.0/24 subnet is not in access list 1, so it gets no translation.

![V2-PC1 ping to the internet fails](images/07-v2-pc1-ping-internet-fail.png)

**Checks on R1:**
- `show ip nat translations` shows ICMP entries where inside local 192.168.71.129 becomes inside global 203.0.113.1, talking to outside address 203.0.113.10. Each entry has its own port number.
- `show ip nat statistics` shows Gi0/2 as the outside interface and Gi0/1 as the inside interface, plus the hit, miss and expired counts.
- `show ip route` shows the default route `S* 0.0.0.0/0 via 203.0.113.10` and the three connected networks.

![R1 NAT statistics, translations and routing table](images/08-r1-nat-statistics-translations-and-route.png)

Translations expire quickly. Run `show ip nat translations` right after the ping, otherwise the table can be empty.

## What I learnt

- NAT needs three things working together: inside and outside interfaces marked, an access list saying who gets translated, and the source list rule that connects them.
- Overload lets many private hosts share one public address, and the port numbers keep each conversation separate.
- The access list decides who gets NAT. A subnet left out of it is not translated.
- The router needs a default route so it knows where internet traffic goes.
- Translations time out, so check the table right after sending traffic.

## Final topology

![Final topology with NAT overload to the Internet-Server](images/09-final-topology-nat-overload.png)
