# 🕵️ El Bandito — Walkthrough (TryHackMe - Difícil)

## 📌 Introducción

El Bandito es un CTF de TryHackMe orientado a la práctica de reconocimiento, enumeración y explotación de servicios comunes en máquinas Linux.

En este writeup seguiremos un flujo típico: reconocimiento pasivo/activo → enumeración de puertos y servicios → enumeración específica por servicio → obtención de acceso inicial → y captura de flags.

## Paso 1 — Recon básico / Ping de comprobación

Objetivo: comprobar que la máquina objetivo está viva y responde a ICMP (ping). Confirmar que la IP 10.10.186.9 está accesible antes de iniciar escaneos activos.

Captura (comando ejecutado)

<img width="634" height="333" alt="Ping" src="https://github.com/user-attachments/assets/41b67383-4736-4e1d-9f06-5d8f3c2adf5d" />

Comando usado:

ping -c 4 10.10.186.9

### Análisis / Observaciones

La máquina 10.10.186.9 responde correctamente a ICMP sin pérdida de paquetes (0% packet loss).

Tiempos de respuesta constantes (≈42–45 ms) indican conectividad estable.

ttl=63 puede indicar un sistema Linux (TTL inicial habitual 64), aunque no es concluyente por sí solo.

### Herramientas utilizadas

ping (comprobación básica de conectividad)

## 🔎 Paso 2 — Escaneo de puertos y servicios (resultado del nmap)

Objetivo: interpretar los resultados del escaneo inicial y planificar la enumeración por servicio.

Captura (salida nmap)

<img width="1199" height="998" alt="nmap puertos" src="https://github.com/user-attachments/assets/b055c8ed-32ea-4221-a5f9-b683d746987a" />

### Resumen de puertos y servicios detectados (extraído de la captura)

22/tcp — open ssh — OpenSSH 8.2p1 (Ubuntu)

80/tcp — open http/ssl — Server: El Bandito Server (respuestas HTTP, 404 Not Found en algunas rutas)

631/tcp — open ipp — CUPS (impression: CUPS 2.4.7 aparece en la salida)

8080/tcp — open http — aparece información relacionada con Spring Java Framework / servidor web (posible aplicación Java en 8080)


### Interpretación y prioridades

Web (puertos 80 y 8080) — prioridad alta

Hay dos entradas HTTP distintas: puerto 80 (servidor "El Bandito Server") y puerto 8080 (aplicación Java / Spring).

Estos son candidatos principales para enumeración de directorios, ficheros expuestos, endpoints de API, formularios, vulnerabilidades típicas (RCE, SSTI, deserialización de Java, etc.).

CUPS (631) — prioridad media

CUPS normalmente tiene una interfaz web administrativa (/admin) y endpoints IPP; puede presentar páginas con información o incluso impresoras compartidas con ficheros.

SSH (22) — prioridad media-baja

Servicio activo; revisar versión. Evitar fuerza bruta salvo que haya permiso. Buscar credenciales encontradas en la enumeración web o archivos expuestos.

### Cabeceras y seguridad

Content-Security-Policy, X-Frame-Options y demás indican un mínimo de protección en headers. No implica que no existan vulnerabilidades en la app.

## 🔎 Paso 3 — Enumeración web con Gobuster (puertos 80 y 8080)

Objetivo: usar los resultados de gobuster para identificar rutas interesantes en los servidores web (puerto 80 y 8080) y decidir la siguiente fase de enumeración/ataque.

Capturas (salida Gobuster)

Puerto 80

<img width="897" height="353" alt="gobuster puerto 80" src="https://github.com/user-attachments/assets/95f610a7-980f-46ba-a39a-6a4e8e5ff282" />

Puerto 8080

<img width="966" height="779" alt="gobuster puerto 8080" src="https://github.com/user-attachments/assets/74818917-c776-4eeb-bdcf-182de131748f" />


### Rutas detectadas (extraídas de las capturas)

#### En http://10.10.207.0:80 (puerto 80)

/login — Status: 405 (Size: 153)

/static — Status: 301 → http://10.10.207.0/static/ (Size: 169)

/access — Status: 200 (Size: 4871)

/messages — Status: 302 (Size: 189) → redirecciones

/logout — Status: 302 (Size: 189)

/save — Status: 405 (Size: 153)

/ping — Status: 200 (Size: 4)

(otros: /admin_images, etc. dependiendo del wordlist)

