# Network Programming

> Socket programming, TCP state machine, socket options, network namespaces, and container networking foundations.

---

## Socket Programming

### TCP Server Pattern

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main() {
    // 1. Create socket
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. Set socket options
    int opt = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 3. Bind to address
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(8080),
        .sin_addr.s_addr = INADDR_ANY
    };
    bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));

    // 4. Listen
    listen(sockfd, 128);  // Backlog = 128

    // 5. Accept connections
    while (1) {
        struct sockaddr_in client_addr;
        socklen_t len = sizeof(client_addr);
        int clientfd = accept(sockfd, (struct sockaddr *)&client_addr, &len);

        char client_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
        printf("Connection from %s:%d\n", client_ip, ntohs(client_addr.sin_port));

        // 6. Read/Write
        char buf[4096];
        ssize_t n = read(clientfd, buf, sizeof(buf));
        write(clientfd, "HTTP/1.1 200 OK\r\n\r\nHello", 24);

        close(clientfd);
    }
    close(sockfd);
    return 0;
}
```

### TCP Client Pattern

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in server_addr = {
    .sin_family = AF_INET,
    .sin_port = htons(8080)
};
inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr));

write(sockfd, "GET / HTTP/1.1\r\n\r\n", 18);
char buf[4096];
ssize_t n = read(sockfd, buf, sizeof(buf));
buf[n] = '\0';
printf("Response: %s\n", buf);

close(sockfd);
```

### UDP Server/Client

```c
// UDP Server
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));

struct sockaddr_in client;
socklen_t clen = sizeof(client);
ssize_t n = recvfrom(sockfd, buf, sizeof(buf), 0,
                     (struct sockaddr *)&client, &clen);
sendto(sockfd, response, rlen, 0,
       (struct sockaddr *)&client, clen);

// UDP Client
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
sendto(sockfd, msg, len, 0, (struct sockaddr *)&server, sizeof(server));
recvfrom(sockfd, buf, sizeof(buf), 0, NULL, NULL);
```

---

## TCP State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│  TCP Connection States                                          │
│                                                                 │
│  Client                              Server                    │
│  ──────                              ──────                    │
│                                      LISTEN                    │
│  CLOSED ──── SYN ────►              SYN_RCVD                  │
│  SYN_SENT ◄── SYN+ACK ──           SYN_RCVD                  │
│  ESTABLISHED ── ACK ──►            ESTABLISHED                │
│              ... data transfer ...                              │
│                                                                 │
│  Active Close (client initiates):                               │
│  FIN_WAIT_1 ── FIN ────►           CLOSE_WAIT                 │
│  FIN_WAIT_2 ◄── ACK ──                                        │
│  TIME_WAIT  ◄── FIN ──             LAST_ACK                   │
│              ── ACK ────►           CLOSED                     │
│  CLOSED (after 2×MSL)                                          │
│                                                                 │
│  MSL = Maximum Segment Lifetime (typically 60s)                │
│  TIME_WAIT lasts 2×MSL = 120s default                          │
└─────────────────────────────────────────────────────────────────┘
```

### Important States

| State | Side | Meaning |
|---|---|---|
| **LISTEN** | Server | Waiting for connection requests |
| **SYN_SENT** | Client | SYN sent, waiting for SYN+ACK |
| **SYN_RCVD** | Server | SYN received, SYN+ACK sent |
| **ESTABLISHED** | Both | Connection active, data flowing |
| **FIN_WAIT_1** | Active closer | FIN sent, waiting for ACK |
| **FIN_WAIT_2** | Active closer | FIN ACK'd, waiting for peer's FIN |
| **CLOSE_WAIT** | Passive closer | Peer sent FIN, pending local close |
| **LAST_ACK** | Passive closer | FIN sent, waiting for final ACK |
| **TIME_WAIT** | Active closer | Waiting 2×MSL before fully closing |
| **CLOSING** | Both | Simultaneous close (rare) |

### Why TIME_WAIT Exists
1. **Ensure last ACK reaches peer**: If lost, peer retransmits FIN, TIME_WAIT side retransmits ACK
2. **Prevent old segments**: Wait for old packets to expire before reusing port
3. **Duration**: 2×MSL (60-120 seconds)

### TIME_WAIT Problems and Solutions

```bash
# Count TIME_WAIT connections
ss -tn state time-wait | wc -l

# Problem: Too many TIME_WAIT sockets → port exhaustion
# Solutions:
sysctl net.ipv4.tcp_tw_reuse=1    # Reuse TIME_WAIT sockets (safe for clients)
sysctl net.ipv4.ip_local_port_range="1024 65535"  # More ephemeral ports
# Use connection pooling / keep-alive to reduce connection churn
```

### Three-Way Handshake Details

```
Client                          Server
  │                                │
  │──── SYN (seq=x) ──────────►   │  Client: SYN_SENT
  │                                │  Server: SYN_RCVD
  │◄──── SYN+ACK (seq=y, ack=x+1) │
  │                                │
  │──── ACK (ack=y+1) ────────►   │  Both: ESTABLISHED
  │                                │
