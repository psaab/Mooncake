# IPv6 Compatibility Audit & Fix Skill

Audit a codebase for IPv4-only bugs and fix them for IPv6 compatibility.

## When to Use This Skill

Use when the user asks to:
- Find and fix IPv4-only bugs
- Add IPv6 support to networking code
- Audit socket/network code for dual-stack compatibility
- Fix host:port parsing for IPv6 addresses

## Search Patterns

Run these searches in parallel to find all IPv4-only code:

### 1. Deprecated IPv4-only APIs
```
gethostbyname        # IPv4-only, not thread-safe, deprecated
inet_addr(           # Returns INADDR_NONE for IPv6, use getaddrinfo
inet_ntoa(           # IPv4-only, not thread-safe, use inet_ntop
inet_aton(           # IPv4-only, use inet_pton with AF_INET6 or AF_UNSPEC
```

### 2. Hardcoded IPv4 socket creation
```
AF_INET[^6]          # AF_INET without AF_INET6 alternative
socket(AF_INET,      # IPv4-only socket, should try AF_INET6 first
sockaddr_in[^6]      # IPv4-only address struct
INADDR_ANY           # IPv4 any-address, IPv6 equivalent is in6addr_any
INADDR_LOOPBACK      # IPv4 loopback, IPv6 equivalent is in6addr_loopback
```

### 3. Host:port splitting that breaks on IPv6
```
.find(":")           # In C++: first colon matches inside IPv6 address
.find(':')           # Same issue
.split(":")          # In Python: splits IPv6 address into many parts
.split(':')          # Same issue
+ ":" +              # String concatenation without bracket notation
```

### 4. Hardcoded bind addresses
```
0.0.0.0              # IPv4-only bind, should be :: for dual-stack
"127.0.0.1"          # Check if ::1 should also be recognized
```

### 5. IPv4-only address format in inet_pton
```
inet_pton(AF_INET,   # Hardcoded to IPv4, rejects IPv6 addresses
```

### 6. Python-specific
```
socket.AF_INET[^6]   # IPv4-only socket
gethostbyname        # Deprecated IPv4-only resolver
```

### 7. RDMA GID selection filtering out native IPv6
```
ipv6_addr_v4mapped.*&&.*ROCE_V2   # Only accepts IPv4-mapped RoCEv2 GIDs
is_ipv4_mapped && is_roce_v2      # Same pattern with variables
```

### 8. Overlay/private network detection that ignores IPv6
```
isOverlayIPv4        # Only checks IPv4-mapped GID ranges, misses IPv6 overlays
is_ipv4_mapped &&.*isOverlay  # Guard that skips overlay check for native IPv6
```

### 9. accept() with undersized address buffer
```
sockaddr_in addr;.*accept(   # sockaddr_in too small for IPv6 addresses
sizeof(sockaddr_in).*accept  # Truncates sockaddr_in6 on IPv6 sockets
```

### 10. getaddrinfo locked to single address family
```
hints\.ai_family.*use_ipv6.*AF_INET6.*AF_INET  # Forces one family, should be AF_UNSPEC
```

## Fix Patterns

### Fix 1: Replace gethostbyname with getaddrinfo

**C:**
```c
// BEFORE (IPv4-only)
struct hostent *he = gethostbyname(hostname);

// AFTER (IPv4/IPv6)
struct addrinfo hints = {}, *result;
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;
getaddrinfo(hostname, port_str, &hints, &result);
// Use result->ai_family, result->ai_addr, result->ai_addrlen
freeaddrinfo(result);
```

**Python:**
```python
# BEFORE
ip = socket.gethostbyname(hostname)

# AFTER
infos = socket.getaddrinfo(hostname, None, socket.AF_UNSPEC, socket.SOCK_STREAM)
for family, _, _, _, sockaddr in infos:
    if family == socket.AF_INET:
        ip = sockaddr[0]
        break
else:
    for family, _, _, _, sockaddr in infos:
        if family == socket.AF_INET6:
            ip = sockaddr[0]
            break
```

### Fix 2: Dual-stack socket with IPv4 fallback

