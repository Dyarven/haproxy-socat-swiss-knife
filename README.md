# 🌐 HAProxy Socat Swiss Knife

[![Star this repo](https://img.shields.io/github/stars/Dyarven/haproxy-socat-swiss-knife?style=social)](https://github.com/Dyarven/haproxy-socat-swiss-knife/stargazers)
[![Follow me](https://img.shields.io/github/followers/Dyarven?style=social)](https://github.com/Dyarven)
[![License](https://img.shields.io/github/license/Dyarven/haproxy-socat-swiss-knife)](https://github.com/Dyarven/haproxy-socat-swiss-knife/blob/main/LICENSE)

A tool to interact with the HAProxy socket using [socat](https://www.kali.org/tools/socat/) to make frontend and server/backend state changes, view and clear stick-tables, enable healthchecks, control network traffic and requests and other stuff more easily.

Managing large HAProxy instances can be complex, so being able to test and review configs, as well as debugging network problems and strange behaviours comfortably becomes a necessity.

---

## Features
- **Interactive and preset mode**: Includes a workaround to support interactive mode in Debian based systems using netcat and socat readline in RHEL based ones.
- **Fast and simple:** Interactive menu with predefined and parsed complex commands to diagnose problems and make changes quickly.
- **Self descriptive:** Pretty much run and gun.
- **Lightweight & Efficient:** Ensures minimal resource usage for smooth performance.
- **Native:** Fully made in bash with native tools.

## Requirements
- Requires socat package and a running haproxy with the stats socket configured with admin permissions (duh)
- Note that my functions assume that your HAProxy configuration doesn't have stick table IDs specified, so they share the backend name.


## Usage
### Run the Script
```bash
# Run as sudo (some commands require root to make changes)
bash haproxy_socat_swiss_knife
```

### Install as a program
```bash
# Copy the script in /usr/local/bin as "hssk" and ensure proper permissions.
chmod +x /usr/local/bin/hssk
# Now you can run it from anywhere simply typing:
hssk
```

### Notes
There is a cool way to interact with the admin stats panel (RW) that allows you to modify the state of servers and backends (this is not a functionality in my script but documented for posterity).
You can use curl and send a payload applying the changes you want, once you find out the backend number checking the source code of the stats panel:
```bash
curl -x "" -u user:pass "http://HAPROXY_PANEL_IPADDR:PORT/haproxy?stats" --data-urlencode "s=server1-name" --data-urlencode "action=maint/ready" --data-urlencode "b=#XX" # your backend number I.E. b=#17
 ```
