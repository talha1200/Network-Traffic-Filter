# Work: Layer 2, Layer 2.5, VLANs, and Packet Filtering Q&A

This document contains a series of questions and detailed responses related to Layer 2, Layer 2.5 protocols, VLANs, Q-in-Q tagging, and packet filtering concepts. It serves as a reference for understanding these networking concepts.

---

## Q&A

### **Q1: What is the maximum size of a Layer 2 header?**
The maximum size of a Layer 2 (Ethernet) header depends on the presence of optional fields such as VLAN tags:
- **Standard Ethernet Header**: 14 bytes.
- **With 802.1Q VLAN Tag**: 18 bytes.
- **With Double (Q-in-Q) VLAN Tags**: 22 bytes.
- **With Additional Fields** (e.g., MACsec, extensions): Size increases further.

---

### **Q2: How many VLAN tags can a packet have?**
A packet can have **multiple VLAN tags**, typically seen in Q-in-Q tagging. There is no strict upper limit in theory, but practical limitations arise due to hardware processing capabilities and MTU (Maximum Transmission Unit) constraints. Packets with **4-5 VLAN tags** are rare but possible.

### **Q3: How Many VLAN Tags Can a Packet Have?**
An Ethernet packet can have **multiple VLAN tags** due to **Q-in-Q tagging** (also known as VLAN stacking). While the standard 802.1Q allows a single VLAN tag, Q-in-Q (802.1ad) allows stacking multiple VLAN tags.  

- **802.1Q VLAN Tagging**: 1 VLAN tag.  
- **802.1ad (Q-in-Q)**: Typically 2 VLAN tags (outer provider tag + inner customer tag).  
- **Vendor-Specific or Custom Implementations**: Technically, there is no hard limit to the number of VLAN tags, as tags are inserted sequentially into the Ethernet frame. However, practical limits depend on the maximum Ethernet frame size and the processing capabilities of networking devices. 

### **Q4: How to Tell if a Packet Has VLAN Tags?**

You can identify the presence of VLAN tags by examining the **EtherType/Length field** in the Ethernet frame. Here's the process:

1. **Inspect EtherType Field**:
   - If the EtherType field contains `0x8100`, the frame contains an **802.1Q VLAN tag**.
   - If the EtherType is `0x88A8`, the frame contains a **Q-in-Q VLAN tag** (used in 802.1ad).

2. **Structure of a VLAN Tag**:
   - **802.1Q Tag (4 bytes)**:
     - **Tag Protocol Identifier (TPID)**: 2 bytes (`0x8100` for standard VLAN).  
     - **Tag Control Information (TCI)**: 2 bytes.
       - **Priority Code Point (PCP)**: 3 bits (QoS).  
       - **Drop Eligible Indicator (DEI)**: 1 bit.  
       - **VLAN Identifier (VID)**: 12 bits (VLAN ID range: 0–4095).

3. **Handle Stacked VLAN Tags (Q-in-Q)**:
   - The outer VLAN tag uses EtherType `0x88A8`.
   - The inner VLAN tag uses EtherType `0x8100`.

4. **Parsing VLAN Tags in a Frame**:
   - Start at the 13th byte of the Ethernet frame (after destination and source MACs).
   - If the EtherType is `0x8100` or `0x88A8`, parse the 4 bytes as a VLAN tag.
   - Continue parsing successive tags until you encounter a non-VLAN EtherType (e.g., `0x0800` for IPv4).

#### **Example Ethernet Frame with VLAN Tags**

##### **Single VLAN Tag**:
```
[Dst MAC (6 bytes)] [Src MAC (6 bytes)] [TPID (0x8100)] [TCI (4 bytes)] [Payload...]
```
##### **Double VLAN Tag (Q-in-Q)**:
```
[Dst MAC (6 bytes)] [Src MAC (6 bytes)] [Outer TPID (0x88A8)] [Outer TCI (4 bytes)] [Inner TPID (0x8100)] [Inner TCI (4 bytes)] [Payload...]
```