```c
// Try IPv6 dual-stack first (accepts both IPv4 and IPv6 on Linux)
int sock = socket(AF_INET6, SOCK_STREAM, 0);
if (sock >= 0) {
    int v6only = 0;
    setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY, &v6only, sizeof(v6only));

    struct sockaddr_in6 addr = {};
    addr.sin6_family = AF_INET6;
    addr.sin6_addr = in6addr_any;  // or in6addr_loopback
    addr.sin6_port = htons(port);

    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) == 0) {
        // success
    }
} else {
    // Fallback to IPv4
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    // ... IPv4 bind as before
}
```

### Fix 3: Use getaddrinfo for connect() instead of inet_addr

```c
// BEFORE
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = inet_addr(ip);  // IPv4-only!
connect(sock, (sockaddr*)&addr, sizeof(addr));

// AFTER
struct addrinfo hints = {}, *result;
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;
getaddrinfo(ip, port_str, &hints, &result);
int sock = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
connect(sock, result->ai_addr, result->ai_addrlen);
freeaddrinfo(result);
```

### Fix 4: Use sockaddr_storage for accept()

```c
// BEFORE
struct sockaddr_in client_addr;
socklen_t len = sizeof(client_addr);
accept(server_fd, (sockaddr*)&client_addr, &len);
LOG(INFO) << inet_ntoa(client_addr.sin_addr);  // IPv4-only!

// AFTER
struct sockaddr_storage client_addr;
socklen_t len = sizeof(client_addr);
accept(server_fd, (sockaddr*)&client_addr, &len);
char addr_str[INET6_ADDRSTRLEN] = {};
if (client_addr.ss_family == AF_INET) {
    auto *a4 = (struct sockaddr_in *)&client_addr;
    inet_ntop(AF_INET, &a4->sin_addr, addr_str, sizeof(addr_str));
} else if (client_addr.ss_family == AF_INET6) {
    auto *a6 = (struct sockaddr_in6 *)&client_addr;
    inet_ntop(AF_INET6, &a6->sin6_addr, addr_str, sizeof(addr_str));
}
```

### Fix 5: IPv6-safe host:port parsing

The core problem: IPv6 addresses contain colons, so `host:port` splitting by `:` breaks. IPv6 uses bracket notation: `[::1]:8080`.

**IMPORTANT:** When fixing host:port parsing, extract a single shared utility function rather than
inlining the logic at each call site. Codebases typically have 3+ places that parse `host:port`
(client connect, server bind, config parsing) and they drift out of sync. Centralizing prevents
regressions and keeps callers readable.

**C++ helper (centralized):**
```cpp
// Put this in a shared header (e.g. a utility or io header).
// Returns {host, port} where port may be empty if not present.
inline std::pair<std::string, std::string> split_host_port(
    std::string_view address) {
    if (!address.empty() && address.front() == '[') {
        // Bracketed IPv6: [addr] or [addr]:port
        auto close = address.find(']');
        if (close != std::string_view::npos) {
            std::string host{address.substr(1, close - 1)};
            std::string port;
            if (close + 2 < address.size() && address[close + 1] == ':')
                port = address.substr(close + 2);
            return {std::move(host), std::move(port)};
        }
    }
    // Count colons: 2+ means bare IPv6 (no port)
    auto first = address.find(':');
    if (first != std::string_view::npos &&
        address.find(':', first + 1) != std::string_view::npos) {
        return {std::string{address}, {}};
    }
    // Single colon: host:port
    if (first != std::string_view::npos) {
        return {std::string{address.substr(0, first)},
                std::string{address.substr(first + 1)}};
    }
    // No colon: plain host
    return {std::string{address}, {}};
}

std::string build_host_port(const std::string& host, int port) {
    if (host.find(':') != std::string::npos)
        return "[" + host + "]:" + std::to_string(port);
    return host + ":" + std::to_string(port);
}
```

