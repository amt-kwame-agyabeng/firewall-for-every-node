# Firewall for Every Node

## 1. Overview

This project implements firewall configuration and SSH hardening across three Ubuntu 24.04 LTS nodes provisioned with Multipass.

The objective was to:

- Install and enable UFW.
- Deny incoming traffic by default.
- Allow outgoing traffic for package installation and normal operation.
- Allow SSH only from the local Multipass network.
- Change SSH from port `22` to `65123`.
- Disable root login over SSH.
- Disable password-based SSH authentication.
- Enforce public-key authentication.
- Validate firewall rules, SSH access, Internet connectivity, and package installation.

---

## 2. Lab Environment

### Host

- Windows 11 Pro
- WSL2
- Ubuntu
- Multipass 1.16.4
- Multipass QEMU driver

### Nodes

| Node | IP Address | OS |
|---|---|---|
| node1 | `10.192.37.59` | Ubuntu 24.04 LTS |
| node2 | `10.192.37.163` | Ubuntu 24.04 LTS |
| node3 | `10.192.37.14` | Ubuntu 24.04 LTS |

### Network

- Internal subnet: `10.192.37.0/24`
- Gateway: `10.192.37.1`
- Multipass bridge: `mpqemubr0`

### Screenshot 1 — Multipass Lab Environment

Place the screenshot showing:

```bash
multipass list
```

here.

**[INSERT SCREENSHOT 1 HERE]**

---

## 3. Task Requirements

The firewall and SSH configuration needed to satisfy the following requirements:

1. Install and enable UFW.
2. Allow outgoing traffic.
3. Allow incoming SSH only from the local network.
4. Deny public incoming traffic by default.
5. Change SSH port from `22` to `65123`.
6. Disable root login over SSH.
7. Disable password authentication.
8. Enable public-key authentication.
9. Reload and validate UFW.
10. Confirm package installation works while the firewall is enabled.
11. Confirm port `22` is blocked and port `65123` is accessible only through the configured network rule.

---

## 4. Initial UFW Configuration

UFW was configured with the following default policies:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

The resulting policy was:

- Incoming traffic: **DENY**
- Outgoing traffic: **ALLOW**
- Routed/forwarded traffic: **DENY**

UFW was initially inactive while the configuration was prepared.

### Screenshot 2 — Initial UFW Configuration

Place the screenshot showing the default UFW policies and initial status here.

**[INSERT SCREENSHOT 2 HERE]**

---

## 5. Configure SSH Firewall Rule

SSH was moved from the default port `22` to port `65123`.

The SSH firewall rule allows connections only from the local Multipass subnet:

```bash
sudo ufw allow from 10.192.37.0/24 to any port 65123 proto tcp
```

This prevents SSH access from outside the configured local network.

Verify the rule:

```bash
sudo ufw show added
```

Expected rule:

```text
ufw allow from 10.192.37.0/24 to any port 65123 proto tcp
```

### Screenshot 3 — UFW SSH Rule

Place the screenshot showing the configured SSH firewall rule here.

**[INSERT SCREENSHOT 3 HERE]**

---

## 6. SSH Hardening

SSH was hardened on all three nodes using:

```text
/etc/ssh/sshd_config.d/99-hardening.conf
```

The configuration contains:

```text
Port 65123
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

This provides the following controls:

| SSH Control | Configuration |
|---|---|
| SSH Port | `65123` |
| Root Login | Disabled |
| Password Authentication | Disabled |
| Public-Key Authentication | Enabled |

The configuration was validated with:

```bash
sudo sshd -t
```

Then the effective SSH configuration was checked:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'
```

Expected result:

```text
port 65123
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

### Screenshot 4 — SSH Hardening Configuration

Place the screenshot showing `/etc/ssh/sshd_config.d/99-hardening.conf` here.

**[INSERT SCREENSHOT 4 HERE]**

### Screenshot 5 — SSH Listening on Custom Port

Place the screenshot showing:

```bash
sudo ss -lntp | grep ssh
```

The output should show SSH listening on port `65123` and not on port `22`.

**[INSERT SCREENSHOT 5 HERE]**

---

## 7. Ubuntu SSH Socket Activation

Ubuntu 24.04 uses SSH socket activation. Changing `sshd_config` alone can leave the socket listener on port `22`.

A systemd socket override was therefore created:

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
```

Create:

```text
/etc/systemd/system/ssh.socket.d/override.conf
```

with:

```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:65123
ListenStream=[::]:65123
```

Then reload systemd and restart the socket:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

The final listening ports were verified with:

```bash
sudo ss -lntp | grep ssh
```

---

## 8. Multipass Management Connection

After SSH was moved to port `65123`, the normal:

```bash
multipass shell node1
```

