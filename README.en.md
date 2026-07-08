**English** | [پارسی](README.md)

این هم نسخه انگلیسی کامل و یکپارچه (README.en.md) که برای مخزن گیت‌هاب شما بهینه‌سازی شده است. تمام متون، جداول و توضیحات به انگلیسی روان برگردانده شده‌اند تا کاربران بین‌المللی نیز بتوانند به راحتی از آن استفاده کنند:

---

```markdown
# OpenWrt 24.10 Psiphon-Core Setup & Automation Guide

This project provides a comprehensive guide and an intelligent script to deploy, manage, and pair the native Linux Psiphon core (`psiphon-core`) with **OpenWrt** routers (specifically optimized for version 24.10 and 64-bit architectures like `aarch64_cortex-a53`, such as the GL-MT3000).

Due to censorship circumvention mechanics, the Psiphon core binds to completely random SOCKS and HTTP ports upon each execution. This script dynamically captures these random ports from the system logs, automatically detects your router's local gateway IP, and forwards them to fixed, predictable ports accessible across your local area network (LAN). Furthermore, it introduces on-the-fly egress region (country) selection.

---

## 📂 Directory Structure & File Paths

Before running the automation script, prepare the following 3 items and transfer them to their respective paths on the router using an SFTP/SCP client (e.g., MobaXterm or WinSCP):

| # | Item Type | File / Folder Name | Target Path on Router | Description |
|---|---|---|---|---|
| 1 | **Binary Executable** | `psiphon-core` | `/usr/bin/psiphon-core` | Compiled binary matching your router's CPU architecture (e.g., ARM64) |
| 2 | **Configuration File** | `psiphon.config` | `/usr/bin/psiphon.config` | Contains client configurations, handshake keys, and basic parameters |
| 3 | **Internal Database** | `psiphon_data` | `/usr/bin/psiphon_data/` | Directory where server entries, handshake cache, and configuration updates are stored |

---

## 🛠️ 1. Environmental Setup & Prerequisites (On Router)

Once the files have been uploaded to the `/usr/bin/` directory, execute the following commands in the router's terminal to grant execution permissions and install the necessary routing wrapper (`socat`):

```bash
# Grant executable permissions to the binary and create the data directory
chmod +x /usr/bin/psiphon-core
mkdir -p /usr/bin/psiphon_data

# Update package feeds and install socat for dynamic traffic redirection
opkg update && opkg install socat

```

---

## 📝 2. Generate the Global Configuration File (`psiphon.config`)

Copy the block below and paste it directly into your router's terminal to initialize a standard configuration template (the region parameter will be managed dynamically by the launcher script):

```bash
cat << 'INPUT_EOF' > /usr/bin/psiphon.config
{
  "SocksProxyPort": 10808,
  "HttpProxyPort": 10809,
  "ClientPlatform": "Windows_10.0.26200_11",
  "ClientVersion": "187",
  "DataRootDirectory": "/usr/bin/psiphon_data",
  "EgressRegion": "ALL",
  "PropagationChannelId": "0000000000000000",
  "SponsorId": "0000000000000000",
  "ServerEntrySignaturePublicKey": "sHuUVTWaRyh5pZwy4UguSgkwmBe0EHtJJkoF5WrxmvA=",
  "UseIndistinguishableTLS": true
}
INPUT_EOF

```

---

## ⚡ 3. Dynamic Launcher Script (With Region Selection)

This script automatically discovers your router's active LAN gateway IP, injects your chosen target region into the configuration profile, parses the JSON log array to extract the randomized internal ports, and binds them to the local firewall interfaces.

> 💡 **Region Selection Guide (TARGET_REGION):**
> Change the `TARGET_REGION` variable on the very first line of the script below to any two-letter country code (capitalized) to route your traffic accordingly.
> * **`ALL`** : Connects to the fastest available server (Best Performance)
> 
> 
> | Code | Country | | Code | Country |
> | :---: | :--- | | :---: | :--- |
> | **`AT`** | Austria | | **`IT`** | Italy |
> | **`BE`** | Belgium | | **`JP`** | Japan |
> | **`CA`** | Canada | | **`NL`** | Netherlands |
> | **`CH`** | Switzerland | | **`NO`** | Norway |
> | **`DE`** | Germany | | **`PL`** | Poland |
> | **`DK`** | Denmark | | **`SE`** | Sweden |
> | **`ES`** | Spain | | **`SG`** | Singapore |
> | **`FI`** | Finland | | **`US`** | United States |
> | **`FR`** | France | | **`GB`** | United Kingdom |

```bash
# 🌍 Define the egress country code (Two-letter uppercase code, or ALL for the fastest server)
TARGET_REGION="ALL"

