# Table of Contents

**General Linux Tasks**
- [1. Provide Systematic Access for Other Users to the Machine](#1-provide-systematic-access-for-other-users-to-the-machine)
- [2. Limit Access of Users to Specific Applications/Folders](#2-limit-access-of-users-to-specific-applicationsfolders)
- [3. Multiple Users Plan to Deploy Workloads on This Machine](#3-multiple-users-plan-to-deploy-workloads-on-this-machine)
- [4. Provide a Proxy to a Webserver Claiming Port 443](#4-provide-a-proxy-to-a-webserver-claiming-port-443)

**Network Task**
- [Network Task - Company Topology Design](#network-task---company-topology-design)
- [1. Provide Systematic Access for Other Users to the Machine](#1-provide-systematic-access-for-other-users-to-the-machine)
- [2. Limit Access of Users to Specific Applications/Folders](#2-limit-access-of-users-to-specific-applicationsfolders)
- [3. Prevent Application Starvation Between Users](#3-prevent-application-starvation-between-users)
- [4. Provide a Proxy to a Webserver Claiming Port 443](#4-provide-a-proxy-to-a-webserver-claiming-port-443)

**Summary**
- [Summary](#Summary)

# General Linux Tasks

## 1. Provide Systematic Access for Other Users to the Machine

**Technologies/Products to Use:**
- SSH with Key-Based Authentication
- Ansible for user provisioning and configuration
- LDAP or FreeIPA for centralized identity management (optional for scale)

**Approach:**
1. Create user accounts:
   ```bash
   sudo adduser alice
   sudo adduser bob
   ```
2. Set up SSH key access:
   ```bash
   mkdir /home/alice/.ssh
   echo "ssh-rsa AAAAB3... alice@host" > /home/alice/.ssh/authorized_keys
   chown -R alice:alice /home/alice/.ssh
   chmod 700 /home/alice/.ssh
   chmod 600 /home/alice/.ssh/authorized_keys
   ```
3. Use `sudo` privileges through a group (e.g., `sudo` or a custom admin group)
   ```bash
   usermod -aG sudo alice
   ```
4. Manage users via Ansible playbooks for automation and consistency.

**Sample Ansible Playbook (add_users.yml):**
```yaml
- name: Add users and configure SSH access
  hosts: all
  become: true
  vars:
    users:
      - name: alice
        key: "ssh-rsa AAAAB3... alice@host"
        groups: ["sudo"]
      - name: bob
        key: "ssh-rsa AAAAB3... bob@host"
        groups: []

  tasks:
    - name: Ensure users exist
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups | join(',') }}"
        append: yes
        shell: /bin/bash
        state: present
      loop: "{{ users }}"

    - name: Create .ssh directory
      file:
        path: "/home/{{ item.name }}/.ssh"
        state: directory
        owner: "{{ item.name }}"
        group: "{{ item.name }}"
        mode: '0700'
      loop: "{{ users }}"

    - name: Add SSH public key
      copy:
        content: "{{ item.key }}"
        dest: "/home/{{ item.name }}/.ssh/authorized_keys"
        owner: "{{ item.name }}"
        group: "{{ item.name }}"
        mode: '0600'
      loop: "{{ users }}"
```

## 2. Limit Access of Users to Specific Applications/Folders

**Approach:**
- Use group-based permissions.
- Use `chroot` jails or `AppArmor`/`SELinux` for stricter isolation if necessary.

**What are AppArmor and SELinux?**
- **AppArmor** (Application Armor) is a Linux security module that uses profiles to restrict the capabilities of programs. It operates on a per-application basis and is relatively simple to configure.
- **SELinux** (Security-Enhanced Linux) is a more advanced and granular security module that uses mandatory access controls (MAC) to enforce policies on all processes, users, and files. It is powerful but has a steeper learning curve and is common in Red Hat-based systems.

**Implementation:**

**A. chroot Jail (Basic Example for `alice`)**
```bash
# Create directory structure
sudo mkdir -p /jail/home/alice
sudo usermod --home /home/alice --shell /bin/bash --move-home alice
sudo usermod -d /home/alice -m alice

# Copy necessary binaries and libraries
sudo cp /bin/bash /jail/bin/
sudo mkdir -p /jail/lib/x86_64-linux-gnu
ldd /bin/bash  # Identify dependent libraries
sudo cp /lib/x86_64-linux-gnu/libtinfo.so.6 /jail/lib/x86_64-linux-gnu/
sudo cp /lib/x86_64-linux-gnu/libc.so.6 /jail/lib/x86_64-linux-gnu/

# Set chroot environment using PAM or manually in SSH config
sudo vim /etc/ssh/sshd_config

# Add:
Match User alice
    ChrootDirectory /jail
    AllowTCPForwarding no
    ForceCommand internal-sftp

sudo systemctl restart sshd
```

**B. AppArmor (Restricting `alice` to specific paths)**
```bash
# Create a simple profile for a restricted application
sudo aa-genprof /usr/bin/vim
# Follow the prompts and run the application to generate access logs
sudo aa-logprof  # Use to refine permissions

# Or manually create profile in /etc/apparmor.d/
```

**C. SELinux (RHEL/CentOS Based)**
```bash
# View current contexts
ls -Z /opt/webapp

# Change type to a more restrictive domain if needed
sudo semanage fcontext -a -t httpd_sys_content_t '/opt/webapp(/.*)?'
sudo restorecon -Rv /opt/webapp

# Use booleans to fine-tune access
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
```

**Example:**
```bash
# Create a group and assign access
sudo groupadd webdev
sudo usermod -aG webdev alice
sudo chown :webdev /opt/webapp
sudo chmod 750 /opt/webapp
```

- Restrict shell access using `rbash`:
```bash
sudo ln -s /bin/bash /bin/rbash
usermod -s /bin/rbash alice
```

## 3. Multiple Users Plan to Deploy Workloads on This Machine

**Approach 1: Systemd Resource Control**
```bash
# Example unit override
sudo systemctl edit myapp.service

[Service]
CPUQuota=50%
MemoryMax=512M
```

**Approach 2: cgroups directly (via systemd or cgroup tools)**
```bash
sudo cgcreate -g memory,cpu:/devgroup
sudo cgset -r memory.limit_in_bytes=512M devgroup
sudo cgset -r cpu.shares=512 devgroup
cgexec -g memory,cpu:devgroup /opt/myapp
```

## 4. Provide a Proxy to a Webserver Claiming Port 443

**Tool: NGINX (recommended for performance + flexibility)**

**Bare-Metal or Host-Based Config:**
```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
}
```

**Alternative: Docker-Based Implementation**

*Advantages: Easier portability, encapsulated config, fast redeployment.*

**Dockerfile (for NGINX reverse proxy)**
```dockerfile
FROM nginx:alpine
COPY default.conf /etc/nginx/conf.d/default.conf
```

**default.conf**
```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**docker-compose.yml**
```yaml
version: '3'
services:
  reverse-proxy:
    build: .
    ports:
      - "443:443"
    volumes:
      - ./certs:/etc/nginx/certs:ro

  backend:
    image: my-backend-app:latest
    expose:
      - "8080"
```

This setup ensures a clean separation of proxy logic, can be easily deployed on any host, and fits well in containerized or hybrid environments.

---

## Network Task - Company Topology Design

### Assumptions
- Company Size: ~100 employees
- Roles: Developers, Management, Sales, HR
- Network Zones: Dev, Mgmt, Sales, HR, DMZ
- Single external static IP address

### Topology Overview (Text-Based Markdown Format)

```markdown
+------------------+       +------------------+
|  Internet        |-------|  Firewall (e.g.,  |
|                  |       |  pfSense/OPNsense)|
+--------+---------+       +--------+---------+
                          | NAT, VPN, ACLs    |
                          +--------+---------+
                                   |
                     +-------------+-------------+
                     | VLAN Trunk via L3 Switch  |
                     +--------------------------+
                       |     |     |     |     |
                    VLAN10 VLAN20 VLAN30 VLAN40 VLAN50
                     Dev   Mgmt   Sales   HR     DMZ

                    [VPN Access via Firewall for each VLAN]

       - Each VLAN uses DHCP with static reservations for key devices
       - Inter-VLAN routing only where necessary (e.g., Dev to DMZ)
       - VLAN tagging handled at switch and AP level
       - Wi-Fi SSIDs mapped to VLANs
```

### Network Hardware
Three alternative setups are proposed based on budget and performance considerations:

**🟢 Cost-Effective Setup (Budget-Conscious)**
- **Firewall:** pfSense on a low-power appliance or virtual machine (e.g., Protectli Vault, used Dell OptiPlex with dual NICs)
- **Switch:** TP-Link JetStream TL-SG3210XHP-M2 or Netgear GS110EMX (L2+/L3-lite with VLAN support)
- **Access Points:** TP-Link Omada EAP245 or Ubiquiti UniFi U6-Lite (affordable, VLAN-capable)

**🔵 Higher-End Setup (Performance & Reliability Focus)**
- **Firewall:** OPNsense running on Netgate 6100 or similar business-class appliance
- **Switch:** UniFi Switch Pro 24 or Aruba 2930F (fully managed L3 switch)
- **Access Points:** Aruba InstantOn AP22 or UniFi U6-Pro (Wi-Fi 6, enterprise-grade)

**🟣 High-End Enterprise Setup (Scalability & Compliance Focus)**
- **Firewall:** Palo Alto PA-440 or Fortinet FortiGate 60F with unified threat management (UTM)
- **Switch:** Cisco Catalyst 9300 or Aruba CX 6200 (full L3 with advanced QoS and telemetry)
- **Access Points:** Cisco Meraki MR46 or Aruba AP-635 (Wi-Fi 6/6E, with cloud-managed control)

All setups support VLAN segmentation and offer scalable security layers, with the high-end solution ideal for regulated or highly secure environments.

### Security Measures
- Firewall ACLs + stateful rules to restrict VLAN traffic
- DMZ isolated with access to necessary services only (e.g., web apps)
- HTTPS with valid certificates
- IPS/IDS like Suricata or Snort on pfSense
- MFA for VPN and admin interfaces
- Central log collection with Graylog/ELK

### External IP Handling
- Use reverse proxy (NGINX/HAProxy) on DMZ host to route traffic based on hostname
- Example:
```nginx
server {
    listen 443 ssl;
    server_name dev.threedy.io;
    proxy_pass http://192.168.50.10;
}
```

### VPN Access
- Per-group OpenVPN/WireGuard profiles via pfSense or standalone VPN server
- Access restricted to specific VLAN based on user role

### Cost and Time Estimate
- **Hardware**: ~€10,000 (firewalls, switches, APs)
- **Initial Setup**: 2–3 weeks
- **Ongoing Maintenance**: ~5 hrs/month
- **Licensing**: Prefer open-source (pfSense, WireGuard, NGINX), optional RADIUS or SSO integration

# Summary

## 📝 Summary of Key Discussion Points

### 🔐 User Management & Access Control
- Created user accounts with SSH key-based login (`adduser`, `.ssh/authorized_keys`)
- Used **Ansible playbook** for automated user provisioning
- Applied **group-based sudo** and file access control
- Introduced optional use of **LDAP/FreeIPA** for scalable identity management

### 🛡️ Application/Folders Access Limitation
- Group permissions + RBAC (`groupadd`, `chmod`, `chown`)
- Explained **chroot jail** setup for shell isolation
- Demonstrated **AppArmor** and **SELinux** usage
  - Defined both
  - Showed example profile application (e.g., restricting `vim` or `httpd`)
- Mentioned `rbash` to limit user command access

### ⚙️ Resource Isolation (Multi-user Workload)
- **Systemd resource control** (`CPUQuota`, `MemoryMax`)
- **cgroups** for CPU/memory limits per workload/user
- Ensures one user/app doesn't starve system resources

### 🌐 Proxy Server Setup
- NGINX reverse proxy config on **port 443** with TLS
- Optimized for **security** (`ssl_ciphers`, `protocols`) and **performance**
- Used proxy headers to retain client info

### 🕸️ Network Architecture Design
- Designed **scalable network** for 100-employee org
- VLANs: Dev, Mgmt, Sales, HR, DMZ (via L3 switch)
- pfSense/OPNsense firewall with **VPN** per zone
- DMZ serves select services using single public IP with reverse proxy

### 🧱 Security Architecture
- Firewall ACLs, IDS/IPS (Suricata), strict VLAN routing
- HTTPS enforcement and **MFA for VPN**
- Centralized logging (Graylog/ELK), future-ready

### 💸 Time & Budget Estimate
- Hardware cost: ~€10K
- Initial setup: 2–3 weeks
- Monthly maintenance: ~5 hours
- Preference for **open-source solutions** (license savings)
