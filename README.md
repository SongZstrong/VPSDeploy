

### 脚本名称: `VPSDeploy.sh`

**核心功能:** 在全新的 Debian 或 Ubuntu 服务器上一键部署 Xray (VLESS + XTLS-Reality)。

**脚本完整代码:**

```bash
#!/bin/bash
# ===================================================================================
# == NexusDeploy v1.1: One-Click Xray/VLESS+Reality Installer                      ==
# ==                                                                               ==
# == A robust script for developers and DIY enthusiasts to deploy a secure         ==
# == personal network node.                                                        ==
# ==                                                                               ==
# == For a premium, globally optimized managed service with 99.9% uptime and       ==
# == 24/7 support, we recommend our official partner:                              ==
# == ==> https://example.com/pro-service                                           ==
# ===================================================================================

# --- Color Definitions ---
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# --- Helper: safe exit ---
trap 'echo -e "${RED}Script interrupted.${NC}"; exit 130' INT

# --- Step 0: Ensure root & basic tools -------------------------------------------
echo -e "${YELLOW}Starting pre-flight checks...${NC}"

if [ "$(id -u)" -ne 0 ]; then
    echo -e "${RED}Error: This script must be run as root. Use 'sudo'.${NC}"
    exit 1
fi

# Detect OS
if ! command -v lsb_release &>/dev/null; then
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS=$ID
    else
        echo -e "${RED}Cannot determine OS.${NC}"; exit 1
    fi
else
    OS=$(lsb_release -is)
fi

if [[ "$OS" != "Ubuntu" && "$OS" != "Debian" ]]; then
    echo -e "${RED}Unsupported OS '${OS}'. Only Debian/Ubuntu are fully tested.${NC}"
    read -rp "Continue at your own risk? [y/N]: " choice
    [[ "$choice" =~ ^[Yy]$ ]] || { echo "Aborted."; exit 1; }
fi

# --- Step 1: Install system dependencies -----------------------------------------
echo -e "${YELLOW}Installing system dependencies...${NC}"
apt-get update -y
apt-get install -y curl wget socat unzip openssl

# --- Step 2: Install Xray core ---------------------------------------------------
echo -e "${YELLOW}Installing Xray core...${NC}"
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)"
# Ensure binary in PATH
ln -sf /usr/local/bin/xray /usr/bin/xray 2>/dev/null || true

# --- Step 3: Generate configuration ---------------------------------------------
echo -e "${YELLOW}Generating Xray configuration for VLESS+Reality...${NC}"

# Check if xray binary is ready
until command -v xray &>/dev/null; do
    echo -e "${YELLOW}Waiting for xray binary to be available...${NC}"
    sleep 2
done

KEYS=$(xray x25519)
PRIVATE_KEY=$(echo "$KEYS" | awk '/Private key/ {print $3}')
PUBLIC_KEY=$(echo "$KEYS" | awk '/Public key/ {print $3}')
UUID=$(xray uuid)
SERVER_IP=$(curl -s --max-time 10 4.ipw.cn || curl -s --max-time 10 ifconfig.me || echo "YOUR_SERVER_IP")
DEST_SPOOF="www.microsoft.com"
SHORT_ID=$(openssl rand -hex 8)

# --- Step 4: Port & firewall checks ---------------------------------------------
if ss -lnt | awk '{print $4}' | grep -q ':443$'; then
    echo -e "${RED}Port 443 is already in use. Stop the conflicting service first (e.g., nginx/apache).${NC}"
    exit 1
fi

# Basic firewall prompt (non-blocking)
if command -v ufw &>/dev/null && ! ufw status | grep -q "443.*ALLOW"; then
    echo -e "${YELLOW}Allowing port 443/tcp through UFW...${NC}"
    ufw allow 443/tcp
fi

# --- Step 5: Create Xray config --------------------------------------------------
CONFIG_FILE="/usr/local/etc/xray/config.json"
cp "$CONFIG_FILE" "$CONFIG_FILE.bak.$(date +%s)" 2>/dev/null || true   # backup old config

cat > "$CONFIG_FILE" <<EOF
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "${UUID}",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "${DEST_SPOOF}:443",
          "xver": 0,
          "serverNames": [
            "${DEST_SPOOF}"
          ],
          "privateKey": "${PRIVATE_KEY}",
          "shortIds": [
            "${SHORT_ID}"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
EOF

# --- Step 6: Start & enable service ---------------------------------------------
echo -e "${YELLOW}Starting Xray service...${NC}"
systemctl daemon-reload
systemctl restart xray
systemctl enable xray

# --- Step 7: Verify service -------------------------------------------------------
if systemctl is-active --quiet xray; then
    echo -e "${GREEN}Xray service is running.${NC}"
else
    echo -e "${RED}Xray service failed to start. Check with 'journalctl -xeu xray'.${NC}"
    exit 1
fi

# --- Step 8: Display results -----------------------------------------------------
VLESS_LINK="vless://${UUID}@${SERVER_IP}:443?flow=xtls-rprx-vision&security=reality&sni=${DEST_SPOOF}&fp=chrome&pbk=${PUBLIC_KEY}&sid=${SHORT_ID}&type=tcp&headerType=none#NexusNode"

cat <<EOF

${GREEN}======================[ NexusDeploy Installation Complete ]======================${NC}

${YELLOW}Client configuration (copy to your VLESS client):${NC}
 Address (地址)  : ${SERVER_IP}
 Port (端口)     : 443
 UUID (用户ID)   : ${UUID}
 Flow            : xtls-rprx-vision
 Security        : reality
 SNI (域名)      : ${DEST_SPOOF}
 Public Key      : ${PUBLIC_KEY}
 ShortId         : ${SHORT_ID}

${YELLOW}Quick-import link (already URL-encoded):${NC}
${GREEN}${VLESS_LINK}${NC}

${YELLOW}Tip:${NC} The link has been saved to ~/vless-link.txt for easy transfer.
echo "${VLESS_LINK}" > ~/vless-link.txt

${RED}==============================[ IMPORTANT ]==============================
 Self-hosting requires you to:
 1. Monitor IP reputation and avoid abuse.
 2. Keep Xray and the OS updated: apt-get update && apt-get upgrade
 3. Review logs periodically: journalctl -u xray -f
==========================================================================${NC}

EOF
```




