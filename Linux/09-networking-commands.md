# Chapter 9: Linux Networking Commands

## Why does a DevOps engineer need networking commands?

A DevOps engineer often deploys applications on servers. If an application stops working, the first thing to check is: **is the server reachable? is the network fine?** You may need to ping a server, check its IP address, see which ports are open, check packet loss or latency. Networking commands do all of this.

Many networking tools are **not installed by default** on servers. You install them, for example:

```bash
sudo apt install net-tools      # netstat, ifconfig, route, arp
sudo apt install traceroute
sudo apt install wireless-tools # iwconfig
sudo apt install whois
sudo apt install nmap
sudo apt install jq
```

---

## 1. `ping`

Checks whether a server is reachable and whether data is being sent and received.

```bash
ping traininwithshubham.com
```

- Ping sends small **packets** of data and waits for replies.
- Example result: "18 packets transmitted, 18 received, 0% packet loss" means the connection is fine.
- Press **Ctrl + C** to stop.
- Packets carry the data you see on websites, video and voice.

---

## 2. `netstat` – network statistics

Shows network connections, listening ports and more.

```bash
netstat
```

At the top you see **active internet connections**:

- **Protocol** (for example TCP = Transmission Control Protocol).
- **Local address** (your machine and port).
- **Foreign address** (the remote machine and port).
- **State** (for example ESTABLISHED means the connection is working).

If you switch from Wi-Fi to a mobile hotspot, a new connection appears with a new foreign address, while the old one may still be shown in cache for a while.

**Sockets:** a socket is an endpoint used to connect one system to another (network or local). You will see many of them in the output. Focus mainly on the TCP connections.

---

## 3. `ss`

`ss` is a newer version of `netstat`. It gives similar information (socket statistics). Interviewers may ask you to list networking commands, so remember: `netstat`, `ss`, `ping`, `mtr`, `traceroute`, `tracepath`, `dig`, etc.

---

## 4. `ifconfig` – interface configuration

Shows **network interfaces** and their addresses.

```bash
ifconfig
```

Common interfaces:

| Interface | Meaning |
|-----------|---------|
| `eth0` / `ens5` | Ethernet interface (connected through a **NIC**, Network Interface Card). Has the private IP, IPv6, broadcast address, MAC address |
| `lo` | **Loopback** interface: the machine talking to itself (127.0.0.1) |
| `docker0` | Network created by Docker if Docker is installed |

### Loopback and localhost

- `lo` is the loopback interface. Its IP address is **127.0.0.1**.
- The name for this address is **localhost**.
- If you run a React or Node.js app on your laptop, you open it with `localhost` or `127.0.0.1`. It is a local, internal server on your own machine.

---

## 5. `ip addr show`

A modern command that shows all IP addresses and interfaces (loopback, ethernet, etc.).

```bash
ip addr show
```

---

## 6. `traceroute` – see the path packets take

When you open a site, your request does not go directly. It goes from server to server, and each jump is called a **hop**.

```bash
traceroute youtube.com
```

- It shows the list of IP addresses (hops) between you and the destination.
- If you look up these IPs on an "IP lookup" site, you can see roughly where they are, for example a data center in the USA.
- `* * *` (or "no reply") means a hop did not answer. This can indicate packet loss or a firewall.
- After 30 hops, traceroute gives up with "too many hops" if it can not reach.

### `tracepath`
Works like `traceroute` but shows the path (and does not need root).

```bash
tracepath youtube.com
```

Interview answer: "To find how many hops a request takes between source and destination, use **traceroute**."

---

## 7. `mtr` – My Traceroute

`mtr` combines **ping** and **traceroute** in one live tool.

```bash
mtr traininwithshubham.com
```
It shows the path (IP hops) and continuously sends packets, showing loss and latency. It is better than running `ping` and `traceroute` separately.

---

## 8. `nslookup` – DNS lookup

Finds the IP address of a domain name (and vice versa).

```bash
nslookup traininwithshubham.com
```
It shows the server used for the lookup and the **IP address** of the domain, so you know if the domain resolves correctly.

---

## 9. `dig` – deep DNS lookup

`dig` digs for more DNS information about a domain (records, servers, response time).

```bash
dig traininwithshubham.com
```
It tells you where a website is hosted and how DNS answered.

---

## 10. `telnet` – connect to a port

`telnet` connects to a host on a **specific port** to see whether it is reachable.

```bash
telnet traininwithshubham.com 80
telnet traininwithshubham.com 443
```

- Port **80** = **HTTP** (not encrypted, public web).
- Port **443** = **HTTPS** (secure, encrypted web).
- If the connection succeeds you see "Connected to ...". If the port is closed it fails.

