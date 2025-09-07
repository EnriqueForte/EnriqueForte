# 🚩 Walkthrough – Bounty Hacker (TryHackMe)

## 📖 Introducción

En este reto de TryHackMe denominado **Bounty Hacker**, nuestro objetivo será comprometer la máquina víctima, obteniendo acceso inicial y escalando privilegios hasta conseguir las flags.  
A lo largo del proceso seguiremos la metodología habitual de un pentest: **reconocimiento, enumeración, explotación y escalada de privilegios**.  

Este CTF resulta ideal para practicar **enumeración de servicios expuestos, fuerza bruta y explotación básica en sistemas Linux**, por lo que es un buen ejercicio de iniciación para quienes estén dando sus primeros pasos en el hacking ético.

---

## 🛠️ Herramientas utilizadas

Durante la resolución de este CTF utilicé las siguientes herramientas y recursos:

- **ping** → Comprobación de conectividad con la máquina objetivo.  
- **nmap** → Escaneo de puertos, servicios y versiones (`-sC -sV`).  
- **navegador web (Firefox/Chromium)** → Exploración inicial del servicio HTTP en el puerto 80.  
- **ftp** → Acceso al servicio FTP con login anónimo y descarga de archivos.  
- **hydra** → Ataque de fuerza bruta contra SSH usando el diccionario `locks.txt`.  
- **ssh** → Acceso remoto al sistema con las credenciales obtenidas.  
- **sudo -l** → Enumeración de privilegios del usuario.  
- **tar** → Abuso de permisos sudo para escalar a root (técnica de GTFOBins).  

---

## 1️⃣ Reconocimiento inicial

Lo primero que hicimos fue comprobar la conectividad con la máquina objetivo mediante un **ping**:

<img width="718" height="217" alt="ping" src="https://github.com/user-attachments/assets/9f624b03-3781-455e-99a8-e6bd96c436a8" />

Observamos que la máquina **10.10.235.47** responde correctamente, con un **tiempo de respuesta de ~47ms** y **0% packet loss**, lo que indica que está activa y accesible en la red.

## 2️⃣ Escaneo de puertos y servicios

A continuación, realizamos un escaneo con **Nmap** utilizando los parámetros `-sC -sV` para detectar servicios y versiones:

```bash
nmap -sC -sV 10.10.235.47
```

<img width="690" height="503" alt="nmap escaneo de puertos" src="https://github.com/user-attachments/assets/e08bc627-aec4-45a0-a701-a84880b79da0" />


Los resultados muestran los siguientes puertos abiertos:

21/tcp – FTP (vsftpd 3.0.5)

El servidor permite acceso anónimo (ftp-anon).

No es posible listar directorios debido a restricciones de permisos.

22/tcp – SSH (OpenSSH 8.2p1 en Ubuntu 20.04)

Servicio de acceso remoto seguro.

Podría ser un punto de acceso si conseguimos credenciales válidas.

80/tcp – HTTP (Apache/2.4.41 en Ubuntu)

Servidor web en ejecución.

Requiere exploración para identificar recursos o posibles vulnerabilidades.

👉 La información más interesante es que el FTP permite login anónimo, lo que puede ser aprovechado para acceder y revisar archivos disponibles.

## 3️⃣ Exploración del servicio web (HTTP – Puerto 80)

Accedimos a la dirección **http://10.10.235.47/** desde el navegador:

<img width="1115" height="802" alt="prueba puerto 80" src="https://github.com/user-attachments/assets/e440eeba-3f84-4154-8bb0-e7be227ca592" />

El sitio web muestra una imagen de la serie *Cowboy Bebop* junto a algunos mensajes de los personajes:

- **Spike**: "Oh look you're finally up. It's about time, 3 more minutes and you were going out with the garbage."
- **Jet**: "Now you told Spike here you can hack any computer in the system. We'd let Ed do it but we need her working on..."

👉 Este contenido no revela directamente información sensible, pero es un indicio de que el puerto **80** está activo y funcionando con un servidor **Apache/2.4.41 (Ubuntu)**.

## 4️⃣ Acceso al servicio FTP (Puerto 21)

Probamos el acceso al servidor **FTP** utilizando credenciales anónimas:

<img width="510" height="277" alt="inicio de sesion con ftp " src="https://github.com/user-attachments/assets/5421c033-2fe9-4d9a-992c-7abe3f14b08b" />

```bash
ftp 10.10.235.47
```

El login fue exitoso con el usuario anonymous, confirmando que el servicio permite conexiones sin autenticación.

👉 Esto nos brinda la oportunidad de listar y posiblemente descargar archivos disponibles en el servidor.

## 5️⃣ Enumeración de archivos en FTP

Una vez dentro del servidor FTP, listamos el contenido del directorio:

<img width="579" height="446" alt="listar ftp y decargar archivos txt" src="https://github.com/user-attachments/assets/367a04ac-2f9a-4015-912e-efbdcfed794d" />

```bash
ftp> ls
```
Se identificaron los siguientes archivos de texto:

locks.txt

task.txt

Descargamos ambos utilizando mget *.txt:

Los archivos fueron transferidos correctamente:

locks.txt → 418 bytes

task.txt → 68 bytes

👉 El siguiente paso será analizar el contenido de estos archivos en busca de credenciales, notas o cualquier información útil para acceder al sistema.

## 6️⃣ Análisis de archivos descargados

Una vez descargados los archivos desde el servidor FTP, analizamos su contenido:

<img width="639" height="629" alt="contenido archivos txt" src="https://github.com/user-attachments/assets/a6fda99c-a929-4cea-8a18-90484054cb53" />

```bash
cat task.txt
```
El archivo task.txt contiene una nota con instrucciones:

1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin

```bash
cat locks.txt
```
El archivo locks.txt contiene una lista extensa de posibles contraseñas:
EdrdR4g0N
ReDr4g0nSynd!cat3
Dr@g0n$yn91cat3
R3DDr460nSYndIC@Te
...

👉 Es muy probable que se trate de un diccionario de contraseñas a probar contra algún servicio como SSH (puerto 22) usando el usuario lin.

## 7️⃣ Ataque de fuerza bruta con Hydra (SSH)

Dado que descubrimos al usuario **lin** en `task.txt` y una lista de contraseñas en `locks.txt`, utilizamos **Hydra** para realizar un ataque de fuerza bruta contra el servicio **SSH**:

<img width="995" height="199" alt="hydra para contraseña" src="https://github.com/user-attachments/assets/d6481739-8809-4a41-ad3b-48f1b44f0f9e" />

```bash
hydra -l lin -s 22 -P ./locks.txt ssh://10.10.235.47
```

El ataque tuvo éxito y encontramos las credenciales válidas:

Usuario: lin

Contraseña: RedDr4gonSynd1cat3

👉 Con estas credenciales ya es posible iniciar sesión en la máquina víctima a través de SSH y obtener acceso al sistema.

## 8️⃣ Acceso inicial a la máquina (SSH)

Con las credenciales obtenidas mediante Hydra, nos conectamos a la máquina víctima a través de **SSH**:

<img width="953" height="445" alt="inicio sesion con ssh" src="https://github.com/user-attachments/assets/2029fa9d-f1b2-46fa-9040-06c88a350eff" />

```bash
ssh lin@10.10.235.47
```

El acceso fue exitoso y conseguimos una shell como el usuario lin en el sistema operativo Ubuntu 20.04.6 LTS.

Al listar el escritorio del usuario encontramos el archivo:

user.txt

👉 Esto indica que hemos alcanzado el primer objetivo (flag de usuario).

## 9️⃣ Obtención de la flag de usuario

Dentro del directorio `Desktop` del usuario **lin**, encontramos el archivo **user.txt**:

<img width="667" height="125" alt="flag user txt" src="https://github.com/user-attachments/assets/c666a790-e9df-4e2a-b851-4c98aa401607" />

```bash
cat user.txt
```
La flag obtenida es:
THM{CR1M3_SyNd1c4T3}

👉 Con esto conseguimos la primera flag (usuario).

## 🔟 Escalada de privilegios con `tar`

Al revisar los privilegios de `lin` con el comando `sudo -l`, observamos lo siguiente:

<img width="944" height="130" alt="sudo l para root" src="https://github.com/user-attachments/assets/4646c9a7-14d5-49c5-9296-b9ba56403daa" />

El usuario **lin** puede ejecutar `/bin/tar` como **root**.  
Esto permite abusar de la funcionalidad de `tar` para escalar privilegios.

De acuerdo con [GTFOBins], podemos aprovecharlo así:

<img width="740" height="561" alt="gtfoBins sudo tar" src="https://github.com/user-attachments/assets/aae21423-c39d-43aa-a151-0c3dcdfc1fc2" />

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
Esto abre una shell con privilegios de root.

## 1️⃣1️⃣ Obtención de la flag de root

Ejecutamos el exploit con `tar` para escalar privilegios:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

<img width="943" height="140" alt=" flag user" src="https://github.com/user-attachments/assets/2c654c64-8ce5-4626-9f69-814fe52e45f0" />

Conseguimos acceso como root:
id
uid=0(root) gid=0(root) groups=0(root)

Finalmente, accedimos al archivo root.txt ubicado en /root/:

La flag obtenida es:

THM{B0UN7Y_h4cK3r}

👉 Con esto completamos el CTF Bounty Hacker con éxito 🎉.

