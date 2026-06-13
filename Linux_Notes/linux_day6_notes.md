## Day 6 — Networking Commands

---

### Why This is Critical for DevOps

You deployed your application but users cannot access it. You check the app — it's running fine. So what's wrong? The problem could be: server not reachable, wrong port, firewall blocking, DNS not resolving, or service not listening on the right interface. Today's commands diagnose every one of these.

---

### Quick Concept — IP Addresses and Ports

**IP Address** — the address of a server on a network. Like a house address. Example: `192.168.1.10` or `13.234.56.78`

**Port** — a specific door on that house. Different services listen on different ports.

| Port | Service |
|------|---------|
| 22 | SSH — remote login |
| 80 | HTTP — websites |
| 443 | HTTPS — secure websites |
| 3306 | MySQL database |
| 5432 | PostgreSQL database |
| 6379 | Redis |
| 8080 | Common alternate web port |
| 27017 | MongoDB |

When you say `http://192.168.1.10:8080` — you mean "go to this IP and knock on door number 8080."

---

### What You Will Learn Today

- ping — is a server reachable?
- curl — make HTTP requests
- wget — download files
- ip / ifconfig — network interfaces
- ss / netstat — open ports
- nslookup / dig — DNS troubleshooting
- traceroute — trace network path

---

### Command 1 — ping

```bash
ping google.com          # ping continuously — press Ctrl+C to stop
ping -c 4 google.com    # send exactly 4 packets then stop
ping -i 2 google.com    # send one packet every 2 seconds
```

**Understanding ping output:**
```
PING google.com (142.250.67.46) 56(84) bytes of data.
64 bytes from 142.250.67.46: icmp_seq=1 ttl=118 time=12.3 ms
```

| Part | Meaning |
|------|---------|
| 142.250.67.46 | IP address Linux resolved for google.com |
| icmp_seq=1 | Packet number 1 |
| ttl=118 | How many routers this packet passed through |
| time=12.3 ms | Round trip time — how long the response took |

| Situation | What it means |
|-----------|--------------|
| ping works | Server is reachable at network level |
| ping fails | Server is down, unreachable, OR blocking ping |
| High response time 500ms+ | Network is slow or congested |
| Packet loss shown | Unstable network connection |

> ⚠️ Some servers block ping for security. Ping failing does NOT always mean the server is down.

**Real DevOps situation:** User says "I can't reach the website." First check:
```bash
ping yourserver.com
```
If ping fails — network level issue. If ping works — problem is at application level (wrong port, app crashed, firewall).

---

### Command 2 — curl (Make HTTP Requests)

Makes HTTP requests from the terminal. Like a browser but in text form. One of the most used tools in DevOps.

```bash
curl http://google.com               # fetch a webpage
curl -I http://google.com           # fetch ONLY headers (faster)
curl -o myfile.html http://google.com  # save output to file
curl -L http://google.com           # follow redirects automatically
curl -v http://google.com           # verbose — full request and response
curl -I http://localhost:8080/health  # check if app is responding
```

**HTTP status codes:**

| Code | Meaning |
|------|---------|
| 200 | OK — success |
| 301/302 | Redirect |
| 400 | Bad request |
| 401 | Unauthorized |
| 403 | Forbidden — permission denied |
| 404 | Not found |
| 500 | Internal server error |
| 502 | Bad gateway — upstream server issue |
| 503 | Service unavailable |

**Testing an API with curl:**
```bash
curl -X GET "https://api.example.com/users"
curl -X POST "https://api.example.com/users" \
     -H "Content-Type: application/json" \
     -d '{"name": "varun", "email": "varun@example.com"}'
```

**Check only the HTTP response code:**
```bash
curl -o /dev/null -s -w "%{http_code}" http://google.com
# Output: 200
```

---

### Command 3 — wget (Download Files)

```bash
wget https://example.com/file.zip              # download file
wget -O myname.zip https://example.com/file.zip  # save with custom name
wget -c https://example.com/largefile.zip     # resume interrupted download
wget -q https://example.com/file.zip          # quiet — no progress output
```

**curl vs wget:**

| Feature | curl | wget |
|---------|------|------|
| Primary use | Making HTTP requests, testing APIs | Downloading files |
| Resuming downloads | Manual | Built in with `-c` |
| Recursive download | No | Yes with `-r` |
| Output | Prints to screen by default | Saves to file by default |

**Real DevOps situation:** Install a tool on a server:
```bash
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
unzip terraform_1.5.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform --version
```

---

### Command 4 — ip / ifconfig (Network Interfaces)

```bash
ip addr show           # show all network interfaces and IPs
ip addr show eth0      # show specific interface
ip route show          # show routing table
ifconfig               # older command — same purpose
```

Example output:
```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 192.168.1.100  netmask 255.255.255.0  broadcast 192.168.1.255
lo:   flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
      inet 127.0.0.1  netmask 255.255.0.0
```

**Important interfaces:**

| Interface | Meaning |
|-----------|---------|
| eth0 or ens3 | Your main ethernet network card |
| lo | Loopback — always 127.0.0.1 — your machine talking to itself |
| wlan0 | Wireless interface |
| docker0 | Docker's virtual network interface |

`127.0.0.1` (localhost) — always refers to your own machine. When an app says "listening on localhost:8080" — only accessible from the same machine, not from outside.

---

### Command 5 — ss / netstat (Open Ports and Connections)

