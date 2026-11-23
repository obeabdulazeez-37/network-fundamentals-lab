# Lesson 6: Connecting Two Routers with Static Routing

## Why this lesson is late

Every IT pro has a story that starts with *"So I was just testing something on the weekend…"* This is mine.

I spent a weekend practising **file server permissions** in my home lab, and I gave some of my files permission they didn't ask for. Specifically, permission to stop existing. 🫠 My working files for this lesson went with them.

This lesson isn't about home labs or file servers. After this weekend, though, I'm clearly qualified to teach that one too, so a home lab / file server learning path is coming. Consider it a lessons-learned session I didn't schedule.

I did have a backup. It was the kind that looks great in the status report and turns out to be "partially complete" when you actually need it. I rebuilt from an older copy of Lesson 6, and the lab did what every rebuilt lab does on its first day: it didn't work.

So I've turned the delay into a feature. *(That's the most corporate sentence I've ever written, and I'm keeping it.)* This lesson is written as a **troubleshooting walkthrough**. It gives you the full configs for the setup and explains anything we haven't covered in earlier lessons.

## What is static routing?

A router automatically knows about the networks plugged directly into it. For anything else, it needs to be told where to go. A **static route** is a route you add by hand. It's basically an email that says *"for anything about 192.168.20.0, please reach out to 10.0.0.2."*

```
ip route <destination network> <subnet mask> <next-hop IP>
```

A rule worth remembering: **route to the network you can't see, through the neighbour you can see.**

The routing table uses a few codes you'll see in this lesson:

| Code | Meaning | In this lab (R1) |
|---|---|---|
| C | A directly connected network | 192.168.10.0/24, 10.0.0.0/30 |
| L | The router's own local address on that network (/32) | 192.168.10.1, 10.0.0.1 |
| S | A static route you added | 192.168.20.0/24 via 10.0.0.2 |
| [1/0] | Administrative distance / metric. Static routes have a distance of 1 | [1/0] on the static route |

## Where the topology changes from lesson 5

Lesson 5 had a single router doing NAT for two switched subnets. Lesson 6 adds a **second router**, and the two routers have to learn about each other's networks:

- Two Cisco 2911 routers, R1 and R2, are connected directly to each other on Gi0/1.
- Each router has one PC on its LAN interface (Gi0/0): PC-A on R1 and PC-B on R2.
- The link between the routers uses a **/30** subnet (255.255.255.252). It has exactly 2 usable addresses, one for each router. A point-to-point link doesn't need 254 addresses any more than a two-person meeting needs a 200-seat auditorium.

