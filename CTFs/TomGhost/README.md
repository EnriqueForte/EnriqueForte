# TomGhost - TryHackMe 🕵️‍♂️

## Introducción 🚀

TomGhost es una máquina vulnerable que entra en la categoría fácil de las máquinas de arranque para root de TryHackMe. Se trata de explotar la vulnerabilidad "Ghostcat" (CVE-2020-1938) en Apache Tomcat para obtener acceso inicial y realizar una escalada de privilegios horizontal para, finalmente, comprometer completamente la máquina y tomar el control del sistema.

---

## Paso 1: Ping a la máquina 🎯

Antes de comenzar cualquier ataque, probamos la conectividad con la máquina objetivo usando el comando `ping`. Este paso es fundamental para asegurarse que la máquina está activa y responde correctamente a las peticiones ICMP.

````bash
ping -c 4 10.10.170.215
````

- En la captura, se muestra cómo la máquina responde correctamente a los 4 paquetes enviados, sin pérdida y con tiempos de respuesta variables pero estables.
- Esto confirma que la máquina es accesible en la red y podemos continuar con el proceso de enumeración y explotación.

<img width="667" height="301" alt="Ping" src="https://github.com/user-attachments/assets/fde7c487-1b1d-4d2c-b47e-85d1cae8b716" />

---

## Paso 2: Escaneo de puertos con Nmap 🔍

El siguiente paso consiste en identificar los servicios activos y sus versiones en la máquina objetivo usando la herramienta Nmap. Este escaneo nos ayuda a determinar vectores de ataque posibles según los puertos y servicios públicos.

````bash
nmap -p- -sV -sC --open --min-rate 5000 -vvv -n -Pn -oN escaneo 10.10.170.215
````

- En la captura se observan los resultados principales: los puertos 22 (SSH), 53 (tcpwrapped) y 8080 (Apache Tomcat 9.0.30) están abiertos.
- El servicio destacado es Apache Tomcat, que será clave para explotar la vulnerabilidad Ghostcat en los siguientes pasos.
- Se detectan métodos HTTP permitidos como `GET`, `HEAD`, `POST`, y `OPTIONS`, lo que indica superficie adicional para pruebas.

<img width="1860" height="681" alt="Nmap" src="https://github.com/user-attachments/assets/b326e25f-84c8-49d2-b184-66a4f85a7219" />

---

## Paso 3: Nmap en modo sigiloso 🕶️

Para una mejor evasión de sistemas IDS/IPS y para identificar puertos clave de manera menos intrusiva, realizamos un escaneo SYN (half-open) usando Nmap con el parámetro `-sS`. Este tipo de escaneo es más sigiloso que un escaneo completo y suele ser menos registrado por sistemas de detección.

````bash
sudo nmap -sS 10.10.170.215
````

- Los resultados muestran de nuevo los puertos 22 (SSH), 53 (domain), 8009 (ajp13) y 8080 (http-proxy) abiertos.
- El puerto 8009 (AJP13) es especialmente relevante porque está relacionado con la vulnerabilidad Ghostcat.
- El escaneo fue rápido y confirma que el objetivo no tiene firewalls bloqueando estos puertos.

<img width="734" height="301" alt="Nmap Sigiloso" src="https://github.com/user-attachments/assets/2d7f5161-2815-419e-9248-10891aa578a9" />

---

## Paso 4: Búsqueda de vulnerabilidades en servicios ✴️

Para identificar posibles vectores de ataque, utilizamos `searchsploit` para buscar vulnerabilidades conocidas sobre OpenSSH, uno de los servicios detectados previamente.

````
searchsploit openssh
````

- La herramienta muestra varias vulnerabilidades relacionadas con distintas versiones de OpenSSH, incluyendo enumeración de usuarios, ejecución remota de comandos y escalada de privilegios.
- En este caso, ninguna de las vulnerabilidades listadas aplica directamente a la versión presente en la máquina víctima, por lo que no se detecta un exploit crítico sobre este servicio.
- Sin embargo, el análisis orienta nuestra atención hacia otros servicios como Apache Tomcat y el protocolo AJP13, que será explorado en los siguientes pasos.

<img width="884" height="650" alt="busqueda de vulnerabilidades de OpenSSH" src="https://github.com/user-attachments/assets/1cbfa44f-f12c-4fef-9819-72b514bd5cc8" />

---

## Paso 5: Explotando Ghostcat y accediendo por SSH 🧑‍💻🔓

Con la información recolectada, identificamos que el servicio Apache Tomcat es vulnerable a Ghostcat (CVE-2020-1938). Usamos Metasploit para explotar esta vulnerabilidad y obtener archivos sensibles del servidor.

### Utilización de Metasploit para extraer credenciales

Ejecutamos el siguiente módulo en `msfconsole` para explotar Ghostcat y leer archivos a través del conector AJP13 (puerto 8009):

````bash
msfconsole -q
search ghostcat
use auxiliary/admin/http/tomcat_ghostcat
set rhosts 10.10.170.215
run
````


- El exploit permite leer el archivo `web.xml`, donde se muestran credenciales y el nombre de usuario `skyfuck`.
- Este archivo revela, en la etiqueta `<description>`, la contraseña que usaremos para acceder por SSH.

