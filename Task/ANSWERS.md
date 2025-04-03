# Table of Contents

**General Linux Tasks**
- [1. Provide Systematic Access for Other Users to the Machine](#1-provide-systematic-access-for-other-users-to-the-machine)
- [2. Limit Access of Users to Specific Applications/Folders](#2-limit-access-of-users-to-specific-applicationsfolders)
- [3. Multiple Users Plan to Deploy Workloads on This Machine](#3-multiple-users-plan-to-deploy-workloads-on-this-machine)
- [4. Provide a Proxy to a Webserver Claiming Port 443](#4-provide-a-proxy-to-a-webserver-claiming-port-443)

**Network Task**
- [Company Topology Design](#company-topology-design)
- [1. Provide Systematic Access for Other Users to the Machine](#1-provide-systematic-access-for-other-users-to-the-machine)
- [2. Limit Access of Users to Specific Applications/Folders](#2-limit-access-of-users-to-specific-applicationsfolders)
- [3. Prevent Application Starvation Between Users](#3-prevent-application-starvation-between-users)
- [4. Provide a Proxy to a Webserver Claiming Port 443](#4-provide-a-proxy-to-a-webserver-claiming-port-443)

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

# Network Task

## Network Architectures

### North–South Traffic Architectures
These architectures are optimized for traffic between end-users and data centers, or from the edge to the core (north–south direction).

#### Collapsed-Core Architecture
A simplified alternative to the traditional three-tier design, combining the core and distribution layers into one. It reduces cost and complexity, making it ideal for small to medium-sized networks.

#### Three-Tier Architecture
A hierarchical design separating the core, distribution, and access layers. It offers better modularity, scalability, and fault tolerance, and is suited for large enterprise networks with significant north–south traffic.

### East–West Traffic Architecture
These architectures are optimized for server-to-server or intra–data center traffic, i.e., east–west traffic.

#### Spine-Leaf Architecture
A modern data center topology where every leaf switch connects to every spine switch, providing consistent low latency and high bandwidth. It’s ideal for environments with heavy east–west traffic and high scalability demands.

## Recommended Architecture: Collapsed-Core with DMZ + VLAN Segmentation

🔹 **Why Collapsed-Core?**
Threedy is a mid-sized company (100 employees), so a full Three-Tier architecture would be overkill—too complex and costly.

Collapsed-Core gives you simplicity (combined core/distribution), while still supporting advanced segmentation and security needs.

Easily supports multiple VLANs, routing between zones, firewall integration, and VPN termination.

### 🔸 Network Zones (VLANs)
Each group will be isolated into a VLAN for segmentation and security:

| Zone         | VLAN ID | Purpose                        |
|--------------|---------|--------------------------------|
| Developers   | 10      | Internal dev tools, Git, etc   |
| Management   | 20      | Exec communications, reports   |
| Sales        | 30      | CRM, emails, VoIP, etc         |
| HR           | 40      | Employee data, payroll         |
| DMZ          | 50      | External-facing services       |
| VPN Clients  | 60      | Remote access per zone         |

### 🧱 Critical Components (L3-capable hardware suggested)
- **L3 Core/Distribution Switch** (e.g., Cisco Catalyst 9300 or equivalent)
- **Firewall** (e.g., pfSense, Fortinet, Cisco ASA)
- **Router / NAT Gateway**
- **Access Layer Switches** (L2 switches with 802.1Q VLAN support)
- **VPN Server** (can be part of firewall or separate OpenVPN/WireGuard box)
- **WAPs** (with VLAN tagging support for wireless access per zone)

### 🌐 DMZ Handling with a Single Public IP
Since you only have one external IP, here’s how to expose multiple services:

- Use **port forwarding** or **reverse proxying** from a DMZ gateway/firewall.
- Example tools:
  - HAProxy, NGINX, or Apache reverse proxy for HTTP/S
  - Firewall rules for custom ports (e.g., SSH, VPN)
- Place only the **minimal exposed services** in the DMZ (e.g., a web frontend, not the DB)

### 🔐 VPN Access (Per-Zone)
- Users connect via VPN (OpenVPN / WireGuard)
- After authentication, they are placed into their VLAN (via VPN policies or RADIUS)
- Access is restricted to their zone using firewall rules or ACLs

### 🧠 Summary

| Feature         | Implementation                            |
|----------------|--------------------------------------------|
| Architecture    | Collapsed-Core                             |
| Segmentation    | VLANs for each dept. + DMZ                 |
| Internet access | NAT via firewall/router                    |
| DMZ with 1 IP   | Port-forwarding + reverse proxy            |
| VPN             | VLAN-aware + per-zone access               |
| Security        | Inter-VLAN ACLs + centralized firewall     |
| Scalability     | Simple, with room for modular growth       |

