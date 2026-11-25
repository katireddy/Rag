# Linux Fundamentals – Distributions, Networking Commands & AWS EC2

Practical revision notes for Linux administration and DevOps interviews. Includes commands, explanations, sample outputs, AWS EC2 login differences, and troubleshooting examples.

---

## 1. Linux distributions (flavours)

Linux has different distributions (also called flavours). They use the Linux kernel but differ in package managers, default configurations, and sometimes service names and utilities.

| Distribution | Common use | Package manager | Package format | Default AWS EC2 username |
|---|---|---|---|---|
| Red Hat Enterprise Linux (RHEL) | Enterprise environments | `dnf` / `yum` | RPM | `ec2-user` |
| Ubuntu | Cloud and DevOps environments | `apt` | DEB | `ubuntu` |
| Amazon Linux 2023 | AWS environments | `dnf` | RPM | `ec2-user` |

**Important:** Many commands are the same across distributions, including `ss`, `ip`, `curl`, `dig`, and `systemctl`, provided the relevant utilities are installed. Differences are more common in package installation commands, package names, default users, and service names.

### Common differences

| Feature | RHEL | Ubuntu |
|---|---|---|
| Package manager | `dnf` / `yum` | `apt` |
| Install package | `dnf install` | `apt install` |
| Refresh package metadata | `dnf makecache` / `dnf check-update` | `apt update` |
| Upgrade packages | `dnf upgrade` | `apt upgrade` |
| Package format | RPM | DEB |
| Common firewall tool | `firewalld` | `ufw` (often) |
| Service manager | `systemd` | `systemd` |

---

## 2. Identify the Linux distribution

When you log in to a Linux machine, first identify which distribution is running.

**Command:**

```bash
cat /etc/os-release
```

**Sample output (Ubuntu):**

```text
PRETTY_NAME="Ubuntu 24.04 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
ID=ubuntu
```

**Sample output (Amazon Linux 2023):**

```text
NAME="Amazon Linux"
VERSION="2023"
ID="amzn"
```

Other useful commands:

```bash
uname -a
uname -r
hostname
whoami
pwd
```

| Command | Purpose |
|---|---|
| `uname -a` | Kernel and system information |
| `uname -r` | Kernel version |
| `hostname` | Displays server hostname |
| `whoami` | Displays current logged-in user |
| `pwd` | Displays current directory |

---

## 3. AWS EC2 – Connect using MobaXterm or SSH

To connect to a Linux EC2 instance, you generally need:

1. Public IP address or public DNS name (or a reachable private address).
2. SSH private key (`.pem`), if the instance uses key-based authentication.
3. Correct SSH username.
4. SSH port (default is TCP 22, unless customized).
5. Network access permitted by the EC2 security group, network ACLs, and routing.

### Common default usernames

| EC2 operating system / AMI | Common default username |
|---|---|
| Amazon Linux 2 | `ec2-user` |
| Amazon Linux 2023 | `ec2-user` |
| Ubuntu | `ubuntu` |
| RHEL | `ec2-user` |
| CentOS | `centos` |
| Debian | `admin` (commonly; check the AMI documentation) |

**Note:** The username depends on the AMI publisher and image configuration. Check the AMI documentation if the default is unclear.

### Example 1: Connect to Ubuntu EC2

```bash
ssh -i mykey.pem ubuntu@13.234.100.10
```

Explanation:

- `ssh` – Starts a Secure Shell connection.
- `-i` – Specifies the identity/private-key file.
- `mykey.pem` – Private key file.
- `ubuntu` – SSH username.
- `13.234.100.10` – Example public IP address.

### Example 2: Connect to Amazon Linux EC2

```bash
ssh -i mykey.pem ec2-user@13.234.100.10
```

### MobaXterm connection checklist

1. Open MobaXterm and create a new SSH session.
2. Enter the EC2 public IP address or DNS name as the remote host.
3. Enable **Specify username** and enter the correct username (`ubuntu` or `ec2-user`, for example).
4. Open **Advanced SSH settings** and select the private key file if key authentication is used.
5. Confirm the SSH port is correct (usually `22`).
6. Connect.