**Callers then become simple:**
```cpp
// Client connect
auto [host, port] = split_host_port(address);
return connect(std::move(host), std::move(port), timeout);

// Server init_address
auto [host, port_str] = split_host_port(address);
address_ = std::move(host);
if (!port_str.empty()) {
    uint16_t port;
    auto [ptr, ec] = std::from_chars(
        port_str.data(), port_str.data() + port_str.size(), port, 10);
    if (ec == std::errc{}) port_ = port;
}
```

**Python:**
```python
# BEFORE (breaks on IPv6)
host, port = addr.split(":")

# AFTER — centralize in a helper
def split_host_port(addr):
    if addr.startswith("["):
        bracket_end = addr.find("]")
        host = addr[1:bracket_end]
        port_str = addr[bracket_end + 2:] if bracket_end + 1 < len(addr) and addr[bracket_end + 1] == ":" else ""
    elif addr.count(":") == 1:
        host, port_str = addr.split(":")
    else:
        host = addr  # bare IPv6, no port
        port_str = ""
    return host, port_str
```

### Fix 6: Default bind addresses

```
0.0.0.0  →  ::        (for bind/listen addresses)
```

`::` with `IPV6_V6ONLY=0` (Linux default) accepts both IPv4 and IPv6 connections.

### Fix 7: Loopback detection

```python
# BEFORE
def is_local(host):
    return host in ("localhost", "127.0.0.1")

# AFTER
def is_local(host):
    return host in ("localhost", "127.0.0.1", "::1", "[::1]")
```

### Fix 8: IPv6-aware address construction for logging/registration

When constructing `ip:port` strings, always check if the IP is IPv6.
Like split_host_port, extract a shared `build_host_port` utility and use it everywhere —
log lines, error messages, address registration, `operator<<` for endpoint types.

**IMPORTANT:** Search for ALL `host << ":" << port`, `host + ":" + port`, and
`.append(":").append(port)` patterns across the codebase and replace them.
These are easy to miss in log lines.

**C++ helper (centralized, pair with split_host_port):**
```cpp
// Put next to split_host_port in the same shared header.
template <typename Port>
inline std::string build_host_port(std::string_view host, Port port) {
    std::string result;
    if (host.find(':') != std::string_view::npos)
        result.append("[").append(host).append("]");
    else
        result.append(host);
    if constexpr (std::is_arithmetic_v<Port>)
        result.append(":").append(std::to_string(port));
    else
        result.append(":").append(port);
    return result;
}
```

**Callers become simple:**
```cpp
// Log lines (most common case — easy to miss during audit)
LOG_INFO << "connecting to " << build_host_port(host, port);
LOG_INFO << "endpoint: " << build_host_port(ep.address().to_string(), ep.port());

// operator<< for endpoint/address structs
std::ostream& operator<<(std::ostream& os, const endpoint& ep) {
    return os << build_host_port(ep.address.to_string(), ep.port);
}

// Building address strings
address = build_host_port(ip_str, port);  // NOT: ip_str + ":" + std::to_string(port)
```

**Python:**
```python
def build_host_port(host, port):
    if ":" in host:
        return f"[{host}]:{port}"
    return f"{host}:{port}"
```

### Fix 9: RDMA GID selection — accept native IPv6 RoCEv2 GIDs

RDMA GID tables contain entries of different types (IB, RoCE v1, RoCE v2).
RoCE v2 GIDs can be either IPv4-mapped (::ffff:x.x.x.x) or native IPv6.
Code that filters on `ipv4_mapped && RoCEv2` rejects native IPv6 RoCEv2 GIDs.

```c
// BEFORE — only accepts IPv4-mapped RoCEv2, skips native IPv6 RoCEv2
if ((ipv6_addr_v4mapped(gid.raw) && gid_type == IBV_GID_TYPE_ROCE_V2) ||
    gid_type == IBV_GID_TYPE_IB) {
    // use this GID
}

// AFTER — accepts all RoCEv2 (IPv4-mapped and native IPv6) plus IB
if (gid_type == IBV_GID_TYPE_ROCE_V2 || gid_type == IBV_GID_TYPE_IB) {
    // use this GID
}
```

### Fix 10: Overlay network detection — handle IPv6 ranges

