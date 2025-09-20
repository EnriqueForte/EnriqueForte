# 🕵️ El Bandito — Walkthrough (TryHackMe - Difícil)

## 📌 Introducción

El Bandito es un CTF de TryHackMe orientado a la práctica de reconocimiento, enumeración y explotación de servicios comunes en máquinas Linux.

En este writeup seguiremos un flujo típico: reconocimiento pasivo/activo → enumeración de puertos y servicios → enumeración específica por servicio → obtención de acceso inicial → escalada de privilegios y captura de flags.

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


Lo que muestran tus pruebas es que el servidor objetivo realiza una petición HTTP hacia la IP que tú controlas. 

En la captura se ve claramente la petición GET / HTTP/1.1 hacia 10.11.147.155:8088 y tu python3 -m http.server 8088 ha registrado la conexión. 

Eso significa SSRF (al menos de tipo “blind/http-request”): el servidor puede alcanzar recursos que tú especifiques en el parámetro url de /isOnline.

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