#### **FPGA Implementation Tip**
1. **Detect VLAN Tags**:
   - Check the EtherType field after the source MAC.
   - Maintain a counter to track multiple VLAN tags if needed.

2. **Logic Example**:
   ```verilog
   always @(posedge clk) begin
       if (ether_type == 16'h8100 || ether_type == 16'h88A8) begin
           vlan_detected <= 1'b1;
           num_vlan_tags <= num_vlan_tags + 1;
       end else begin
           vlan_detected <= 1'b0;
       end
   end
   ```
3. **Store VLAN Information**:
   - Store VLAN IDs (from TCI) for filtering.
   - Ensure support for both single and stacked VLAN tags.

---

### **Q5: How can we identify if a packet has a VLAN tag?**
To identify VLAN tags:
1. Look for the **EtherType/TPID** field in the Ethernet header:
    - **0x8100**: Indicates a standard 802.1Q VLAN tag.
    - **0x88A8**: Indicates a Q-in-Q VLAN tag.
2. If the EtherType matches these values, the next 4 bytes contain the VLAN Tag Control Information (TCI).

---

### **Q6: Can ARP, ICMP, and other non-IP packets have VLAN tags in them?**
Yes, **ARP**, **ICMP**, and other non-IP packets can have **VLAN tags**. VLAN tagging occurs at Layer 2 (Data Link Layer), independent of the Layer 3 (Network Layer) protocol.

#### **ARP with VLAN Tags**
- ARP (Address Resolution Protocol) operates directly over Ethernet and does not depend on an IP header.
- An ARP packet can include a VLAN tag if the Ethernet frame carrying it belongs to a VLAN.

**Example: ARP with VLAN Tag**
```
[Dst MAC (6 bytes)] [Src MAC (6 bytes)] [TPID (0x8100)] [TCI (4 bytes)] [EtherType (0x0806 for ARP)] [ARP Payload...]
```
#### **ICMP with VLAN Tags**
- ICMP (Internet Control Message Protocol) is encapsulated within IP packets (usually IPv4 or IPv6).
- If an Ethernet frame carrying an ICMP packet is tagged, the VLAN tag will be present before the EtherType field for IPv4/IPv6.

**Example: ICMP with VLAN Tag**
```
[Dst MAC (6 bytes)] [Src MAC (6 bytes)] [TPID (0x8100)] [TCI (4 bytes)] [EtherType (0x0800 for IPv4)] [IP Header...] [ICMP Payload...]
```

#### **Other Non-IP Packets with VLAN Tags**
Any protocol encapsulated in an Ethernet frame can have VLAN tags, as tagging occurs at the Ethernet frame level.

##### Examples:
1. **Non-IP Protocols (e.g., STP, LLDP, or proprietary protocols)**:
   - Spanning Tree Protocol (STP): EtherType = `0x0100`
   - Link Layer Discovery Protocol (LLDP): EtherType = `0x88CC`

   **Example Frame**:
   ```
   [Dst MAC (6 bytes)] [Src MAC (6 bytes)] [TPID (0x8100)] [TCI (4 bytes)] [EtherType] [Protocol Payload...]
   ```

2. **Control Protocols**:
   - Protocols like ARP (`0x0806`) or Wake-on-LAN (`0x0842`) can also carry VLAN tags.

#### **Summary of When VLAN Tags Appear**
1. **Ethernet Frame Level**:
   - VLAN tagging is agnostic to the higher-layer protocol (e.g., IP, ARP, ICMP).
   - Any frame belonging to a VLAN will include a VLAN tag, regardless of its payload.

2. **Detecting VLAN Tags in Non-IP Packets**:
   - The **TPID field** (e.g., `0x8100`) after the source MAC indicates a VLAN tag.
   - This applies to both IP and non-IP packets.

