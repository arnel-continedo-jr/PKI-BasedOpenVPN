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

### Step 1 — Update the system
```bash
sudo apt update
sudo apt upgrade -y
```
### Step 2 — Install OpenVPN + Easy-RSA
```bash
sudo apt install -y openvpn easy-rsa
```
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

### Step 4 — Generate Server Certificate and Key

#### 4.1 Create the Server Private Key and Certificate Request

```bash
./easyrsa gen-req server nopass
```


