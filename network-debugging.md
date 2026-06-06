# Network Debugging

> Why can't my app connect? Trace packets, understand DNS, debug TLS, spoof hosts.
> Tools: [tcpdump](https://www.tcpdump.org), [Wireshark](https://www.wireshark.org), [netcat](https://netcat.sourceforge.net), [curl](https://curl.se)

## Quick Diagnosis — is it the network?

```bash
# 1. Can I reach the host at all?
ping -c 4 google.com                       # basic ICMP
ping -c 4 8.8.8.8                          # by IP (bypasses DNS)

# 2. Is DNS working?
dig google.com                               # DNS lookup (Linux/macOS)
nslookup google.com                          # DNS lookup (all platforms)
host google.com                              # short DNS lookup

# 3. Is the port open?
nc -zv google.com 443 -w 3                  # TCP connect test (3s timeout)
curl -v telnet://google.com:443              # another way to test port
# PowerShell:
Test-NetConnection google.com -Port 443

# 4. What's the route?
traceroute google.com                        # Linux/macOS
tracert google.com                           # Windows
mtr google.com                               # continuous traceroute (install: mtr)

# 5. What's listening locally?
ss -tlnp                                   # Linux: TCP listening + process
netstat -an | grep LISTEN                  # all platforms
lsof -i :8080                              # macOS/Linux: what's on port 8080
# PowerShell:
Get-NetTCPConnection -State Listen | Select-Object LocalPort,OwningProcess
```

## tcpdump — capture packets

```bash
# Install: sudo apt install tcpdump / brew install tcpdump

# Capture all traffic on default interface
sudo tcpdump -i any -c 100                 # first 100 packets

# Filter by host
sudo tcpdump -i any host 192.168.1.100

# Filter by port
sudo tcpdump -i any port 8080
sudo tcpdump -i any port 443 or port 80    # HTTP + HTTPS

# Filter by protocol
sudo tcpdump -i any tcp                    # TCP only
sudo tcpdump -i any udp                    # UDP only
sudo tcpdump -i any icmp                   # ping only

# Save to file (open in Wireshark later)
sudo tcpdump -i any -w capture.pcap host api.example.com

# Read saved capture
tcpdump -r capture.pcap

# Show packet contents (hex + ASCII)
sudo tcpdump -i any -XX port 8080

# Show only SYN packets (connection attempts)
sudo tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0'

# Show only RST packets (connection refused)
sudo tcpdump -i any 'tcp[tcpflags] & tcp-rst != 0'

# Filter by app (by PID — Linux)
sudo tcpdump -i any -c 100 | grep "$(lsof -t -i :8080)"

# DNS queries only
sudo tcpdump -i any port 53
```

## Wireshark — GUI packet analysis

```bash
# Install: scoop install wireshark / sudo apt install wireshark / brew install --cask wireshark
wireshark                                    # launch GUI
wireshark capture.pcap                       # open saved capture

# Useful display filters (type in filter bar):
# http                                    — HTTP traffic only
# dns                                     — DNS queries/responses
# tls                                     — TLS/SSL handshake
# tcp.port == 8080                        — specific port
# ip.addr == 192.168.1.100                — specific host
# tcp.flags.syn == 1 && tcp.flags.ack == 0  — SYN only (connection attempts)
# tcp.flags.reset == 1                    — RST (connection refused)
# http.request                            — HTTP requests only
# dns.qry.name contains "example"         — DNS for specific domain
# tcp.stream eq 0                         — follow one TCP conversation
# frame contains "password"               — search packet contents
```

### Decrypt TLS in Wireshark

```bash
# Set SSLKEYLOGFILE before running your app
export SSLKEYLOGFILE=~/sslkeys.log

# Run your app (curl, Chrome, Firefox, etc.)
curl https://api.example.com

# In Wireshark:
# Edit → Preferences → Protocols → TLS
# Set "(Pre)-Master-Secret log filename" to ~/sslkeys.log
# Now you can see decrypted HTTP content!

# For Chromium-based browsers:
chromium --ssl-key-log-file=~/sslkeys.log
```

## curl — HTTP debugging

```bash
# Verbose — see full request/response
curl -v https://api.example.com/health

# Show timing breakdown
curl -o /dev/null -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nFirst byte: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://api.example.com

# Follow redirects + show each hop
curl -vL https://example.com 2>&1 | grep -E "< HTTP|< Location|> Host"

# Test specific HTTP methods
curl -X POST -H "Content-Type: application/json" -d '{"key":"val"}' https://api.example.com/endpoint

# Test with specific TLS version
curl --tlsv1.2 -v https://api.example.com
curl --tlsv1.3 -v https://api.example.com

# Test with specific IP (bypass DNS)
curl --resolve api.example.com:443:93.184.216.34 https://api.example.com

# Test with specific host header
curl -H "Host: api.example.com" https://93.184.216.34

# Show only response headers
curl -I https://api.example.com

# Download + show progress
curl -# -O https://example.com/large-file.zip

# Retry with backoff
curl --retry 5 --retry-delay 2 --retry-all-errors https://api.example.com
```

## DNS Debugging

```bash
# Full DNS trace (follow the chain)
dig +trace google.com

# Query specific DNS server
dig @8.8.8.8 google.com                    # Google DNS
dig @1.1.1.1 google.com                    # Cloudflare DNS

# Check all record types
dig ANY example.com

# Reverse DNS lookup
dig -x 8.8.8.8

# Check if DNS is cached (macOS)
dscacheutil -cachedump -entries

# Flush DNS cache
sudo systemd-resolve --flush-caches        # Linux (systemd)
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder  # macOS
ipconfig /flushdns                          # Windows

# Test DNS resolution speed
time dig google.com
time dig @8.8.8.8 google.com
```

## netcat — TCP/UDP Swiss army knife

```bash
# Test if port is open
nc -zv host 8080 -w 3                       # TCP connect test

# Simple TCP server (listen on port 8080)
nc -l 8080

# Simple TCP client (connect to server)
nc host 8080

# Send HTTP request manually
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80

# UDP test
nc -u -zv host 53                           # UDP DNS port

# Port scan (quick check)
nc -zv host 1-1024 2>&1 | grep succeeded

# File transfer
# Sender:
nc -l 9999 < file.txt
# Receiver:
nc host 9999 > file.txt

# Simple chat
# Server:
nc -l 9999
# Client:
nc host 9999
# Type messages, they appear on the other side
```

## Host Spoofing & Testing

```bash
# /etc/hosts — redirect domain to different IP
sudo sh -c 'echo "127.0.0.1 api.example.com" >> /etc/hosts'
# Now api.example.com points to localhost

# Windows: C:\Windows\System32\drivers\etc\hosts (admin)

# curl --resolve (without modifying hosts file)
curl --resolve api.example.com:443:127.0.0.1 https://api.example.com

# Proxy all traffic through local server
export http_proxy=http://127.0.0.1:8080
export https_proxy=http://127.0.0.1:8080
# Now all HTTP/HTTPS goes through your local proxy

# mitmproxy — intercept and modify HTTPS traffic
# Install: pip install mitmproxy
mitmproxy                                    # interactive TUI
mitmweb                                      # web UI at http://127.0.0.1:8081
# Set proxy + install mitmproxy CA cert to decrypt HTTPS
```

## Firewall & Port Debugging

```bash
# Linux — iptables/nftables
sudo iptables -L -n                          # list rules
sudo nft list ruleset                        # nftables
sudo ufw status                              # Ubuntu firewall

# macOS — pf
sudo pfctl -sr                               # show rules
sudo pfctl -d                                # disable firewall temporarily

# Windows — firewall
netsh advfirewall show allprofiles state
netsh advfirewall firewall show rule name=all | findstr "8080"

# Check if something is blocked by firewall
# Try from another machine on same network:
nc -zv your-ip 8080

# Common "it works on localhost but not from outside":
# 1. Firewall blocking the port
# 2. App binding to 127.0.0.1 instead of 0.0.0.0
# 3. NAT/router not forwarding port
```

## MTU & Fragmentation Issues

```bash
# Test MTU (find maximum packet size)
ping -c 4 -M do -s 1472 host               # Linux: 1472 + 28 = 1500 MTU
ping -c 4 -D -s 1472 host                  # macOS
ping -f -l 1472 host                        # Windows

# If ping fails at 1472, reduce by 10 until it works:
ping -c 4 -M do -s 1462 host               # try 1462
ping -c 4 -M do -s 1400 host               # try 1400
# Working size + 28 = your path MTU

# Common MTU issues:
# - VPN reduces MTU (often 1400 or less)
# - Docker overlay network: MTU 1450
# - "Works sometimes" = MTU mismatch for large packets
```

## Connection State Debugging

```bash
# See all TCP connections and their states
ss -tnp                                     # Linux
netstat -anp tcp                            # macOS
# PowerShell:
Get-NetTCPConnection | Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State

# TCP states explained:
# ESTABLISHED  — connected and working
# TIME_WAIT    — connection closed, waiting for stray packets (normal)
# CLOSE_WAIT   — remote closed, local hasn't closed yet (possible leak!)
# SYN_SENT     — trying to connect (stuck here = connection refused/firewall)
# SYN_RECV     — received connection request (stuck = half-open)
# FIN_WAIT_2   — local closed, waiting for remote (stuck = remote not closing)

# Count connections by state
ss -tn | awk '{print $1}' | sort | uniq -c | sort -rn

# Find connections to specific host
ss -tn | grep 192.168.1.100

# Find what process owns a connection
ss -tnp | grep :443                        # shows pid
lsof -i :443                               # macOS/Linux
```

## WebSocket Debugging

```bash
# Install websocat:
# scoop install websocat / brew install websocat / cargo install websocat

# Connect to WebSocket
websocat ws://localhost:8080/ws

# With headers
websocat -H "Authorization: Bearer token" ws://localhost:8080/ws

# Test with curl (upgrade request)
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  http://localhost:8080/ws
```

## Quick Network Health Check Script

```bash
#!/bin/bash
HOST=${1:-"google.com"}
PORT=${2:-443}

echo "=== Network Health Check: $HOST:$PORT ==="
echo ""
echo "1. DNS Resolution:"
dig +short $HOST | head -3
echo ""
echo "2. Ping (latency):"
ping -c 3 $HOST 2>&1 | tail -1
echo ""
echo "3. Port connectivity:"
nc -zv $HOST $PORT -w 3 2>&1
echo ""
echo "4. TLS handshake:"
echo | openssl s_client -connect $HOST:$PORT -servername $HOST 2>/dev/null | grep -E "Protocol|Cipher|Verify"
echo ""
echo "5. HTTP response:"
curl -sI -w "HTTP %{http_code} | Time: %{time_total}s\n" -o /dev/null https://$HOST
echo ""
echo "6. Route:"
traceroute -m 15 $HOST 2>&1 | tail -5
```

---

**Related**: [File tracking](README.md#11-file-tracking) — strace/procmon for file I/O
**Related**: [Shell tricks](shell-tricks.md) — process substitution, pipes
**Related**: [Wireshark docs](https://www.wireshark.org/docs/) — full display filter reference
**Related**: [tcpdump man page](https://www.tcpdump.org/manpages/tcpdump.1.html) — complete filter syntax
