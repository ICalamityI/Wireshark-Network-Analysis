# 🦈 Wireshark Network Analysis Lab

**Environment:** Windows 11 · Wireshark 4.x · Ethernet interface

---

## 📋 What This Lab Is

This is a hands-on network analysis lab I completed to build practical packet capture skills using Wireshark. The goal was to move beyond theory and actually see how common protocols behave at the packet level — DNS lookups, TCP connections being established, credentials moving across an unencrypted HTTP connection, and how individual packets reassemble into a full conversation.

The skills practised here map directly to real work in network engineering, SOC analysis, and cloud security — anywhere you need to know what is *actually* moving across a network rather than what you *think* should be.

---

## 🏗️ Architecture — How Wireshark Captures Traffic

```
┌─────────────────────────────────────────────┐
│              Local Machine (Windows)        │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  Wireshark                           │   │
│  │  Capture + display filter engine     │   │
│  └──────────────┬───────────────────────┘   │
│                 │ monitors                  │
│  ┌──────────────▼───────────────────────┐   │
│  │  Network Interface (NIC)             │   │
│  │  Ethernet / Wi-Fi                    │   │
│  └──────────────┬───────────────────────┘   │
│       outbound  │  inbound                  │
│  ┌──────────────▼───────────────────────┐   │
│  │  Command Prompt + Browser            │   │
│  │  nslookup · HTTP requests            │   │
│  └──────────────────────────────────────┘   │
└──────────────┬──────────────────────────────┘
               │
    DNS · TCP · HTTP · ICMP
               │
┌──────────────▼──────────────────────────────┐
│              Internet                       │
│  DNS servers · Web servers · HTTP endpoints │
└─────────────────────────────────────────────┘
```

Wireshark hooks into the NIC and passively records every frame passing through it in both directions. The browser and Command Prompt are what *generate* the traffic — Wireshark only observes. Nothing inside Wireshark changes or sends data.

---

## 🧠 Concepts I Applied

Understanding these before starting made the exercises click rather than feel like blind instruction-following.

**Packets** are small units of data that travel across a network. When you load a page or send a request, that data gets broken into hundreds or thousands of individual packets, each carrying a header with source IP, destination IP, and port number, plus a payload with the actual content. They travel independently and get reassembled at the destination. Wireshark shows you each one.

**Protocols** are the rules that define how data is formatted and transmitted. DNS handles name resolution. HTTP transfers web content. TCP ensures reliable delivery. ICMP covers ping and diagnostics. Each has its own port and packet structure, which is what makes filtering by protocol so powerful.

**The TCP three-way handshake** is the connection setup sequence every TCP session starts with. Your machine sends SYN (*I want to connect*), the server replies SYN-ACK (*accepted*), your machine replies ACK (*ready*). A SYN with no SYN-ACK following it means the connection failed. A RST means it was forcibly terminated. Recognising these patterns instantly tells you whether a connection succeeded or was blocked.

**DNS** translates domain names into IP addresses. Every single network action — browser requests, API calls, emails — starts with a DNS query. Seeing DNS in packets gives you a foundation that transfers directly to cloud DNS troubleshooting in tools like Azure Network Watcher.

**HTTP vs HTTPS** — HTTP is completely unencrypted. Anyone on the network path can read every byte of every request and response. HTTPS wraps HTTP in TLS so the content is unreadable even if captured. This lab makes that difference viscerally obvious.

**Promiscuous mode** allows the NIC to capture packets addressed to *any* machine on the segment, not just your own. Wireshark enables this automatically on start. On a modern switched network you'll mostly see your own traffic and broadcast traffic, but the mode is what makes packet capture possible at all.

---

## 🛠️ Step 1 — Installing Wireshark on Windows

