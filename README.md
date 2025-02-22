# **Complete Guide: VPN with Ubuntu 24.04 VM, WireGuard, and FRP**
### **Bypass CGNAT & Securely Access Your Home Network**
---

## **Overview**
This guide will walk you through setting up a **secure VPN** where all traffic routes through your **Home Ubuntu 24.04 VM**, even if it’s behind **CGNAT** or lacks a **public IP**. This is achieved by:
- **Using WireGuard (via PiVPN)** to create a VPN server.
- **Using FRP (Fast Reverse Proxy)** on a **Vultr cloud server** to forward VPN traffic to the local VM.

✅ **Bypass CGNAT** – Forward VPN traffic using FRP.  
✅ **Privacy & Security** – Encrypt all internet traffic via your home network.  
✅ **Persistent Auto-Restart** – Ensure VPN & FRP services restart on boot.  
✅ **Lightweight** – Minimal resource usage.

---

## **Prerequisites**
- **Local Machine (VPN Server):** Ubuntu 24.04 (x86_64 VM).
- **Cloud Server (FRP Proxy):** Vultr, running **Debian 12**.
- **Client Device (VPN Client):** Phone, laptop, or any device with WireGuard.
- **SSH Client:** Terminal (macOS/Linux) or PuTTY (Windows).

---

## **1. Set Up Ubuntu 24.04 VM (VPN Server)**
### **1.1 Install Ubuntu 24.04 in VirtualBox/Proxmox**
1. **Download Ubuntu 24.04 ISO:**  
   👉 [Ubuntu Server Download](https://ubuntu.com/download/server)
2. **Create a New Virtual Machine (VM):**
   - **CPU:** At least 1 vCPU.
   - **RAM:** At least 1GB.
   - **Disk:** 10GB or more.
   - **Network:** **Bridged Adapter** (for external access).
3. **Install Ubuntu**:
   - Select **Minimal Installation**.
   - Enable **OpenSSH Server** during setup.
4. **Access the VM via SSH:**
   ```bash
   ssh user@<UBUNTU_VM_IP>
   ```

### **1.2 Update Ubuntu**
```bash
sudo apt update && sudo apt upgrade -y
```

---

## **2. Install PiVPN (WireGuard)**
1. **Run the PiVPN installation script:**
   ```bash
   curl -L https://install.pivpn.io | bash
   ```
2. **Follow the setup:**
   - Choose **WireGuard** as the VPN protocol.
   - Use **port 51820**.
   - Allow PiVPN to configure the firewall.

3. **Create a VPN Client:**
   ```bash
   pivpn add
   ```
4. **Transfer `.conf` file to the client device:**
   ```bash
   scp user@<UBUNTU_VM_IP>:/home/user/configs/<client>.conf .
   ```

5. **Test VPN locally before proceeding:**
   ```bash
   curl ifconfig.me
   ```
   - If the **public IP matches your home network**, VPN is working.

---

## **3. Set Up Vultr Cloud Server (FRP Proxy)**
### **3.1 Deploy a Vultr Instance**
1. **Create an account:**  
   👉 [Vultr Sign-Up](https://www.vultr.com/)
2. **Deploy a Debian 12 server**:
   - **Location:** Nearest to your home network.
   - **OS:** Debian 12.
   - **Size:** Smallest instance (1 vCPU, 512MB RAM).
   - **Note the public IP**.

### **3.2 Configure Vultr Server**
1. **SSH into the server:**
   ```bash
   ssh root@<VULTR_SERVER_IP>
   ```
2. **Update the system:**
   ```bash
   apt update && apt upgrade -y
   ```

---

## **4. Install and Configure FRP**
### **4.1 Install FRP on Vultr Server (FRP Server)**
1. **Download the latest FRP version dynamically:**
   ```bash
   latest_version=$(curl -s https://api.github.com/repos/fatedier/frp/releases/latest | grep '"tag_name":' | cut -d'"' -f4 | sed 's/v//')
   wget https://github.com/fatedier/frp/releases/download/v${latest_version}/frp_${latest_version}_linux_amd64.tar.gz
   ```
2. **Extract and install:**
   ```bash
   tar -xzf frp_${latest_version}_linux_amd64.tar.gz
   sudo mv frp_${latest_version}_linux_amd64/frps /usr/local/bin/
   sudo chmod +x /usr/local/bin/frps
   sudo mkdir -p /etc/frp
   ```

3. **Configure FRP Server:**
   ```bash
   sudo nano /etc/frp/frps.ini
   ```
   Add:
   ```ini
   [common]
   bind_port = 7000

   [wireguard]
   type = udp
   listen_port = 51820
   ```
4. **Create systemd service for FRP:**
   ```bash
   sudo nano /etc/systemd/system/frps.service
   ```
   ```ini
   [Unit]
   Description=FRP Server
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/frps -c /etc/frp/frps.ini
   Restart=always
   User=root

   [Install]
   WantedBy=multi-user.target
   ```
5. **Enable and start FRP:**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable frps
   sudo systemctl start frps
   sudo systemctl status frps
   ```

---

### **4.2 Install FRP on Ubuntu 24.04 VM (FRP Client)**
1. **Download FRP Client:**
   ```bash
   latest_version=$(curl -s https://api.github.com/repos/fatedier/frp/releases/latest | grep '"tag_name":' | cut -d'"' -f4 | sed 's/v//')
   wget https://github.com/fatedier/frp/releases/download/v${latest_version}/frp_${latest_version}_linux_amd64.tar.gz
   tar -xzf frp_${latest_version}_linux_amd64.tar.gz
   sudo mv frp_${latest_version}_linux_amd64/frpc /usr/local/bin/
   sudo chmod +x /usr/local/bin/frpc
   sudo mkdir -p /etc/frp
   ```
2. **Configure FRP Client:**
   ```bash
   sudo nano /etc/frp/frpc.ini
   ```
   ```ini
   [common]
   server_addr = <VULTR_SERVER_IP>
   server_port = 7000

   [wireguard]
   type = udp
   local_ip = 127.0.0.1
   local_port = 51820
   remote_port = 51820
   ```
3. **Create systemd service for FRP Client:**
   ```bash
   sudo nano /etc/systemd/system/frpc.service
   ```
   ```ini
   [Unit]
   Description=FRP Client
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.ini
   Restart=always
   User=root

   [Install]
   WantedBy=multi-user.target
   ```
4. **Enable and start FRP Client:**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable frpc
   sudo systemctl start frpc
   sudo systemctl status frpc
   ```

---

## **5. Firewall Configuration**
### **On Vultr Server**
```bash
sudo ufw allow 7000/tcp
sudo ufw allow 51820/udp
sudo ufw reload
```

---

## **6. Testing the VPN**
1. **Check if FRP is listening on the Vultr server:**
   ```bash
   sudo ss -tulnp | grep 7000
   ```
2. **Test VPN connectivity from a client device:**
   ```bash
   curl ifconfig.me
   ```
   - The output should show your **home network’s public IP**.