**Interview point:** Different AMIs define different default users. Ubuntu EC2 images commonly use `ubuntu`; Amazon Linux and RHEL EC2 images commonly use `ec2-user`.

---

## 4. Linux networking commands

Networking commands are essential for DevOps work and production troubleshooting.

### A. Check listening ports – `ss`

**Command:**

```bash
sudo ss -tulnp
```

Meaning of each option:

| Option | Meaning |
|---|---|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets |
| `-n` | Numeric addresses and port numbers |
| `-p` | Process information |

**Illustrative sample output:**

```text
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
tcp   LISTEN 0      128    0.0.0.0:22          0.0.0.0:*         users:(("sshd",pid=900,fd=3))
tcp   LISTEN 0      128    0.0.0.0:80          0.0.0.0:*         users:(("nginx",pid=1200,fd=6))
tcp   LISTEN 0      128    0.0.0.0:443         0.0.0.0:*         users:(("nginx",pid=1200,fd=7))
tcp   LISTEN 0      128    127.0.0.1:5432      0.0.0.0:*         users:(("postgres",pid=1500,fd=5))
udp   UNCONN 0      0      0.0.0.0:68          0.0.0.0:*
```

**Understanding the output:**

- Port `22` – Commonly SSH.
- Port `80` – HTTP.
- Port `443` – HTTPS.
- Port `5432` – PostgreSQL.
- Port `68` – Commonly DHCP client traffic.

Check one particular port:

```bash
sudo ss -tulnp | grep ':8080'
```

Illustrative output:

```text
tcp LISTEN 0 128 0.0.0.0:8080 0.0.0.0:* users:(("java",pid=2200,fd=45))
```

This indicates that a TCP service is listening on port 8080.

**Troubleshooting examples:**

- Application not accessible? Check whether its port is listening.
- SSH not working? Check whether `sshd` is listening on port 22.
- Website not loading? Check ports 80 and 443.

**Remember:** A listening port does not automatically mean it is reachable externally. Also check host firewall rules, AWS Security Groups, network ACLs, routing, and the application configuration.

### B. Check IP address – `ip`

Linux uses the `ip` command suite to inspect and manage network interfaces and routes. Windows has the `ipconfig` command; Linux generally does not use `ipconfig` by default.

**Commands:**

```bash
ip addr
ip a
```

**Illustrative sample output:**

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001
    inet 172.31.10.25/20 brd 172.31.15.255 scope global dynamic eth0
    inet6 fe80::1234/64 scope link
```

Explanation:

- `eth0` – Network interface name.
- `172.31.10.25` – Example private IPv4 address.
- `/20` – CIDR prefix length.
- `UP` – Interface is administratively up.

Other useful commands:

```bash
ip link show
ip route
ip route get 8.8.8.8
hostname -I
```

| Command | Purpose |
|---|---|
| `ip addr` | Displays IP addresses |
| `ip link` | Displays network interfaces and link state |
| `ip route` | Displays the routing table |
| `ip route get 8.8.8.8` | Shows the route selected to reach a destination |
| `hostname -I` | Displays assigned IP addresses |

### C. Check DNS – `dig`

`dig` is used to query DNS and troubleshoot name resolution.

**Command:**

```bash
dig google.com
```

**Illustrative sample output:**

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; QUESTION SECTION:
;google.com.             IN      A

;; ANSWER SECTION:
google.com.      300     IN      A       142.250.183.14

;; Query time: 20 msec
;; SERVER: 127.0.0.53#53
```

Explanation:

- `QUESTION SECTION` – The DNS question sent.
- `ANSWER SECTION` – The DNS answer returned.
- `A` – IPv4 address record.
- `NOERROR` – The DNS query completed without a DNS error.
- `Query time` – Time taken for the query.

Other useful commands:

```bash
dig google.com A
dig google.com AAAA
dig google.com MX
dig google.com NS
dig +short google.com
```

| DNS record | Purpose |
|---|---|
| A | IPv4 address |
| AAAA | IPv6 address |
| MX | Mail server |
| CNAME | Canonical name / alias |
| NS | Name server |
| TXT | Text record |

### D. Test HTTP/API connectivity – `curl`

