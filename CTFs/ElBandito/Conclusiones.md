# Conclusiones — El Bandito

## Resumen final

En esta máquina (“El Bandito”) se explotó una **vulnerabilidad SSRF** en el endpoint `/isOnline`, lo que permitió interactuar con servicios internos de la aplicación. 

Aprovechando esa vector se enumeraron endpoints internos (Actuator, trace) y se extrajeron credenciales administrativas en `/admin-creds`.

Con esas credenciales se autenticó en la interfaz web y, mediante interacciones con el sistema de mensajería (`POST /send_message` y `GET /getMessages`), se recuperaron flags del reto.

También se observaron componentes de infraestructura relevantes (Varnish como reverse-proxy y intentos de WebSocket upgrade) que condicionaron la explotación.

---

## Hallazgos clave
- **SSRF sin validación** en `/isOnline` — permite al servidor realizar peticiones a URLs arbitrarias, incluyendo `127.0.0.1` y puertos internos.  
- **Endpoints Spring Boot / Actuator** y trazas (`/trace`) accesibles vía SSRF que revelaron rutas internas útiles (`/admin-creds`, `/admin-flag`).  
- **Credenciales administrativas** expuestas en `/admin-creds` (usadas para login web).  
- **Sistema de mensajería**: `POST /send_message` y `GET /getMessages` permitieron insertar y recuperar contenido; mediante variaciones de peticiones se recuperaron flags.  
- **Varnish** como reverse-proxy: en ocasiones devolvió `503 Backend fetch failed`, lo que requirió reintentos y adaptación de payloads.  
- **CSP (Content-Security-Policy)** presente: `script-src 'self'` limita ejecución de XSS inline; con sesión admin es posible subir JS al mismo origen y sortear esta limitación.

---

## Lecciones aprendidas
1. **SSRF es una vía potente hacia la red interna**: permite identificar y leer servicios no expuestos públicamente.  
2. **Actuator / debug endpoints son riesgosos**: no deben exponerse sin protección.  
3. **Mitigaciones en el cliente (CSP) ayudan pero no sustituyen a la validación server-side**.  
4. **Proxies y caches (p. ej. Varnish) alteran la explotación**: pueden producir respuestas intermitentes; hay que adaptar la táctica (reintentos, variar longitud/headers).  
5. **Encadenamiento de vectores** (SSRF → credenciales → sesión admin → lectura/exfiltración) suele ser necesario para una explotación completa.

---

## Recomendaciones de mitigación
- **Bloquear/filtrar SSRF**: implementar whitelist de hosts/puertos, bloquear esquemas peligrosos (`file:`, `gopher:`, `ws:`, etc.), evitar permitir peticiones a `localhost`/`127.0.0.1`.  
- **Proteger Actuator**: deshabilitar endpoints sensibles o protegerlos con autenticación y ACLs.  
- **No exponer secretos**: eliminar endpoints que devuelvan credenciales o datos sensibles en texto plano.  
- **Escapar/sanitizar el contenido almacenado** que luego se renderiza, y complementar con CSP estricta.  
- **Revisar configuración de proxy (Varnish/NGINX)**: ajustar timeouts, backends y manejo de errores para evitar `503` inesperados.  
- **Registro y monitoreo**: alertas para peticiones inusuales a `/isOnline` o accesos internos vía SSRF.

---

## Resultado final
- Explotación exitosa: flags recuperadas mediante la cadena SSRF → extracción de credenciales → uso de sesión admin → lectura de recursos/flags.  

---
