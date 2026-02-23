
```markdown
# Deep Dive: Custom Network Stack & Reliable UDP (RUDP)

## The Core Concept (In Plain English)
Imagine trying to send a certified document using a delivery service that randomly loses mail, drops packages, and doesn't tell you if the package arrived. That delivery service is **UDP** (User Datagram Protocol) – it's fast, but completely unreliable. 

This project solves that problem from scratch. It builds a custom **Reliable UDP (RUDP)** protocol on top of basic UDP, adding tracking, retries, and delivery confirmations. To make it a complete network ecosystem, it also implements custom **DHCP** (to assign IP addresses) and **DNS** (to translate names into IPs) servers. 

Everything is built using Python's bare-metal `socket` library, without relying on external networking frameworks.

---

## The Network Components

### 1. DHCP Server (Dynamic Host Configuration Protocol)
* **What it does:** Think of it as the "housing authority" of the network. When a new computer (client) joins, it doesn't have an address. The DHCP server leases it an available IP address so it can communicate.
* **Technical Implementation:** Listens on a specific port for discovery broadcasts. It manages a pool of available IP addresses, assigns them to requesting MAC addresses, and prevents IP conflicts.

### 2. DNS Server (Domain Name System)
* **What it does:** It's the "Contacts App" or "Phonebook" of the network. Humans use names (like `server.local`), but computers need IP addresses (like `192.168.1.50`). 
* **Technical Implementation:** Maintains a dictionary mapping hostnames to IPs. When a client wants to connect to the RUDP server, it first asks the DNS server to resolve the hostname into a usable IP address.

---

## The Heart of the Project: Reliable UDP (RUDP)
Standard UDP just fires packets into the network and hopes for the best. Our custom RUDP adds the reliability of TCP while maintaining control over the exact flow of data.

### Connection Establishment (The 3-Way Handshake)
Before sending data, the client and server must agree to talk.
1. **SYN:** Client sends a synchronization request ("Are you there?").
2. **SYN-ACK:** Server responds with an acknowledgment ("I'm here, are you ready?").
3. **ACK:** Client confirms ("Yes, let's start sending data.").


### Ensuring Delivery (Sequence & ACKs)
Every packet of data is assigned a **Sequence Number**. When the server receives packet #1, it sends back an **ACK 1** (Acknowledgment). If the client doesn't receive the ACK within a specific timeframe (Timeout), it assumes the packet was lost in transit and retransmits it.

### Congestion Control (Not breaking the network)
If we send data too fast, the network routers will choke and drop packets. We manage this using two algorithms:

* **Go-Back-N (GBN):** Instead of waiting for an ACK after *every single packet*, the client sends a "window" of packets (e.g., 5 at a time). If packet #3 is lost, the server drops everything after it, and the client "goes back" and resends from packet #3 onwards.


* **AIMD (Additive Increase / Multiplicative Decrease):** The protocol constantly probes the network's capacity. 
  * *Additive Increase:* If packets are arriving successfully, we slowly increase the transmission speed (adding to the window size).
  * *Multiplicative Decrease:* The moment a packet is lost, we assume the network is congested and instantly cut the transmission speed in half to let the network recover.


---

## How to Run the Ecosystem

To see the system in action, you must start the infrastructure in the correct order, simulating a real network boot-up:

**Step 1: Start the Network Services**
Open a terminal and start the DHCP server to begin assigning IPs:
```bash
python DHCP_Server.py

```

Open a second terminal and start the DNS resolver:

```bash
python DNS_Server.py

```

**Step 2: Start the Application Server**
Open a third terminal and start the RUDP Server. This server will bind to an IP and wait for reliable connections:

```bash
python RUDP_Server.py

```

**Step 3: Connect a Client**
Open a fourth terminal and run the client. The client will dynamically get an IP, resolve the server's name via DNS, perform the 3-way handshake, and securely download the data payload:

```bash
python Client.py

```

```

ז
