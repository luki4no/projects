# Table of Contents

- [1. Provide Systematic Access for Other Users to the Machine](#1-provide-systematic-access-for-other-users-to-the-machine)
- [2. Limit Access of Users to Specific Applications/Folders](#2-limit-access-of-users-to-specific-applicationsfolders)
- [3. Prevent Application Starvation Between Users](#3-prevent-application-starvation-between-users)
- [4. Provide a Proxy to a Webserver Claiming Port 443](#4-provide-a-proxy-to-a-webserver-claiming-port-443)
- [Network Task - Company Topology Design](#network-task---company-topology-design)

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

## 3. Prevent Application Starvation Between Users

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

**Config:**
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
- L3 Managed Switch (e.g., UniFi Switch Pro or Aruba 2930)
- Firewall (pfSense/OPNsense appliance)
- Access Points with VLAN support (UniFi or Aruba InstantOn)

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
```

---

