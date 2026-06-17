# AI suggestions for topics to further explore

This document contains a list of suggested improvements, RFC alignments, and technical considerations to explore for future revisions of the mobile L4S deployment BCP.

## 1. Host OS Requirements

### Per-Packet ECN Control for Multiplexed Sockets
*   **Topic**: Sockets that multiplex multiple traffic types (e.g., WebRTC multiplexing RTP media, RTCP control, and SCTP data channels on a single UDP socket) cannot use simple socket-wide `setsockopt(IP_TOS)` because some flows (like SCTP) are queue-building and must not be marked `ECT(1)`.
*   **Suggestion**: Recommend that host OSes expose per-packet ECN marking APIs (such as `sendmsg()` with `IP_TOS` / `IPV6_TCLASS` control messages in `cmsg` data) so applications can selectively mark L4S-compatible packets on shared sockets.

### RFC 8888 Alignment for UDP Interactive Media
*   **Topic**: Standardizing ECN feedback for RTP-based real-time interactive media streams.
*   **Suggestion**: Align UDP requirements with **RFC 8888** ("RTP Control Protocol (RTCP) Feedback for Congestion Control on Interactive Real-Time Media"), recommending that interactive media applications use the RFC 8888 feedback format to echo ECN CE markings back to the sender.

### ECN/L4S Path Capability Caching Granularity
*   **Topic**: Caching path ECN capabilities and AccECN Option blocking behaviors at the host OS level to optimize connection bootstrapping and avoid timeout fallback penalties.
*   **Suggestion**: Recommend that host OS TCP stacks implement a hybrid cache to store ECN/L4S path support, using the following granularities:
    *   **Local Interface ID**:
        *   *Wi-Fi*: Keyed on the Access Point's physical MAC address (**BSSID**), as SSID is too coarse and does not uniquely identify the local queue and ISP router.
        *   *Cellular*: Keyed on the carrier network (**PLMN** - MCC+MNC) combined with the active Radio Access Technology (**RAT** - e.g., LTE vs 5G), as middlebox option-blocking policies are carrier-wide.
    *   **Destination Host**: Keyed on the **Destination IP Subnet** (e.g., `/24` for IPv4 or `/48` for IPv6) rather than individual host IPs, to accommodate CDN/load-balancer rotation while caching the path bottleneck properties.
    *   **Expiry**: Cached capability status MUST age-out after a reasonable duration (e.g., 24 hours) to account for interface roaming and network upgrades.

---

## 2. Link-Layer Subsystems Requirements (Modem & Wi-Fi)

### Dynamic Time-Based Queue Sizing
*   **Topic**: Static byte-limit queues (e.g., a hard 16kB queue size) do not scale well on mobile networks where throughput fluctuates from <1 Mbps on weak LTE to 1+ Gbps on 5G.
*   **Suggestion**: Explore recommending dynamic time-based queue sizing (sojourn time limits, e.g., sizing the queue to hold no more than 10-15 ms of data at the current transmission capacity) rather than a static byte/packet limit.

### Cellular Radio Bearer Configuration (RLC Mode)
*   **Topic**: Bypassing in-order link-layer delivery to reduce latency spikes.
*   **Suggestion**: Provide specific guidance recommending that cellular networks map L4S (`ECT(1)`) and NQB (`DSCP-45`) traffic to radio bearers configured with **RLC Unacknowledged Mode (UM)** or with in-order delivery disabled at the radio link control layer, rather than standard RLC Acknowledged Mode (AM).

### RFC 9332 Alignment (DualQ Coupled AQM)
*   **Topic**: Scheduling and coupling configurations for multi-queue schedulers.
*   **Suggestion**: Ensure that multi-queue scheduling and coupling algorithms in the modem or Wi-Fi driver align with the coupled marking and AQM parameters defined in **RFC 9332** to ensure fair resource sharing between L4S and Classic queues.

### RFC 8325 Alignment (DiffServ to 802.11 Wi-Fi Mapping)
*   **Topic**: Mapping DSCP-45 (NQB) to Wi-Fi User Priorities and Access Categories.
*   **Suggestion**: Align Wi-Fi classification with **RFC 8325**, suggesting that Wi-Fi access points and client drivers map NQB (`DSCP-45`) traffic to appropriate WMM Access Categories (typically `AC_VI` or `AC_BE` with optimized queue settings).

---

## 3. Middlebox & Carrier Requirements

### NQB Marking Demotion at Carrier Ingress Boundaries
*   **Topic**: Handling DSCP-45 traffic in carrier networks that do not support NQB queue protection.
*   **Suggestion**: Recommend that ingress nodes (UPFs/PGWs) that lack queue protection capabilities demote unrecognized `DSCP-45` traffic to `DSCP-0` (Best Effort) rather than dropping the packets, to maintain robust backward compatibility.

---

## 4. Path Traversal & Option Blocking Scenarios

### Detailed Handling of AccECN Option Dropping (Blackholing)
*   **Topic**: Middleboxes that drop/blackhole packets containing unrecognized TCP Option Kind 172/174, rather than just stripping them.
*   **Suggestion**: Ensure the BCP explicitly details how host OS stacks must handle option blackholing across all connection phases:
    *   *SYN/ACK Option Drop*: Retransmitting the SYN/ACK without the AccECN option on first timeout, and fallback to non-AccECN handshake on subsequent timeouts.
    *   *Data Segment Option Drop*: Retransmitting the data segment without the option on timeout.
    *   *Pure ACK Option Drop*: Detecting ACK loss (via unexpected sender data retransmissions) and disabling option usage on subsequent ACKs.

### Header Flags Mangling and Zeroing (ECN-Silence)
*   **Topic**: Middleboxes that zero out the ECN bits in the IP header or the ECN flags (AE, CWR, ECE) in the TCP header, creating "ECN-Silence".
*   **Suggestion**: Detail host OS safety behavior when feedback is zeroed out: the sender receives no CE marks, which leads to buffer overflow and packet drops at the bottleneck. The sender MUST react to these packet drops with a standard, safe Classic ECN/Reno response (maintaining connection survival).

## 5. Wed 17 June review

*   "receive-side L4S for TCP" should be renamed to be consistent with RFC 9330

