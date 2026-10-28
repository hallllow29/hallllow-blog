---
title: HTB - MetaTwo
date: 2026-10-28
tags:
  - wordpress
  - sqli
  - xxe
  - ftp
  - passpie
  - password-cracking
categories:
  - HackTheBox
os: Linux
difficulty: Easy
description: "HTB MetaTwo (Easy). A WordPress box: a BookingPress SQL injection (CVE-2022-0739) dumps the manager hash, cracked to log into wp-admin. A WordPress media XXE (CVE-2021-29447) reads wp-config.php for the DB and FTP creds. The FTP share has a send_email.php with jnelson's SSH password, and root comes from cracking a passpie GPG keyring to export the stored root password."
---

<p style="text-align:center">
  <img src="featured.png" alt="MetaTwo box" width="250">
</p>

<p>
  <img src="https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat&logo=hackthebox&logoColor=black" alt="Platform">
  <img src="https://img.shields.io/badge/OS-Linux-E95420?style=flat&logo=linux&logoColor=white" alt="OS">
  <img src="https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat" alt="Difficulty">
</p>

> **TL;DR** - A WordPress site vulnerable to CVE-2022-0739 (a BookingPress SQL injection) dumps the `manager` and `admin` hashes. Cracking `manager` gives `partylikearockstar` and a wp-admin login. From there, CVE-2021-29447 (a WordPress media XXE) lets me read files off the box, including `wp-config.php`, which has the DB and FTP creds inside. On the FTP share there's a `send_email.php` with `jnelson`'s password in plaintext, and that same password works over SSH (user.txt). For root, a hidden `.passpie` keyring holds GPG keys, so I crack the passphrase with John, unlock passpie, and export the vault to get the root password (root.txt).

## Recon

Like always, I started with a full nmap scan:

```bash
nmap -sV -sC -p- 10.129.80.192 -oA nmap/metatwo
```

![](Images/MetaTwo-scan.png)

Three ports open. I started with the FTP one on 21 and tried anonymous access, but no luck, so I moved on to port 80.

![](Images/MetaTwo-web.png)

In the browser it looked like a simple WordPress site, so I ran `wpscan` to see if I could catch something. It told me the install was vulnerable to `CVE-2022-0739`, a SQL injection in the BookingPress plugin, so I grabbed a [PoC](https://github.com/viardant/CVE-2022-0739) and ran it against it.

![](Images/MetaTwo-ranpoc.png)

Sweet, I got two hashes, one for `manager` and one for `admin`. I tried to crack both offline but only `manager` cracked.

![](Images/MetaTwo-cracknmanager.png)

So now I had creds, `manager:partylikearockstar`. Off to the wp-admin page to log in, and I was in.

![](Images/MetaTwo-wrdorpesslogin.png)

## File read via XXE (CVE-2021-29447)

Inside the manager account I saw it didn't have admin rights. But I already knew the WordPress version from the wpscan earlier (5.6.2), so I went looking for CVEs on that version and found `CVE-2021-29447`, an XXE in the media library that lets you read files off the machine. I found a [PoC](https://github.com/Val-Resh/CVE-2021-29447-POC) online and tried it out, starting with `/etc/passwd`.

![](Images/MetaTwo-etcpasswd.png)

And it worked. Now that I could read files from the server, I wanted something useful, maybe the WordPress php config. After some trial and error guessing the path, I found `wp-config.php` at `/var/www/metapress.htb/blog/wp-config.php`.

![](Images/MetaTwo-wpconfig.png)

Inside were MySQL creds for `blog:635Aq@TdqrCwXFUZ` and FTP creds for `metapress.htb:9NYS_ii@FyL_p5M2NvJ` (on `ftp.metapress.htb`). FTP was open and I had valid creds now, so I figured that was the next step.

![](Images/MetaTwo-insideftp.png)

## FTP and SSH as jnelson

And we're in. After digging around the FTP share for a bit, I found a file called `send_email.php` with some creds sitting right there in plaintext.

![](Images/MetaTwo-plaintextcreds.png)

That gave me `jnelson:Cb4_JmWM8zUZWMu@Ys`, so I tried logging in over SSH.

![](Images/MetaTwo-insidejnelson.png)

And we're in as jnelson.

![](Images/MetaTwo-userflag.png)

Got the user flag, `5acd8a600fc1d76f7170266769266066`.

## Privilege escalation (passpie)

Next was getting root. The first thing I usually do is `sudo -l` for a quick win, but no luck this time. So I kept digging, and an `ls -la` turned up a hidden file.

![](Images/MetaTwo-hiddenfile.png)

A `.passpie` directory, which isn't something you see every day, so I looked into what it was. Turns out passpie is a password manager for the CLI. Inside there was a hidden `.keys` folder where it keeps its keys, and there were two keys in there.

![](Images/MetaTwo-keys.png)

I passed those to john and cracked them.

![](Images/MetaTwo-cracked.png)

That got me `blink182`, the passpie master password. With it I used passpie to export the stored passwords.

![](Images/MetaTwo-passpie.png)

And there it was, the root password, `root:p7qfAZt4_A1xo_0x`. So let's log in as root.

![](Images/MetaTwo-rootflag.png)

And we're in as root. Got the flag, `083c84b7976c26f1f01b55b9327b9060`.

## Kill chain summary

| Phase | Vector | Result |
|---|---|---|
| Recon | nmap: FTP / SSH / WordPress on 80 | attack surface |
| Foothold | BookingPress SQLi (CVE-2022-0739) | `manager:partylikearockstar`, wp-admin |
| File read | WordPress media XXE (CVE-2021-29447) | `wp-config.php`, DB + FTP creds |
| Lateral | FTP `send_email.php` plaintext creds | `jnelson` over SSH (user.txt) |
| Root | crack passpie GPG keyring, export vault | `root:p7qfAZt4_A1xo_0x` (root.txt) |