echo "Setting Psiphon egress region to: $TARGET_REGION"
# Dynamically update the configuration profile inline
sed -i "s/\"EgressRegion\": \".*\"/\"EgressRegion\": \"$TARGET_REGION\"/g" /usr/bin/psiphon.config

# 1. Terminate any overlapping instances to avoid port collision
killall -9 psiphon-core socat 2>/dev/null

# 2. Spin up the Psiphon core in the background (storing volatile logs in temporary RAM)
/usr/bin/psiphon-core -config /usr/bin/psiphon.config -dataRootDirectory /usr/bin/psiphon_data > /tmp/psiphon.log 2>&1 &

# 3. Wait for the tunnel handshake to establish and open local random proxy interfaces
echo "Initializing Psiphon Core... Sleeping for 7 seconds..."
sleep 7

# 4. Safely parse the exact randomized local proxy ports from the JSON log nodes
SOCKS_PORT=$(grep "ListeningSocksProxyPort" /tmp/psiphon.log | grep -o '"port":[0-9]*' | cut -d':' -f2)
HTTP_PORT=$(grep "ListeningHttpProxyPort" /tmp/psiphon.log | grep -o '"port":[0-9]*' | cut -d':' -f2)

echo "Captured randomized internal ports -> SOCKS: $SOCKS_PORT | HTTP: $HTTP_PORT"

# 5. Dynamically query the router's local gateway IP address (LAN IP) via ubus
ROUTER_IP=$(ubus call network.interface.lan status | jsonfilter -e '@["ipv4-address"][0].address')
echo "Detected Router LAN IP: $ROUTER_IP"

# 6. Establish a stable socket proxy bridge via socat mapping to static ports 10808 & 10809
socat TCP-LISTEN:10809,fork,bind=$ROUTER_IP TCP:127.0.0.1:$HTTP_PORT &
socat TCP-LISTEN:10808,fork,bind=$ROUTER_IP TCP:127.0.0.1:$SOCKS_PORT &

# 7. Append local zone rule to native OpenWrt 24.10 nftables (fw4) pipeline to accept incoming downstream traffic
nft add rule inet fw4 input iifname "br-lan" tcp dport 10808-10809 accept 2>/dev/null

echo "Psiphon core successfully paired with static LAN ports 10808 & 10809 under location: $TARGET_REGION!"

```

---

## 🔍 4. Troubleshooting & Connection Verification

To ensure that the cryptographic tunnel has successfully broken through and traffic is actively routed, execute the following validation commands:

* **Inspect live handshake logs:** (Look for the node key `"noticeType":"ConnectedServerRegion"`)
```bash
tail -n 20 /tmp/psiphon.log

```


* **Verify active downstream socket listeners:**
```bash
netstat -tulpn | grep -E '10808|10809'

```


* **Perform a WAN exit point check from within the router terminal:** (Should print an uncensored foreign IP address)
```bash
curl -x [http://127.0.0.1:10809](http://127.0.0.1:10809) [https://ifconfig.me](https://ifconfig.me)

```



---

## 🛑 5. Process Teardown & Full Disabling Command

To gracefully kill the core runtime engine, clean up system memory, and tear down open listeners, run the following sequence:

```bash
# Terminate core binary threads and socket forwards
killall -9 psiphon-core socat 2>/dev/null

# Refresh native firewall hooks to flush temporary ingress rule adjustments
/etc/init.d/firewall restart

echo "Psiphon engine and local bridge sockets have been completely turned off."

```

---

## 💻 6. Downstream Client Configurations (OS / Browser)

To pass proxy instructions down to your connected endpoints (Phones, Laptops, smart devices), navigate to your Operating System network rules or browser proxy manager (e.g., FoxyProxy) and configure the following parameters:

* **Proxy Protocol Type:** `HTTP` or `SOCKS5`
* **Server IP / Hostname:** Your router's management gateway IP (e.g., `192.168.18.1`)
* **HTTP Port:** `10809`
* **SOCKS Port:** `10808`

```
---

```