<img width="1629" height="932" alt="utilizo metasploit consigo credenciales" src="https://github.com/user-attachments/assets/cbc43d00-778a-4c93-9d38-7b09786e28ba" />

### Accediendo a la máquina por SSH

Con las credenciales extraídas, establecemos una conexión SSH exitosa con el usuario identificado:

````bash
ssh skyfuck@10.10.170.215
````

- El acceso es concedido y ya estamos dentro del sistema como usuario, donde observamos archivos potencialmente interesantes para la fase de escalada de privilegios.

<img width="843" height="543" alt="establezco conexion ssh" src="https://github.com/user-attachments/assets/9f7b0ff0-c911-4c85-8518-b3fd75056efb" />

---

## Paso 6: Descarga de archivos y descifrado de hash 🗝️📥

Tras el acceso SSH, encontramos archivos potencialmente interesantes: `credential.pgp` y `tryhackme.asc`. El siguiente paso es exfiltrarlos a nuestra máquina para analizarlos y tratar de obtener credenciales adicionales o flags.

### Descargando los archivos a la máquina local

Utilizamos `scp` (o simplemente transferencia desde el entorno de la máquina CTF si se permite) para traer los dos archivos a nuestro entorno de trabajo:

- `credential.pgp`
- `tryhackme.asc`

<img width="1808" height="575" alt="me descargo los dos arhcivo en mi maquina" src="https://github.com/user-attachments/assets/89492783-cbbe-4159-ae4b-9ebb68075e92" />

### Extrayendo y descifrando el hash con John the Ripper

Para tratar de obtener la contraseña protegida, primero convertimos el archivo `.asc` a un hash compatible con John the Ripper usando `gpg2john` y luego lanzamos el crackeo con la popular lista `rockyou.txt`:

````bash
gpg2john tryhackme.asc > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
````

- John the Ripper identifica el hash y realiza con éxito el crackeo, mostrando la contraseña correspondiente bajo el nombre de usuario encontrado anteriormente.
- El proceso resulta fundamental para la posible escalada de privilegios posterior.

<img width="874" height="543" alt="descifro hash y ontengo usuario y clave" src="https://github.com/user-attachments/assets/96f97824-0c2e-4373-95b2-9386c2b6de66" />

---

## Paso 7: Escalada horizontal y obtención de flag de usuario 🔑🏳️

Después de descifrar la password secreta con John the Ripper, la utilizamos para cambiar de usuario en la máquina comprometida, accediendo como el usuario `merlin`.

### Importando la clave y descifrando con GPG

- Importamos la clave `tryhackme.asc` en el sistema.
- Usamos la contraseña obtenida para descifrar el archivo `credential.pgp`, revelando credenciales útiles para la escalada.

````bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
````

- La clave descifrada corresponde a la contraseña de `merlin`.

<img width="950" height="626" alt="en la consola ssh probamos la credencial y descifro otroas credenciales" src="https://github.com/user-attachments/assets/ad221aac-f364-4366-9514-ffa2a9dfb8ac" />

### Cambio de usuario y captura de la flag

- Cambiamos de usuario con `su merlin` usando la contraseña recuperada.
- Accedemos al directorio home de `merlin` y leemos el contenido de `user.txt`, obteniendo la flag de usuario.

````bash
su merlin
cd ~
cat user.txt
````

<img width="943" height="322" alt="cambiamos de usuario y obtneemos la flag user" src="https://github.com/user-attachments/assets/709ffe9f-8c85-4d9a-90ae-c0298deff0df" />

---

## Paso 8: Escalada de privilegios y obtención de la flag root 🛡️👑

Tras acceder como el usuario `merlin`, analizamos los privilegios sudo disponibles para encontrar una posible vía de escalada.

### Comprobando privilegios sudo y binarios vulnerables

- Usamos `sudo -l` y observamos que el usuario puede ejecutar `/usr/bin/zip` como root sin necesidad de contraseña (`NOPASSWD`):

````bash
sudo -l
````

<img width="959" height="176" alt="vemos privilegios del usuario y encontramos bin" src="https://github.com/user-attachments/assets/3ca9903c-ff3e-4b24-96a7-0349e74914e0" />

### Consultando GTFOBins para explotación

- En el sitio GTFOBins encontramos que es posible escalar privilegios explotando la configuración insegura de `zip` para obtener una shell de root:

````bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
````

<img width="959" height="208" alt="buscamos en pag GTFOBins" src="https://github.com/user-attachments/assets/2097da2d-430a-464e-9a02-257e64f39fae" />

### Obteniendo shell root y la flag definitiva

- Ejecutamos la técnica obtenida en GTFOBins y conseguimos una shell interactiva como root.
- Accedemos al directorio `/root`, donde obtenemos la flag final de la máquina: `root.txt`.

````
cd /root
cat root.txt
````

<img width="587" height="298" alt="elevo privilegios y obtengo flag root" src="https://github.com/user-attachments/assets/002a20a4-1325-4db2-80bf-ed6ab83e0cf2" />

---