`curl` is used to make requests to URLs and APIs.

Get a response body:

```bash
curl https://google.com
```

Display only response headers:

```bash
curl -I https://google.com
```

**Illustrative sample output:**

```text
HTTP/2 200
content-type: text/html
server: gws
```

Useful options:

```bash
curl -v https://google.com
curl -o output.html https://google.com
curl -L https://google.com
curl -k https://example.com
```

| Command | Purpose |
|---|---|
| `curl -I URL` | Displays response headers |
| `curl -v URL` | Verbose connection and TLS debugging |
| `curl -o file URL` | Saves response body to a file |
| `curl -L URL` | Follows redirects |
| `curl -k URL` | Skips TLS certificate verification (testing only; avoid in production) |

**Real-world DevOps example:** An application is expected to run on port 8080. Test it locally:

```bash
curl -v http://localhost:8080
```

If it responds locally but cannot be reached externally, investigate the host firewall, AWS Security Group, network ACL, routing, load balancer, and application bind address.

---

## 5. Other useful networking commands

| Command | Purpose | Example |
|---|---|---|
| `ping` | Tests basic reachability (if ICMP is allowed) | `ping -c 4 google.com` |
| `traceroute` | Traces network path | `traceroute google.com` |
| `nslookup` | DNS lookup | `nslookup google.com` |
| `hostname -I` | Shows assigned IP addresses | `hostname -I` |
| `ss` | Displays socket information | `ss -tulnp` |
| `netstat` | Legacy network statistics/socket tool | `netstat -tulnp` |
| `nc` | Tests TCP/UDP connectivity | `nc -zv example.com 443` |
| `ip route` | Displays routing information | `ip route` |
| `ethtool` | Displays network interface details | `sudo ethtool eth0` |

**Note:** `traceroute`, `netstat`, `dig`, and `ethtool` may need to be installed separately. Some environments use `tracepath` instead of `traceroute`.

---

## 6. Package installation differences

Package installation commands differ by distribution.

### Ubuntu

```bash
sudo apt update
sudo apt install curl -y
sudo apt install dnsutils -y
sudo apt install net-tools -y
sudo apt install traceroute -y
```

- `apt update` refreshes package indexes; it does not itself upgrade installed packages.
- `apt upgrade` upgrades installed packages.

### Red Hat Enterprise Linux

```bash
sudo dnf makecache
sudo dnf install curl -y
sudo dnf install bind-utils -y
sudo dnf install net-tools -y
sudo dnf install traceroute -y
```

### Amazon Linux 2023

```bash
sudo dnf update -y
sudo dnf install curl -y
sudo dnf install bind-utils -y
sudo dnf install net-tools -y
```

### Common utility package names

| Utility | Ubuntu package | RHEL / Amazon Linux package |
|---|---|---|
| `dig` | `dnsutils` | `bind-utils` |
| `curl` | `curl` | `curl` |
| `netstat` | `net-tools` | `net-tools` |
| `traceroute` | `traceroute` | `traceroute` |

---

## 7. Service management – `systemctl`

Ubuntu and RHEL-based distributions commonly use `systemd` to manage services.

Check SSH service:

**Ubuntu:**

```bash
sudo systemctl status ssh
```

**RHEL / Amazon Linux:**

```bash
sudo systemctl status sshd
```

