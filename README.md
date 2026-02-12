<img width="9000" height="500" src="https://github.com/arnel-continedo-jr/PKI-BasedOpenVPN/blob/main/images/PKI-based%20OpenVPN%20infrastructure%20design.png" />


# PKI-BasedOpenVPN
Design and implementation of a PKI-based OpenVPN infrastructure on Ubuntu. This project demonstrates certificate generation with Easy-RSA, secure client-server configuration, NAT and IP forwarding, multi-client support (Windows, macOS, Linux), and encrypted full-tunnel VPN deployment in a virtualized lab environment.

# Overview

This project demonstrates the design and implementation of a PKI-based OpenVPN infrastructure deployed on Ubuntu. It covers secure certificate generation using Easy-RSA, server and multi-client configuration, NAT and IP forwarding, encrypted tunneling over a dedicated VPN subnet (10.8.0.0/24), and cross-platform client connectivity (Windows, macOS, Linux).

The lab simulates a real-world VPN deployment, including secure key management, TLS encryption, full-tunnel routing, and secure remote access within a virtualized environment.

## 🎯 Core Capabilities

| Function | Purpose |
|----------|---------|
| PKI Infrastructure | Generates and manages certificates using Easy-RSA for secure client authentication. |
| Encrypted Tunneling | Establishes AES-encrypted VPN tunnels over TLS 1.3. |
| Multi-Client Support | Allows multiple devices (Windows, macOS, Linux) to connect simultaneously. |
| Dedicated VPN Subnet | Creates isolated tunnel network (10.8.0.0/24) for secure traffic segmentation. |
| NAT & IP Forwarding | Enables full-tunnel internet routing through the VPN server. |
| Cross-Platform Connectivity | Supports OpenVPN clients across major operating systems. |
| Secure Key Management | Implements certificate-based authentication instead of passwords. |
| Service Automation | Uses systemd to manage OpenVPN as a persistent background service. |

## 📋 Requirements

### 🖥 Server
- Ubuntu Server (20.04+ recommended)
- OpenVPN (2.5+)
- Easy-RSA (3.x)
- OpenSSH Server (for remote access)
- UFW or iptables for firewall configuration

### 🌐 Network
- Static private IP for VPN server
- UDP port 1194 open (or custom port)
- IP forwarding enabled
- NAT (MASQUERADE rule) configured

### 💻 Client Devices
- OpenVPN Client (Windows, macOS, or Linux)
- Tunnelblick (macOS) or OpenVPN GUI (Windows)
- Secure transfer method for `.ovpn` files (SCP/SFTP)

### 🛠 Virtualization (Lab Environment)
- VirtualBox or VMware (optional)
- Bridged or NAT networking properly configured

## 🔧 Step-by-Step Guide

---

### Step 1 — Update the system
```bash
sudo apt update
sudo apt upgrade -y
```

---

### Step 2 — Install OpenVPN + Easy-RSA
```bash
sudo apt install -y openvpn easy-rsa
```
---

### Step 3 — Set up the Certificate Authority (Easy-RSA)

#### 3.1 Create the Easy-RSA working directory
```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```

#### 3.2 Edit the vars file (certificate identity fields)
```bash
nano vars
```

Add or update values like these (use your own values — they are labels/metadata only):

```bash
set_var EASYRSA_REQ_COUNTRY    "US"
set_var EASYRSA_REQ_PROVINCE   "California"
set_var EASYRSA_REQ_CITY       "San Francisco"
set_var EASYRSA_REQ_ORG        "MyOrg"
set_var EASYRSA_REQ_EMAIL      "email@example.com"
set_var EASYRSA_REQ_OU         "MyOrgUnit"
```

Note: These fields do not control where your VPN traffic appears to come from. Only the VPN server’s actual IP/location affects that.

#### 3.3 Build the Certificate Authority
```bash
•/easyrsa init-pki
•/easyrsa build-ca
```

---

### Step 4 — Generate Server Certificate and Key

#### 4.1 Create the Server Private Key and Certificate Request

```bash
./easyrsa gen-req server nopass
```

This creates:

server.key (private key)
server.req (certificate request) 

#### 4.2 Sign the Server Certificate with the CA
```bash
./easyrsa sign-req server server
```

When prompted, type:
yes

This generates:
pki/issued/server.crt (signed server certificate)

#### 4.3 Generate Diffie-Hellman Parameters
```bash
./easyrsa gen-dh
```
This creates:
pki/dh.pem

#### 4.4 Generate HMAC TLS Key
```bash
openvpn --genkey --secret ta.key
```
This creates:
ta.key (used for TLS authentication and protection against DoS attacks)

---

### Step 5 — Configure the OpenVPN Server

#### 5.1 Copy Required Certificate and Key Files

```bash
sudo mkdir -p /etc/openvpn/server
sudo cp pki/ca.crt \
pki/private/server.key \
pki/issued/server.crt \
pki/dh.pem \
ta.key \
/etc/openvpn/server/
```