Downloaded the **Windows x64 Installer (.exe)** from [wireshark.org/download.html](https://www.wireshark.org/download.html) — free, no account required. During installation, accepted all defaults and made sure to install **Npcap** when prompted. Npcap is the packet capture driver that bridges Wireshark and the network interface — skipping it means Wireshark opens but can't capture anything. After the install completed, launched Wireshark from the Start menu.

---

## 📡 Step 2 — First Capture

Opened Wireshark and on the welcome screen saw a list of network interfaces, each with a live wave graph showing real-time activity. Double-clicked the Ethernet interface — the one with the most movement in the graph.

Wireshark started capturing immediately. Opened a browser, navigated to a few sites, let it run for about 30 seconds, then clicked the **red square Stop button** in the toolbar.

The result was thousands of packets from just 30 seconds of normal browsing. That raw volume made it immediately clear why just staring at an unfiltered capture is useless — and why display filters are the actual skill to develop.

---

## 🔍 Step 3 — Display Filters

Typed filters directly into the filter bar at the top of the window and pressed Enter. The packet list updated instantly to show only matching traffic.

One thing that stood out early: Wireshark has two filter types and confusing them wastes time. **Capture filters** restrict what gets recorded before a capture starts. **Display filters** restrict what you *see* after the fact, without discarding anything from the capture. Using display filters means you can apply `dns`, look at DNS traffic, remove it, apply `tcp`, look at something else — the full capture is always sitting underneath, unchanged.

These are the filters I worked with throughout the lab:

| Filter | What it shows | When to use it |
|---|---|---|
| `dns` | All DNS queries and responses | Troubleshooting name resolution, spotting unusual domain lookups |
| `http` | Unencrypted HTTP traffic only | Finding cleartext data, debugging web apps without HTTPS |
| `tcp` | All TCP traffic | Starting point for connectivity investigations |
| `tcp.flags.syn == 1` | TCP SYN packets — connection attempts | Seeing which hosts are trying to connect and where |
| `tcp.flags.reset == 1` | TCP RST packets — connection resets | Finding refused or forcibly closed connections |
| `icmp` | All ICMP traffic including ping | Verifying basic reachability between hosts |
| `ip.addr == 192.168.1.1` | All traffic to or from a specific IP | Isolating a single host in a busy capture |
| `ip.src == 10.0.0.5` | Traffic from a specific source only | Isolating outbound traffic from one machine |
| `tcp.port == 443` | All HTTPS traffic | Identifying encrypted web traffic by port |
| `http.request` | HTTP GET and POST requests only | Spotting web requests, identifying potential exfiltration |

---

## 🧪 Step 4 — Exercises

### Exercise A — Capturing a DNS Lookup

The objective was to generate a DNS query on demand and find it in Wireshark. Used `nslookup` in Command Prompt — a built-in Windows tool that manually triggers a DNS lookup and shows the result in the terminal. The workflow was: start a capture in Wireshark, switch to Command Prompt, run the lookup, switch back and stop the capture.

Opened Command Prompt via Windows key → `cmd` → Enter, then ran:

```
nslookup google.com
```

The terminal returned the IP address(es) for `google.com`. Applied the `dns` display filter in Wireshark and found two packets in the Info column — `Standard query A google.com` (my machine asking) and `Standard query response A google.com` (the DNS server answering). Clicked the response packet, expanded the **Domain Name System (response)** → **Answers** section in the detail pane, and confirmed the A record IP matched exactly what the terminal had shown.

What made this click: that lookup is happening silently before *every* network action my machine takes — browser tabs, app updates, background services. In a security context, an unexpected DNS query to an unusual domain in a capture is often the earliest visible indicator of malware phoning home.

---

### Exercise B — The TCP Three-Way Handshake

Started a new capture, navigated to `http://example.com` in the browser (HTTP deliberately, not HTTPS — TLS would obscure the handshake), then stopped. Ran `nslookup example.com` to get the IP, then applied:

```
tcp and ip.addr == [IP from nslookup]
```

Found the three packets that establish every TCP connection:

| Packet | Flags | What it means |
|---|---|---|
| 1st | `SYN` | My machine: *I want to connect. Here is my sequence number.* |
| 2nd | `SYN, ACK` | The server: *Got it. Here is my sequence number. Connection accepted.* |
| 3rd | `ACK` | My machine: *Confirmed. Connection is open. Ready to send data.* |

**📸 Screenshot — TCP handshake captured:**

![TCP Three-Way Handshake](TCP_handshake.png)

In the screenshot, packets 5246, 5252, and 5253 are the SYN, SYN-ACK, and ACK completing the handshake to port 443. The packet detail pane at the bottom fully decodes the TCP layer — sequence numbers, acknowledgement numbers, and flags all visible. The filter `tcp and ip.addr == 172.66.147.243` is what isolated these three packets from the 8,000+ others in the capture.

The practical takeaway: if a SYN exists with no SYN-ACK following it, the server is unreachable or blocking the connection. A RST after the SYN means it was actively refused. You don't need to guess — the packets tell you exactly what happened.

---

### Exercise C — Cleartext Credentials Over HTTP

> ⚠️ **Educational exercise only** — performed on a test environment I own. Never use this technique against systems or networks you don't have explicit permission to analyse.

Started a capture, submitted a login form over plain HTTP, stopped the capture, and applied:

```
http.request.method == POST
```

Clicked the POST packet and expanded the **HTML Form URL Encoded** layer in the detail pane.

**📸 Screenshot — Credentials visible in plaintext:**

![Cleartext credentials in packet capture](Packet_capture.png)

The screenshot shows packet 1473 — a POST to `/signin.html`. The detail pane shows `user_login = "hello"` and `user_password = "notsosafehuh!"` sitting in the open, readable by anyone on the network path at the time. No cracking, no decryption — just opening the packet.

This exercise made the HTTP vs HTTPS argument concrete in a way that no diagram ever could. Without TLS, credentials are not protected — they're just text moving across the wire. This is the demonstration security teams use when developers push back on enforcing HTTPS.

---

### Exercise D — Following a Full TCP Stream

Captured HTTP traffic, right-clicked a packet, and selected **Follow → TCP Stream**. Wireshark reassembled all packets from that session into a single readable conversation.

**📸 Screenshot — Full TCP stream reconstructed:**

![Follow TCP Stream](Stream_follow.png)

The screenshot shows the TCP stream window with filter `tcp.stream eq 1` applied. The complete XML payload exchanged between `10.0.0.4` and `168.63.129.16` is fully reconstructed — Azure VM configuration data including container IDs, role instance details, and configuration endpoints, all visible as a coherent conversation rather than scattered packet fragments.

Individual packets are fragments of a story. The stream view is the whole story — which is exactly what incident responders need when reconstructing what happened during a network event, determining what data moved, and what commands were executed.

---

## 💾 Step 5 — Saving Captures

Saves all notable captures using **File → Save As → `.pcapng` format**. To export only the packets matching a current filter: apply the filter first, then **File → Export Specified Packets → Displayed**.

---

## 💡 What I Took Away

A few things that weren't obvious before doing this hands-on:

The sheer *volume* of background traffic on a normal machine is surprising. Even with no browser open, background services are generating DNS queries, TCP connections, and HTTPS traffic constantly. Without filters you see none of it clearly — with the right filter you can isolate exactly the conversation you're looking for in seconds.

Seeing cleartext credentials in a live capture was the moment the HTTP vs HTTPS discussion stopped being abstract. Reading `user_password = "notsosafehuh!"` out of a packet detail pane with no special tooling makes the point better than any documentation.

The TCP handshake and stream reconstruction exercises built a mental model that I can already see applying to Azure Network Watcher logs and VPC flow log analysis — the same patterns, just at a higher level of abstraction.

---

## 📎 Resources

- [Wireshark Official Documentation](https://www.wireshark.org/docs/)
- [Wireshark Display Filter Reference](https://www.wireshark.org/docs/dfref/)
- [Wireshark Sample Captures (for practice)](https://wiki.wireshark.org/SampleCaptures)