Overlay detection that only checks IPv4 private ranges misses IPv6 overlay networks.

```c
// BEFORE — only checks IPv4-mapped addresses
static bool isOverlayIPv4(const struct in6_addr* addr) {
    if (!ipv6_addr_v4mapped(addr)) return false;
    // ... check 10.0.0.0/8, 172.16.0.0/12, 100.64.0.0/10
}

// AFTER — checks both IPv4 and IPv6 overlay ranges
static bool isOverlayAddress(const struct in6_addr* addr) {
    if (ipv6_addr_v4mapped(addr)) {
        uint32_t ipv4 = ntohl(addr->s6_addr32[3]);
        uint8_t o1 = (ipv4 >> 24) & 0xFF, o2 = (ipv4 >> 16) & 0xFF;
        if (o1 == 10) return true;                              // 10.0.0.0/8
        if (o1 == 172 && o2 >= 16 && o2 <= 31) return true;    // 172.16.0.0/12
        if (o1 == 100 && o2 >= 64 && o2 <= 127) return true;   // 100.64.0.0/10
        return false;
    }
    // IPv6 overlay ranges
    if (addr->s6_addr[0] == 0xfd) return true;                  // fd00::/8 ULA
    if ((addr->s6_addr[0] == 0xfe) &&
        ((addr->s6_addr[1] & 0xc0) == 0x80)) return true;       // fe80::/10 link-local
    return false;
}
```

### Fix 11: getaddrinfo with AF_UNSPEC instead of forced family

```c
// BEFORE — locks to one address family
hints.ai_family = use_ipv6 ? AF_INET6 : AF_INET;

// AFTER — resolves both, iterates results
hints.ai_family = AF_UNSPEC;
// The getaddrinfo loop (for rp = result; rp; rp = rp->ai_next)
// already tries all results, so AF_UNSPEC just works.
```

## Triage Guide

**Fix immediately (production code):**
- `gethostbyname` / `inet_addr` / `inet_ntoa` in non-test code
- `AF_INET`-only socket creation for bind/listen/connect
- Host:port splitting by first `:` that breaks on IPv6
- `0.0.0.0` as default bind address in servers
- Missing bracket notation in IPv6 address:port construction
- RDMA GID selection requiring `ipv4_mapped` for RoCEv2 (skips native IPv6)
- `accept()` using `sockaddr_in` on dual-stack/IPv6 sockets (buffer truncation)
- Overlay detection that only checks IPv4 private ranges (misses ULA/link-local)
- `getaddrinfo` hints locked to single AF via config flag (should be `AF_UNSPEC`)

**Fix but lower priority (test code):**
- Test utilities that discover local IP (add IPv6 fallback)
- Test data using `split(":")` for port extraction
- Test assertions checking for `0.0.0.0` defaults
- Hard-coded `127.0.0.1` test defaults — change to `[::1]` for IPv6-only compat

**Skip (requires architectural changes):**
- Data structures storing IPs as `in_addr` (32-bit) — need struct redesign
- External library APIs that only accept IPv4 types (e.g., vendor SDK)
- `127.0.0.1` used intentionally for loopback-only access
- HCCL transport `RankInfo`/`RankControlInfo` structs with `struct in_addr` fields
  (tied to Huawei's `HcclIpAddress` API which only accepts `in_addr`).
  All `inet_pton(AF_INET)`, `inet_ntoa()`, and `s_addr` comparisons in
  `hccl_transport.cpp` and `hccl_transport_mem_c.cpp` are consequences of this.
  Fixing requires upstream library changes or a wire protocol version bump.

**Not bugs (leave alone):**
- `AF_INET` in IPv4 fallback branches of dual-stack code
- `AF_INET` in conditional `use_ipv6 ? AF_INET6 : AF_INET` code
- `AF_INET` preference in two-pass IP discovery (IPv4 first, then IPv6)
- RFC 1918 range checks (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- `type:index` parsers (e.g., `cuda:0`) — not network addresses
- Comments mentioning IPv4 addresses as examples
- Segment IDs that look like IPs but are string identifiers (e.g., `openSegment("192.168.3.76")`)