#### **Practical Considerations**
- When filtering or processing packets in hardware (e.g., FPGA):
  - Always check for VLAN tags after the source MAC address.
  - Extract VLAN information before inspecting the EtherType for the payload type.

---

### **Q7: Provide a detailed structure for a VLAN tag. What information does it contain, and is it variable in size?**
The VLAN tag is **fixed in size (4 bytes)** and contains the following fields:

| **Field** | **Size (bits)** | **Description**                                                                 |
|-----------|-----------------|---------------------------------------------------------------------------------|
| TPID      | 16              | Tag Protocol Identifier (e.g., `0x8100` for 802.1Q, `0x88A8` for Q-in-Q).       |
| PCP       | 3               | Priority Code Point (used for QoS).                                             |
| DEI       | 1               | Drop Eligible Indicator (frame discard eligibility).                            |
| VID       | 12              | VLAN Identifier (range: `0–4095`).                                              |

---

### **Q8: Provide an example of a packet containing 4–5 VLAN tags.**
An Ethernet packet with multiple VLAN tags (e.g., Q-in-Q):
Ethernet packet structure containing **4 VLAN tags**. Each VLAN tag adds **4 bytes** (2 bytes for TPID + 2 bytes for TCI).

#### **Packet Breakdown**
##### **Header Fields**
1. **Destination MAC (6 bytes)**: `FF:FF:FF:FF:FF:FF` (broadcast).
2. **Source MAC (6 bytes)**     : `00:11:22:33:44:55`.
3. **VLAN Tag 1 (Outer)**       :
   - **TPID**                   : `0x88A8` (indicates Q-in-Q VLAN).
   - **TCI**                    : `0x1001` (PCP: 1, DEI: 0, VLAN ID: 1).
4. **VLAN Tag 2**               :
   - **TPID**                   : `0x8100` (indicates 802.1Q VLAN).
   - **TCI**                    : `0x2002` (PCP: 2, DEI: 0, VLAN ID: 2).
5. **VLAN Tag 3**               :
   - **TPID**                   : `0x8100` (indicates 802.1Q VLAN).
   - **TCI**                    : `0x3003` (PCP: 3, DEI: 0, VLAN ID: 3).
6. **VLAN Tag 4 (Inner)**       :
   - **TPID**                   : `0x8100` (indicates 802.1Q VLAN).
   - **TCI**                    : `0x4004` (PCP: 4, DEI: 0, VLAN ID: 4).
7. **EtherType**                : `0x0800` (IPv4).
8. **Payload (IPv4 header)**    : 

#### **Hexadecimal Representation**
```
FF FF FF FF FF FF                     // Destination MAC (6 bytes)
00 11 22 33 44 55                     // Source MAC (6 bytes)
88 A8 10 01                           // Outer VLAN Tag 1 (TPID: 0x88A8, TCI: PCP=1, DEI=0, VID=1)
81 00 20 02                           // VLAN Tag 2 (TPID: 0x8100, TCI: PCP=2, DEI=0, VID=2)
81 00 30 03                           // VLAN Tag 3 (TPID: 0x8100, TCI: PCP=3, DEI=0, VID=3)
81 00 40 04                           // Inner VLAN Tag 4 (TPID: 0x8100, TCI: PCP=4, DEI=0, VID=4)
08 00                                 // EtherType (IPv4)
45 00 00 3C 1C 46 40 00 40 06 B1 E6   // IPv4 Header (20 bytes, simplified)
C0 A8 01 01                           // Source IP (192.168.1.1)
C0 A8 01 02                           // Destination IP (192.168.1.2)
```

#### **Explanation**
1. **Destination MAC**  : Broadcast (`FF:FF:FF:FF:FF:FF`).
2. **Source MAC**       : `00:11:22:33:44:55`.
3. **VLAN Tags**        :
   - **Outer VLAN Tag** : Indicates Q-in-Q with VLAN ID 1.
   - **Inner Tags**     : Stack three additional 802.1Q VLAN tags with IDs 2, 3, and 4.
