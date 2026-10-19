[README.md](https://github.com/user-attachments/files/31200565/README.md)
# Tor Node Manager

A simple Bash-based manager for creating **local Tor SOCKS5 exit nodes** by country.

This tool creates isolated Tor instances by exit country, each running on its own local SOCKS5 port. It is designed to be used as a local SOCKS5 outbound source for tools like Xray, 3x-ui, or any other application that can use a SOCKS5 proxy.

> **Important:** This script does **not** modify x-ui, 3x-ui, Xray, firewall routing rules, or any panel configuration. It only creates local Tor SOCKS5 services on `127.0.0.1:<port>`.

---

## Features

- Create country-based Tor SOCKS5 nodes
- Run each country as a separate systemd service
- Use dedicated Tor config, data directory, and log file per node
- Check whether the SOCKS5 port is alive
- Verify that traffic is actually going through Tor
- Verify the Tor exit country using multiple geo-check providers
- Restart and retry automatically until the selected country is confirmed
- Print ready-to-use Xray outbound JSON
- Delete nodes cleanly
- View logs per node
- Interactive menu mode
- Live menu of every country that currently has a running Tor exit relay
- Automatic selection of the first available local port
- Web and Telegram connectivity checks
- Upgrade/fix command for existing nodes

---

## Available Countries

The interactive menu queries the Tor Project's Onionoo service and shows every country that currently has at least one running exit relay. The list therefore follows the live Tor network instead of being hard-coded. If Onionoo is temporarily unavailable, a built-in fallback list is shown.

Custom ISO-3166 alpha-2 country codes are also accepted. Installation can still fail when no usable exit exists for the selected country.

---

## How It Works

For each country, the script creates a dedicated Tor instance.

Example for Turkey:

```text
Service:       tor-exit-tr
SOCKS5:        127.0.0.1:9053
Tor config:    /etc/tor/tor-exit-tr.torrc
Data dir:      /var/lib/tor-multi/tor-exit-tr
Log file:      /var/log/tor-multi/tor-exit-tr.log
Metadata:      /etc/tor-multi/nodes/tr.env
```

The generated Tor configuration uses:

```text
SocksPort 127.0.0.1:9053 IsolateSOCKSAuth
ClientOnly 1
ExitNodes {tr}
StrictNodes 1
SafeSocks 0
ClientUseIPv6 0
```

This means Tor will only expose a local SOCKS5 proxy and will try to use an exit node from the selected country. With `StrictNodes 1`, the node will fail instead of silently using the wrong country.

---

## Installation

### Install directly from GitHub

Replace the URL with your own repository URL if you forked or renamed the project.

```bash
curl -fsSL https://raw.githubusercontent.com/AMasoudKaveh/tor-node-manager/main/tor-node-manager.sh -o /usr/local/bin/tor-node-manager
chmod +x /usr/local/bin/tor-node-manager
tor-node-manager menu
```

### Manual install

```bash
chmod +x tor-node-manager.sh
sudo ./tor-node-manager.sh menu
```

The script must be run as `root` because it creates systemd services and writes to `/etc`, `/var/lib`, and `/var/log`.

---

## Usage

### Open interactive menu

```bash
tor-node-manager menu
```

### Install a Turkey node on port 9053

```bash
tor-node-manager install tr 9053
```

Omit the port to automatically select the first available port starting at `9053`:

```bash
tor-node-manager install tr
```

### Install Germany, Netherlands, and France

```bash
tor-node-manager install de 9054
tor-node-manager install nl 9055
tor-node-manager install fr 9056
```

### Install a custom country

Example: Spain on port `9060`:

```bash
tor-node-manager install es 9060
```

---

## Health Check

After installation, the script checks:

1. The SOCKS5 port is listening on `127.0.0.1:<port>`
2. The connection is going through Tor
3. The exit IP country matches the selected country
4. At least one geo-check provider confirms the expected country
5. A normal HTTPS website is reachable
6. Telegram's API endpoint is reachable

Example output:

```text
Country expected: TR
SOCKS5: 127.0.0.1:9053
Tor check: true
Exit IP: 83.217.9.73
Geo votes: TR|TR|TR|TR
OK: Node is healthy. Confirmed Tor exit country: TR
```

The exit IP is not used inside Xray. It is only shown so you can see the current public Tor exit address.

---

## Xray / 3x-ui Outbound Example

After installing a node, get the outbound JSON:

```bash
tor-node-manager json tr
```

Example output:

```json
{
  "tag": "tor-tr",
  "protocol": "socks",
  "settings": {
    "address": "127.0.0.1",
    "port": 9053
  },
  "targetStrategy": "AsIs"
}
```

Inside your panel, create the outbound manually:

```text
Protocol: socks
Address: 127.0.0.1
Port: 9053
Tag: tor-tr
Username: empty
Password: empty
```

Do **not** use the Tor exit IP as the outbound address. Always use the local SOCKS5 address:

```text
127.0.0.1:9053
```

---

## Routing Example

After creating the outbound manually in your panel, route the desired inbound to the Tor outbound.

Example routing idea:

```json
{
  "type": "field",
  "inboundTag": [
    "your-inbound-tag"
  ],
  "outboundTag": "tor-tr",
  "network": "tcp"
}
```

Use your actual inbound tag from your Xray or panel configuration.

---

## Commands

### List installed nodes

```bash
tor-node-manager list
```

### Upgrade/fix all existing nodes

Applies the latest Tor compatibility settings while preserving country and port assignments:

```bash
tor-node-manager upgrade
```

### Test an installed node

```bash
tor-node-manager test tr
```

### Restart a node and try to get a fresh exit circuit

```bash
tor-node-manager restart tr
```

### Show Xray outbound JSON

```bash
tor-node-manager json tr
```

### Show logs

```bash
tor-node-manager logs tr
```

### Delete a node

```bash
tor-node-manager delete tr
```

---

## File Structure

```text
/etc/tor-multi/
└── nodes/
    └── tr.env

/etc/tor/
└── tor-exit-tr.torrc

/etc/systemd/system/
└── tor-exit-tr.service

/var/lib/tor-multi/
└── tor-exit-tr/

/var/log/tor-multi/
└── tor-exit-tr.log
```

---

## Notes and Limitations

- Tor is not designed for high-speed commercial proxy traffic.
- Speed and stability are not guaranteed.
- Some websites block Tor exit nodes.
- UDP is not suitable for this setup.
- This is best used for lightweight TCP traffic, location testing, and country-based SOCKS5 routing.
- Exit country accuracy is checked at install/test time, but Tor exit availability can change later.
- If a node stops matching the expected country, run:

```bash
tor-node-manager restart tr
```

or:

```bash
tor-node-manager test tr
```

---

## Troubleshooting

### Check whether the SOCKS5 port is listening

```bash
ss -ltnp | grep 9053
```

### Test through SOCKS5 manually

```bash
curl --socks5-hostname 127.0.0.1:9053 https://check.torproject.org/api/ip
```

### Check service status

```bash
systemctl status tor-exit-tr --no-pager -l
```

### View logs

```bash
tor-node-manager logs tr
```

### Delete and recreate a node

```bash
tor-node-manager delete tr
tor-node-manager install tr 9053
```

---

## Safety

All SOCKS5 ports are bound only to localhost:

```text
127.0.0.1:<port>
```

This prevents the SOCKS5 proxy from being exposed publicly on the internet.

---

## License

No license has been selected yet. Add a license before distributing or publishing this project for public use.

# Author

**Masoud Kaveh**

GitHub:  
https://github.com/AMasoudKaveh

Telegram:  
https://t.me/MasoudOpenSourceLab

---

If this project is useful to you, consider giving the repository a ⭐ on GitHub.

