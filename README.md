# README.md
Overview & The "Why" -

This project details the bare-metal provisioning and network configuration of a dedicated WireGuard VPN server.

While commercial VPNs are useful for geographic anonymity, they do not provide access to local home infrastructure and are frequently blocked by commercial firewalls. By building a self-hosted VPN, I established a secure, encrypted tunnel from any unsecured public Wi-Fi network directly into my private home network. This architecture grants me 100% control over my data routing, allows for active network monitoring, and provides secure remote access to my local hardware—an essential pipeline for remote administration and data analysis.

The Hardware & OS Provisioning -

<img width="4284" height="5712" alt="Image" src="https://github.com/user-attachments/assets/4bfc5ea3-158d-4e74-9822-98a7c7ae49e7" />

Motherboard/System: LGA 1150 H81T ITX Motherboard, Core i7-4770S, 8GB of RAM DDR3, 256GB mSATA SSD

Operating System: Ubuntu Server LTS 26.04

Peripherals: Headless (Managed entirely via SSH)

I flashed a USB drive using Rufus to deploy a minimal installation of Ubuntu Server LTS. The LTS release guarantees maximum stability and a multi-year window of enterprise security patches. The installation was kept entirely headless (no GUI) and bypassed logical volume management (LVM) to ensure all system resources and disk space are dedicated strictly to network routing and cryptographic handshakes.

Network Topology -

Traffic Flow: Mobile Client ➔ Public ISP ➔ Modem ➔ Router (Port Forwarding UDP 51820) ➔ Ubuntu ITX Server

Implementation & Code Execution -

Prior to configuring the server, I bound the ITX server's physical MAC address to a permanent static IP in the router's DHCP settings. I then configured the firewall to forward all inbound UDP traffic on port 51820 directly to that local IP. Once the perimeter routing was established, I disconnected all physical peripherals and executed the entire build remotely via SSH.


### PHASE 1: REMOTE CONNECTION & INSTALLATION

# Establish remote connection to the headless ITX server via SSH
ssh username@192.168.x.x

# Update the system's package list to ensure we get the latest software
sudo apt update

# Upgrade all installed packages to their latest security patches
sudo apt upgrade -y

# Install WireGuard, its dependencies, and the QR code generator tool
sudo apt install wireguard wireguard-tools qrencode -y

### PHASE 2: CRYPTOGRAPHIC KEY GENERATION

# Generate the server's private key, save it, generate the public key, and save it
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Generate the mobile client's private key, save it, generate the public key, and save it
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key

### PHASE 3: SERVER CONFIGURATION

# Open the WireGuard server configuration file in a text editor
sudo nano /etc/wireguard/wg0.conf

[Interface]
Address = 10.8.0.1/24                                   # Assigns the VPN its own internal subnet
ListenPort = 51820                                      # The UDP port the server will listen on
PrivateKey = <REDACTED_SERVER_PRIVATE_KEY>              # The server's secret decryption key
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o enp2s0 -j MASQUERADE    # Firewall rules to push traffic out to the internet
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o enp2s0 -j MASQUERADE  # Firewall rules to clean up when the VPN turns off

[Peer]
PublicKey = <REDACTED_CLIENT_PUBLIC_KEY>                # The mobile phone's public lock
AllowedIPs = 10.8.0.2/32                                # The specific IP assigned to the phone

### PHASE 4: KERNEL ROUTING & PERSISTENCE

# Open the Linux system configuration file to enable routing
sudo nano /etc/sysctl.conf

net.ipv4.ip_forward=1                                   # Tells the Linux kernel to allow IP forwarding (act as a router)

# Apply the IP forwarding kernel changes immediately without rebooting
sudo sysctl -p

# Tell Ubuntu to start WireGuard automatically every time the server boots up
sudo systemctl enable wg-quick@wg0

# Start the WireGuard tunnel immediately
sudo systemctl start wg-quick@wg0

### PHASE 5: CLIENT DEPLOYMENT

# Create a new text file for the mobile client's profile
nano client.conf

[Interface]
PrivateKey = <REDACTED_CLIENT_PRIVATE_KEY>              # The phone's secret decryption key
Address = 10.8.0.2/24                                   # The phone's IP address inside the VPN tunnel
DNS = 1.1.1.1, 1.0.0.1                                  # Cloudflare's secure DNS servers for web browsing

[Peer]
PublicKey = <REDACTED_SERVER_PUBLIC_KEY>                # The ITX server's public lock
AllowedIPs = 0.0.0.0/0                                  # Tells the phone to send ALL internet traffic through the VPN
Endpoint = <REDACTED_PUBLIC_WAN_IP>:51820               # The home network's public IP and the open firewall port
PersistentKeepalive = 25                                # Keeps the connection alive if the phone is idle

# Generate a terminal-based QR code from the client config file for instant device scanning
qrencode -t ansiutf8 < client.conf

Summary & Real-World Application -

The infrastructure is now fully operational. When connected to an unsecured public Wi-Fi network, all mobile data is symmetrically encrypted and tunneled directly to this ITX machine before hitting the open web. Local threats and network administrators cannot packet-sniff the traffic, and I retain full administrative access to my home network as if I were sitting securely in my own house.
