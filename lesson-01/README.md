# Lesson 1: Two PCs and a Switch

**Topic:** Building a small LAN in Packet Tracer
**Goal:** Connect two PCs through a switch, give them static IP addresses, and confirm they can reach each other.
**Why I learned it:** My home lab maps kept coming out wrong because I did not understand what a switch does or how addressing works. This is the smallest network I could build to test that.

## Step 1: Place the devices
Add a 2960 switch and two PCs to the workspace.

![Choosing the switch](images/step2.png)
![Devices placed](images/step1.png)

## Step 2: Cable and name them
Connect each PC to the switch with a copper straight-through cable. Green triangles mean the link is up. I renamed the devices SW-V2, V2-PC1 and V2-PC2 to match the V2 network in my lab. The finished topology is shown at the end of this lesson.

![Cable type](images/step2-1.png)

## Step 3: Set a static IP address on the first PC
On V2-PC1, open Desktop, then IP Configuration. Choose Static and enter:
- IPv4 address: 192.168.50.10
- Subnet mask: 255.255.255.0

The address identifies this PC. The mask tells it which part of the address is the network (192.168.50) and which part is the device (10). Gateway and DNS stay at 0.0.0.0 because the traffic never leaves this one network.

![Desktop tab](images/step3.png)
![IP configuration](images/step3-1.png)

## Your turn: configure the second PC
Now do the same on V2-PC2. Three rules:
1. Same network: the first three numbers must match (192.168.50).
2. Unique device number: no two devices can share an address, so use .20 instead of .10.
3. Same subnet mask: 255.255.255.0.

Try it before reading on. When you are done, hover over V2-PC2 and confirm FastEthernet0 shows 192.168.50.20/24.

Stretch challenge: change V2-PC2 to 192.168.60.20 and run the ping in the next step again. Write down what happens and why, then change it back.

## Step 4: Test with ping
With both PCs configured, open Command Prompt on V2-PC1 and run `ping 192.168.50.20`. Four packets sent, four received, 0% loss. The first reply took 203 ms and the rest took under 1 ms.

![Ping result](images/step4.png)

## Step 5: Verify the ports
Hovering over V2-PC1 shows FastEthernet0 is up with 192.168.50.10/24. Hovering over the switch shows only FastEthernet0/1 and 0/2 are up, and every port is in VLAN 1.

![PC port status](images/step5.png)
![Switch port status](images/step5-5.png)

## Conclusion
Two devices can talk when they share the same network (same subnet and mask) and a switch connects them. No gateway is needed inside one network. Each device needs a unique address within that network.

## Final topology
![Final topology](images/final-topology.png)