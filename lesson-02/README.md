# Lesson 2: Two isolated networks

## Goal

Add a second network next to the first one, with no link between them, and show that devices on different networks cannot reach each other.

## Why it matters

My home lab has two networks that are kept apart on purpose: a closed lab network (V2) and an update network (V8). Before connecting anything between them, I wanted to see what "isolated" looks like and why pings fail across it.

The same check is one of the first things to try when a user can reach some devices but not others: are the two devices on the same network?

## Final topology

![Final topology with addresses](images/final-topology.png)

Two separate networks, each with its own switch and two PCs. There is no cable and no router between the two switches.

## Address table

| Device | Network | IP address | Subnet mask | Default gateway |
|---|---|---|---|---|
| V2-PC1 | 192.168.50.0/24 | 192.168.50.10 | 255.255.255.0 | none |
| V2-PC2 | 192.168.50.0/24 | 192.168.50.20 | 255.255.255.0 | none |
| V8-PC1 | 192.168.71.0/24 | 192.168.71.10 | 255.255.255.0 | none |
| V8-PC2 | 192.168.71.0/24 | 192.168.71.20 | 255.255.255.0 | none |

## Steps

### 1. Start from Lesson 1

SW-V2 with V2-PC1 and V2-PC2, addressed in Lesson 1. On V2-PC1, `ipconfig` shows 192.168.50.10 with an empty default gateway, and `ping 192.168.50.20` gets four replies.

![V2-PC1 pinging V2-PC2](images/ping-test-v2pc1-to-v2pc2.png)

### 2. Add the second network

1. Place a second 2960 switch and rename it SW-V8.
2. Place two PCs and rename them V8-PC1 and V8-PC2.
3. Connect each PC to SW-V8 with the same cable type as Lesson 1.
4. Do not connect SW-V8 to SW-V2.
5. Set V8-PC1 to 192.168.71.10 and V8-PC2 to 192.168.71.20, mask 255.255.255.0, and leave the gateway empty.

![Both networks placed, no link between the switches](images/two-networks.png)

Why: the two networks use different first three numbers, 192.168.50 and 192.168.71, so they are different networks.

### 3. Check that the new network works

On V8-PC1, open Desktop, then Command Prompt. Run `ipconfig`, then `ping 192.168.71.20`.

![V8-PC1 ipconfig and ping](images/ping-test-v8pc1-to-v8pc2.png)

Result: four replies, 0% loss. The address is 192.168.71.10 and the default gateway is empty. Devices on the same network reach each other through the switch with no gateway.

### 4. Try to cross between the networks

On V2-PC1, run `ping 192.168.71.10`.

![V2-PC1 ping timeout](images/ping-test-v2pc1-to-v8pc1.png)

Result: four timeouts, 100% loss.

Why it fails:
- 192.168.71.10 is outside V2-PC1's own network (192.168.50.0/24).
- A PC sends traffic for another network to its default gateway. V2-PC1 has none, so the packet has nowhere to go.
- Even with a gateway, nothing connects the two switches.

## Tests

| Test | From | To | Expected | Actual |
|---|---|---|---|---|
| Same network (V2) | V2-PC1 | V2-PC2 | Replies | 4 of 4 replies, 0% loss |
| Same network (V8) | V8-PC1 | V8-PC2 | Replies | 4 of 4 replies, 0% loss |
| Different networks | V2-PC1 | V8-PC1 | Timeout | 4 of 4 timed out, 100% loss |

Not tested in this lesson: pings from V8 to V2. The result should match the V2 to V8 test.

## What I learned

- Devices share a network when their addresses start the same way and the mask matches. Here that is the first three numbers.
- Two networks with nothing between them are isolated. This is the same setup as my V2 and V8 lab networks.
- A failed ping across networks is the expected result here, not a fault. The next lesson adds a router to change it.

## Conclusion

The two networks work on their own and cannot reach each other. That is the design I want for my lab, and it sets up Lesson 3, where a router joins them and I see what changes.

[Next: Lesson 3, a router between two networks](../lesson-03-router/README.md)