```bash
ss -tulnp                    # show all open ports and listeners
ss -tulnp | grep :8080       # check if port 8080 is in use
ss -tulnp | grep :80         # what is listening on port 80?
netstat -tulnp               # older command — same purpose
netstat -an                  # all connections with numbers
```

| Flag | Meaning |
|------|---------|
| -t | TCP connections |
| -u | UDP connections |
| -l | Show only listening ports |
| -n | Numbers instead of service names |
| -p | Show which process uses each port |

Example output:
```
Netid  State   Local Address:Port   Process
tcp    LISTEN  0.0.0.0:22           sshd
tcp    LISTEN  0.0.0.0:80           nginx
tcp    LISTEN  127.0.0.1:3306       mysqld
```

Reading this:
- SSH is listening on port 22 — accessible from anywhere (0.0.0.0)
- Nginx is on port 80 — accessible from anywhere
- MySQL is on port 3306 — only accessible locally (127.0.0.1)

**Real DevOps situation:** You deployed your app on port 8080 but users can't connect:
```bash
ss -tulnp | grep 8080
```
If nothing shows — app is not listening on that port (crashed or wrong port). If it shows `127.0.0.1:8080` — it's listening only locally. Change app config to listen on `0.0.0.0:8080`.

---

### Command 6 — nslookup / dig (DNS Lookup)

DNS converts domain names (google.com) to IP addresses (142.250.67.46). When DNS is broken, your domain stops working even if the server is perfectly fine.

```bash
nslookup google.com                # basic DNS lookup
dig google.com                     # detailed DNS lookup
dig google.com A                   # get IPv4 address record
dig google.com MX                  # get mail server records
dig @8.8.8.8 google.com           # query Google's DNS server specifically
```

**Real DevOps situation:** You updated DNS records but the site still shows old content:
```bash
dig yourwebsite.com
```
Check the IP in response. If still old IP — DNS hasn't propagated yet. Wait and check again. If shows new IP but site still broken — problem is on your server, not DNS.

---

### Command 7 — traceroute

Shows every router (hop) your packet passes through to reach a destination. Like GPS tracking for your network packet.

```bash
traceroute google.com
```

Example output:
```
1  192.168.1.1 (router)       1.234 ms
2  10.0.0.1 (ISP gateway)     5.678 ms
3  ...
15 142.250.67.46 (google)     12.3 ms
```

If traceroute stops at hop 5 — that router is where the problem is. You can tell your network team exactly where the issue is.

---

### Putting It All Together — Real Scenario

Your manager says: "Users in Bangalore can't access our website. Investigate."

```bash
ping yourwebsite.com             # is server reachable at all?
curl -I http://yourwebsite.com   # is web server responding?
nslookup yourwebsite.com         # is DNS resolving correctly?
traceroute yourwebsite.com       # where is the connection failing?
ss -tulnp | grep :80             # is nginx actually listening on port 80?
systemctl status nginx           # is nginx running?
```

Six commands. Complete picture. That's professional DevOps troubleshooting.

---

### Full Command Summary — Day 6

| Command | What it does | Key flag |
|---------|-------------|---------|
| `ping host` | Check if server is reachable | `-c 4` for 4 packets |
| `curl URL` | Make HTTP request | `-I` headers only, `-v` verbose |
| `wget URL` | Download file | `-c` resume, `-O` custom name |
| `ip addr show` | Show IP addresses | |
| `ifconfig` | Show network interfaces (older) | |
| `ss -tulnp` | Show open ports and listeners | `| grep port` to filter |
| `netstat -tulnp` | Same as ss (older) | |
| `nslookup domain` | Basic DNS lookup | |
| `dig domain` | Detailed DNS lookup | |
| `traceroute host` | Trace network path | |

---

### Interview Questions — Day 6

**Q1. How do you check if a remote server is reachable?**
Using `ping servername` — sends ICMP packets and shows if the server responds. Note: some servers block ping so a failed ping doesn't always mean the server is down.

**Q2. What is the difference between `curl` and `wget`?**
`curl` is primarily for making HTTP requests and testing APIs — outputs to screen by default. `wget` is primarily for downloading files — saves to disk by default and supports resuming interrupted downloads.

**Q3. How do you find which process is using port 3306?**
`ss -tulnp | grep 3306` — shows the process listening on that port.

**Q4. What is DNS and how do you troubleshoot a DNS issue?**
DNS converts domain names to IP addresses. To troubleshoot, use `nslookup` or `dig` to check what IP a domain resolves to and verify it matches the expected server IP.

**Q5. What is the difference between `0.0.0.0` and `127.0.0.1` when a service is listening?**
`0.0.0.0` means the service accepts connections from any network interface — publicly accessible. `127.0.0.1` means only from the same machine — not accessible from outside.

**Q6. A deployment went fine but the application is unreachable from outside. What do you check?**
Check if the app is listening on the right port with `ss -tulnp`. Check if it's on `0.0.0.0` not just `127.0.0.1`. Check firewall. Check DNS. Check service status with `systemctl status`.

---

### Homework — Before Day 7

1. Run `ping -c 4 google.com` — what is the average response time?
2. Run `curl -I http://google.com` — what HTTP status code do you get?
3. Run `ip addr show` — what is your machine's IP address?
4. Run `ss -tulnp` — list 3 services currently listening on your machine
5. Run `nslookup github.com` — what IP does it resolve to?

---
