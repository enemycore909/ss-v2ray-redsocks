# 🛡️ Shadowsocks + V2Ray + Redsocks 🛡️
## Transparent SOCKS5 proxy stack with WebSocket/TLS obfuscation. 
``
 This content is published for educational purposes only. Do not use on your systems if prohibited by the laws of your country.
``

## 🔎 Project Overview

This project sets up a **transparent proxy/VPN** using:

- **Shadowsocks** (encrypted tunnel)
- **V2Ray-plugin** (WebSocket + TLS obfuscation, looks like normal HTTPS to `www.bing.com`)
- **redsocks + iptables** (transparent redirect of all TCP traffic on the client)

Key properties:

- **Encryption**: `chacha20-ietf-poly1305` (AEAD)
- **Transport**: WebSocket over TLS, `Host: www.bing.com`
- **Transparency**: all outgoing TCP is redirected, apps do not know about proxy

---

## 🧰 Tech Stack

| Component | Role |
| :-- | :-- |
| Shadowsocks | Encrypted TCP/UDP proxy |
| V2Ray-plugin | WebSocket/TLS obfuscation |
| redsocks | Transparent SOCKS5 redirector |
| iptables | NAT + packet redirection |
| systemd | Service management |
| Client OS 
| VPS OS 


---

## 🖥 Server Setup (VPS)

### 1. Install components

```bash
# Shadowsocks-libev
sudo apt update
sudo apt install -y shadowsocks-libev

# V2Ray-plugin
wget https://github.com/teddysun/v2ray-plugin/releases/download/v1.3.2/v2ray-plugin-linux-amd64.tar.gz
tar xzf v2ray-plugin-linux-amd64.tar.gz
sudo mv v2ray-plugin /usr/local/bin/
sudo chmod +x /usr/local/bin/v2ray-plugin
```


### 2. Shadowsocks server config

`/etc/shadowsocks-libev/config-v2ray.json`:

```json
{
  "server": "0.0.0.0",
  "server_port": 9388,
  "password": "CHANGE_ME_TO_STRONG_PASSWORD",
  "timeout": 300,
  "method": "chacha20-ietf-poly1305",
  "mode": "tcp_and_udp",
  "plugin": "/usr/local/bin/v2ray-plugin",
  "plugin_opts": "server;mode=websocket;host=www.bing.com"
}
```


### 3. systemd unit

`/etc/systemd/system/shadowsocks-v2ray.service`:

```ini
[Unit]
Description=Shadowsocks V2Ray Server
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/ss-server -c /etc/shadowsocks-libev/config-v2ray.json -u
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable \& start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable shadowsocks-v2ray.service
sudo systemctl start shadowsocks-v2ray.service
sudo systemctl status shadowsocks-v2ray.service
```

Quick checks:

```bash
sudo journalctl -u shadowsocks-v2ray.service -f
sudo ss -tulpen | grep 9388
ps aux | grep -E "(ss-server|v2ray-plugin)"
```


---

## 💻 Client Setup (Linux / Fedora)

### 1. Install components (example for Fedora)

```bash
# shadowsocks-rust (via cargo)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
cargo install shadowsocks-rust

# v2ray-plugin
wget https://github.com/teddysun/v2ray-plugin/releases/download/v1.3.2/v2ray-plugin-linux-amd64.tar.gz
tar xzf v2ray-plugin-linux-amd64.tar.gz
mv v2ray-plugin $HOME/.cargo/bin/
chmod +x $HOME/.cargo/bin/v2ray-plugin

# redsocks + iptables
sudo dnf install -y redsocks iptables-services
```


### 2. Directory layout

```text
~/shadowsocks/
├── client.json      # Shadowsocks client config
└── vpn-start.sh     # Start script (transparent proxy)
```


### 3. Client config

`~/shadowsocks/client.json`:

```json
{
  "server": "YOUR_IP",
  "server_port": 9388,
  "local_address": "127.0.0.1",
  "local_port": 1080,
  "password": "CHANGE_ME_TO_SAME_PASSWORD_AS_SERVER",
  "method": "chacha20-ietf-poly1305",
  "plugin": "/home/your_user/.cargo/bin/v2ray-plugin",
  "plugin_opts": "mode=websocket;host=www.bing.com"
}
```


### 4. Redsocks config

`/etc/redsocks.conf`:

```ini
base {
    log_debug = off;
    log_info = on;
    log = "file:/var/log/redsocks.log";
    daemon = on;
    redirector = iptables;
}

redsocks {
    local_ip = 127.0.0.1;
    local_port = 12345;
    ip = 127.0.0.1;
    port = 1080;
    type = socks5;
}
```


