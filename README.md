# Network-Traffic-Filter
This repository contains an implementation of a **live network traffic filter** capable of filtering Ethernet traffic from Layer 2 to Layer 4.

The filtering engine supports the following protocols and criteria:
- **Layer 2 (Ethernet)** : Source/Destination MAC address, VLAN, Ether Type
- **Layer 3 (IP)**       : Source/Destination IP address, ToS/DSCP, IPv4/IPv6
- **Layer 4 (Transport)**: Source/Destination port, Protocol type (TCP, UDP, ICMP, etc.)

This project is designed to be easily extendable and can be adapted to different hardware and network configurations. The filter can be used in various applications such as network traffic analysis, intrusion detection systems (IDS), and quality of service (QoS) management.