Observación: el puerto 80 parece ofrecer rutas de una app con login, mensajes, accesos y una ruta simple /ping que devuelve respuesta corta.

#### En http://10.10.207.0:8080 (puerto 8080 — aplicación Java / Spring)

/info — Status: 200 (Size: 2)

/admin — Status: 403 (Size: 146)

/health — Status: 200 (Size: 150)

/assets — Status: 200 (Size: 0)

/trace — Status: 403 (Size: 146)

/environment — Status: 403 (Size: 146)

/envelope_small — Status: 403 (Size: 146)

/error — Status: 500 (Size: 88)

/token — Status: 200 (Size: 8)

/metrics, /beans, /env, /dump, /actuator (variantes y protegidas con 403)

Observación: aparecen muchas rutas típicas de Spring Boot Actuator (/info, /health, /env, /metrics, /trace, etc.). Varias devuelven 403 (protegidas) pero /info, /health y /token responden 200, lo cual es relevante.

## 🔎 Paso 4 — Análisis de endpoints detectados y siguientes intentos (HTTP, CUPS y Actuator)

Objetivo: interpretar la respuesta vista en :631 (CUPS) y continuar la enumeración / prueba de los endpoints web importantes (/token, /info, /health, /ping, /access, /static) buscando información útil o vectores de explotación.

Captura — intento de acceso a CUPS (puerto 631)

<img width="680" height="284" alt="puerto 631 no se puede acceder" src="https://github.com/user-attachments/assets/ace3b02f-b17a-4ec9-82a1-f1c9fa039a31" />

### Observación: la interfaz web de CUPS está presente pero devuelve Forbidden / You cannot access this page. 

Esto indica que el servicio está corriendo, pero la interfaz está protegida (política de acceso, autenticación o binding a localhost). 

No significa que no haya información útil vía IPP/NSE o endpoints mal configurados.

### Interpretación rápida

Forbidden en el navegador → CUPS activo pero acceso web bloqueado (403).

Aun así, CUPS puede exponer información vía el protocolo IPP (Puerto 631) que nmap --script ipp-info puede devolver.

No intentes fuerza bruta web/admin sin encontrar credenciales o permiso — primero enumerar y buscar info reveladora (versiones, impresoras, archivos subidos, etc.).

## 🔎 Paso 5 — Análisis de la respuesta en https://10.10.186.9:80

<img width="648" height="219" alt="puerto 80 ejecuta en https" src="https://github.com/user-attachments/assets/2ea1a256-8afe-4e2a-a330-c3c5548e2080" />

Contexto: se accede por navegador a https://10.10.186.9:80 y la página muestra nothing to see. 

Esto confirma que el servidor web en el puerto 80 está respondiendo (incluso vía HTTPS forzado) pero la página pública no revela contenido interesante. 

Aun así, por los resultados previos (Gobuster) sabemos que hay rutas útiles (/login, /access, /ping, /static, y el servicio en :8080 con endpoints tipo Actuator).

## 🔎 Paso 6 — Encontrado /static/messages.js codigo fuente puerto 80

Inspeccionado el código fuente y vemos que la página raiz incluye el script:

<img width="676" height="179" alt="codigo fuente p80" src="https://github.com/user-attachments/assets/dee21266-6a10-4021-a8d4-9a34df6d6f4c" />

<script src='/static/messages.js'></script>

Eso es buena noticia: los ficheros JS suelen contener endpoints, rutas internas, tokens hardcodeados o pistas. 

## 🔎 Paso 7 — Análisis del messages.js y pruebas concretas

<img width="874" height="854" alt="archivo messages js" src="https://github.com/user-attachments/assets/dd698713-2247-4adf-b2cc-1bba5ba830b5" />

El messages.js muestra que la app hace fetch("/getMessages") y manipula el DOM con appendMessage(...). Eso nos da pistas claras:

Hay un endpoint GET /getMessages que devuelve los mensajes (JSON).

Existe lógica cliente para mostrar mensajes directamente en el DOM — si el código inserta innerHTML con el contenido del mensaje, podría no sanitizar y permitir stored XSS.

## 🔎 Paso 8 — Revisión puerto 8080 y 8080/services.html detecta un vector SSRF vía /isOnline (todo en Markdown)

<img width="1445" height="773" alt="pagina puerto 8080" src="https://github.com/user-attachments/assets/4dd742fe-7118-4291-8412-c74e02cf7fde" />


<img width="1449" height="809" alt="puerto 8080 services" src="https://github.com/user-attachments/assets/198db954-57bc-4e02-af1a-1e2a8d33bbd5" />