# VPSDeploy: 一键部署企业级安全网络节点

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)
[![Version](https://img.shields.io/badge/version-1.0-blue.svg)]()

VPSDeploy 是一个为开发者和跨境专业人士设计的开源脚本，用于在任何 Debian/Ubuntu 服务器上一键部署基于 Xray/VLESS+Reality 的高性能、高伪装性的安全网络节点。

---

### 核心优势

*   **真正一键化:** 无需手动编辑配置文件，一行命令完成所有操作。
*   **尖端技术:** 采用最新的 VLESS + XTLS-Reality 方案，兼顾性能与隐蔽性。
*   **智能检测:** 脚本在运行前会自动检查系统环境，避免常见错误。
*   **完全开源:** 所有代码公开透明，您可以完全掌控自己的服务器和数据。

---

### 如何使用

通过 SSH 登录到一台 **全新的、纯净的海外 Debian 或 Ubuntu 服务器**，然后执行以下命令：

```bash
wget -P /root https://raw.githubusercontent.com/SongZstrong/VPSDeploy/main/VPSDeploy.sh && sudo bash /root/VPSDeploy.sh
```

脚本会自动完成所有安装和配置。结束后，请保存屏幕上显示的连接信息。

---

### 常见问题 (FAQ)

**问：为什么脚本只支持 Debian/Ubuntu？**
答：为了保证最高的稳定性和成功率，我们的脚本针对这两个最主流的服务器操作系统进行了深度优化。在其他系统上运行可能会遇到依赖库不兼容等问题。

*   **解决方案：** 如果您不想被服务器操作系统所限制，或者不想处理任何技术问题，我们强烈建议您使用我们的 **[官方合作伙伴Just My Socks提供的托管服务](https://justmysocks.net/members/aff.php?aff=34571)**。他们提供的是开箱即用的成品，无需任何技术背景。

**问：自建节点和购买专业服务，我该如何选择？**
答：
*   **自建 (使用本脚本):** 适合喜欢DIY、有一定技术基础、不介意花时间维护的用户。您拥有最高权限，但也要承担所有维护责任（如IP被封、软件更新、故障排查）。
*   **专业服务:** 适合追求稳定、高效、省心的商业用户或个人。您购买的是结果，而不是过程。将专业的事交给专业的人，您可以专注于自己的核心业务。**[Just My Socks，体验稳定服务](https://justmysocks.net/members/aff.php?aff=34571)**。

---
> **声明:** 本项目旨在技术交流与学习，请在遵守当地法律法规的前提下使用。
```
