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