(Note: `nslookup` gives the address of a domain, but for testing a **port number** use `telnet`.)

Also, in the recorded lesson the speaker said 443 gives an IPv6 address. That is a mistake in wording: 80 and 443 are just port numbers; IPv4 and IPv6 are different address types.

---

## 11. `hostname`

Shows the **host name** of the computer.

```bash
hostname
```
The hosts file `/etc/hosts` maps IP addresses to host names. For example `127.0.0.1` maps to `localhost`.

---

## 12. `ip a` vs `ifconfig`

`ip addr show` (or `ip a`) is the newer way to see the same information, including loopback and ethernet details, so if `ifconfig` is missing you can still use `ip`.

---

## 13. `iwconfig` – wireless interfaces

For wireless (Wi-Fi) network details you need wireless tools.

```bash
sudo apt install wireless-tools
iwconfig
```
On a cloud EC2 server you will get "no wireless extensions", because an EC2 instance has no Wi-Fi card. On your laptop it would list the Wi-Fi interface.

---

## 14. `whois` – domain registration details

```bash
sudo apt install whois
whois traininwithshubham.com
```
It shows domain name, registrar (where the domain was bought), creation date, expiry date, name servers and more. Big companies also have whois records (for example Google's domain is registered with a corporate registrar).

---

## 15. `nc` (netcat)

`nc` (netcat) can open network connections. It is not commonly used, but good to know.

---

## 16. `arp` – Address Resolution Protocol

Shows the mapping between IP addresses and **MAC addresses** (hardware addresses).

```bash
arp
```
A **MAC address** is the unique hardware address of a network device. A host asks the router for the MAC address of a device, and `arp` shows that mapping. Use it to find the MAC address of an interface like `eth0`.

---

## 17. `ifplugstatus`

Shows whether each network interface is plugged in / working.

```bash
sudo apt install ifplugd
ifplugstatus
```
Example: `lo: link beat detected`, `eth0: link beat detected`, `docker0: unplugged` (meaning not running).

---

## 18. `curl` vs `wget`

### curl – call APIs

An **API** is a middleman. Example: at McDonald's you do not enter the kitchen, you tell the counter person "one veg burger", they bring it. The counter person is the API. The **client** requests, the **server** responds.

`curl` gets a response from a server, and is mostly used to test **API calls**.

```bash
curl -X GET https://dummy.restapiexample.com/api/v1/employees
```
- `-X GET` means use the GET method (fetch data).
- The output may look messy (one long line).

### jq – pretty print JSON

```bash
sudo apt install jq
curl -X GET <api-url> | jq
```
The pipe `|` sends curl output to `jq` which formats the JSON nicely.

### wget – download files

```bash
mkdir downloads
cd downloads
wget <link-address-of-file>
```
`wget` downloads a public file from the internet into the current folder. To get the link, right-click a download and choose "Copy link address".

**Difference:** `curl` is mainly for API calls; `wget` is mainly for downloading files. They can replace each other, but use each for its main purpose.

---

## 19. `route` – routing table

```bash
route
```
Shows the **routing table**: where traffic goes. The destination `0.0.0.0` (default) is for internet access via the **internet gateway**, through the `eth0` interface. This links to the AWS VPC route table concept (which tells how subnets route traffic).

## 20. `iptables`

`iptables` is the firewall rule table (used by Docker as well). It needs root:

```bash
sudo iptables -L
```

---

## 21. `watch` – repeat a command

Runs any command again and again at intervals and shows the live output.

```bash
watch mtr traininwithshubham.com     # runs every 2 seconds
watch -n 5 top                        # runs every 5 seconds
```
Useful when you must repeatedly check the status of something.

---

## 22. `nmap` – network mapper

Scans hosts to see which **ports are open** and what services run.

```bash
sudo apt install nmap
nmap -v traininwithshubham.com
```
`-v` is verbose. It performs DNS resolution then scans, and reports for example that port 80 is open.

Use it to check that only the ports you want are open. **Use nmap only on servers you own or have permission to test.** Scanning other people's systems can be illegal.

---

## Quick reference table

| Command | Use |
|---------|-----|
| `ping` | Is the server reachable? |
| `netstat`, `ss` | Show connections and ports |
| `ifconfig`, `ip addr show` | Show interfaces and IPs |
| `traceroute`, `tracepath` | Show the hops to a destination |
| `mtr` | Live ping + traceroute |
| `nslookup`, `dig` | DNS lookups |
| `telnet` | Test a port |
| `whois` | Domain details |
| `arp` | IP to MAC mapping |
| `curl` | API calls |
| `wget` | Download files |
| `route` | Routing table |
| `iptables` | Firewall rules |
| `watch` | Repeat a command |
| `nmap` | Scan open ports |