4. **Payload**          :
   - An IPv4 packet with header details.

#### **How to Parse This in Hardware (FPGA)**
1. **Start after Source MAC**:
   - Read the first 2 bytes to detect the **TPID** (`0x88A8` or `0x8100`).
   - If TPID matches, process the 4-byte VLAN tag and increment the parser offset.
2. **Continue for Additional VLAN Tags**:
   - Stop when a non-VLAN **EtherType** is detected (e.g., `0x0800` for IPv4).
3. **Output VLAN Information**:
   - Extract **PCP**, **DEI**, and **VID** for all tags.

Would you like a Verilog or SystemVerilog module to parse multiple VLAN tags from an Ethernet packet?
| **Field**              | **Size (bytes)** | **Example Value**              |
|------------------------|------------------|--------------------------------|
| **Destination MAC**    | 6                | `FF:FF:FF:FF:FF:FF`            |
| **Source MAC**         | 6                | `00:11:22:33:44:55`            |
| **Outer Q-in-Q Tag**   | 4                | `88A8 1001` (PCP=1, VLAN ID=1) |
| **Middle Q-in-Q Tag**  | 4                | `88A8 2002` (PCP=2, VLAN ID=2) |
| **Inner Q-in-Q Tag**   | 4                | `88A8 3003` (PCP=3, VLAN ID=3) |
| **EtherType**          | 2                | `0800` (IPv4)                  |
| **Payload**            | Variable         | IPv4 data                      |


### **Q9: What is the name of the layer between Layer 2 and Layer 3?**
The layer between Layer 2 (Data Link Layer) and Layer 3 (Network Layer) is often referred to as the **"Layer 2.5"** or **"Network Access Layer"**. This term is not part of the official OSI model but is used informally to describe functionality that doesn't neatly fit into Layer 2 or Layer 3.

#### **Examples of Layer 2.5 Technologies**
1. **MPLS (Multiprotocol Label Switching)**:
   - MPLS operates between Layer 2 and Layer 3.
   - It uses labels instead of IP headers to forward packets, enabling faster and more efficient routing.

2. **PPP (Point-to-Point Protocol)**:
   - PPP is another protocol often considered part of Layer 2.5.
   - It encapsulates network-layer packets (like IP) for transmission over a Layer 2 link.

3. **VLAN Tagging (802.1Q)**:
   - VLAN tagging, while officially part of Layer 2, adds an additional logical separation and can sometimes be seen as operating in the Layer 2.5 space.

4. **Metro Ethernet Services**:
   - Services that bridge Layer 2 Ethernet with Layer 3 IP/MPLS networks often function as Layer 2.5.

5. **GRE (Generic Routing Encapsulation)**:
   - GRE encapsulates Layer 3 packets within Layer 2 frames, often making it a candidate for Layer 2.5 functionality.

#### **Why Use the Term "Layer 2.5"?**
- To describe protocols or features that **extend Layer 2 functionality** while enabling some Layer 3-like capabilities.
- It helps network engineers conceptualize hybrid roles in modern networking.

---

### **Q10: Can a packet have two or more Q-in-Q tags in it?**
Yes, a packet can have **two or more Q-in-Q (802.1ad)** tags, effectively creating a **stack of VLAN tags**. This is often referred to as **VLAN stacking** or **multiple Q-in-Q tags**. Each Q-in-Q tag is a standard VLAN tag but uses a different **TPID (Tag Protocol Identifier)** to distinguish it from regular VLAN tags.

#### **Why Use Multiple Q-in-Q Tags?**
1. **Service Provider Networks**:
   - Multiple Q-in-Q tags allow service providers to segregate customer traffic further while also adding their own VLAN tags for network management.
   - This technique is commonly used in **Metro Ethernet** and **carrier-grade networks**.