```

### Four-Way Teardown

```
Active Closer                   Passive Closer
  │                                │
  │──── FIN ────────────────►      │  FIN_WAIT_1 / CLOSE_WAIT
  │◄──── ACK ──────────────        │  FIN_WAIT_2
  │                                │  (app calls close())
  │◄──── FIN ──────────────        │  LAST_ACK
  │──── ACK ────────────────►      │  CLOSED
  │                                │
  │  TIME_WAIT (2×MSL)            │
  │  → CLOSED                     │
```

---

## Socket Options

### SOL_SOCKET Level

| Option | Purpose | Typical Value |
|---|---|---|
| `SO_REUSEADDR` | Reuse address in TIME_WAIT | 1 (enable) |
| `SO_REUSEPORT` | Multiple listeners on same port (load balance) | 1 (enable) |
| `SO_KEEPALIVE` | Send keepalive probes to detect dead connections | 1 (enable) |
| `SO_RCVBUF` | Receive buffer size | 65536-16777216 |
| `SO_SNDBUF` | Send buffer size | 65536-16777216 |
| `SO_LINGER` | Control close() behavior | struct linger |
| `SO_RCVTIMEO` | Receive timeout | struct timeval |
| `SO_SNDTIMEO` | Send timeout | struct timeval |

### IPPROTO_TCP Level

| Option | Purpose | Typical Value |
|---|---|---|
| `TCP_NODELAY` | Disable Nagle's algorithm (send immediately) | 1 for low-latency |
| `TCP_CORK` | Batch small writes (opposite of NODELAY) | 1 for throughput |
| `TCP_KEEPIDLE` | Time before first keepalive probe | 60 (seconds) |
| `TCP_KEEPINTVL` | Interval between keepalive probes | 10 (seconds) |
| `TCP_KEEPCNT` | Number of probes before declaring dead | 5 |
| `TCP_QUICKACK` | Disable delayed ACK | 1 (must set per-packet) |
| `TCP_FASTOPEN` | TFO: send data in SYN | Queue length |

### Detailed Examples

```c
// SO_REUSEADDR — Allow binding to address in TIME_WAIT
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
// Without this: bind() fails with EADDRINUSE after server restart

// SO_REUSEPORT — Multiple processes listen on same port
setsockopt(sockfd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
// Kernel distributes incoming connections across listeners
// Used by: Nginx, HAProxy for zero-downtime reload

// TCP_NODELAY — Disable Nagle's algorithm
setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
// Nagle: delays small packets to batch them (good for throughput)
// NODELAY: send immediately (good for latency)
// Use for: RPC, interactive protocols, trading systems

// TCP_CORK — Batch small writes
setsockopt(sockfd, IPPROTO_TCP, TCP_CORK, &opt, sizeof(opt));
// Cork: accumulate data until uncorked or MSS reached
// Pattern: cork → write header + body → uncork → send full-sized segments
// Use for: HTTP response headers + body

// SO_KEEPALIVE with custom timers
setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
int idle = 60;    // Start after 60s idle
int intvl = 10;   // Probe every 10s
int cnt = 5;      // Give up after 5 probes (110s total)
setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPIDLE, &idle, sizeof(idle));
setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPINTVL, &intvl, sizeof(intvl));
setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPCNT, &cnt, sizeof(cnt));

// SO_LINGER — Control close() behavior
struct linger ling;
ling.l_onoff = 1;
ling.l_linger = 0;  // 0 = send RST immediately (abort connection)
setsockopt(sockfd, SOL_SOCKET, SO_LINGER, &ling, sizeof(ling));
// l_linger = 0: close() sends RST, skip TIME_WAIT
// l_linger > 0: close() blocks until data sent or timeout

// Buffer sizes
int rcvbuf = 4 * 1024 * 1024;  // 4MB
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
// Kernel doubles the value (for internal bookkeeping)
// Max: net.core.rmem_max
```

### Nagle's Algorithm vs TCP_NODELAY

```
With Nagle (default):
  write("H")  → buffer (wait for ACK or MSS)
  write("e")  → buffer
  write("l")  → buffer
  write("lo") → buffer
  ACK arrives → send "Hello" as one segment

With TCP_NODELAY:
  write("H")  → send immediately: [H]
  write("e")  → send immediately: [e]
  write("l")  → send immediately: [l]
  write("lo") → send immediately: [lo]
  4 separate TCP segments (higher overhead, lower latency)