### 5. Start script

`~/shadowsocks/vpn-start.sh`:

```bash
#!/bin/bash
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo -e "${BLUE}        SHADOWSOCKS VPN START      ${NC}"
echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo ""

# Cleanup previous state
sudo pkill -9 sslocal v2ray-plugin 2>/dev/null
sudo systemctl stop redsocks 2>/dev/null
sudo iptables -t nat -F 2>/dev/null
sudo iptables -t nat -X REDSOCKS 2>/dev/null
sleep 2

# Start Shadowsocks client (+ v2ray plugin via SIP003)
echo -e "${BLUE}[1/3]${NC} Starting Shadowsocks + V2Ray..."
/home/your_user/.cargo/bin/sslocal -c ~/shadowsocks/client.json >/dev/null 2>&1 &
SSLOCAL_PID=$!
sleep 15

# Start redsocks
echo -e "${BLUE}[2/3]${NC} Starting redsocks..."
sudo systemctl start redsocks
sleep 3

# Configure iptables REDSOCKS chain
echo -e "${BLUE}[3/3]${NC} Configuring iptables..."
sudo iptables -t nat -N REDSOCKS
# Exclude VPS and local networks
sudo iptables -t nat -A REDSOCKS -d YOUR_IP -j RETURN
sudo iptables -t nat -A REDSOCKS -d 0.0.0.0/8 -j RETURN
sudo iptables -t nat -A REDSOCKS -d 10.0.0.0/8 -j RETURN
sudo iptables -t nat -A REDSOCKS -d 127.0.0.0/8 -j RETURN
sudo iptables -t nat -A REDSOCKS -d 169.254.0.0/16 -j RETURN
sudo iptables -t nat -A REDSOCKS -d 172.16.0.0/12 -j RETURN
sudo iptables -t nat -A REDSOCKS -d 192.168.0.0/16 -j RETURN
# Redirect all other TCP to redsocks
sudo iptables -t nat -A REDSOCKS -p tcp -j REDIRECT --to-ports 12345
sudo iptables -t nat -A OUTPUT -p tcp -j REDSOCKS

sleep 5
IP=$(curl -s --max-time 10 icanhazip.com)

echo ""
echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
if [ "$IP" == "YOUR_IP" ]; then
    echo -e "${GREEN}✓ VPN ACTIVE${NC}"
    echo -e "   IP: ${GREEN}$IP${NC}"
    echo -e "${GREEN}✓ Shadowsocks PID: $SSLOCAL_PID${NC}"
else
    echo -e "${YELLOW}⚠ Current IP: $IP${NC}"
fi
echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
```

Make it executable:

```bash
chmod +x ~/shadowsocks/vpn-start.sh
```


---

## ▶️ Usage

Start transparent proxy:

```bash
~/shadowsocks/vpn-start.sh
```

Check external IP:

```bash
curl -s icanhazip.com
```

(Should match `YOUR_IP` if tunnel is active.)

Simple stop sequence (can be separate script):

```bash
sudo pkill -9 sslocal v2ray-plugin
sudo systemctl stop redsocks
sudo iptables -t nat -F
sudo iptables -t nat -X REDSOCKS
```


---

## 🛠 Troubleshooting

### 1. No VPN / IP not changed

```bash
# Check processes
ps aux | grep -E "(sslocal|v2ray-plugin|redsocks)" | grep -v grep

# Check iptables NAT rules
sudo iptables -t nat -L -n -v

# Test SOCKS proxy directly
curl --socks5 127.0.0.1:1080 icanhazip.com
```


### 2. Port unreachable from client

On server:

```bash
sudo systemctl status shadowsocks-v2ray.service
sudo ss -tulpen | grep 9388
```

From client:

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/YOUR_IP/9388' && echo "✓ Port reachable" || echo "✗ Port blocked"
```


### 3. Redsocks issues

```bash
sudo systemctl status redsocks
sudo journalctl -u redsocks -f
tail -n 50 /var/log/redsocks.log
```


---

## 🔐 Security Notes

- Replace `CHANGE_ME_TO_STRONG_PASSWORD` with a strong unique password.
- Consider firewall rules on VPS/VDS (allow only needed ports).
- Treat this as a **learning / lab** setup, not a production-grade VPN.

---

## 📚 References

- https://github.com/shadowsocks/shadowsocks-libev
- https://github.com/shadowsocks/shadowsocks-rust
- https://github.com/teddysun/v2ray-plugin
- https://github.com/darkk/redsocks
- https://wiki.archlinux.org/title/iptables