#### 5.2 Create the OpenVPN Server Configuration File
```bash
sudo nano /etc/openvpn/server/server.conf
```
Add the following configuration:

```conf
port 1194
proto udp
dev tun

ca ca.crt
cert server.crt
key server.key
dh dh.pem

tls-auth ta.key 0

server 10.8.0.0 255.255.255.0

cipher AES-256-CBC
auth SHA256

user nobody
group nogroup

persist-key
persist-tun

status openvpn-status.log
log-append /var/log/openvpn.log
verb 3
```

#### 5.3 Enable IP Forwarding

Edit sysctl configuration:
```bash
sudo nano /etc/sysctl.conf
```

Uncomment:
```ini
net.ipv4.ip_forward=1
```

Apply changes:
```bash
sudo sysctl -p
```

#### 5.4 Configure UFW Firewall Rules

Allow OpenVPN and SSH:
```bash
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
```

Edit UFW rules:
```bash
sudo nano /etc/ufw/before.rules
```

Add the following lines at the top of the file:
```conf
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
COMMIT
```

Replace eth0 with your actual network interface (e.g., enp0s3).

Enable UFW:
```bash
sudo ufw enable
```
---

### Step 6 — Start and Enable the OpenVPN Server

Start the OpenVPN service:

```bash
sudo systemctl start openvpn-server@server
```
Enable the service to start automatically on boot:
```bash
sudo systemctl enable openvpn-server@server
```

Verify that the service is running:
```bash
sudo systemctl status openvpn-server@server
```

If successful, the service should show as:
active (running)

---

### Step 7 — Generate Client Certificates and Configuration

#### 7.1 Generate Client Certificate and Key
```bash
cd ~/openvpn-ca
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

Confirm with yes when prompted.

This generates:

client1.key
client1.crt

#### 7.2 Create the Client Configuration Directory
```bash
mkdir -p ~/client-configs/keys
```

Copy required files:
```bash
cp pki/ca.crt \
pki/issued/client1.crt \
pki/private/client1.key \
~/client-configs/keys/

cp ta.key ~/client-configs/keys/
```

#### 7.3 Create the Base Client Configuration File
```bash
nano ~/client-configs/base.conf
```

Add:
```conf
client
dev tun
proto udp
remote YOUR_SERVER_IP 1194

resolv-retry infinite
nobind

persist-key
persist-tun

remote-cert-tls server

cipher AES-256-CBC
auth SHA256

key-direction 1
verb 3
```

Replace YOUR_SERVER_IP with your actual server IP address.

#### 7.4 Create Script to Package Client Configuration

Create directory:
```bash
mkdir -p ~/client-configs/files
```

Create script:
```bash
nano ~/client-configs/make_config.sh
```

Add:
```bash
#!/bin/bash

KEY_DIR=~/client-configs/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```

Make it executable:
```bash
chmod +x ~/client-configs/make_config.sh
```

Generate client configuration file:
```bash
cd ~/client-configs
./make_config.sh client1
```

This will generate:
```bash
~/client-configs/files/client1.ovpn
```

---

### Step 8 — Configure the Client

#### 8.1 Transfer the `.ovpn` File to the Client Device

Use a secure transfer method such as:

- SCP
- SFTP
- USB transfer
- Encrypted cloud storage

Example using SCP:

```bash
scp ~/client-configs/files/client1.ovpn user@CLIENT_IP:/home/user/
```

#### 8.2 Install OpenVPN on the Client
🐧 Linux
Install OpenVPN:
```bash
sudo apt install -y openvpn
```

Connect using:
```bash
sudo openvpn --config client1.ovpn
```

🪟 Windows

1. Download OpenVPN GUI from:
https://openvpn.net/community-downloads/

2. Install the application.

3. Place client1.ovpn in:
```arduino
C:\Program Files\OpenVPN\config\
```

4. Run OpenVPN GUI as Administrator.

5, Right-click the tray icon → Connect.

🍎 macOS

1. Download Tunnelblick:
https://tunnelblick.net/

2. Install the application.

3. Double-click client1.ovpn to import.

4. Click Connect.
   
--- 

### Step 9 — Test the VPN Connection
#### 9.1 Connect to the VPN

Start the OpenVPN client using client1.ovpn.

If successful, you should see:
```mathematica
Initialization Sequence Completed
```

#### 9.2 Verify Your Public IP Address

Check if your public IP matches the VPN server’s IP.

Linux / macOS
```bash
curl ifconfig.me
```

Windows

Open Command Prompt:
```bash
curl ifconfig.me
```
If the IP matches your Ubuntu server, traffic is routed through the VPN.

#### 9.3 Verify Tunnel Interface

On Linux/macOS:
```bash
ip a
```

You should see a tun0 interface with an IP like:
```bash
10.8.0.x
```

#### 9.4 Test Network Access

Ping the VPN server:
```bash
ping 10.8.0.1
```

You should receive replies, confirming the encrypted tunnel is active.