Best practice: Use TCP_NODELAY + application-level buffering
(write complete messages at once)
```

---

## Network Namespaces

### Concept

```
┌────────────────────────────────────────────────────┐
│  Host Network Namespace (default)                  │
│  ┌──────────┐  ┌──────────┐                       │
│  │ eth0     │  │ lo       │                       │
│  │ 10.0.0.1 │  │ 127.0.0.1│                       │
│  └──────────┘  └──────────┘                       │
│  Routes: default via 10.0.0.1                     │
│  iptables: host rules                             │
│                                                    │
│  ┌────────────────────────────────────────┐        │
│  │  Container Network Namespace           │        │
│  │  ┌──────────┐  ┌──────────┐           │        │
│  │  │ eth0     │  │ lo       │           │        │
│  │  │ 172.17.  │  │ 127.0.0.1│           │        │
│  │  │ 0.2      │  └──────────┘           │        │
│  │  └────┬─────┘                         │        │
│  │       │ veth pair                     │        │
│  └───────┼───────────────────────────────┘        │
│          │                                         │
│   ┌──────▼─────┐                                  │
│   │ docker0     │ (bridge)                        │
│   │ 172.17.0.1  │                                 │
│   └─────────────┘                                 │
└────────────────────────────────────────────────────┘
```

Each network namespace has its own:
- Network interfaces
- IP addresses
- Routing table
- iptables/nftables rules
- Socket tables
- /proc/net

### Working with Network Namespaces

```bash
# Create namespace
ip netns add myns

# Run command in namespace
ip netns exec myns ip addr show
ip netns exec myns ping 127.0.0.1

# Create veth pair (virtual ethernet tunnel)
ip link add veth0 type veth peer name veth1

# Move one end to namespace
ip link set veth1 netns myns

# Configure addresses
ip addr add 10.0.0.1/24 dev veth0
ip link set veth0 up
ip netns exec myns ip addr add 10.0.0.2/24 dev veth1
ip netns exec myns ip link set veth1 up
ip netns exec myns ip link set lo up

# Now can ping between namespaces
ping 10.0.0.2                           # From host
ip netns exec myns ping 10.0.0.1        # From namespace

# List namespaces
ip netns list

# Delete
ip netns del myns
```

---

## Container Networking Foundations

### Bridge Networking (Docker default)

```
┌─────────────────────────────────────────────────────┐
│  Container Bridge Networking                        │
│                                                     │
│  Container 1        Container 2                     │
│  ┌──────────┐      ┌──────────┐                    │
│  │ eth0     │      │ eth0     │                    │
│  │ 172.17.  │      │ 172.17.  │                    │
│  │ 0.2      │      │ 0.3      │                    │
│  └────┬─────┘      └────┬─────┘                    │
│       │ veth             │ veth                     │
│  ┌────▼──────────────────▼─────┐                   │
│  │        docker0 bridge        │                   │
│  │        172.17.0.1            │                   │
│  └──────────────┬──────────────┘                   │
│                 │ NAT (iptables MASQUERADE)         │
│  ┌──────────────▼──────────────┐                   │
│  │          eth0 (host)         │                   │
│  │          10.0.0.1            │                   │
│  └──────────────────────────────┘                   │
│                                                     │
│  Container→Container: via bridge (layer 2)         │
│  Container→Internet: NAT via host                  │
│  Host→Container: port mapping (-p 8080:80)         │
└─────────────────────────────────────────────────────┘
```

### Container Networking Types

| Type | Description | Use Case |
|---|---|---|
| **Bridge** | Default, containers on virtual bridge | Single-host, isolated |
| **Host** | Container uses host network stack | Performance-critical |
| **Overlay** | Multi-host networking (VXLAN) | Docker Swarm, Kubernetes |
| **Macvlan** | Container gets own MAC, appears as physical device | Legacy integration |
| **None** | No networking | Security, custom setup |

---

## Useful Network Debugging

```bash
# Connection states
ss -tn state established             # Established connections
ss -tn state time-wait | wc -l       # Count TIME_WAIT
ss -tn state close-wait              # CLOSE_WAIT (app not calling close())

# Socket info with process
ss -tnlp                             # Listening sockets with PID
ss -tnp dst :8080                    # Connections to port 8080

# Network statistics
ss -s                                 # Summary statistics
cat /proc/net/snmp | grep Tcp         # TCP statistics
nstat -az | grep -i retrans           # Retransmissions
netstat -s | grep -i retransmit

# Interface statistics
ip -s link show eth0                  # TX/RX packets, bytes, errors, drops
ethtool -S eth0                       # Detailed NIC statistics
```

---

## Interview Tips

1. **TCP three-way handshake?**
   → SYN(seq=x) → SYN+ACK(seq=y, ack=x+1) → ACK(ack=y+1). Establishes sequence numbers for both directions.

2. **What is TIME_WAIT and why?**
   → Active closer waits 2×MSL (120s). Ensures last ACK reaches peer and old packets expire. Fix exhaustion: tcp_tw_reuse, connection pooling.

3. **CLOSE_WAIT accumulation?**
   → Application received FIN but hasn't called close(). Bug in application code. Fix: ensure proper close() in error paths.

4. **TCP_NODELAY vs TCP_CORK?**
   → NODELAY: disable Nagle, send immediately (latency). CORK: accumulate data, send full segments (throughput). Best: NODELAY + app-level message buffering.

5. **SO_REUSEPORT purpose?**
   → Multiple processes/threads bind to same port. Kernel distributes connections. Used for zero-downtime reload, multi-process servers.

6. **How do containers get networking?**
   → Network namespace (isolated stack) + veth pair (virtual ethernet) + bridge (layer 2 switching) + iptables NAT (outbound). Each container has its own IP.

---

*Last updated: March 13, 2026*