Other useful commands:

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
sudo systemctl disable nginx
```

| Command | Purpose |
|---|---|
| `start` | Starts a service now |
| `stop` | Stops a service now |
| `restart` | Restarts a service |
| `status` | Shows service status |
| `enable` | Configures service to start at boot |
| `disable` | Disables automatic startup at boot |

**Remember:** Ubuntu commonly names its SSH service `ssh`; RHEL and Amazon Linux commonly use `sshd`. Service names can vary by image and package.

---

## 8. Practical troubleshooting scenario – application not accessible on AWS EC2

**Scenario:** An application deployed on an AWS EC2 Ubuntu instance is not accessible from a browser.

Use a systematic troubleshooting sequence.

### Step 1: Check whether the server is up

```bash
uptime
```

### Step 2: Check the private IP address and interface

```bash
ip addr
```

### Step 3: Check whether the application port is listening

```bash
sudo ss -tulnp
```

For a specific port:

```bash
sudo ss -tulnp | grep ':8080'
```

### Step 4: Check the application service

```bash
sudo systemctl status nginx
```

### Step 5: Test the application locally

```bash
curl -I http://localhost:8080
```

### Step 6: Check DNS resolution

```bash
dig example.com
```

### Step 7: Test remote TCP connectivity

```bash
nc -zv example.com 443
```

### Step 8: Check AWS network configuration

- Security Groups – inbound and outbound rules.
- Network ACLs – subnet-level inbound and outbound rules.
- Route Tables – correct routes for the subnet.
- Internet Gateway – required for appropriate public internet routing.
- Load Balancer – listener rules and target health.
- Subnet and instance addressing – verify the instance has the expected network path.

**Important:** A successful local `curl` test only confirms the local application endpoint responds. It does not prove that external clients can reach it.

---

## 9. Quick revision cheat sheet

| Requirement | Command |
|---|---|
| Check Linux distribution | `cat /etc/os-release` |
| Check kernel | `uname -r` |
| Check logged-in user | `whoami` |
| Check hostname | `hostname` |
| Check IP address | `ip addr` |
| Check listening TCP/UDP ports | `sudo ss -tulnp` |
| Check a specific port | `sudo ss -tulnp \| grep ':8080'` |
| DNS resolution | `dig google.com` |
| Test HTTP response | `curl -I https://google.com` |
| Test basic connectivity | `ping -c 4 google.com` |
| Test TCP port | `nc -zv host 443` |
| Check routing | `ip route` |
| Check service | `systemctl status nginx` |
| Check disk space | `df -h` |
| Check memory | `free -h` |
| Check processes | `ps aux` |
| Monitor processes | `top` |

---

## 10. Interview questions – quick self-test

1. Which command displays listening TCP and UDP ports along with process information?
   - Answer: `sudo ss -tulnp`

2. What is the common default AWS EC2 username for Ubuntu?
   - Answer: `ubuntu`

3. What is the common default AWS EC2 username for Amazon Linux?
   - Answer: `ec2-user`

4. Which command is used for DNS troubleshooting?
   - Answer: `dig`

5. Which package manager does Ubuntu use?
   - Answer: `apt`

6. Which command checks the local application's HTTP response?
   - Answer: `curl`

7. Does a listening port guarantee that the application is reachable from the internet?
   - Answer: No. Check the host firewall, Security Groups, network ACLs, routing, and application configuration.

8. What is the difference between `systemctl start` and `systemctl enable`?
   - Answer: `start` starts the service now; `enable` configures it to start automatically during boot.

---

## 11. Interview explanation – sample answer

"Linux has multiple distributions, such as Red Hat, Ubuntu, and Amazon Linux. The core Linux commands are largely similar, but package managers, package names, default usernames, and some service names can differ.

For example, Ubuntu uses `apt` for package management, whereas RHEL uses `dnf` or `yum`. When connecting to AWS EC2, the common default username is `ubuntu` for Ubuntu AMIs and `ec2-user` for Amazon Linux and RHEL AMIs.

For networking troubleshooting, I use `ip addr` to check IP addresses, `ss -tulnp` to identify listening ports and processes, `dig` to troubleshoot DNS resolution, and `curl` to test HTTP endpoints. I also use `systemctl` to check application services. If the application works locally but is not accessible externally, I investigate the host firewall, AWS Security Groups, network ACLs, route tables, and load balancer configuration."

---

## 12. Practical revision routine

Try these commands on a test Linux VM or EC2 instance:

```bash
cat /etc/os-release
whoami
hostname
ip addr
ip route
sudo ss -tulnp
dig google.com
curl -I https://google.com
df -h
free -h
ps aux
systemctl status ssh
```

For each command, practise explaining:

1. What does the command do?
2. What does its output mean?
3. When would you use it in a production incident?
4. What would you check next if the output showed a problem?

**Final interview tip:** Do not only memorize Linux commands. Explain what problem each command solves and how you use its output to decide the next troubleshooting step.