2. **Nested Segmentation**:
   - Enterprises or providers can use multiple levels of tagging to create hierarchical network segmentation.

3. **Traffic Isolation**:
   - It ensures precise isolation of traffic from different customers or services.

#### **Structure of Multiple Q-in-Q Tags**
Each Q-in-Q tag is **4 bytes** (just like a standard VLAN tag) and includes:
- **TPID (Tag Protocol Identifier)** : Typically `0x88A8` for Q-in-Q or `0x8100` for 802.1Q.
- **TCI (Tag Control Information)**  : Contains:
  - **PCP (3 bits)**                 : Priority Code Point (QoS).
  - **DEI (1 bit)**                  : Drop Eligible Indicator.
  - **VID (12 bits)**                : VLAN ID (0–4095).

#### **Example: Ethernet Packet with 3 Q-in-Q Tags**
| **Field**               | **Size (bytes)** | **Example Value**              |
|-------------------------|------------------|--------------------------------|
| **Destination MAC**     | 6                | `FF:FF:FF:FF:FF:FF`            |
| **Source MAC**          | 6                | `00:11:22:33:44:55`            |
| **Outer Q-in-Q Tag**    | 4                | `88A8 1001` (PCP=1, VLAN ID=1) |
| **Middle Q-in-Q Tag**   | 4                | `88A8 2002` (PCP=2, VLAN ID=2) |
| **Inner Q-in-Q Tag**    | 4                | `88A8 3003` (PCP=3, VLAN ID=3) |
| **EtherType**           | 2                | `0800` (IPv4)                  |
| **Payload**             | Variable         | IPv4 data                      |

#### **How to Detect Multiple Q-in-Q Tags**
1. **Parse TPID (Tag Protocol Identifier)**:
   - If the TPID is `0x88A8` or `0x8100`, it's a VLAN tag.
2. **Move to Next Tag**:
   - Skip 4 bytes (length of a VLAN tag) and check the next 2 bytes.
   - If the next TPID is again `0x88A8` or `0x8100`, another tag is present.
3. **Continue Parsing**:
   - Repeat until a non-VLAN EtherType (e.g., `0x0800` for IPv4) is found.

#### **Use Cases for Multiple Q-in-Q Tags**
1. **Customer Traffic Isolation**:
   - Each customer gets a unique outer VLAN ID (S-VLAN), while internal VLAN IDs (C-VLANs) remain unchanged.
2. **Nested Service Segmentation**:
   - Different services (e.g., video, VoIP) within the same customer traffic can use multiple VLAN layers.
3. **Provider Backbone Bridging (PBB)**:
   - Adds even more scalability by combining Q-in-Q with MAC-in-MAC encapsulation.

---

### **Q11: Table of all Layer 2.5 protocols with their structure.**

#### **Table of Layer 2.5 Protocols**