management connection could no longer be used because Multipass management expected the original SSH port.

The nodes were instead accessed directly using the Multipass private key:

```bash
sudo ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa \
  -p 65123 ubuntu@<NODE-IP>
```

Example:

```bash
sudo ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa \
  -p 65123 ubuntu@10.192.37.14
```

This also provided evidence that key-based authentication was working.

---

## 9. Enable UFW

After the SSH rule had been staged and SSH hardening had been validated, UFW was enabled:

```bash
sudo ufw enable
```

Then the configuration was verified:

```bash
sudo ufw status verbose
```

Expected final configuration:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
65123/tcp                  ALLOW IN    10.192.37.0/24
```

### Screenshot 6 — UFW Enabled

Place the screenshot showing the final UFW status on one of the nodes here.

**[INSERT SCREENSHOT 6 HERE]**

---

## 10. Node Configuration

The same security configuration was applied to all three nodes.

### node1

IP:

```text
10.192.37.59
```

Security configuration:

- UFW enabled.
- Default incoming policy: DENY.
- Default outgoing policy: ALLOW.
- SSH port: `65123`.
- SSH allowed only from `10.192.37.0/24`.
- Root SSH login disabled.
- Password authentication disabled.
- Public-key authentication enabled.

### Screenshot 7 — node1 SSH and Firewall Validation

Place the screenshot showing node1 SSH hardening and UFW validation here.

**[INSERT SCREENSHOT 7 HERE]**

### node2

IP:

```text
10.192.37.163
```

The same firewall and SSH hardening configuration was applied and validated.

### node3

IP:

```text
10.192.37.14
```

The same firewall and SSH hardening configuration was applied and validated.

### Screenshot 8 — node3 UFW Validation

Place the screenshot showing:

```bash
sudo ufw status verbose
```

on node3 here.

**[INSERT SCREENSHOT 8 HERE]**

---

## 11. Outbound Connectivity and Package Installation

UFW was configured to allow outgoing traffic:

```text
Default: deny (incoming), allow (outgoing), disabled (routed)
```

Internet connectivity was verified using:

```bash
ping -c 4 8.8.8.8
```

Repository access was verified using:

```bash
curl -4 -I --connect-timeout 10 https://archive.ubuntu.com
```

Package installation was validated with:

```bash
sudo apt update
```

The command completed successfully on all three nodes while UFW was enabled.

### Screenshot 9 — Package Installation with Firewall Enabled

Place the screenshot showing successful:

```bash
sudo apt update
```

while UFW is active here.

**[INSERT SCREENSHOT 9 HERE]**

---

## 12. Multipass/WSL Networking Issue

During testing, the nodes could reach the Multipass gateway and resolve DNS records, but external Internet access was initially unavailable.

The issue was not caused by UFW. The Multipass bridge required host-side NAT and forwarding so that traffic from the `10.192.37.0/24` network could reach the WSL host's external interface.

The following rules were configured on the WSL host:

```bash
sudo iptables -t nat -A POSTROUTING \
  -s 10.192.37.0/24 -o eth0 -j MASQUERADE

sudo iptables -A FORWARD \
  -i mpqemubr0 -o eth0 \
  -s 10.192.37.0/24 -j ACCEPT

sudo iptables -A FORWARD \
  -i eth0 -o mpqemubr0 \
  -d 10.192.37.0/24 \
  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

After configuring host-side NAT and forwarding, outbound Internet and package repository access worked with UFW enabled.

### Screenshot 10 — Host-Side NAT/Forwarding

Place the screenshot showing the NAT/forwarding configuration here.

**[INSERT SCREENSHOT 10 HERE]**

---

## 13. Final SSH Port Validation

SSH access was tested from the WSL host against all three nodes.

### Port 22

Port `22` was tested with:

```bash
nc -vz 10.192.37.59 22
nc -vz 10.192.37.163 22
nc -vz 10.192.37.14 22
```

All three connections failed/timed out, confirming that the default SSH port was no longer accessible.

### Port 65123

The custom SSH port was tested with:

```bash
nc -vz 10.192.37.59 65123
nc -vz 10.192.37.163 65123
nc -vz 10.192.37.14 65123
```

All three connections succeeded.

### Screenshot 11 — SSH Port Hardening Validation

Place the screenshot containing the combined `nc` tests here.

**[INSERT SCREENSHOT 11 HERE]**

Expected result:

| Node | Port 22 | Port 65123 |
|---|---|---|
| node1 | Blocked | Accessible |
| node2 | Blocked | Accessible |
| node3 | Blocked | Accessible |

---

## 14. Key-Based Authentication Validation

Key-based authentication was validated using the Multipass private key:

```bash
sudo ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa \
  -p 65123 ubuntu@<NODE-IP>
```

