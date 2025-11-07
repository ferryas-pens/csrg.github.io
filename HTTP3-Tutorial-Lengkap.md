# Tutorial Hands-On HTTP/3 & QUIC
## Panduan Lengkap dari Teori hingga Praktik

**Versi:** 1.0  
**Untuk:** Pembelajaran HTTP/3 dan QUIC dari dasar hingga mahir  
**Durasi:** 8-10 jam hands-on  
**Prerequisites:** Pemahaman HTTP/2 (recommended)

---

# DAFTAR ISI

1. [BAB 1: DASAR TEORI HTTP/3 & QUIC](#bab-1-dasar-teori-http3--quic)
2. [BAB 2: DIAGRAM PERCOBAAN](#bab-2-diagram-percobaan)
3. [BAB 3: PERSIAPAN ENVIRONMENT](#bab-3-persiapan-environment)
4. [BAB 4: LANGKAH-LANGKAH INSTALASI](#bab-4-langkah-langkah-instalasi)
5. [BAB 5: EKSPERIMEN](#bab-5-eksperimen)
6. [BAB 6: UJI EKSPERIMEN](#bab-6-uji-eksperimen)
7. [BAB 7: KESIMPULAN](#bab-7-kesimpulan-dan-langkah-selanjutnya)

---

# BAB 1: DASAR TEORI HTTP/3 & QUIC

## 1.1 Pengenalan HTTP/3

HTTP/3 adalah versi ketiga major protocol HTTP yang merevolusi transport layer dengan menggunakan **QUIC** (Quick UDP Internet Connections) sebagai pengganti TCP. Dipublikasikan sebagai **RFC 9114** pada Juni 2022, HTTP/3 mengatasi keterbatasan fundamental yang masih ada di HTTP/2.

### Evolusi HTTP

```
HTTP/0.9 (1991)  → Single line protocol
    ↓
HTTP/1.0 (1996)  → Headers, methods
    ↓
HTTP/1.1 (1997)  → Persistent connections, pipelining
    ↓
HTTP/2 (2015)    → Binary, multiplexing over TCP
    ↓
HTTP/3 (2022)    → Multiplexing over QUIC/UDP ⭐
```

### Mengapa HTTP/3 Diperlukan?

**Masalah HTTP/2 yang Tersisa:**

1. **TCP Head-of-Line Blocking**
   - HTTP/2 mengatasi HOL blocking di HTTP layer
   - TAPI masalah masih ada di TCP layer
   - Satu packet loss memblokir SEMUA streams

2. **Slow Connection Establishment**
   - TCP handshake: 1 RTT
   - TLS handshake: 1-2 RTT
   - Total: 2-3 RTT sebelum data transfer

3. **Connection Migration Issues**
   - TCP connection tied to IP address
   - Network change = new connection
   - Mobile users suffer

4. **Ossified Middleboxes**
   - Firewalls, NATs, proxies expect TCP
   - Hard to upgrade TCP protocol
   - Innovation stagnated

**Solusi HTTP/3:**

1. **QUIC Protocol (UDP-based)**
   - No TCP HOL blocking
   - Each stream independent
   - Packet loss affects only one stream

2. **0-RTT Connection Setup**
   - Resume connections instantly
   - Cached credentials
   - Near-zero latency

3. **Connection Migration**
   - Connection ID, not IP-based
   - Seamless network switching
   - Perfect for mobile

4. **Built-in Encryption**
   - Encryption is mandatory
   - TLS 1.3 integrated
   - Better security by default

5. **Better Congestion Control**
   - Improved algorithms
   - Faster recovery from packet loss
   - Adaptive to network conditions

## 1.2 Arsitektur HTTP/3

### Protocol Stack Comparison

```
HTTP/1.1 & HTTP/2:          HTTP/3:
┌─────────────┐             ┌─────────────┐
│    HTTP     │             │   HTTP/3    │
├─────────────┤             ├─────────────┤
│  TLS (opt)  │             │             │
├─────────────┤             │    QUIC     │
│    TCP      │             │ (TLS 1.3)   │
├─────────────┤             ├─────────────┤
│     IP      │             │     UDP     │
└─────────────┘             ├─────────────┤
                            │     IP      │
                            └─────────────┘
```

### QUIC Layer Details

```
┌────────────────────────────────────────┐
│           HTTP/3 Layer                 │
│  (Methods, Headers, Body, Frames)      │
├────────────────────────────────────────┤
│          QUIC Transport Layer          │
│  ┌──────────────────────────────────┐ │
│  │   Connection Management          │ │
│  ├──────────────────────────────────┤ │
│  │   Stream Multiplexing            │ │
│  ├──────────────────────────────────┤ │
│  │   Flow Control                   │ │
│  ├──────────────────────────────────┤ │
│  │   Congestion Control             │ │
│  ├──────────────────────────────────┤ │
│  │   Loss Detection & Recovery      │ │
│  ├──────────────────────────────────┤ │
│  │   TLS 1.3 (Integrated)          │ │
│  └──────────────────────────────────┘ │
├────────────────────────────────────────┤
│              UDP                       │
└────────────────────────────────────────┘
```

## 1.3 QUIC Protocol Deep Dive

### 1.3.1 QUIC Connection Establishment

**Traditional TCP+TLS (3 RTT):**
```
Client                              Server
   │                                   │
   │  ① SYN                            │
   │──────────────────────────────────>│
   │  ② SYN-ACK                        │
   │<──────────────────────────────────│
   │  ③ ACK                            │
   │──────────────────────────────────>│
   │  [TCP established: 1 RTT]         │
   │                                   │
   │  ④ ClientHello (TLS)              │
   │──────────────────────────────────>│
   │  ⑤ ServerHello + Certificate      │
   │<──────────────────────────────────│
   │  ⑥ Finished                       │
   │──────────────────────────────────>│
   │  [TLS established: +2 RTT]        │
   │                                   │
   │  ⑦ HTTP Request                   │
   │──────────────────────────────────>│
   │  [Total: 3 RTT before data]       │
```

**QUIC 1-RTT Connection:**
```
Client                              Server
   │                                   │
   │  ① Initial + ClientHello          │
   │     + HTTP Request                │
   │──────────────────────────────────>│
   │                                   │
   │  ② Initial + ServerHello          │
   │     + HTTP Response               │
   │<──────────────────────────────────│
   │  [Total: 1 RTT]                   │
```

**QUIC 0-RTT Connection (Resume):**
```
Client                              Server
   │                                   │
   │  ① 0-RTT + HTTP Request           │
   │     (using cached credentials)    │
   │──────────────────────────────────>│
   │                                   │
   │  ② 1-RTT + HTTP Response          │
   │<──────────────────────────────────│
   │  [Total: 0 RTT for request!]      │
```

### 1.3.2 QUIC Packet Structure

```
QUIC Packet:
┌────────────────────────────────────────────┐
│ Header (Long or Short)                     │
├────────────────────────────────────────────┤
│ ┌────────────────────────────────────────┐ │
│ │ Frame 1: STREAM                        │ │
│ │  - Stream ID                           │ │
│ │  - Offset                              │ │
│ │  - Length                              │ │
│ │  - Data                                │ │
│ └────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────┐ │
│ │ Frame 2: ACK                           │ │
│ │  - Largest Acknowledged                │ │
│ │  - ACK Delay                           │ │
│ │  - ACK Ranges                          │ │
│ └────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────┐ │
│ │ Frame 3: CRYPTO                        │ │
│ │  - TLS handshake data                  │ │
│ └────────────────────────────────────────┘ │
├────────────────────────────────────────────┤
│ Packet Protection (AEAD)                   │
└────────────────────────────────────────────┘
```

**QUIC Frame Types:**

| Frame Type | Purpose | Key Feature |
|-----------|---------|-------------|
| STREAM | Carry data | Per-stream ordering |
| ACK | Acknowledgment | Precise feedback |
| RESET_STREAM | Cancel stream | Granular control |
| STOP_SENDING | Request stop | Flow control |
| CRYPTO | TLS handshake | Integrated security |
| NEW_TOKEN | 0-RTT token | Fast resume |
| PING | Keep-alive | Connection health |
| CONNECTION_CLOSE | Terminate | Graceful shutdown |
| PATH_CHALLENGE | Validate path | Migration support |
| PATH_RESPONSE | Confirm path | Migration support |

### 1.3.3 Stream Independence

**TCP/HTTP/2 Problem:**
```
Stream 1: [Data][Data][Data]
Stream 2: [Data][Data][Data]
Stream 3: [Data][Data][Data]
           ↓
TCP Layer: [.........X...........]
                      ↑
                 Packet Loss
                      ↓
           ALL streams blocked!
```

**QUIC/HTTP/3 Solution:**
```
Stream 1: [Data][Data][Data] → Continues ✓
Stream 2: [Data][Data][X]    → Only this blocked
Stream 3: [Data][Data][Data] → Continues ✓
           ↓
QUIC Layer: Independent streams
            Only affected stream waits
```

### 1.3.4 Connection Migration

**TCP Connection:**
```
Device: WiFi (IP: 192.168.1.100)
   │
   │  Established Connection
   │  ═══════════════════════════>  Server
   │
   ▼ [Switch to 4G]
   
Device: 4G (IP: 10.0.0.50)
   │
   │  ✗ Connection LOST
   │  Must reconnect from scratch
   │  New TCP handshake required
   │  ═══════════════════════════>  Server
```

**QUIC Connection:**
```
Device: WiFi (IP: 192.168.1.100)
   │  Connection ID: 0xABCD1234
   │  ═══════════════════════════>  Server
   │
   ▼ [Switch to 4G]
   
Device: 4G (IP: 10.0.0.50)
   │  Same Connection ID: 0xABCD1234
   │  ✓ Connection CONTINUES
   │  PATH_CHALLENGE sent
   │  ═══════════════════════════>  Server
   │  PATH_RESPONSE received
   │  ═══════════════════════════>  Server
   │  [Seamless migration!]
```

### 1.3.5 Loss Detection & Recovery

**QUIC Advantages:**

1. **Precise ACK Information**
```
TCP ACK: "I received up to byte 1000"
QUIC ACK: "I received packets: 1,2,3,5,7,8,9"
         → Missing: 4, 6 (precise!)
```

2. **Separate Packet Number Space**
```
TCP: Retransmissions use same sequence number
     → Ambiguity in RTT measurement

QUIC: Each packet has unique number
      → No retransmission ambiguity
      → Accurate RTT calculation
```

3. **Faster Loss Detection**
```
TCP: Wait for 3 duplicate ACKs or timeout
QUIC: Can detect loss earlier with ACK ranges
```

### 1.3.6 Congestion Control

QUIC implements multiple congestion control algorithms:

1. **NewReno** (default, like TCP NewReno)
2. **CUBIC** (optimized for high-bandwidth)
3. **BBR** (Bottleneck Bandwidth and RTT)
   - Focuses on bandwidth and latency
   - Better for lossy networks
   - Google's innovation

**BBR Advantage:**
```
Traditional: Packet loss = reduce rate
BBR: Measures actual bottleneck bandwidth
     → More accurate congestion detection
     → Better performance on lossy networks
```

## 1.4 HTTP/3 Specific Features

### 1.4.1 HTTP/3 Frames

HTTP/3 uses different frames than HTTP/2:

```
HTTP/2 Frames:               HTTP/3 Frames:
- DATA                       - DATA
- HEADERS                    - HEADERS
- PRIORITY                   (removed - use prioritization)
- RST_STREAM                 - RESET_STREAM
- SETTINGS                   - SETTINGS
- PUSH_PROMISE               - PUSH_PROMISE
- PING                       (use QUIC PING)
- GOAWAY                     - GOAWAY
- WINDOW_UPDATE              (use QUIC flow control)
- CONTINUATION              (not needed)

New in HTTP/3:
- CANCEL_PUSH
- MAX_PUSH_ID
```

### 1.4.2 QPACK Header Compression

HTTP/3 uses **QPACK** instead of HPACK:

**Why QPACK?**
- HPACK required ordered delivery (TCP)
- QUIC streams are independent
- QPACK allows out-of-order delivery

**QPACK Features:**
```
1. Dynamic Table Updates on separate stream
2. Header blocks can be decoded independently
3. Blocked insertions don't block all streams
4. Better for lossy networks
```

**QPACK Streams:**
```
┌─────────────────────────────────────┐
│ Stream 0: Control Stream            │
│  - Settings                         │
│  - Max table size                   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Stream 2: QPACK Encoder Stream      │
│  - Dynamic table insertions         │
│  - Table size updates               │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Stream 3: QPACK Decoder Stream      │
│  - Acknowledgments                  │
│  - Known received count             │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Request Streams (4, 8, 12, ...)     │
│  - Encoded headers using QPACK      │
│  - Can reference dynamic table      │
└─────────────────────────────────────┘
```

### 1.4.3 Server Push in HTTP/3

HTTP/3 maintains server push but with improvements:

```
Client                              Server
   │                                   │
   │  SETTINGS (ENABLE_PUSH=1)         │
   │──────────────────────────────────>│
   │                                   │
   │  Request: GET /index.html         │
   │──────────────────────────────────>│
   │                                   │
   │  PUSH_PROMISE (Push ID: 0)        │
   │    → /style.css                   │
   │<──────────────────────────────────│
   │                                   │
   │  Response: index.html             │
   │<──────────────────────────────────│
   │                                   │
   │  Pushed: style.css (Push ID: 0)   │
   │<──────────────────────────────────│
   │                                   │
   │  [Client can CANCEL_PUSH if       │
   │   already cached]                 │
```

## 1.5 HTTP/2 vs HTTP/3 Comparison

### Detailed Comparison Table

| Feature | HTTP/2 | HTTP/3 | Winner |
|---------|--------|--------|--------|
| **Transport** | TCP | QUIC/UDP | HTTP/3 |
| **Encryption** | Optional (TLS) | Mandatory (TLS 1.3) | HTTP/3 |
| **Connection Setup** | 2-3 RTT | 0-1 RTT | HTTP/3 ✓ |
| **HOL Blocking** | TCP layer only | None | HTTP/3 ✓ |
| **Multiplexing** | Yes | Yes | Tie |
| **Header Compression** | HPACK | QPACK | HTTP/3 ✓ |
| **Connection Migration** | No | Yes | HTTP/3 ✓ |
| **Packet Loss Impact** | All streams | Single stream | HTTP/3 ✓ |
| **Mobile Performance** | Poor | Excellent | HTTP/3 ✓ |
| **Middlebox Compatibility** | Good | Variable | HTTP/2 |
| **Server Support** | Widespread | Growing | HTTP/2 |
| **Browser Support** | Universal | Modern only | HTTP/2 |
| **CPU Overhead** | Lower | Higher | HTTP/2 |

### Performance Improvements

```
Connection Establishment:
HTTP/2: ████████████ (3 RTT)
HTTP/3: ████ (1 RTT)
        ▼ (0 RTT on resume)

Packet Loss (1% loss rate):
HTTP/2: ██████████████████████ (20% slowdown)
HTTP/3: ████████████ (5% slowdown)

Mobile Network Switching:
HTTP/2: ████████████████ (new connection)
HTTP/3: ██ (seamless migration)

High Latency Networks (200ms RTT):
HTTP/2: ████████████████████████
HTTP/3: ████████████
        ↑ 2x faster!
```

## 1.6 Use Cases & Benefits

### When HTTP/3 Shines ✨

1. **Mobile Networks**
   - Frequent IP changes
   - High packet loss
   - Variable latency
   - Connection migration critical

2. **High-Latency Connections**
   - Satellite
   - International connections
   - 0-RTT helps significantly

3. **Lossy Networks**
   - WiFi with interference
   - Congested networks
   - Independent streams help

4. **Real-Time Applications**
   - Video streaming
   - Gaming
   - Live broadcasts
   - VoIP

5. **API Services**
   - Mobile apps
   - IoT devices
   - Microservices
   - Connection reuse important

### Challenges & Considerations ⚠️

1. **UDP Blocking**
   - Some networks block UDP
   - Fallback to HTTP/2 needed
   - 3-5% of networks problematic

2. **CPU Overhead**
   - QUIC processing more intensive
   - User-space implementation
   - More CPU than kernel TCP

3. **Middlebox Issues**
   - NATs with UDP timeout
   - Firewalls blocking QUIC
   - Load balancers learning curve

4. **Deployment Complexity**
   - New protocol stack
   - Different configuration
   - Monitoring tools immature

5. **Browser Support**
   - Modern browsers only
   - No IE support
   - Requires fallback

## 1.7 QUIC Versions

### QUIC Evolution

```
gQUIC (Google QUIC)
    ↓
IETF QUIC (Draft versions)
    ↓
QUIC v1 (RFC 9000) ⭐ Current
    ↓
QUIC v2 (RFC 9369) Future enhancements
```

**Key IETF Changes from gQUIC:**
- Removed built-in flow control (use QUIC's)
- Standardized crypto (TLS 1.3)
- Better extensibility
- Cleaner separation of layers

---

# BAB 2: DIAGRAM PERCOBAAN

## 2.1 Arsitektur Lab Environment

```
┌───────────────────────────────────────────────────────────────────┐
│                    HTTP/3 LAB ENVIRONMENT                         │
│                                                                   │
│  ┌────────────────┐        ┌────────────────┐                   │
│  │    Browser     │        │   Command      │                   │
│  │    Client      │        │   Line Tools   │                   │
│  │                │        │                │                   │
│  │  - Chrome 90+  │        │  - curl 7.88+  │                   │
│  │  - Firefox 88+ │        │  - quiche      │                   │
│  │  - Edge 91+    │        │  - aioquic     │                   │
│  └────────┬───────┘        └────────┬───────┘                   │
│           │                         │                            │
│           │  HTTPS/3 (UDP)          │  QUIC                      │
│           │  Port 443 (UDP)         │                            │
│           │                         │                            │
│  ┌────────┴─────────────────────────┴────────┐                  │
│  │                                            │                  │
│  │         Four Test Servers                  │                  │
│  │                                            │                  │
│  │  ┌──────────────────────────────────────┐ │                  │
│  │  │  HTTP/2 Server (Baseline)            │ │                  │
│  │  │  Port: 8443 (TCP)                    │ │                  │
│  │  │  Protocol: h2                        │ │                  │
│  │  │  Purpose: Performance comparison     │ │                  │
│  │  └──────────────────────────────────────┘ │                  │
│  │                                            │                  │
│  │  ┌──────────────────────────────────────┐ │                  │
│  │  │  HTTP/3 Server (Basic)               │ │                  │
│  │  │  Port: 4433 (UDP)                    │ │                  │
│  │  │  Protocol: h3                        │ │                  │
│  │  │  Implementation: aioquic             │ │                  │
│  │  │  Features: Basic QUIC + HTTP/3       │ │                  │
│  │  └──────────────────────────────────────┘ │                  │
│  │                                            │                  │
│  │  ┌──────────────────────────────────────┐ │                  │
│  │  │  HTTP/3 Server (0-RTT)               │ │                  │
│  │  │  Port: 4434 (UDP)                    │ │                  │
│  │  │  Protocol: h3                        │ │                  │
│  │  │  Features: 0-RTT resumption enabled  │ │                  │
│  │  │  Session tickets: Yes                │ │                  │
│  │  └──────────────────────────────────────┘ │                  │
│  │                                            │                  │
│  │  ┌──────────────────────────────────────┐ │                  │
│  │  │  HTTP/3 Server (Migration Test)      │ │                  │
│  │  │  Port: 4435 (UDP)                    │ │                  │
│  │  │  Protocol: h3                        │ │                  │
│  │  │  Features: Connection migration      │ │                  │
│  │  │  Multi-path support                  │ │                  │
│  │  └──────────────────────────────────────┘ │                  │
│  │                                            │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Static Content                                 │ │
│  │  - index.html (HTTP/3 demo page)                           │ │
│  │  - style.css                                               │ │
│  │  - app.js (HTTP/3 client code)                            │ │
│  │  - test-files/ (various sizes for testing)                │ │
│  │      - small.txt (1KB)                                     │ │
│  │      - medium.jpg (100KB)                                  │ │
│  │      - large.mp4 (1MB)                                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Network Simulation Tools                       │ │
│  │  - tc (traffic control) - simulate packet loss             │ │
│  │  - iptables - firewall rules                               │ │
│  │  - tcpdump / wireshark - packet capture                    │ │
│  └────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

## 2.2 Connection Flow Diagrams

### Eksperimen 1: 0-RTT vs 1-RTT vs 3-RTT

```
Traditional TCP+TLS (HTTP/2):
═══════════════════════════════

Client                                          Server
  │                                               │
  │ ─────── ① TCP SYN ─────────────────────────> │
  │                                               │  
  │ <────── ② TCP SYN-ACK ──────────────────────  │
  │                                               │
  │ ─────── ③ TCP ACK ─────────────────────────> │
  │ [TCP Connection: 1 RTT]                       │
  │                                               │
  │ ─────── ④ TLS ClientHello ─────────────────> │
  │                                               │
  │ <────── ⑤ TLS ServerHello + Cert ──────────  │
  │                                               │
  │ ─────── ⑥ TLS Finished ────────────────────> │
  │ [TLS Handshake: +2 RTT]                       │
  │                                               │
  │ ─────── ⑦ HTTP Request ────────────────────> │
  │ [FIRST DATA: 3 RTT] ⏱️                        │
  │                                               │
  │ <────── ⑧ HTTP Response ─────────────────────│
  │                                               │

Total: 3 Round Trip Times


QUIC 1-RTT (HTTP/3 First Connection):
═════════════════════════════════════

Client                                          Server
  │                                               │
  │ ─────── ① Initial Packet ──────────────────> │
  │         - CRYPTO frame (ClientHello)          │
  │         - HTTP/3 SETTINGS                     │
  │                                               │
  │ <────── ② Initial + Handshake ──────────────  │
  │         - CRYPTO (ServerHello + Cert)         │
  │         - HTTP/3 SETTINGS                     │
  │                                               │
  │ ─────── ③ Handshake Complete ───────────────>│
  │         - CRYPTO (Finished)                   │
  │         - HTTP/3 Request                      │
  │ [FIRST DATA: 1 RTT] ⏱️                        │
  │                                               │
  │ <────── ④ 1-RTT Packet ──────────────────────│
  │         - HTTP/3 Response                     │
  │                                               │

Total: 1 Round Trip Time
Improvement: 66% faster! 🚀


QUIC 0-RTT (HTTP/3 Resumed Connection):
═══════════════════════════════════════

Client                                          Server
  │ [Has cached session ticket]                   │
  │                                               │
  │ ─────── ① 0-RTT Packet ─────────────────────>│
  │         - Encrypted with 0-RTT key            │
  │         - HTTP/3 Request (IMMEDIATELY!)       │
  │ [FIRST DATA: 0 RTT] ⚡                         │
  │                                               │
  │ <────── ② 1-RTT Packet ──────────────────────│
  │         - HTTP/3 Response                     │
  │         - New session ticket                  │
  │                                               │

Total: 0 Round Trip Times for request!
Improvement: Instant! 🚀🚀🚀

Time Comparison:
TCP+TLS:  ████████████ (3 RTT = 300ms @ 100ms RTT)
QUIC 1-RTT: ████ (1 RTT = 100ms @ 100ms RTT)
QUIC 0-RTT: ▌(0 RTT = 0ms for request!)
```

### Eksperimen 2: Head-of-Line Blocking

```
HTTP/2 over TCP (HOL Blocking):
═══════════════════════════════

Stream 1: [P1][P2][P3][P4][P5] → Video frame 1
Stream 2: [P6][P7][P8][P9]     → Critical CSS
Stream 3: [P10][P11]           → JavaScript

TCP Layer: [P1][P2][X][P4][P5][P6][P7][P8][P9][P10][P11]
                     ↑ Packet 3 LOST!
                     
Result:
─────────────────────────────────────────────
Stream 1: BLOCKED ❌ (waiting for P3)
Stream 2: BLOCKED ❌ (waiting for P3)
Stream 3: BLOCKED ❌ (waiting for P3)

ALL streams must wait!
User sees: Loading... 😫


HTTP/3 over QUIC (No HOL Blocking):
═══════════════════════════════════

Stream 1: [P1][P2][P3][P4][P5] → Video frame 1
Stream 2: [P6][P7][P8][P9]     → Critical CSS
Stream 3: [P10][P11]           → JavaScript

QUIC Layer: [P1][P2][X][P4][P5][P6][P7][P8][P9][P10][P11]
                      ↑ Packet 3 LOST!

Result:
─────────────────────────────────────────────
Stream 1: BLOCKED ❌ (only this stream)
Stream 2: CONTINUES ✅ [P6][P7][P8][P9] delivered
Stream 3: CONTINUES ✅ [P10][P11] delivered

Only affected stream waits!
User sees: Page rendering! 😊

Performance Impact (1% packet loss):
HTTP/2: ████████████████████ (95% slowdown)
HTTP/3: ███ (5% slowdown)
```

### Eksperimen 3: Connection Migration

```
Scenario: User switching from WiFi to 4G
═════════════════════════════════════════

HTTP/2 (TCP-based):
───────────────────

Device @ WiFi (192.168.1.100)
   │  Active download: file.mp4 (50% complete)
   │  ════════════════════════════> Server
   │
   ▼ [User leaves WiFi coverage]
   
Device @ 4G (10.5.23.45)
   │  ✗ Connection TERMINATED
   │  ✗ Download ABORTED
   │  ✗ Must reconnect: TCP handshake
   │  ✗ Must restart: from beginning
   │  ════════════════════════════> Server
   │  [New connection, restart download]

Result:
- Connection lost: ~2-3 seconds
- Data wasted: 50% of download lost
- User experience: Frustrating 😤


HTTP/3 (QUIC-based):
────────────────────

Device @ WiFi (192.168.1.100)
   │  Connection ID: 0xABCDEF12
   │  Active download: file.mp4 (50% complete)
   │  ════════════════════════════> Server
   │
   ▼ [User leaves WiFi coverage]
   
Device @ 4G (10.5.23.45)
   │  Same Connection ID: 0xABCDEF12
   │  ✓ Connection MAINTAINED
   │  ✓ Download CONTINUES from 50%
   │  PATH_CHALLENGE ────────────> Server
   │  PATH_RESPONSE <──────────── Server
   │  ════════════════════════════> Server
   │  [Seamless migration, resume download]

Result:
- Migration time: ~100-200ms
- Data preserved: Continue from 50%
- User experience: Seamless! 😊

Timeline Comparison:

HTTP/2 Network Switch:
[Download]───X [Reconnect]═══[Restart Download]═══
   ↑           ↑ 2-3s        ↑ Lost progress
   Loss                      

HTTP/3 Network Switch:
[Download]──✓─[Path Validate]─[Continue Download]══
   ↑           ↑ 100ms        ↑ No data loss
   Seamless
```

### Eksperimen 4: Packet Loss Impact

```
Scenario: Network with 5% Packet Loss
═════════════════════════════════════

HTTP/2 Behavior:
────────────────

Sending 20 packets (P1-P20):
[P1][P2][P3][P4][P5][P6][P7][P8][P9][P10]
[P11][P12][X13][P14][P15][X16][P17][P18][P19][P20]
           ↑                  ↑
        Lost (5%)          Lost (5%)

TCP Layer Response:
─────────────────────
P1-P12: Delivered ✓
P13: Lost → Fast retransmit triggered
      ↓
[BLOCK ALL DELIVERY]
      ↓
Wait for P13 retransmission
      ↓
P13 retransmitted, received
      ↓
P14-P15: Can now deliver
P16: Lost → Fast retransmit triggered
      ↓
[BLOCK ALL DELIVERY AGAIN]
      ↓
Wait for P16 retransmission
      ↓
Continue...

Timeline:
Request ──[Delays]──[Delays]──[Complete]
         100ms    150ms    Time: 250ms+

Average slowdown: 2-3x slower


HTTP/3 Behavior:
────────────────

Sending 20 packets to 3 different streams:
Stream 1: [P1][P2][P3][P4][P5][P6][P7]
Stream 2: [P8][P9][P10][P11][P12][X13]
Stream 3: [P14][P15][X16][P17][P18][P19][P20]

QUIC Layer Response:
───────────────────
Stream 1: All packets delivered ✓
Stream 2: P8-P12 delivered ✓
          P13 lost → Retransmit P13
          [Only Stream 2 waits]
Stream 3: P14-P15 delivered ✓
          P16 lost → Retransmit P16
          P17-P20 delivered ✓
          [Only wait for P16]

Timeline:
Stream 1: ════════════[Complete] ✓ (no delay)
Stream 2: ════[Wait]══[Complete] (minor delay)
Stream 3: ════[Wait]══[Complete] (minor delay)

Average slowdown: 1.1-1.2x slower

Performance Comparison:
HTTP/2 @ 5% loss: ████████████████ (3x slower)
HTTP/3 @ 5% loss: █████ (1.2x slower)
                   ↑ 2.5x better!
```

## 2.3 QUIC Internal Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  QUIC CONNECTION STATE                       │
│                                                              │
│  Connection ID: 0xABCDEF1234567890                          │
│  State: ESTABLISHED                                         │
│  Version: QUIC v1 (RFC 9000)                               │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           CRYPTO STATE (TLS 1.3)                   │    │
│  │  - Encryption Level: 1-RTT                         │    │
│  │  - Cipher Suite: TLS_AES_128_GCM_SHA256           │    │
│  │  - Keys: Client/Server write keys                  │    │
│  │  - Session Ticket: Available for 0-RTT             │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           ACTIVE STREAMS                           │    │
│  │                                                     │    │
│  │  Stream 0 (Control):                               │    │
│  │    State: OPEN                                     │    │
│  │    Purpose: HTTP/3 settings, QPACK                 │    │
│  │                                                     │    │
│  │  Stream 4 (Request):                               │    │
│  │    State: OPEN                                     │    │
│  │    Method: GET /index.html                         │    │
│  │    Bytes Sent: 1024                                │    │
│  │    Bytes Received: 8192                            │    │
│  │                                                     │    │
│  │  Stream 8 (Request):                               │    │
│  │    State: OPEN                                     │    │
│  │    Method: GET /style.css                          │    │
│  │    Bytes Sent: 512                                 │    │
│  │    Bytes Received: 2048                            │    │
│  │                                                     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           FLOW CONTROL                             │    │
│  │  Connection-level:                                 │    │
│  │    Max Data: 1,048,576 bytes                       │    │
│  │    Data Sent: 524,288 bytes                        │    │
│  │    Data Received: 786,432 bytes                    │    │
│  │                                                     │    │
│  │  Stream-level (per stream):                        │    │
│  │    Max Stream Data: 65,536 bytes                   │    │
│  │                                                     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           CONGESTION CONTROL                       │    │
│  │  Algorithm: BBR                                    │    │
│  │  Congestion Window: 32,768 bytes                   │    │
│  │  RTT: 45ms (smoothed)                              │    │
│  │  RTT Variance: 5ms                                 │    │
│  │  Bandwidth: 10 Mbps (estimated)                    │    │
│  │  In Flight: 16,384 bytes                           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           LOSS DETECTION                           │    │
│  │  Largest Acked Packet: 142                         │    │
│  │  Loss Detection Timer: 90ms                        │    │
│  │  Lost Packets: [137, 139]                          │    │
│  │  Retransmission Queue: 2 packets                   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │           PATH VALIDATION                          │    │
│  │  Local Address: 192.168.1.100:54321                │    │
│  │  Remote Address: 93.184.216.34:443                 │    │
│  │  Path Status: VALIDATED                            │    │
│  │  Alternative Paths: None                           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 2.4 Data Flow Visualization

```
Complete HTTP/3 Request/Response Flow:
══════════════════════════════════════

① Application Layer (HTTP/3):
   ┌─────────────────────────────────┐
   │ GET /api/data HTTP/3            │
   │ Host: example.com               │
   │ Accept: application/json        │
   └─────────────────────────────────┘
                ↓

② HTTP/3 Framing:
   ┌─────────────────────────────────┐
   │ HEADERS Frame:                  │
   │   Stream ID: 4                  │
   │   :method = GET                 │
   │   :path = /api/data             │
   │   :authority = example.com      │
   │                                 │
   │ [Encoded with QPACK]            │
   └─────────────────────────────────┘
                ↓

③ QUIC Stream Layer:
   ┌─────────────────────────────────┐
   │ STREAM Frame:                   │
   │   Stream ID: 4                  │
   │   Offset: 0                     │
   │   Length: 156                   │
   │   FIN: 1                        │
   │   Data: [HEADERS frame data]    │
   └─────────────────────────────────┘
                ↓

④ QUIC Packet Layer:
   ┌─────────────────────────────────┐
   │ Short Header Packet:            │
   │   Header Form: 0 (short)        │
   │   Connection ID: 0xABCD...      │
   │   Packet Number: 143            │
   │   Protected Payload:            │
   │     [STREAM frame]              │
   │     [Encrypted]                 │
   └─────────────────────────────────┘
                ↓

⑤ UDP Datagram:
   ┌─────────────────────────────────┐
   │ UDP Header:                     │
   │   Source Port: 54321            │
   │   Dest Port: 443                │
   │   Length: 1200                  │
   │   Checksum: 0x4A2F              │
   │                                 │
   │ Payload: [QUIC packet]          │
   └─────────────────────────────────┘
                ↓

⑥ IP Packet:
   ┌─────────────────────────────────┐
   │ IP Header:                      │
   │   Source: 192.168.1.100         │
   │   Dest: 93.184.216.34           │
   │   Protocol: UDP (17)            │
   │                                 │
   │ Payload: [UDP datagram]         │
   └─────────────────────────────────┘
                ↓
           [Network]
                ↓
    Server processes and responds...
```

---

# BAB 3: PERSIAPAN ENVIRONMENT

## 3.1 Kebutuhan Sistem

### Hardware Requirements

| Component | Minimum | Recommended | Purpose |
|-----------|---------|-------------|---------|
| CPU | Dual-core 2GHz | Quad-core 2.5GHz+ | QUIC crypto operations |
| RAM | 4GB | 8GB+ | Multiple servers + testing |
| Disk | 1GB free | 5GB+ free | Logs, captures, test files |
| Network | 10 Mbps | 100 Mbps+ | Bandwidth testing |

**Note:** HTTP/3 is more CPU-intensive than HTTP/2 due to user-space implementation and encryption overhead.

### Software Requirements

| Software | Minimum Version | Recommended | Purpose |
|----------|----------------|-------------|---------|
| **Python** | 3.8 | 3.10+ | aioquic server |
| **pip** | 20.0 | Latest | Package management |
| **OpenSSL** | 1.1.1 | 3.0+ | Crypto operations |
| **Node.js** | 16.0 | 20.x LTS | HTTP/2 baseline server |
| **curl** | 7.88+ | Latest | HTTP/3 client support |
| **Chrome** | 90+ | Latest | H3 browser support |
| **Firefox** | 88+ | Latest | H3 browser support |

### Optional Tools

| Tool | Purpose | Installation |
|------|---------|--------------|
| **Wireshark** | Packet analysis | `sudo apt install wireshark` |
| **tcpdump** | Packet capture | `sudo apt install tcpdump` |
| **qlog viewer** | QUIC logs visualization | https://qvis.quictools.info/ |
| **h3spec** | HTTP/3 conformance | https://github.com/kazu-yamamoto/h3spec |
| **quiche-client** | Cloudflare QUIC client | Build from source |

## 3.2 Verifikasi Sistem

### Check Python & pip

```bash
# Check Python version
python3 --version
# Expected: Python 3.8.0 or higher

# Check pip
pip3 --version
# Expected: pip 20.0 or higher

# Test Python SSL support
python3 << 'EOF'
import ssl
print(f"SSL version: {ssl.OPENSSL_VERSION}")
print(f"TLS 1.3 support: {ssl.HAS_TLSv1_3}")
EOF
# Should show OpenSSL 1.1.1+ and TLS 1.3: True
```

### Check curl HTTP/3 Support

```bash
# Check curl version and features
curl --version

# Expected output should include:
# - Version 7.88 or higher
# - Features: ... HTTP3 ...

# If HTTP3 not listed, need to build curl with HTTP/3 support
# or install from repository with HTTP/3

# Ubuntu 23.04+:
# sudo apt install curl

# Manual build (if needed):
# See instructions in installation section
```

### Check Browser Support

**Chrome:**
```
1. Open chrome://version
2. Check version: 90 or higher
3. Go to chrome://flags
4. Search: "quic"
5. Verify: "Experimental QUIC protocol" enabled (default)
```

**Firefox:**
```
1. Open about:config
2. Search: network.http.http3.enable
3. Verify: true (default in Firefox 88+)
```

### Check UDP Connectivity

```bash
# Test if UDP port 443 is accessible
# Install nc (netcat) if needed
sudo apt install netcat-openbsd

# Test UDP port (will fail but shows if blocked)
nc -u -v 8.8.8.8 443
# Should connect (even if no response)

# Check local firewall
sudo iptables -L -n | grep 443
# Should not block UDP 443

# Check if UDP is working
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(('0.0.0.0', 4433))
print('UDP socket OK')
s.close()
"
```

## 3.3 Platform-Specific Setup

### Linux (Ubuntu/Debian)

```bash
# Update package lists
sudo apt update

# Install Python and dev tools
sudo apt install -y python3 python3-pip python3-dev

# Install OpenSSL development files
sudo apt install -y libssl-dev

# Install Node.js (for HTTP/2 baseline)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install build tools (needed for some Python packages)
sudo apt install -y build-essential

# Install network tools
sudo apt install -y tcpdump wireshark

# Allow non-root packet capture
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/tcpdump

# Install curl with HTTP/3 (Ubuntu 23.04+)
sudo apt install -y curl

# Verify installations
python3 --version
node --version
curl --version | grep HTTP3
```

### macOS

```bash
# Install Homebrew if not installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Python
brew install python@3.10

# Install Node.js
brew install node

# Install OpenSSL
brew install openssl@3

# Install curl with HTTP/3 support
brew install curl

# Verify HTTP/3 support
/opt/homebrew/opt/curl/bin/curl --version | grep HTTP3

# Install Wireshark (optional)
brew install --cask wireshark

# Verify installations
python3 --version
node --version
```

### Windows (WSL2 Recommended)

```powershell
# Enable WSL2
wsl --install

# Install Ubuntu from Microsoft Store
# Then follow Linux instructions inside WSL

# Alternative: Native Windows setup
# Install Python from python.org
# Install Node.js from nodejs.org
# Use Windows Terminal for commands
```

## 3.4 Struktur Direktori

```
http3-lab/
├── README.md                    # Main documentation
├── requirements.txt             # Python dependencies
├── package.json                 # Node.js dependencies
├── .gitignore
├── certs/
│   ├── cert.pem                # SSL certificate
│   ├── key.pem                 # Private key
│   └── generate-certs.sh       # Certificate generation script
├── servers/
│   ├── http2-baseline.js       # HTTP/2 server (comparison)
│   ├── http3-basic.py          # Basic HTTP/3 server
│   ├── http3-0rtt.py          # 0-RTT enabled server
│   └── http3-migration.py      # Connection migration server
├── clients/
│   ├── test-connection.py      # Basic connectivity test
│   ├── test-0rtt.py           # 0-RTT resumption test
│   ├── test-migration.py       # Connection migration test
│   ├── benchmark.py            # Performance comparison
│   └── packet-loss-test.py     # Packet loss simulation
├── public/
│   ├── index.html              # Demo page
│   ├── style.css               # Styling
│   ├── app.js                  # Client-side JavaScript
│   └── test-files/             # Test files of various sizes
│       ├── small.txt           # 1KB
│       ├── medium.jpg          # 100KB
│       └── large.mp4           # 1MB
├── scripts/
│   ├── setup.sh                # Complete setup script
│   ├── simulate-packet-loss.sh # Network simulation
│   ├── start-all-servers.sh    # Start all servers
│   └── capture-packets.sh      # Packet capture helper
├── logs/
│   ├── .gitkeep
│   └── qlog/                   # QLOG files for analysis
├── results/
│   └── .gitkeep                # Experiment results
└── docs/
    ├── theory.md               # Extended theory
    ├── experiments.md          # Experiment guides
    └── troubleshooting.md      # Problem solving
```

## 3.5 Network Configuration

### Firewall Configuration

**Linux (UFW):**
```bash
# Allow HTTP/3 (UDP 443, 4433-4435)
sudo ufw allow 4433/udp
sudo ufw allow 4434/udp
sudo ufw allow 4435/udp

# For production (UDP 443)
sudo ufw allow 443/udp

# Allow HTTP/2 baseline (TCP 8443)
sudo ufw allow 8443/tcp

# Check status
sudo ufw status
```

**iptables (Advanced):**
```bash
# Allow incoming UDP on QUIC ports
sudo iptables -A INPUT -p udp --dport 4433:4435 -j ACCEPT

# Allow outgoing UDP
sudo iptables -A OUTPUT -p udp --sport 4433:4435 -j ACCEPT

# Save rules
sudo netfilter-persistent save
```

### Network Simulation Setup

For realistic testing, we'll simulate various network conditions:

```bash
# Install traffic control tools
sudo apt install iproute2

# Create script for packet loss simulation
cat > scripts/simulate-packet-loss.sh << 'EOF'
#!/bin/bash
# Simulate packet loss on loopback interface

INTERFACE="lo"
LOSS_RATE="${1:-5}"  # Default 5% loss

echo "Simulating ${LOSS_RATE}% packet loss on ${INTERFACE}"

# Add packet loss
sudo tc qdisc add dev ${INTERFACE} root netem loss ${LOSS_RATE}%

echo "Packet loss simulation active"
echo "To remove: sudo tc qdisc del dev ${INTERFACE} root"
EOF

chmod +x scripts/simulate-packet-loss.sh

# Create script for latency simulation
cat > scripts/simulate-latency.sh << 'EOF'
#!/bin/bash
# Simulate network latency

INTERFACE="lo"
LATENCY="${1:-100}"  # Default 100ms

echo "Adding ${LATENCY}ms latency to ${INTERFACE}"

sudo tc qdisc add dev ${INTERFACE} root netem delay ${LATENCY}ms

echo "Latency simulation active"
echo "To remove: sudo tc qdisc del dev ${INTERFACE} root"
EOF

chmod +x scripts/simulate-latency.sh
```

### Loopback Configuration

```bash
# Verify loopback is up
ip addr show lo

# Should show:
# 1: lo: <LOOPBACK,UP,LOWER_UP> ...
#     inet 127.0.0.1/8 ...
#     inet6 ::1/128 ...

# Increase UDP buffer sizes for better QUIC performance
sudo sysctl -w net.core.rmem_max=2500000
sudo sysctl -w net.core.wmem_max=2500000

# Make permanent
echo "net.core.rmem_max=2500000" | sudo tee -a /etc/sysctl.conf
echo "net.core.wmem_max=2500000" | sudo tee -a /etc/sysctl.conf
```

## 3.6 SSL/TLS Certificates

### Generate Certificates

```bash
# Create certs directory
mkdir -p certs
cd certs

# Generate certificate generation script
cat > generate-certs.sh << 'EOF'
#!/bin/bash
# Generate self-signed certificates for HTTP/3 testing

echo "Generating SSL certificates for HTTP/3 lab..."

# Generate private key
openssl genrsa -out key.pem 4096

# Generate certificate
openssl req -new -x509 -key key.pem -out cert.pem -days 365 \
  -subj "/C=ID/ST=East Java/L=Surabaya/O=HTTP3 Lab/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,DNS:*.localhost,IP:127.0.0.1,IP:::1"

# Verify certificate
echo ""
echo "Certificate generated successfully!"
echo "Verifying certificate..."
openssl x509 -in cert.pem -text -noout | grep -A2 "Subject:"

echo ""
echo "Files created:"
ls -lh key.pem cert.pem
EOF

chmod +x generate-certs.sh

# Run generation
./generate-certs.sh

cd ..
```

**Important Notes:**
- HTTP/3 **requires** TLS 1.3
- Self-signed certificates will show browser warnings
- For production, use Let's Encrypt or proper CA

## 3.7 Security Considerations

### Certificate Trust (Optional)

**For Local Testing:**
```bash
# Chrome: Type "thisisunsafe" on warning page
# Or import certificate:
# Settings → Privacy → Security → Manage Certificates
# Import cert.pem as Trusted Root

# Firefox: Accept risk on warning page
# Or Settings → Privacy → Certificates → View Certificates
# Import cert.pem
```

### UDP Security

HTTP/3/QUIC has built-in protection against:
- ✅ Amplification attacks (validated addresses)
- ✅ Replay attacks (packet numbers)
- ✅ Connection hijacking (Connection IDs)
- ✅ Eavesdropping (mandatory encryption)

However, be aware:
- ⚠️ UDP can be blocked by firewalls
- ⚠️ Some networks rate-limit UDP
- ⚠️ DDoS protection may need tuning

---

# BAB 4: LANGKAH-LANGKAH INSTALASI

## 4.1 Initial Setup

```bash
# Create project directory
mkdir http3-lab
cd http3-lab

# Initialize git (optional)
git init

# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
node_modules/
*.pem
*.log
*.qlog
.DS_Store
.env
results/*.png
results/*.har
logs/*.pcap
EOF

# Create directory structure
mkdir -p servers clients public public/test-files scripts logs logs/qlog results docs certs
```

## 4.2 Install Python Dependencies

### Create requirements.txt

```bash
cat > requirements.txt << 'EOF'
# HTTP/3 Server Implementation
aioquic==0.9.21

# Additional useful packages
cryptography>=41.0.0
pylsqpack>=0.3.17
certifi>=2023.7.22

# Testing and benchmarking
aiofiles>=23.2.1
asyncio>=3.4.3

# Optional: For advanced testing
# pyshark>=0.6  # Packet analysis
# scapy>=2.5.0  # Packet crafting
EOF

# Install Python packages
pip3 install -r requirements.txt

# Verify installation
python3 << 'EOF'
import aioquic
print(f"aioquic version: {aioquic.__version__}")
from aioquic.h3.connection import H3Connection
print("HTTP/3 support: OK")
EOF
```

## 4.3 Install Node.js Dependencies

```bash
# Create package.json
cat > package.json << 'EOF'
{
  "name": "http3-lab",
  "version": "1.0.0",
  "description": "HTTP/3 and QUIC Hands-on Laboratory",
  "main": "servers/http2-baseline.js",
  "scripts": {
    "http2": "node servers/http2-baseline.js",
    "http3-basic": "python3 servers/http3-basic.py",
    "http3-0rtt": "python3 servers/http3-0rtt.py",
    "http3-migration": "python3 servers/http3-migration.py",
    "test": "python3 clients/test-connection.py",
    "benchmark": "python3 clients/benchmark.py"
  },
  "keywords": ["http3", "quic", "performance", "web"],
  "author": "",
  "license": "MIT"
}
EOF

# No npm packages needed for this lab
# (HTTP/2 server uses only Node.js built-ins)
```

## 4.4 Generate SSL Certificates

```bash
cd certs
./generate-certs.sh
cd ..

# Verify certificate files exist
ls -lh certs/

# Should show:
# cert.pem  (certificate)
# key.pem   (private key)
```

## 4.5 Create Server Files

### 4.5.1 HTTP/2 Baseline Server

```bash
cat > servers/http2-baseline.js << 'EOFILE'
const http2 = require('http2');
const fs = require('fs');
const path = require('path');

const PORT = 8443;

const server = http2.createSecureServer({
  key: fs.readFileSync('certs/key.pem'),
  cert: fs.readFileSync('certs/cert.pem')
});

const mimeTypes = {
  '.html': 'text/html',
  '.css': 'text/css',
  '.js': 'application/javascript',
  '.json': 'application/json',
  '.txt': 'text/plain',
  '.jpg': 'image/jpeg',
  '.png': 'image/png',
  '.mp4': 'video/mp4'
};

server.on('stream', (stream, headers) => {
  const method = headers[':method'];
  let pathname = headers[':path'];
  
  console.log(`[HTTP/2] ${method} ${pathname}`);

  if (pathname === '/') {
    pathname = '/index.html';
  }

  const filePath = path.join(__dirname, '..', 'public', pathname);
  const ext = path.extname(filePath);
  const contentType = mimeTypes[ext] || 'application/octet-stream';

  fs.stat(filePath, (err, stat) => {
    if (err) {
      stream.respond({ ':status': 404 });
      stream.end('Not Found');
      return;
    }

    stream.respondWithFile(filePath, {
      'content-type': contentType,
      ':status': 200
    });
  });
});

server.listen(PORT, () => {
  console.log(`HTTP/2 Baseline Server running on https://localhost:${PORT}`);
  console.log('Protocol: h2 (for comparison with HTTP/3)');
});
EOFILE
```

### 4.5.2 Basic HTTP/3 Server

```bash
cat > servers/http3-basic.py << 'EOFILE'
#!/usr/bin/env python3
"""
Basic HTTP/3 Server using aioquic
Demonstrates fundamental HTTP/3 functionality
"""

import asyncio
import os
from pathlib import Path
from typing import Dict, Optional

from aioquic.asyncio import QuicConnectionProtocol, serve
from aioquic.h3.connection import H3_ALPN, H3Connection
from aioquic.h3.events import (
    DataReceived,
    H3Event,
    HeadersReceived,
    WebTransportStreamDataReceived,
)
from aioquic.quic.configuration import QuicConfiguration
from aioquic.quic.events import QuicEvent


class HttpServerProtocol(QuicConnectionProtocol):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._http = H3Connection(self._quic)
        self._request_handlers = {}

    def quic_event_received(self, event: QuicEvent) -> None:
        # Handle QUIC events and convert to HTTP/3 events
        for http_event in self._http.handle_event(event):
            self.http_event_received(http_event)

    def http_event_received(self, event: H3Event) -> None:
        if isinstance(event, HeadersReceived):
            # New request received
            headers = {}
            for name, value in event.headers:
                headers[name.decode()] = value.decode()
            
            method = headers.get(':method', 'GET')
            path = headers.get(':path', '/')
            
            print(f"[HTTP/3] {method} {path}")
            
            # Handle request
            asyncio.ensure_future(
                self.handle_request(event.stream_id, headers)
            )

        elif isinstance(event, DataReceived):
            # Request body data (if any)
            pass

    async def handle_request(self, stream_id: int, headers: Dict[str, str]):
        """Handle HTTP request"""
        path = headers.get(':path', '/')
        
        # Default to index.html
        if path == '/':
            path = '/index.html'
        
        # Build file path
        public_dir = Path(__file__).parent.parent / 'public'
        file_path = (public_dir / path.lstrip('/')).resolve()
        
        # Security check: ensure file is within public directory
        try:
            file_path.relative_to(public_dir)
        except ValueError:
            self.send_response(stream_id, 403, b'Forbidden')
            return
        
        # Check if file exists
        if not file_path.exists():
            self.send_response(stream_id, 404, b'Not Found')
            return
        
        # Determine content type
        content_type = self.get_content_type(file_path)
        
        # Read and send file
        try:
            with open(file_path, 'rb') as f:
                data = f.read()
            
            self.send_response(
                stream_id,
                200,
                data,
                content_type=content_type
            )
        except Exception as e:
            print(f"Error reading file: {e}")
            self.send_response(stream_id, 500, b'Internal Server Error')

    def send_response(
        self,
        stream_id: int,
        status: int,
        data: bytes,
        content_type: str = 'text/plain'
    ):
        """Send HTTP response"""
        headers = [
            (b':status', str(status).encode()),
            (b'content-type', content_type.encode()),
            (b'content-length', str(len(data)).encode()),
            (b'server', b'aioquic-http3'),
        ]
        
        # Send headers
        self._http.send_headers(stream_id, headers)
        
        # Send data
        self._http.send_data(stream_id, data, end_stream=True)
        
        # Transmit
        self.transmit()

    @staticmethod
    def get_content_type(file_path: Path) -> str:
        """Determine content type from file extension"""
        ext = file_path.suffix.lower()
        types = {
            '.html': 'text/html',
            '.css': 'text/css',
            '.js': 'application/javascript',
            '.json': 'application/json',
            '.txt': 'text/plain',
            '.jpg': 'image/jpeg',
            '.jpeg': 'image/jpeg',
            '.png': 'image/png',
            '.gif': 'image/gif',
            '.mp4': 'video/mp4',
        }
        return types.get(ext, 'application/octet-stream')


async def main():
    """Start HTTP/3 server"""
    # Configuration
    configuration = QuicConfiguration(
        alpn_protocols=H3_ALPN,
        is_client=False,
        max_datagram_frame_size=65536,
    )
    
    # Load certificate and key
    cert_path = Path(__file__).parent.parent / 'certs' / 'cert.pem'
    key_path = Path(__file__).parent.parent / 'certs' / 'key.pem'
    
    configuration.load_cert_chain(cert_path, key_path)
    
    # Start server
    await serve(
        host='localhost',
        port=4433,
        configuration=configuration,
        create_protocol=HttpServerProtocol,
    )
    
    print("HTTP/3 Basic Server running on https://localhost:4433")
    print("Protocol: h3 (QUIC + HTTP/3)")
    print("Features: Multiplexing, 0-HOL blocking, QPACK")
    print("Press Ctrl+C to stop")
    
    # Run forever
    await asyncio.Future()


if __name__ == '__main__':
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\nServer stopped")
EOFILE

chmod +x servers/http3-basic.py
```

### 4.5.3 HTTP/3 Server with 0-RTT

```bash
cat > servers/http3-0rtt.py << 'EOFILE'
#!/usr/bin/env python3
"""
HTTP/3 Server with 0-RTT (Zero Round Trip Time) resumption
Demonstrates session resumption and early data
"""

import asyncio
import os
from pathlib import Path
from typing import Dict

from aioquic.asyncio import QuicConnectionProtocol, serve
from aioquic.h3.connection import H3_ALPN, H3Connection
from aioquic.h3.events import H3Event, HeadersReceived
from aioquic.quic.configuration import QuicConfiguration
from aioquic.quic.events import QuicEvent
from aioquic.tls import SessionTicket


# Store session tickets for 0-RTT
session_ticket_store = {}


class Http0RTTProtocol(QuicConnectionProtocol):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._http = H3Connection(self._quic)
        
    def quic_event_received(self, event: QuicEvent) -> None:
        # Pass to HTTP/3 handler
        for http_event in self._http.handle_event(event):
            self.http_event_received(http_event)
    
    def http_event_received(self, event: H3Event) -> None:
        if isinstance(event, HeadersReceived):
            headers = {}
            for name, value in event.headers:
                headers[name.decode()] = value.decode()
            
            method = headers.get(':method', 'GET')
            path = headers.get(':path', '/')
            
            # Check if this is 0-RTT data
            if self._quic.early_data_accepted:
                print(f"[HTTP/3 0-RTT] {method} {path} ⚡ (Early Data)")
            else:
                print(f"[HTTP/3 1-RTT] {method} {path}")
            
            asyncio.ensure_future(
                self.handle_request(event.stream_id, headers)
            )
    
    async def handle_request(self, stream_id: int, headers: Dict[str, str]):
        """Handle HTTP request"""
        path = headers.get(':path', '/')
        
        if path == '/':
            path = '/index.html'
        
        public_dir = Path(__file__).parent.parent / 'public'
        file_path = (public_dir / path.lstrip('/')).resolve()
        
        try:
            file_path.relative_to(public_dir)
        except ValueError:
            self.send_response(stream_id, 403, b'Forbidden')
            return
        
        if not file_path.exists():
            self.send_response(stream_id, 404, b'Not Found')
            return
        
        content_type = self.get_content_type(file_path)
        
        try:
            with open(file_path, 'rb') as f:
                data = f.read()
            
            self.send_response(stream_id, 200, data, content_type)
        except Exception as e:
            print(f"Error: {e}")
            self.send_response(stream_id, 500, b'Internal Server Error')
    
    def send_response(
        self,
        stream_id: int,
        status: int,
        data: bytes,
        content_type: str = 'text/plain'
    ):
        """Send HTTP response"""
        headers = [
            (b':status', str(status).encode()),
            (b'content-type', content_type.encode()),
            (b'content-length', str(len(data)).encode()),
            (b'server', b'aioquic-http3-0rtt'),
            (b'alt-svc', b'h3=":4434"; ma=3600'),  # Advertise 0-RTT capability
        ]
        
        self._http.send_headers(stream_id, headers)
        self._http.send_data(stream_id, data, end_stream=True)
        self.transmit()
    
    @staticmethod
    def get_content_type(file_path: Path) -> str:
        """Get content type from extension"""
        ext = file_path.suffix.lower()
        types = {
            '.html': 'text/html',
            '.css': 'text/css',
            '.js': 'application/javascript',
            '.json': 'application/json',
            '.txt': 'text/plain',
            '.jpg': 'image/jpeg',
            '.png': 'image/png',
            '.mp4': 'video/mp4',
        }
        return types.get(ext, 'application/octet-stream')


def save_session_ticket(ticket: SessionTicket) -> None:
    """
    Save session ticket for 0-RTT resumption
    In production, store in Redis/database
    """
    session_ticket_store[ticket.ticket] = ticket
    print(f"[0-RTT] Session ticket saved (ID: {ticket.ticket[:16].hex()}...)")


async def main():
    """Start HTTP/3 server with 0-RTT"""
    configuration = QuicConfiguration(
        alpn_protocols=H3_ALPN,
        is_client=False,
        max_datagram_frame_size=65536,
    )
    
    # Load certificates
    cert_path = Path(__file__).parent.parent / 'certs' / 'cert.pem'
    key_path = Path(__file__).parent.parent / 'certs' / 'key.pem'
    configuration.load_cert_chain(cert_path, key_path)
    
    # Enable 0-RTT
    configuration.max_early_data = 0xFFFFFFFF  # Allow unlimited early data
    
    # Set session ticket handler
    configuration.session_ticket_handler = save_session_ticket
    
    await serve(
        host='localhost',
        port=4434,
        configuration=configuration,
        create_protocol=Http0RTTProtocol,
    )
    
    print("HTTP/3 Server with 0-RTT running on https://localhost:4434")
    print("Protocol: h3 + 0-RTT")
    print("Features: Zero Round Trip resumption, Session tickets")
    print("Early data: Enabled")
    print("Press Ctrl+C to stop")
    
    await asyncio.Future()


if __name__ == '__main__':
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\nServer stopped")
EOFILE

chmod +x servers/http3-0rtt.py
```

### 4.5.4 HTTP/3 Server with Connection Migration

```bash
cat > servers/http3-migration.py << 'EOFILE'
#!/usr/bin/env python3
"""
HTTP/3 Server with Connection Migration support
Demonstrates seamless network switching
"""

import asyncio
from pathlib import Path
from typing import Dict

from aioquic.asyncio import QuicConnectionProtocol, serve
from aioquic.h3.connection import H3_ALPN, H3Connection
from aioquic.h3.events import H3Event, HeadersReceived
from aioquic.quic.configuration import QuicConfiguration
from aioquic.quic.events import QuicEvent
from aioquic.quic.packet import PACKET_TYPE_INITIAL


class HttpMigrationProtocol(QuicConnectionProtocol):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._http = H3Connection(self._quic)
        self._connection_id = None
        
    def connection_made(self, transport):
        super().connection_made(transport)
        # Store connection ID for tracking
        self._connection_id = self._quic.host_cid.hex()
        print(f"[HTTP/3] New connection: {self._connection_id[:16]}...")
        
    def quic_event_received(self, event: QuicEvent) -> None:
        # Detect path changes
        if hasattr(event, 'data') and hasattr(event, 'addr'):
            addr = event.addr if hasattr(event, 'addr') else None
            if addr and addr != self._quic._network_paths[0].addr:
                print(f"[HTTP/3] Path changed: {addr}")
                print(f"[HTTP/3] Connection migrated (ID: {self._connection_id[:16]}...)")
        
        for http_event in self._http.handle_event(event):
            self.http_event_received(http_event)
    
    def http_event_received(self, event: H3Event) -> None:
        if isinstance(event, HeadersReceived):
            headers = {}
            for name, value in event.headers:
                headers[name.decode()] = value.decode()
            
            method = headers.get(':method', 'GET')
            path = headers.get(':path', '/')
            
            print(f"[HTTP/3] {method} {path} (CID: {self._connection_id[:8]}...)")
            
            asyncio.ensure_future(
                self.handle_request(event.stream_id, headers)
            )
    
    async def handle_request(self, stream_id: int, headers: Dict[str, str]):
        """Handle HTTP request"""
        path = headers.get(':path', '/')
        
        # Special endpoint to test migration
        if path == '/connection-info':
            info = {
                'connection_id': self._connection_id,
                'local_addr': str(self._quic._network_paths[0].local_addr),
                'remote_addr': str(self._quic._network_paths[0].addr),
                'migration_supported': True
            }
            
            import json
            data = json.dumps(info, indent=2).encode()
            self.send_response(stream_id, 200, data, 'application/json')
            return
        
        if path == '/':
            path = '/index.html'
        
        public_dir = Path(__file__).parent.parent / 'public'
        file_path = (public_dir / path.lstrip('/')).resolve()
        
        try:
            file_path.relative_to(public_dir)
        except ValueError:
            self.send_response(stream_id, 403, b'Forbidden')
            return
        
        if not file_path.exists():
            self.send_response(stream_id, 404, b'Not Found')
            return
        
        content_type = self.get_content_type(file_path)
        
        try:
            with open(file_path, 'rb') as f:
                data = f.read()
            
            self.send_response(stream_id, 200, data, content_type)
        except Exception as e:
            print(f"Error: {e}")
            self.send_response(stream_id, 500, b'Internal Server Error')
    
    def send_response(
        self,
        stream_id: int,
        status: int,
        data: bytes,
        content_type: str = 'text/plain'
    ):
        """Send HTTP response"""
        headers = [
            (b':status', str(status).encode()),
            (b'content-type', content_type.encode()),
            (b'content-length', str(len(data)).encode()),
            (b'server', b'aioquic-http3-migration'),
        ]
        
        self._http.send_headers(stream_id, headers)
        self._http.send_data(stream_id, data, end_stream=True)
        self.transmit()
    
    @staticmethod
    def get_content_type(file_path: Path) -> str:
        ext = file_path.suffix.lower()
        types = {
            '.html': 'text/html',
            '.css': 'text/css',
            '.js': 'application/javascript',
            '.json': 'application/json',
            '.txt': 'text/plain',
            '.jpg': 'image/jpeg',
            '.png': 'image/png',
            '.mp4': 'video/mp4',
        }
        return types.get(ext, 'application/octet-stream')


async def main():
    """Start HTTP/3 server with migration support"""
    configuration = QuicConfiguration(
        alpn_protocols=H3_ALPN,
        is_client=False,
        max_datagram_frame_size=65536,
    )
    
    cert_path = Path(__file__).parent.parent / 'certs' / 'cert.pem'
    key_path = Path(__file__).parent.parent / 'certs' / 'key.pem'
    configuration.load_cert_chain(cert_path, key_path)
    
    # Enable connection migration
    configuration.idle_timeout = 300.0  # 5 minutes
    
    await serve(
        host='localhost',
        port=4435,
        configuration=configuration,
        create_protocol=HttpMigrationProtocol,
    )
    
    print("HTTP/3 Server with Connection Migration on https://localhost:4435")
    print("Protocol: h3 + Connection Migration")
    print("Features: Seamless network switching, Connection ID based")
    print("Test endpoint: /connection-info")
    print("Press Ctrl+C to stop")
    
    await asyncio.Future()


if __name__ == '__main__':
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\nServer stopped")
EOFILE

chmod +x servers/http3-migration.py
```

Saya akan melanjutkan tutorial ini. Apakah Anda ingin saya:
1. Melanjutkan dengan client files dan eksperimen lengkap?
2. Atau membuat file terpisah untuk bagian selanjutnya?

Tutorial ini akan sangat panjang (200+ halaman), jadi saya bisa membuat beberapa file terpisah atau melanjutkan di file yang sama.