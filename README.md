# Bare-Metal Network Infrastructure & Reliable UDP (RUDP) Stack

## Overview
This repository contains a complete, custom-built network ecosystem implemented from scratch in Python using only bare-metal OS `socket` libraries. Bypassing high-level networking frameworks, the project explores low-level architecture by building a custom Reliable UDP transport protocol alongside fundamental network infrastructure services (DHCP and DNS).

The system simulates a complete network lifecycle: obtaining an IP, resolving a domain, establishing a handshake, and reliably transferring data over an inherently unreliable network.

## Core Architecture & Components

### 1. Network Services
* **DHCP Server (Dynamic Host Configuration Protocol):** Listens on UDP port 6767 for incoming `DISCOVER` broadcasts. It manages IP allocation by responding with a JSON `OFFER` assigning a designated IP (e.g., `127.0.0.2`) to the joining client.
* **DNS Server (Domain Name System):** A custom resolver running on UDP port 5353. It acts as the directory service, mapping human-readable domain names like `my-app-server.local` to their assigned IP addresses (e.g., `127.0.0.3`) using a static record store.

### 2. TCP Application Framing
To handle TCP's stream nature and prevent partial reads, the TCP Application Server (`app_server.py`) implements a custom length-prefix framing protocol.
* Before transmitting any payload, it prepends a **10-byte zero-padded ASCII length header** (e.g., `0000000065`).
* The receiver strictly reads this 10-byte header first, converts it to an integer, and loops the `recv()` call until exactly that payload size is accumulated, ensuring complete message boundaries.

### 3. Custom Reliable UDP (RUDP) Protocol
The crown jewel of this infrastructure is a robust transport protocol built over standard UDP, ensuring reliable, ordered delivery of data streams.

* **11-Byte Binary Header:** Every packet utilizes a strict binary header packed via `struct.pack('!IIcH')` containing:
    * **Sequence Number (4 bytes):** Tracks packet ordering.
    * **Acknowledgment Number (4 bytes):** Validates delivery.
    * **Flag (1 byte):** Defines packet state (`S`=SYN, `A`=ACK, `D`=DATA, `F`=FIN).
    * **Payload Length (2 bytes):** Size of the attached data chunk.

* **Congestion Control & Window Management:**
        * **Go-Back-N (GBN):** Implements a sliding window allowing multiple unACKed packets in flight. On a 1-second timeout, the server resets `next_seq = base_seq` and retransmits the entire window.
        * **AIMD (Additive Increase / Multiplicative Decrease):** Dynamically shapes traffic. The `window_size` increases by 1 upon valid ACKs (capped at `MAX_WINDOW = 5`), and is halved (`window_size //= 2`) immediately upon detecting packet loss via timeout.

* **Network Fault Simulation:**
    To prove algorithmic resilience, the RUDP client deliberately injects network faults:
    * `SIMULATE_PACKET_LOSS`: Randomly drops ~30% of incoming DATA packets without ACKing them, forcing the server's timeout and GBN loop.
    * `SIMULATE_LATENCY`: Introduces dynamic delays (`time.sleep(0.1 - 0.4s)`) to test the timeout thresholds and sliding window behavior.

## Execution Flow

To simulate the network environment, boot the components sequentially:

1.  **Initialize Local File Server (for test payloads):**
    ```bash
    python -m http.server 8080
    ```
2.  **Start Infrastructure Services:**
    ```bash
    python dhcp_server.py
    python dns_server.py
    ```
3.  **Start RUDP File Server:**
    ```bash
    python app_server_rudp.py
    ```
4.  **Execute Client Transfer:**
    ```bash
    python client_rudp.py
    ```
