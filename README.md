# HAProxy Socat Swiss Knife

[![Star this repo](https://img.shields.io/github/stars/Dyarven/haproxy-socat-swiss-knife?style=social)](https://github.com/Dyarven/haproxy-socat-swiss-knife/stargazers)
[![Follow me](https://img.shields.io/github/followers/Dyarven?style=social)](https://github.com/Dyarven)
[![License](https://img.shields.io/github/license/Dyarven/haproxy-socat-swiss-knife)](https://github.com/Dyarven/haproxy-socat-swiss-knife/blob/main/LICENSE)

A tool to interact with the HAProxy socket using [socat](https://www.kali.org/tools/socat/) to make server/backend state changes and view stick-tables, session cookies, stats, traffic... easily.

---

## 🛠 Features
- **Simplified use:** Interactive menu with preconfigured complex commands for fast supervision.
- **Speed:** Diagnose network problems quickly.
- **Self descriptive:** Pretty much run and gun.
- **Lightweight & Efficient:** Ensures minimal resource usage for smooth performance.
- **Native:** Fully made in bash.
## ⚠️ Important
- Requires socat package and a running haproxy with the stats socket configured with admin permissions (duh)
- Note that my functions assume that your HAProxy configuration doesn't have stick table IDs specified, so they share the backend name.
  
### Run the Script
```bash
# run as sudo (some commands require root to make changes)
bash haproxy_socat_swiss_knife.sh
```