Captura: services.html construye dinámicamente URLs y llama a:

fetch(`/isOnline?url=${serviceUrl}`, { method: 'GET' })

Eso significa que hay un endpoint servidor /isOnline que recibe una URL y el servidor web hará una petición a esa URL para comprobar si está online.

Si el backend no valida/filtra el parámetro url, esto es un claro vector SSRF (Server Side Request Forgery). 

Con SSRF podemos pedir al servidor que consulte recursos internos (ej.: http://127.0.0.1:8080/actuator/health, http://localhost:631/, http://10.10.186.9:8080/actuator/env) y devolver la respuesta al atacante.

## 🔎 Paso 9 — WebSocket (/ws) en burn.html: pruebas y explotación (TODO en Markdown)

<img width="1440" height="758" alt="puerto 8080 burn" src="https://github.com/user-attachments/assets/1f7d3874-fe79-4ce3-a46c-75bb38728b45" />


<img width="1336" height="789" alt="codigo fuente burn" src="https://github.com/user-attachments/assets/11c99da6-adfb-4338-be0f-9cd34afce0f7" />


Encontrado en burn.html la lógica del WebSocket:

var wsUri = (window.location.protocol === 'https:' ? 'wss://' : 'ws://') + window.location.host + '/ws';

webSocket = new WebSocket(wsUri);
```
// En el submit:
var message = {
  action: "burn",
  address: $("#address").val(),
  amount: $("#amount").val()
};
webSocket.send(JSON.stringify(message));
```

Esto nos dice 3 cosas importantes:

Existe (o existió) un endpoint WebSocket en /<host>/ws (ej. ws://10.10.186.9:8080/ws).

El cliente envía JSON con action, address y amount (p. ej. {"action":"burn","address":"0x...","amount":100}).

En la consola aparece WebSocket is closed now. y errores — por tanto el WS está cerrado / no disponible desde el navegador en el momento de prueba, pero podemos intentar conectarlo de forma directa y/o investigar por qué está cerrado.

## 🔎 Paso 10 Resultado y siguientes pasos — SSRF confirmado (Burp Suite y servidor en Python)

<img width="404" height="269" alt="prueba para SSRF en burn puerto 8080" src="https://github.com/user-attachments/assets/e3a5acb3-161e-4d1b-b829-9fc42a901675" />

<img width="568" height="145" alt="respuesta prueba para SSRF en burn puerto 8080" src="https://github.com/user-attachments/assets/a8e068cf-f759-40ff-b4a3-0a248b7463d4" />


Lo que muestran tus pruebas es que el servidor objetivo realiza una petición HTTP hacia la IP que controlamos. 

En la captura se ve claramente la petición GET / HTTP/1.1 hacia 10.11.147.155:8088 y python3 -m http.server 8088 ha registrado la conexión. 

Eso significa SSRF (al menos de tipo “blind/http-request”): el servidor puede alcanzar recursos que especifiquemos en el parámetro url de /isOnline.

### ¿Qué hemos confirmado?

El endpoint vulnerable es:

````
GET /isOnline?url=<URL>
````
El servidor intenta conectar a la URL — hemos verificado porque nuestra máquina vio la petición HTTP entrante.

## 🔎 Paso 11 — Servidor HTTP controlado en Python para explotar / probar SSRF

<img width="646" height="326" alt="creamos script python server" src="https://github.com/user-attachments/assets/f5f2cadf-72c3-44a6-a56d-9c7230b4934b" />

Creo un servidor HTTP propio para responder a las peticiones que genere el /isOnline es la forma más práctica para confirmar SSRF, 

obtener cabeceras completas, probar distintos cuerpos de respuesta, simular actuator y hacer timing attacks.

## 🔎 Paso 12 Análisis rápido: probando servidor creado en Python


<img width="760" height="301" alt="Burn suite probando servidor hacia script python" src="https://github.com/user-attachments/assets/131c9eb3-a5ff-408a-aa36-ba59570acdff" />

<img width="544" height="101" alt="probamos servidor mediante script" src="https://github.com/user-attachments/assets/5529abbf-a4fb-453f-8a43-8c5d124744f0" />

1 .Petición GET /isOnline?url=http://10.11.147.155:8087 el objetivo realmente hizo la petición al servidor (ssrf_server.py la registró).

2 .La respuesta que devolvió el objetivo en la pestaña Response fue HTTP/1.1 101 (Switching Protocols) y en las cabeceras aparece X-Application-Context: application:8081.

### Interpretación inmediata:

El backend del objetivo respondió al handshake con 101 Switching Protocols — es un indicio de que el servidor objetivo intentó upgrade (probablemente WebSocket) hacia la URL que le proporcionamos o hacia algún upstream.

La cabecera X-Application-Context: application:8081 sugiere además que la petición fue manejada/pasada por un componente (probablemente Spring/Nginx) 

que está vinculado a una aplicación interna escuchando en :8081. Esa información es valiosa para la enumeración interna.

## 🔎 Paso 13 — Resultado del SSRF: hemos descubierto /admin-creds en la app interna (MD)

<img width="903" height="512" alt="probando socket puerto 8080 ⁄ trace" src="https://github.com/user-attachments/assets/c411b08f-56a8-497f-9d1a-aad0319f00b2" />

### Resumen breve:

Las últimas pruebas muestran que al invocar /isOnline?url=... el backend no sólo conecta, sino que estamos recibiendo trazas/httptrace internas que revelan peticiones hechas por la aplicación. 

En las trazas aparece la ruta /admin-creds (método GET, status: 200) en la aplicación interna (X-Application-Context: application:8081). 

Eso significa que podemos pedir explícitamente esa URL vía SSRF para leer su contenido (probablemente credenciales administrativas).

### Interpretación de la captura

La respuesta del servidor a tu SSRF mostró JSON con entradas que contienen "path": "/admin-creds" y response: { "X-Application-Context": "application:8081", ..., "status":"200" }.

Esto indica que la aplicación interna tiene un endpoint /admin-creds y que internamente responde con 200 — es muy probable que devuelva credenciales o información de admin.

## 🔎 Paso 14 — Credenciales extraídas y siguientes pasos (todo en Markdown)

Resultado: mediante SSRF (/isOnline) y lectura de trazas internas (/trace) hemos obtenido directamente las credenciales administrativas expuestas en /admin-creds:

Captura: la respuesta mostró HTTP/1.1 200 y el body username:hAckLIEN password:YouCanCatchUsInYourDreams404

<img width="902" height="388" alt="clave del admin en directorio trace" src="https://github.com/user-attachments/assets/767a17f9-fa5c-4a5b-a49b-fdd6472324a0" />

### Qué significa esto

Tener credenciales administrativas abre varias vías:

Autenticación en la web (formulario /login) para acceder a paneles/funcionalidad protegida.

Si las credenciales también sirven para SSH (posible si hay un usuario local con ese nombre), podríamos iniciar sesión por SSH.

Acceso a endpoints internos o acciones administrativas (reactivar WebSocket, ver flags, leer archivos, etc.).

### 🔎 Paso 15 — Explotación final: extracción de la flag vía SSRF (Primera Flag)

**Vulnerabilidad encontrada:** la página `services.html` hace `fetch('/isOnline?url=...')`. El endpoint `/isOnline` no valida el parámetro `url`, permitiendo SSRF.

**Ejecución:** apunté el parámetro `url` a la aplicación interna indicada por `X-Application-Context: application:8081` y probé varias rutas internas. Al solicitar `/admin-flag` (vía SSRF) obtuve la bandera:

<img width="902" height="352" alt="primera flag en directorio admin-flag" src="https://github.com/user-attachments/assets/09810c13-d404-41d6-83d0-b531556c466c" />


Resultado: la respuesta contenía la flag:

THM{...}

Conclusión: mediante SSRF se pudo leer recursos internos expuestos por la app en :8081, incluida la ruta /admin-flag que contenía la flag del reto.

### 🔎 Paso 16 Acceso al panel Access del puerto 80 mediante credenciales obtenidas

#### Objetivo: usar las credenciales extraídas (hAckLIEN:YouCanCatchUsInYourDreams404) para autenticarse en la aplicación y comprobar funciones disponibles en el panel (por ejemplo el chat / mensajes).

<img width="1027" height="627" alt="accedo al menu puerto 80 con claves obtenidas" src="https://github.com/user-attachments/assets/2dbfe569-ccbc-47e8-98fe-325fb10efb09" />


<img width="1276" height="511" alt="accedo a un chat" src="https://github.com/user-attachments/assets/c1c7e018-f091-44b0-a0f4-0ddf81a9288d" />

### 🔎 Paso 16 Prueba XSS (envío de mensaje)

Petición observada en Burp: `POST /send_message` con `data=hello` y cookie de sesión.

<img width="520" height="487" alt="capturo solcitud enviar mensaje" src="https://github.com/user-attachments/assets/02e8e1dd-5cfe-4181-9828-76a597f56ce4" />

### La captura muestra lo siguiente (resumen):

POST /send_message

Content-Type: application/x-www-form-urlencoded

Cookie de sesión presente (session=...) — estás autenticado.

Payload del formulario: data=hello

Eso significa que sabemos cómo enviar mensajes con la sesión autenticada. Si messages.js inserta esos mensajes con innerHTML (como vimos antes),

podriamos probar stored XSS inyectando un payload controlado y luego comprobar en la interfaz si se ejecuta.

### 🔎 Paso 17 Prueba envío de mensajes

<img width="1094" height="686" alt="probando summuglin" src="https://github.com/user-attachments/assets/f957ed21-66b4-4138-a02f-11c4f12d1595" />

### Descripción de la prueba realizada

Objetivo: comprobar cómo procesa y almacena la aplicación los mensajes enviados desde la interfaz autenticada, evaluar si es posible un stored XSS o si hay mitigaciones (escape/CSP).

Qué se capturó (en la imagen):

Una petición POST /messages seguida por POST /send_message (la app usa /send_message para publicar mensajes).

Content-Type: application/x-www-form-urlencoded.

Cookie de sesión presente en la petición (session=...) — indica que estabas autenticado.

Payload enviado en el cuerpo: data=what (ejemplo de prueba).

Respuesta del servidor al acceder a /messages (HTTP/2 200) con cabeceras relevantes:

Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';

X-Content-Type-Options: nosniff

X-Frame-Options: SAMEORIGIN

En el body de la respuesta se ve el HTML de la página messages (estructura del chat y los mensajes ya renderizados).

No hay respuestas en el chat por lo que no funcionó la prueba


<img width="1259" height="634" alt="no hay respuestas aun " src="https://github.com/user-attachments/assets/dae660be-2791-456b-a017-be96f0ea4b74" />


## 🔎 Paso 18 Prueba variando parámetros en POST /send_message (resultado: 503 Backend fetch failed)

###Descripción breve

Hice una nueva prueba enviando un mensaje con parámetros diferentes al endpoint de mensajes (POST /send_message). 

La petición fue aceptada por el front-end/reverse-proxy pero la respuesta devolvió un error 503 Backend fetch failed con una cabecera/plantilla de Varnish cache server. 

Esto indica que el proxy (Varnish) no pudo conectar o recibir respuesta del backend al procesar la petición.

Qué hice (captura y reproducción):

<img width="1070" height="534" alt="probando summuglin 2 varnish server" src="https://github.com/user-attachments/assets/6db2d8e6-0999-49a6-adf8-704037f56f35" />


Envié un mensaje con un cuerpo distinto (data=vamos nosotros) desde la sesión autenticada (cookie presente en la petición).

Observé la respuesta: HTTP/2 503 Service Unavailable con contenido HTML que contiene 503 Backend fetch failed y mención de Varnish cache server.

## 🔎 Paso 19 Prueba: variación de parámetros / envío múltiple (smuggling de mensajes) y lectura vía puerto 80/getMessages (Segunda Flag)

### Objetivo

Probar si variando cabeceras y enviando múltiples POST al endpoint de envío de mensajes (/send_message) se pueden insertar entradas visibles en el endpoint de lectura (/getMessages) 

y si esto permite introducir/recuperar contenido sensible (por ejemplo flags).

### Descripción de la prueba (lo que hice): 

<img width="1089" height="480" alt="utlimo smugglin variando content" src="https://github.com/user-attachments/assets/eb26ed20-3f04-47a5-9352-aa1928fe78b5" />


Desde una sesión autenticada (cookie session=...) envié varias peticiones POST /send_message con distinto Content-Length y distintos cuerpos (data=a, data=a).

En uno de los intentos la respuesta del proxy fue 503 Backend fetch failed (Varnish), así que reintenté alterando longitudes y añadiendo otro POST inmediatamente después.

Después descargué el endpoint /getMessages y busqué el contenido almacenado/visible — allí apareció el texto inyectado 

y, en uno de los resultados, la segunda flag (formato THM{...}) quedó visible en la salida de getMessages.

<img width="1424" height="646" alt="segunda flag dentro de get message" src="https://github.com/user-attachments/assets/7dd9718d-8d6d-4311-b333-752fd0425843" />