The connection succeeded without password authentication.

The effective SSH configuration was also checked:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'
```

Expected:

```text
port 65123
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

### Screenshot 12 — Key-Based SSH Validation

Place the screenshot showing a successful SSH connection using port `65123` and the key here.

**[INSERT SCREENSHOT 12 HERE]**

---

## 15. Final Security State

The final state across all nodes was:

| Security Control | node1 | node2 | node3 |
|---|---:|---:|---:|
| UFW enabled | ✅ | ✅ | ✅ |
| Default incoming DENY | ✅ | ✅ | ✅ |
| Default outgoing ALLOW | ✅ | ✅ | ✅ |
| SSH on 65123 | ✅ | ✅ | ✅ |
| Port 22 blocked | ✅ | ✅ | ✅ |
| SSH restricted to local subnet | ✅ | ✅ | ✅ |
| Root SSH disabled | ✅ | ✅ | ✅ |
| Password authentication disabled | ✅ | ✅ | ✅ |
| Public-key authentication enabled | ✅ | ✅ | ✅ |
| Internet connectivity | ✅ | ✅ | ✅ |
| `apt update` successful | ✅ | ✅ | ✅ |

---

## 16. Complete Validation Commands

### UFW

```bash
sudo ufw status verbose
sudo ufw show added
```

### SSH

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'
sudo ss -lntp | grep ssh
```

### Network

```bash
ip addr
ip route
ping -c 4 8.8.8.8
curl -4 -I --connect-timeout 10 https://archive.ubuntu.com
```

### Package installation

```bash
sudo apt update
```

### External SSH port validation

```bash
nc -vz 10.192.37.59 22
nc -vz 10.192.37.59 65123

nc -vz 10.192.37.163 22
nc -vz 10.192.37.163 65123

nc -vz 10.192.37.14 22
nc -vz 10.192.37.14 65123
```

---

## 17. Screenshots Checklist

Insert the screenshots at the locations marked above.

| Figure | Screenshot |
|---|---|
| Figure 1 | Multipass lab environment |
| Figure 2 | Initial UFW configuration |
| Figure 3 | UFW SSH rule |
| Figure 4 | SSH hardening configuration |
| Figure 5 | SSH listening on port 65123 |
| Figure 6 | UFW enabled |
| Figure 7 | node1 SSH and firewall validation |
| Figure 8 | node3 UFW validation |
| Figure 9 | Successful `apt update` |
| Figure 10 | Host-side NAT/forwarding |
| Figure 11 | Port 22 blocked / 65123 accessible |
| Figure 12 | Key-based SSH validation |

### Recommended screenshot placement

For the cleanest GitHub README presentation:

```text
README.md
│
├── Screenshot 1 → after Lab Environment
├── Screenshot 2 → after Initial UFW Configuration
├── Screenshot 3 → after SSH Firewall Rule
├── Screenshot 4 → after SSH Hardening
├── Screenshot 5 → after SSH Listening Port
├── Screenshot 6 → after Enable UFW
├── Screenshot 7 → after node1 validation
├── Screenshot 8 → after node3 validation
├── Screenshot 9 → after apt update
├── Screenshot 10 → after NAT/forwarding
├── Screenshot 11 → after final SSH port validation
└── Screenshot 12 → after key-based authentication
```

When the screenshots are added to the repository, replace each placeholder such as:

```text
**[INSERT SCREENSHOT 1 HERE]**
```

with Markdown image syntax, for example:

```markdown
![Multipass Lab Environment](screenshots/01-multipass-list.png)
```

A recommended repository structure is:

```text
firewall-for-every-node/
├── README.md
└── screenshots/
    ├── 01-multipass-list.png
    ├── 02-ufw-initial.png
    ├── 03-ufw-ssh-rule.png
    ├── 04-ssh-hardening.png
    ├── 05-ssh-port.png
    ├── 06-ufw-enabled.png
    ├── 07-node1-validation.png
    ├── 08-node3-validation.png
    ├── 09-apt-update.png
    ├── 10-nat-forwarding.png
    ├── 11-ssh-port-tests.png
    └── 12-key-authentication.png
```

---

## 18. Conclusion

The firewall and SSH hardening requirements were successfully implemented across all three Ubuntu nodes.

The final configuration provides:

- Default-deny incoming firewall policy.
- Allowed outgoing traffic.
- SSH restricted to the local Multipass subnet.
- SSH moved from port `22` to `65123`.
- Root SSH login disabled.
- Password authentication disabled.
- Public-key authentication enabled.
- Port `22` inaccessible.
- Port `65123` accessible through the configured firewall rule.
- Successful Internet connectivity.
- Successful package repository access and `apt update`.

All three nodes were validated individually and through final external port tests.
