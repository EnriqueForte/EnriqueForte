# Conclusiones

## Resumen del ejercicio

En este CTF What's Your Name / WorldWAP se realizó una auditoría de seguridad orientada a la enumeración, explotación y obtención de banderas.

Las pruebas permitieron identificar varias vulnerabilidades críticas que dieron acceso a cuentas privilegiadas y exposición de información sensible (credenciales en texto plano, cookies sin HttpOnly, XSS persistente).

### Hallazgos principales

XSS persistente (alta)

Campos del formulario de registro aceptan y almacenan HTML/JS sin sanitizar.

La cookie de sesión PHPSESSID no estaba marcada como HttpOnly, lo que permitió robarla vía XSS.

Cookie de sesión sin HttpOnly / Secure (alta)

Documentado y explotado: lectura desde JavaScript y secuestro de sesión.

Ficheros de código fuente expuestos (alta)

admin.py accesible vía HTTP y con credenciales hardcoded en texto claro.

Esto permite autenticarse como administrador y acceder a recursos privilegiados.

Endpoints y ficheros informativos accesibles (media/alta)

/logs.txt, /config.php (o similares) expuestos o accesibles → potencial information disclosure.

Múltiples endpoints API y páginas administrativas detectadas (/api/*, /phpmyadmin, /admin.py, etc.).

Autenticación/Flujos duplicados (media)

Diferencias de comportamiento entre hosts/puertos (:80, :8081, login.worldwap.thm) que podían ser aprovechadas para bypass de lógica.

### Impacto

Obtención de sesiones privilegiadas: acceso a cuentas Moderator/Admin sin credenciales válidas inicialmente (secuestro por XSS y credenciales reveladas).

Acceso a backend/DB: posibilidad de consultar y modificar la base de datos (ej. marcar usuarios como verificados, extraer flags, modificar roles).

Exposición de información sensible: contraseñas y datos de configuración en código o logs.

Riesgo de escalada y persistencia: si existieran endpoints de upload o ejecución, podría conseguirse ejecución remota o webshell.

### Recomendaciones de mitigación (prioridad y acciones)

#### Inmediatas (críticas)

Marcar cookies de sesión como HttpOnly y Secure (si se usa HTTPS):

En PHP:

session_set_cookie_params(['httponly'=>true,'secure'=>true,'samesite'=>'Lax']);

session_start();


Cerrar acceso web a ficheros de código fuente (*.py, *.env, config.php), y revisar reglas del servidor (Nginx/Apache) para no servir directorios de aplicación.

Eliminar credenciales hardcoded; rotar contraseñas encontradas y usar variables de entorno o un gestor de secretos (vault).

#### Corto plazo (alta prioridad)

Sanitizar y escapar todas las entradas que se persisten y se muestran (OWASP XSS Prevention). Implementar librerías de sanitización en servidor.

Revisar y restringir phpMyAdmin: acceso solo desde IPs de administración o vía VPN; cambiar credenciales por defecto.

Eliminar/limitar logs.txt o cualquier archivo que pueda exponer información sensible y asegurar que los logs no se publican en rutas públicas.

#### Mediano plazo

Implementar Content Security Policy (CSP) para mitigar XSS y carga de scripts externos.

Auditoría de código y pruebas automatizadas (SAST/DAST), escaneo de endpoints y pruebas regulares de penetración.

Política de gestión de secretos: rotación, almacenamiento seguro y revisión de commits para evitar fugas.

## Recomendaciones operativas

Auditoría completa de logs para identificar accesos no autorizados y posibles abusos durante el periodo de exposición.

Rotación inmediata de credenciales encontradas y forzado de sesión a todos los usuarios si procediera.

Aplicar parches y hardening del servidor web y dependencias (actualizar Apache, PHP, servicios).

Formación/concienciación: para desarrolladores sobre no incluir secretos en código y buenas prácticas de input/output encoding.

## Ética y legalidad

Todas las pruebas fueron realizadas únicamente en el entorno controlado del CTF.

No está permitido replicar estos ataques contra sistemas ajenos sin autorización explícita.

Los materiales del repositorio han sido depurados para no exponer flags ni credenciales reales públicamente.
