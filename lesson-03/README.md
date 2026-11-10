# Lesson 3: A router between two networks

## Goal

Join the two isolated networks from Lesson 2 with a router, and show what a device needs before it can reach the other network.

## Why it matters

In my home lab, the closed lab network (V2) and the update network (V8) are separate. Anything that links two networks is a router, or a device acting as one, with an address in each network. Every device then needs a default gateway that points at it.

When a user can reach local devices but nothing else, a missing or wrong default gateway is one of the first things to check.

## Final topology

![Final topology with addresses and gateways](images/final-topology.png)

## Address table

| Device | Network | IP address | Subnet mask | Default gateway |
|---|---|---|---|---|
| V2-PC1 | 192.168.50.0/24 | 192.168.50.10 | 255.255.255.0 | 192.168.50.1 |
| V2-PC2 | 192.168.50.0/24 | 192.168.50.20 | 255.255.255.0 | 192.168.50.1 |
| V8-PC1 | 192.168.71.0/24 | 192.168.71.10 | 255.255.255.0 | 192.168.71.1 |
| V8-PC2 | 192.168.71.0/24 | 192.168.71.20 | 255.255.255.0 | 192.168.71.1 |
| R1 Gi0/0 (to SW-V2) | 192.168.50.0/24 | 192.168.50.1 | 255.255.255.0 | none |
| R1 Gi0/1 (to SW-V8) | 192.168.71.0/24 | 192.168.71.1 | 255.255.255.0 | none |

## Steps

### 1. Add the router and cable it

1. Place a 2911 router.
2. Connect Gi0/0 to SW-V2 and Gi0/1 to SW-V8.

A new router starts with its ports shut down, so both links show red. The router still has its default name, Router0, which I renamed to R1.

![Router added, both links red](images/router-no-interface-config.png)

### 2. Give each interface an address

In the router CLI:

```
enable
configure terminal
interface Gi0/0
ip address 192.168.50.1 255.255.255.0
interface Gi0/1
ip address 192.168.71.1 255.255.255.0
no shutdown
```

![R1 CLI](images/router-config.png)

Why: each router interface gets an address inside the network it connects to. Those addresses become the default gateways for the PCs.

### 3. Turn the interfaces on

I entered `no shutdown` on Gi0/1 only. The link to SW-V8 turned green, and the link to SW-V2 stayed red.

![Only the SW-V8 link is up](images/router-one-interface-config.png)

Router interfaces are off by default, and each one needs its own `no shutdown`.

I had left `no shutdown` off Gi0/0, so I went back in and turned it on. The capture below shows both commands. `shutdown` turns the interface off: the link goes red and the router reports "administratively down". `no shutdown` turns it back on: the link goes green and the router reports "changed state to up".

```
configure terminal
interface Gi0/0
shutdown
no shutdown
end
write memory
```

![Turning Gi0/0 off and back on](images/router-interface-no-shutdown.png)

| Command | Effect | Message from the router |
|---|---|---|
| `shutdown` | Turns the interface off | changed state to administratively down |
| `no shutdown` | Turns the interface on | changed state to up |

"Administratively down" means someone turned the interface off on purpose. A plain "down" usually points to a cable or the device on the other end.

In the capture, `wr` is short for `write memory`, and `[OK]` confirms the configuration was saved.

After enabling Gi0/0, both links turned green.

![Both links up](images/router-both-interface-config.png)

### 4. Set the default gateway on each PC

Each PC points at the router interface in its own network. On V8-PC1, `ipconfig` shows address 192.168.71.10 and default gateway 192.168.71.1.

![V8-PC1 ipconfig](images/v8pc1-ipconfig.png)

The V2 PCs use 192.168.50.1.

V2-PC2's `ipconfig /all` in Step 6 shows this gateway. V2-PC1's reply to V8-PC1 in Step 5 also confirms its gateway works, because the reply had to cross the router.

### 5. Test across the router

From V8-PC1, ping the router (`ping 192.168.71.1`): four replies.

![V8-PC1 pings its gateway](images/v8pc1-ping-gateway.png)

From V8-PC1, ping V2-PC1 on the other network (`ping 192.168.50.10`): four replies, so traffic now crosses between the two networks.

![V8-PC1 pings V2-PC1](images/v8pc1-ping-v2pc1.png)

### 6. Troubleshooting: V2-PC2 reaches its router but not the other network

1. From V2-PC2, `ping 192.168.50.1` works, but `ping 192.168.71.1` times out.

   ![V2-PC2 ping results without a gateway](images/v2pc2-no-gateway.png)

2. `ipconfig` on V2-PC2 shows the default gateway is empty. That is the root cause.

   ![V2-PC2 ipconfig, no gateway](images/v2pc2-root-cause.png)

3. Fix: set the default gateway to 192.168.50.1 under Desktop, then IP Configuration.

   ![Assigning the gateway on V2-PC2](images/v2pc2-assign-gateway.png)

4. After the fix, `ipconfig /all` on V2-PC2 shows the default gateway 192.168.50.1, and `ping 192.168.71.10` (V8-PC1) gets four replies.

   ![V2-PC2 pings V8-PC1 after the fix](images/v2pc2-ping-v8pc1.png)

Why: 192.168.50.1 is on V2-PC2's own network, so it needs no gateway. 192.168.71.1 is on another network, and with no gateway the PC has nowhere to send the packet.

## Reading the TTL

| Reply came from | TTL | Meaning |
|---|---|---|
| A PC on the same network (Lesson 2) | 128 | Started at 128, no router in between |
| The router itself | 255 | Routers start at 255 |
| A PC on the other network | 127 | 128 minus one hop through the router |

## Tests

| Test | From | To | Expected | Actual |
|---|---|---|---|---|
| Router, own network | V8-PC1 | 192.168.71.1 | Replies | 4 of 4 replies, TTL 255 |
| Across the router | V8-PC1 | V2-PC1 (192.168.50.10) | Replies | 4 of 4 replies, TTL 127 |
| Router, own network, no gateway | V2-PC2 | 192.168.50.1 | Replies | 4 of 4 replies, TTL 255 |
| Other network, no gateway | V2-PC2 | 192.168.71.1 | Timeout | 4 of 4 timed out |
| Across the router, after fix | V2-PC2 | V8-PC1 (192.168.71.10) | Replies | 4 of 4 replies, TTL 127 |

## What I learned

- A router interface needs both an address and `no shutdown`. A red link light on a new router usually means the port is still off.
- `shutdown` turns an interface off and `no shutdown` turns it on. "Administratively down" means someone turned it off on purpose.
- Each router interface sits in one network and acts as that network's gateway.
- A PC reaches its own network without a gateway, but needs one for any other network.
- The TTL in a reply shows how many routers the packet crossed.

## Conclusion

The router joins the two networks, and traffic now passes between V2 and V8 as long as each PC has the right gateway. Lesson 4 sets up DHCP, so devices on V8 get their addresses automatically.

[Next: Lesson 4, DHCP](../lesson-04-dhcp/README.md)