| **Protocol**                 | **Purpose**                                   | **Structure**                                                                                      | **Key-Features**                                              |
|------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| **MPLS (Multiprotocol Label Switching)** | Efficient packet forwarding and routing.    | **MPLS Label (32 bits):** <br> - Label (20 bits) <br> - TC (3 bits) <br> - S (1 bit) <br> - TTL (8 bits) | - Operates between L2 and L3. <br> - Enables traffic engineering and QoS. |
| **VLAN Tagging (802.1Q)**    | Logical network segmentation.                 | **VLAN Tag (4 bytes):** <br> - TPID (16 bits, `0x8100`) <br> - PCP (3 bits) <br> - DEI (1 bit) <br> - VID (12 bits) | - Adds VLAN ID to Ethernet frame. <br> - Fixed 4-byte size.   |
| **Q-in-Q VLAN (802.1ad)**    | Stacked VLANs for service provider networks.  | **Outer Tag (4 bytes):** <br> - TPID (16 bits, `0x88A8`) <br> - TCI (16 bits) <br> **Inner Tag (4 bytes)** | - Allows multiple VLAN tags. <br> - Used in metro Ethernet.  |
| **GRE (Generic Routing Encapsulation)** | Tunneling protocol for encapsulation.       | **GRE Header (4 bytes or more):** <br> - Flags (16 bits) <br> - Protocol Type (16 bits) <br> [Optional Fields] | - Encapsulates Layer 3 packets. <br> - Flexible for VPNs.    |
| **PPP (Point-to-Point Protocol)** | Encapsulation for point-to-point links.     | **PPP Frame:** <br> - Flag (1 byte) <br> - Address (1 byte) <br> - Control (1 byte) <br> - Protocol (2 bytes) <br> - Payload | - Encapsulates network-layer data (e.g., IP).                |
| **Metro Ethernet OAM**       | Operation, Administration, and Maintenance.  | **OAM Frame:** <br> - Destination MAC <br> - Source MAC <br> - Ethertype (e.g., `0x8809`) <br> - Payload (OAM-specific) | - Layer 2.5 for managing Ethernet services.                  |
| **Ethernet over MPLS (EoMPLS)** | Transport Ethernet over MPLS networks.       | **EoMPLS Frame:** <br> - MPLS Label(s) <br> - Ethernet Frame (starting with Dest MAC)               | - Encapsulates Ethernet in MPLS.                             |
| **L2TP (Layer 2 Tunneling Protocol)** | Secure tunneling over IP networks.          | **L2TP Header:** <br> - Flags (1 byte) <br> - Length (2 bytes) <br> - Tunnel ID (2 bytes) <br> - Session ID (2 bytes) | - Tunnels Layer 2 traffic over IP.                           |
| **MAC-in-MAC (802.1ah)**     | Encapsulates Ethernet frames with another MAC header. | **Header Structure:** <br> - Outer Dest MAC (6 bytes) <br> - Outer Src MAC (6 bytes) <br> - B-TAG (4 bytes) <br> - Inner Ethernet Frame | - Extends Ethernet scalability in service provider networks. |

#### **Detailed Protocol Structures**

##### **1. MPLS Label**
| **Field** | **Size (bits)** | **Description**                                   |
|-----------|----------------|----------------------------------------------------|
| Label     | 20             | Unique identifier for the MPLS path (LSP).         |
| TC        | 3              | Traffic Class (used for QoS).                      |
| S         | 1              | Bottom-of-Stack indicator (1 = last MPLS label).   |
| TTL       | 8              | Time-to-Live (prevents infinite forwarding loops). |

##### **2. VLAN Tag (802.1Q)**
| **Field** | **Size (bits)** | **Description**                                    |
|-----------|----------------|-----------------------------------------------------|
| TPID      | 16             | Tag Protocol Identifier (e.g., `0x8100`).           |
| PCP       | 3              | Priority Code Point (used for QoS).                 |
| DEI       | 1              | Drop Eligible Indicator (frame discard eligibility).|
| VID       | 12             | VLAN Identifier (range: `0–4095`).                  |

##### **3. GRE Header**
| **Field**          | **Size (bits)** | **Description**                                               |
|---------------------|----------------|---------------------------------------------------------------|
| Flags               | 16             | Includes optional fields (checksum, key, etc.).               |
| Protocol Type       | 16             | Indicates the encapsulated protocol (e.g., IPv4 = `0x0800`).  |
| Optional Fields     | Variable       | Optional checksum, sequence number, or key for identification.|

##### **4. L2TP Header**
| **Field**          | **Size (bits)** | **Description**                              |
|---------------------|----------------|----------------------------------------------|
| Flags               | 8              | Indicates optional fields in the header.     |
| Length              | 16             | Length of the L2TP header and payload.       |
| Tunnel ID           | 16             | Identifies the tunnel between L2TP endpoints.|
| Session ID          | 16             | Identifies the session within a tunnel.      |

---


This file is intended for networking professionals and developers working with Ethernet and VLAN-related protocols. 

