AWS Security Lab
A hands-on cloud security infrastructure project demonstrating enterprise-grade security architecture built on AWS. This lab covers VPC design, EC2 hardening, reverse proxy configuration, VPN setup, firewall management, and IAM access control — all deployed on a live Ubuntu instance.

Architecture Overview

Internet
    │
    ▼
AWS Security Group (perimeter firewall)
    │
    ▼
VPC (10.0.0.0/16)
    └── Public Subnet (10.0.1.0/24)
            └── EC2 Ubuntu t2.micro
                    ├── UFW (host-based firewall)
                    ├── Nginx (reverse proxy + SSL termination)
                    ├── WireGuard VPN (10.8.0.0/24)
                    ├── Honeypot Alert System
                    │       ├── HTTP :80
                    │       ├── SSH :2222
                    │       ├── FTP :21
                    │       └── Telnet :23
                    ├── Flask Dashboard (Gunicorn, auth-protected)
                    └── MongoDB (local, honeypot_db)

                    **Components**
1. VPC & Networking

- Custom VPC with CIDR 10.0.0.0/16
- Public subnet 10.0.1.0/24 with auto-assign public IP
- Internet Gateway attached and associated
- Route table with 0.0.0.0/0 → IGW for outbound internet access

Key lesson: A subnet is not public just because it has a public IP. The route table association is what makes it reachable — a misconfiguration here is one of the most common real-world mistakes in AWS environments.

2. Security Group (Perimeter Firewall)
AWS-level network access control applied before traffic reaches the instance.
Port    Protocol    Direction    Purpose
22      TCP         Inbound      Real SSH access
80      TCP         Inbound      HTTP (Nginx redirect)
443     TCP         Inbound      HTTPS (Nginx)
51820   UDP         Inbound      WireGuard VPN
2222    TCP         Inbound      SSH Honeypot
21      TCP         Inbound      FTP Honeypot
23      TCP         Inbound      Telnet Honeypot
5000    TCP         Inbound      DENIED — Flask internal only
27017   TCP         Inbound      DENIED — MongoDB internal only

3. UFW (Host-Based Firewall)
A second firewall layer running at the OS level, independent of the AWS Security Group. Defence in depth — if a security group misconfiguration opens a port, UFW provides a fallback.

sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 51820/udp
sudo ufw allow 2222/tcp
sudo ufw allow 21/tcp
sudo ufw allow 23/tcp
sudo ufw deny 5000/tcp
sudo ufw deny 27017/tcp
sudo ufw default deny incoming
sudo ufw default allow outgoing

4. Nginx (Reverse Proxy + SSL)
Nginx sits in front of all internal services. Benefits:

- SSL termination — all traffic is HTTPS via Let's Encrypt
- Internal ports (5000) never exposed directly to the internet
- Single entry point for all web traffic
- HTTP → HTTPS redirect enforced

SSL certificate issued via Let's Encrypt / Certbot for adesinaijo.duckdns.org (auto-renewing).

server {
    listen 443 ssl;
    server_name adesinaijo.duckdns.org;

    location /dashboard {
        proxy_pass http://127.0.0.1:5000;
    }

    location /data {
        proxy_pass http://127.0.0.1:5000;
    }

    location / {
        root /var/www/html;
        index index.html;
    }
}

5. WireGuard VPN
WireGuard server running on UDP port 51820, providing an encrypted tunnel for secure access.

Server subnet: 10.8.0.0/24
Server IP on VPN: 10.8.0.1
Split-tunnel configuration — only VPN subnet traffic routes through the tunnel
Key-pair based authentication (no passwords)

Design decision: Split tunnel was chosen over full tunnel to avoid routing all client internet traffic through a t2.micro instance. Full tunnel is available as a configuration upgrade when scaling to a larger instance.

6. Honeypot Alert System
A multi-service deception system deployed to capture and analyse attacker behaviour. See the honeypot repository for full details.
Services emulated:

HTTP on port 80 — captures automated scanners and web bots
SSH on port 2222 — captures brute force credential attacks
FTP on port 21 — captures FTP login attempts
Telnet on port 23 — captures legacy protocol attacks

All events are geolocated (ip-api.com), logged to MongoDB, and visualised on a live dashboard at https://adesinaijo.duckdns.org/dashboard/.
Within minutes of deployment — without any public announcement — automated scanners from the Netherlands, USA, India, Tunisia, and other countries had already discovered and probed the system. This is a live demonstration of why every public-facing server requires layered security.

7. IAM Access Control
AWS IAM groups demonstrating role-based access control (RBAC):

Group                Policies                                                                            Access Level
security-admins      EC2FullAccess, VPCFullAccess, CloudWatchFullAccessFull                              infrastructure
developers           EC2ReadOnly, CloudWatchReadOnly, VPCReadOnly                                        Read-only infrastructure
hr-team              IAM                                                                                 ReadOnlyUser and role visibility only
accounting-team      BillingReadOnly, EC2ReadOnly                                                        Cost and resource visibility

Deployment
Prerequisites

AWS account with free tier access
Domain name or DuckDNS subdomain
Ubuntu 24.04 EC2 instance (t2.micro)

Setup Order

- Create VPC, subnet, internet gateway, route table
- Launch EC2 instance, configure security group
- Install and configure UFW
- Install Nginx, obtain SSL certificate via Certbot
- Install WireGuard, generate server and client keys
- Deploy Honeypot Alert System
- Configure MongoDB
- Set up systemd services for auto-restart
- Configure IAM groups and users

Systemd Services
Both the honeypot and dashboard run as persistent systemd services:

sudo systemctl enable honeypot honeypot-dashboard
sudo systemctl start honeypot honeypot-dashboard

Security Layers Summary
Layer                  Tool                                                            Purpose
Perimeter              AWS Security Group                                              Cloud-level port filtering
Host                   UFW                                                             OS-level firewall, defence in depth
Transport              Let's Encrypt SSL                                               Encrypted traffic
Proxy                  Nginx                                                           Hides internal ports, SSL termination
Access                 Flask Authentication                                            Dashboard credential protection
Tunnel                 WireGuard                                                       Encrypted VPN for direct accessIdentity
AWS                    IAM                                                             Role-based access control

Live Infrastructure

Dashboard: https://adesinaijo.duckdns.org/dashboard/
Landing page: https://adesinaijo.duckdns.org


Related Projects

Honeypot Alert System
Global Vulnerability Aggregator
Security Policy Generator


Author
Ifeoluwa Josiah Adesina
Cybersecurity Specialist | System Architect
GitHub
