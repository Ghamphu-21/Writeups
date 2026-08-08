## About

**Objectives**
1. Web Flag: `thm{0ae72f7870c3687129f7a824194be09d}`
2. User Flag: `thm{3693fc86661faa21f16ac9508a43e1ae}`
3. Root Flag: `thm{a4f6adb70371a4bceb32988417456c44}`


**Room:** https://tryhackme.com/room/overpass3hosting

**Difficulty:** Medium

**OS:** Linux

**Target IP:** `10.49.152.7`

## Walkthrough


I started, as always, with a full port scan to enumerate open ports, running services, and the OS:

```
nmap -sV -sC -O -v -p- 10.49.152.7 -oA Overpass3
```

![](Assets/2026-08-06_18-52.png)

This confirmed **port 80 (HTTP)** was open, so I decided to check out the web application first. Browsing to the target IP just gave me a default landing page - nothing useful.

![](Assets/2026-08-06_18-48.png)

I had no luck in finding sensitive information in the source code. So, I decided to perform directory brute forcing with `ffuf`:

```
ffuf -u http://10.49.152.7/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![](Assets/2026-08-06_19-00.png)

This turned up an interesting `backups` directory. I navigated into the directory and found a zip file that looked promising, so I downloaded it to dig through later.

![](Assets/2026-08-06_18-58.png)

After downloading and extracting the zip file, two files were present:
- `CustomerDetails.xlsx.gpg` - an encrypted spreadsheet
- `priv.key` - initially assumed to be an SSH private key based on the name, but this needed verification

![](Assets/2026-08-06_19-04.png)

Despite the `.key` extension suggesting SSH, inspecting the `priv.key` revealed it was actually a GPG private key. The header confirmed a PGP key block rather than an OpenSSH one.

![](Assets/2026-08-06_19-22.png)

Let's import the GPG key:

```
gpg --import priv.key
```

The output confirmed a successful import with no passphrase required. This revealed the key owner as `paradox@overpass.thm`.

![](Assets/2026-08-06_19-25.png)

Now, it's time to decrypt the `CustomerDetails.xlsx.gpg` file:

```
gpg --output CustomerDetails.xlsx --decrypt CustomerDetails.xlsx.gpg
```


![](Assets/2026-08-06_19-29.png)

I could've opened this in LibreOffice, but I went with a quick Python script instead to dump the rows:

```
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('CustomerDetails.xlsx')
ws = wb.active
for row in ws.iter_rows(values_only=True):
    print(row)
"
```

This gave us some sensitive info, including a username and password, which I figured I'd try against the SSH or FTP services I'd found earlier.

![](Assets/2026-08-06_19-33.png)

SSH didn't work with these creds, but I got lucky with **FTP** - logged in fine using `paradox:ShibesAreGreat123`.

![](Assets/2026-08-06_19-39.png)

From here, we can upload a reverse shell and get a connection. We can use https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php.

![](Assets/2026-08-06_19-55.png)

Now, we can a start a listener (Here i used Penelope, netcat works fine too) and visit `http://10.49.152.7/shell.php` to trigger the connection.

![](Assets/2026-08-06_20-00.png)


The first web flag is located inside the `/usr/share/httpd` directory and we can read our first flag `thm{0ae72f7870c3687129f7a824194be09d}`.

![](Assets/2026-08-06_20-04.png)

Poking around the home directory, we found two users: `james` and `paradox`. I tried reusing the password I'd recovered from the spreadsheet for `paradox`, and it worked!

![](Assets/2026-08-06_20-13.png)

Now, I uploaded **linpeas** through the FTP server and ran it. Going through the output, I noticed it flagged **NFS** as possibly misconfigured.

![](Assets/2026-08-06_20-35.png)

I tried listing and mounting the NFS share directly from my attack machine, but that didn't work.

![](Assets/2026-08-06_20-38.png)

This means that the share can only be accessed locally as user james. For that, we can use SSH local port forwarding.

First, let's generate a SSH key pair on our Attack host:

```
ssh-keygen -f paradox
```


This generates a private key (paradox) and a public key (paradox.pub), used to authenticate as the paradox user without needing a password.

![](Assets/2026-08-06_21-00.png)


Now, we can insert the contents of `paradox.pub` to the target's authorized keys file:

```
echo "<content of public key>" >> /home/paradox/.ssh/authorized_keys
```

Now, open a new terminal and ssh into the target system.

```
ssh -i paradox paradox@10.49.152.7
```

Now, we can check which ports are used for listening to NFS connections.

```
rpcinfo -p
```

This lists registered RPC services and their ports. Output confirmed NFS was listening on port 2049.

![](Assets/2026-08-06_21-12.png)


Now, we can establish SSH local port forwarding so that all the traffic sent to the SSH daemon would be redirected to the NFS, enabling us to access the share.

```
ssh paradox@10.49.152.7 -i paradox -L 2049:localhost:2049
```

Because of the active tunnel, this connects to the real NFS share on the target, despite targeting localhost.

![](Assets/2026-08-06_21-29.png)

The mount succeeded, we can now read the contents of user flag `thm{3693fc86661faa21f16ac9508a43e1ae}`


![](Assets/2026-08-06_21-31.png)

Moving ahead, we need to create a new SSH key pair:

```
ssh-keygen -f james_key
cat james_key.pub
```

![](Assets/2026-08-06_22-08.png)

Then, from your NFS-mounted access:

```
echo "<contents of james_key.pub>" >> ~/nfs/.ssh/authorized_keys
```

Now, we can login as `james` and run  `./bash -p` to escalate our privileges to root:

```
ssh -i james_key james@10.49.152.7
```

In the victim machine, we copied the bash binary to james home directory.

```
cp /bin/bash .
```

![](Assets/2026-08-06_22-38.png)

On our attacking machine, we changed the ownership and permissions of the bash binary.

```
sudo chown root:root bash
sudo chmod +s bash
```

![](Assets/2026-08-06_22-38_1.png)

In the victim machine, execute the binary with `./binary -p` to gain root shell!

![](Assets/2026-08-06_22-39.png)

Now, we can read the root flag `thm{a4f6adb70371a4bceb32988417456c44}`.

![](Assets/2026-08-06_22-42.png)

---
