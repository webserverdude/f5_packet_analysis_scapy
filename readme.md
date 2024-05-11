# Packet Analysis with Scapy and tcpdump: Checking Compatibility with F5 SSL Orchestrator

## Introduction

In this guide I want to demonstrate how you can use Scapy (https://scapy.net/) and tcpdump for reproducing and troubleshooting layer 2 issues with F5 BIG-IP devices.</br>
Just in case you get into a fingerpointing situation...

## Starting situation

This is a quite recent story from the trenches: My customer uses a Bypass Tap to forward or mirror data traffic to inline tools such as IDS/IPS, WAF or threat intelligence systems. This ByPass Tap offers a feature called Network Failsafe (also known as Fail-to-Wire). This is a fault tolerance feature that protects the flow of data in the event of a power outage and/or system failure. It allows traffic to be rerouted while the inline tools (IDS/IPS, WAF or threat intelligence systems) are shutting down, restarting, or unexpectedly losing power (see red line named _Fallback_ in the picture below).

Since the ByPass Tap itself does not have support for SSL decryption and re-encryption, an F5 BIG-IP SSL Orchestrator shall be introduced as an inline tool in a Layer 2 inbound topology. Tools directly connected to the Bypass Tap will be connected to the SSL Orchestrator for better visibility.</br>
To check the status of the inline tools, the Bypass Tap sends health checks through the inline tools. What is sent on one interface must be seen on the other interace and vice versa.

So if all is OK (health check is green), traffic will be forwarded to the SSL Orchestrator, decrypted and sent to the IDS/IPS and the TAP, and then reencrypted and sent back to the Bypass Tap.</br>
If the Bypass Tap detects that the SSL Orchestrator is in a failure state, it will just forward the traffic to the switch.

This is the traffic flow of the health checks:

![Bypass Tap HC](/assets/bypass-tap-hc.png)

## Target topology

This results in the following topology:

![Target Topology](/assets/target-topology.png)

## Problem description

During commissioning of the new topology, it turned out that the health check packets are not forwarded through the vWire configuried on the BIG-IP.
A packet analysis with Wireshark revealed that the manufacturer uses ARP-like packets with opcode 512 (HEX 02 00). This opcode is not defined in the RFC that describes ARP (https://datatracker.ietf.org/doc/html/rfc826), the RFC only describes the opcodes Request (2 or HEX 00 01) and 2 (2 or HEX 00 02).

![Wireshark analysis lab - customer](/assets/wireshark-customer-env.png)

__NOTE:__ Don't get confused that you see ARP packets on port 1.1 and 1.2. They are not passing through, the Bypass Tap is just send those packets from both sides of the vWire, as explained above. The source MAC on port 1.1 and 1.2 are different.

Since the Bypass Tap is located right behind the customer's edge firewall, lengthy and time-consuming tests on the live system are not an option, since it would result in a massive service interuption. 
Therefore, a BIG-IP i5800 (the same model as the customer's) was set up as SSL Orchestrator and a vWire configuration was build in my employers lab. The vWire configuration can be found in this guide (https://clouddocs.f5.com/sslo-deployment-guide/chapter2/page2.7.html).

__INFO:__ For those not familiar with vWire: "Virtual wire … creates a layer 2 bridge across the defined interfaces. Any traffic that does not match a topology listener will pass across this bridge."

## Lab Topology

The following topology was used for the lab:

![Lab Topology](/assets/lab-topology.png)

I build a vWire configuration on the SSL Orchestrator, as in the customer's environment. A Linux system with Scapy installed was connected to Interface 1.1. With Scapy TCP, UDP and ARP packets can be crafted or sent like a replay from a Wireshark capture.
Interface 1.3 was connected to another Linux system that should receive the ARP packets.
All tcpdumps were captured on the F5 and analyzed on the admin system (not plotted).

## Validating vWire Configuration

To check the functionality of the F5 and the vWire configuration, two tests were performed. 

1. A replay of the Healthcheck packets from the Bypass Tap and 
2. a test with RFC-compliant ARP requests.

## Use Scapy to resend the faulty packets

First, I used Wireshark to extract a single packet from packet analysis we took in the customer environment and saved it to a pcap file. I replayed this pcap file to the F5 with Scapy.
The sendp() function will work at layer 2, it requires the parameters _rdpcap_ (location of the pcap file for replay) and _iface_ (which interface it shall use for sending). 

```bash
webserverdude@tux480:~$ sudo scapy -H
WARNING: IPython not available. Using standard Python shell instead.
AutoCompletion, History are disabled.
Welcome to Scapy (2.5.0)
>>> sendp(rdpcap("/home/webserverdude/cusomter-case/bad-example.pcap"),iface="enp0s31f6")
.
Sent 1 packets.
```

This test confirmed the behavior that was observed in the customer's environment. 
The F5 BIG-IP does not forward this packet.

![Wireshark analysis lab - bad](/assets/wireshark-lab-bad.png)

## Use PING and Scapy to send RFC-compliant ARP packets

To create RFC-compliant ARP requests, I first sent an ARP request (opcode 1) through the vWire via PING command. 
As expected, this was sent through the vWire. To ensure that this also works with Scapy, I also resent this packet with Scapy.

```bash
>>> sendp(rdpcap("/home/webserverdude/cusomter-case/good-example.pcap"),iface="enp0s31f6")
.
Sent 1 packets.
```

In the Wireshark analysis it can be seen that this packet is incoming on port 1.1 and then forwarded to port 1.3 through the vWire.

![Wireshark analysis lab - good1](/assets/wireshark-lab-good1.png)

![Wireshark analysis lab - good2](/assets/wireshark-lab-good2.png)

## Moving on

It became evident that the BIG-IP was dropping ARP packets that failed to meet RFC compliance, rendering the Bypass Tap from this particular vendor seemingly incompatible with the BIG-IP. Following my analysis, the vendor was able to develop and provide a new firmware release addressing this issue. 
To verify that the issue was resolved in this firmware release, my customer's setup, the exact same model of the Bypass Tap and a BIG-IP i5800, were deployed in my lab, where the new firmware underwent thorough testing. With this approach I could test the functionality and compatibility of the systems under controlled conditions.  

In this Wireshark analysis it can be seen that the Healthcheck packets are incoming on port 1.1 and then forwarded to port 1.3 through the vWire (marked in green) and also the other way round, coming in on port 1.3 and then forwarded to port 1.1 (marked in pink).

![Wireshark analysis lab new FW - good1](/assets/wireshark-lab-newFW1.png)

Also now you can see that the packet is a proper gratuitous ARP reply (https://wiki.wireshark.org/Gratuitous_ARP).

![Wireshark analysis lab new FW - good2](/assets/wireshark-lab-newFW2.png)

Because the Healthcheck packets were not longer dropped by the BIG-IP, but were forwarded through the vWire the Bypass Tap subsequently marked the BIG-IP as healthy and available. 

The new firmware resolved the issue. Consequently, my customer could confidently proceed with this project, free from the constraints imposed by the compatibility issue.