Think of R1 and R2 as two department managers. Their teams can only talk to each other if the managers know how to route the message. *(It's a bit like real life, but faster.)*

![Connecting and configuring devices](01-connect-and-config-devices.png)

*The two routers connected to each other, with one PC on each LAN.*

## The lab

| Device | Interface | Address | Notes |
|---|---|---|---|
| PC-A | NIC | 192.168.10.10 | Subnet 192.168.10.0/24, gateway R1 Gi0/0 (192.168.10.1) |
| PC-B | NIC | 192.168.20.10 | Subnet 192.168.20.0/24, gateway R2 Gi0/0 (192.168.20.1) |
| R1 | Gi0/0 | 192.168.10.1 /24 | LAN toward PC-A |
| R1 | Gi0/1 | 10.0.0.1 /30 | Link to R2 |
| R2 | Gi0/0 | 192.168.20.1 /24 | LAN toward PC-B |
| R2 | Gi0/1 | 10.0.0.2 /30 | Link to R1 |

**Goal:** PC-A (192.168.10.10) can reach PC-B (192.168.20.10) across the two routers.

## 1. Configure the devices

**Configure R1.**

```
enable
configure terminal
interface gi0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface gi0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
ip route 192.168.20.0 255.255.255.0 10.0.0.2
```

**Configure R2.**

```
enable
configure terminal
interface gi0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown
interface gi0/1
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
ip route 192.168.10.0 255.255.255.0 10.0.0.1
```

**Configure the PCs.** Go to **Desktop → IP Configuration** on each PC:
- PC-A: IP 192.168.10.10, mask 255.255.255.0, gateway 192.168.10.1
- PC-B: IP 192.168.20.10, mask 255.255.255.0, gateway 192.168.20.1

What each part does:
- `ip address 10.0.0.1 255.255.255.252` gives each router one of the two usable addresses on the /30 link.
- `ip route 192.168.20.0 255.255.255.0 10.0.0.2` on R1 says that PC-B's network is reached through R2.
- `ip route 192.168.10.0 255.255.255.0 10.0.0.1` on R2 says that PC-A's network is reached through R1. Without it, replies can't find their way back.
- The PC gateway is the router interface on the PC's own LAN. It's where the PC sends anything that isn't local.

Use a **subnet mask** (255.255.255.0) in `ip route`, not a wildcard mask (0.0.0.255). Wildcard masks belong to access lists and OSPF. Putting one in `ip route` is like replying-all with the wrong attachment: technically you sent something, but nobody's happy.

## 2. Run tests and confirmations

**Check the interfaces on both routers.** `show ip interface brief` is the networking version of "have you tried turning it off and on again?"

![R1 show ip interface brief](02-r1-show-ip-interface-brief.png)

*R1: Gi0/0 and Gi0/1 are up/up with the correct addresses.*

![R2 show ip interface brief](03-r2-show-ip-interface-brief.png)

*R2: Gi0/0 and Gi0/1 are up/up with the correct addresses.*

How to read the Status / Protocol columns:
- **up / up**: cabled and working.
- **administratively down / down**: turned off on purpose, because `no shutdown` was never entered. Gi0/2 and Vlan1 are unused here. They're the coworkers on PTO, so leave them alone.
- **up / down**: the cable is connected but something's wrong at Layer 2. That's a story for another lesson.

**Ping from PC-A to PC-B.** This is where my rebuilt lab let me down.

![First ping and traceroute test fails](04-pc-a-ping-test-fails.png)

*PC-A pings 192.168.20.10 and 192.168.20.1. Both fail with 100% loss, and `tracert` fails at hop 1.*

Every interface said "up", yet nothing got through. Picture a project status dashboard that's all green while the client is on the phone yelling. 📞🔥

`tracert` shows each router (hop) your traffic passes through, and **where it stops tells you where to look**. Failing at hop 1 means the traffic never even reached R1, so the problem is close to PC-A: its own settings, or R1.

## 3. Troubleshoot and fix

**Problem 1: PC-A had no default gateway.** `ipconfig` on PC-A showed the default gateway as **0.0.0.0**. PC-A had no way out of its own network. It was like a new hire with no manager assigned: lots of potential, and no idea who to send anything to.

**Fix:** set PC-A's gateway to **192.168.10.1**, R1's LAN interface.

![PC-A with no default gateway](05-pc-a-no-gateway.png)

*PC-A's `ipconfig` shows the default gateway as 0.0.0.0.*

![PC-A default gateway configured](06-pc-a-gateway-config.png)

*PC-A's default gateway set to 192.168.10.1.*

> ⚠️ *Editing note: check that this screenshot shows the gateway as **192.168.10.1**. The earlier version showed 192.168.20.1, which is PC-B's gateway. If it still does, mention it ("I fixed it… then fixed my fix 😅") or retake it.*

**Problem 2: R1 had no route to PC-B's network.** On R1, `show ip route` listed only **C** and **L** routes and had **no entry for 192.168.20.0/24**. My "partially complete" backup had struck again. R1 knew its own two networks but had been left off the distribution list for PC-B's.

**Fix:**

```
enable
configure terminal
ip route 192.168.20.0 255.255.255.0 10.0.0.2
exit
```

![R1 root cause analysis](07-r1-missing-static-route.png)

*R1's routing table with no route to 192.168.20.0/24, then the static route being added.*

**Confirm the fix.** `show ip route` now has the static route:

```
S    192.168.20.0/24 [1/0] via 10.0.0.2
```

![R1 route fix confirmed](08-r1-static-route-fixed.png)

*The S entry shows R1 now sends traffic for 192.168.20.0/24 to R2 at 10.0.0.2.*

## 4. Run tests and confirmations

**PC-A traces and pings PC-B.** The traceroute shows the full path: 192.168.10.1 (R1), then 10.0.0.2 (R2), then 192.168.20.10 (PC-B). The ping gets all four replies with 0% loss. Nothing beats that feeling, except maybe a meeting that could've been an email actually becoming an email.

![Successful ping and traceroute](09-pc-a-ping-test-success.png)

*PC-A reaches PC-B: three hops, 4 of 4 replies, TTL 126.*

The replies come back with **TTL 126**. The PC starts at 128 and each router subtracts 1, so 126 confirms that the reply passed through 2 routers (R2, then R1). It's a free sanity check built into every ping.

## What I learnt

- Static routing has to be set up on **both** routers. Communication only works when it goes both ways, in networks as well as in teams.
- Route to the network you can't see, through the neighbour you can see.
- `ip route` takes a subnet mask, not a wildcard mask.
- Troubleshoot from the bottom up: interfaces (`show ip interface brief`), then routes (`show ip route`), then end-to-end connectivity (`ping`, `tracert`).
- `tracert` tells you where to start looking, and the TTL in a ping reply tells you how many routers the traffic crossed.
- Save your configs with `copy running-config startup-config`, and test-restore your backups at least once. A backup you've never restored is just a hope with a file extension. I learned this the hard way so you don't have to. 🙃

## Final topology

![Final topology](10-final-topology.png)
