# 🕵️ CTF: Hammer

📌 **Plataforma**: [TryHackMe – Hammer](https://tryhackme.com/room/hammer)  
🎯 **Dificultad**: Media  
📅 **Fecha de resolución**: [03/09/2025]  

---

## 🎯 Objetivo del reto
El objetivo del CTF Hammer es **comprometer la máquina y obtener las flags ocultas**.  
Se centra en técnicas de **enumeración web, explotación de vulnerabilidades y escalada de privilegios en Linux**.  

---

## 🛠️ Herramientas utilizadas
- `nmap` – Escaneo de puertos y servicios  
- `gobuster` / `dirsearch` – Descubrimiento de directorios  
- `nikto` – Detección de vulnerabilidades web  
- `hydra` – Ataques de fuerza bruta  
- `Burp Suite` – Análisis e inyección en aplicaciones web  
- `john` / `hashcat` – Cracking de contraseñas  
- `linpeas.sh` – Escalada de privilegios  

---

## 📌 Pasos realizados

## 🔎 Paso 1: Verificación de conectividad

Lo primero fue comprobar que la máquina objetivo estaba activa y accesible desde mi Kali.  
Para ello ejecuté un **ping** con 4 paquetes hacia la dirección IP proporcionada (`10.10.48.44`):

<img width="799" height="429" alt="Ping" src="https://github.com/user-attachments/assets/dcdc7ac9-9701-4478-8381-383026026e73" />

📌 Resultado:

Se recibieron las 4 respuestas correctamente.

No hubo pérdida de paquetes.

El tiempo medio de respuesta fue de ~49 ms.

✅ Con esto confirmamos que la máquina está activa y lista para comenzar con la fase de enumeración.

## 🔎 Paso 2: Prueba de acceso vía navegador

Después de confirmar la conectividad con **ping**, probé abrir la dirección IP de la máquina (`10.10.48.44`) en el navegador para comprobar si había un servicio web disponible.

📌 **Acceso probado**:  
- `http://10.10.48.44`  
- `https://10.10.48.44`
  
<img width="898" height="794" alt="prueba explorador" src="https://github.com/user-attachments/assets/179649c5-ce5c-4e27-9c34-bce6ab10ce6e" />

❌ **Resultado:**  
No se obtuvo respuesta desde el navegador. La conexión fue rechazada, indicando que **no hay un servicio web activo en los puertos estándar (80/443)** o bien que está restringido.
✅ Esto confirma la necesidad de realizar un **escaneo de puertos con Nmap** para descubrir qué servicios realmente están abiertos en la máquina.

## 🔎 Paso 3: Escaneo de puertos con Nmap

Dado que el navegador no devolvía respuesta en los puertos estándar (80/443), procedí a realizar un **escaneo de todos los puertos TCP** con Nmap para identificar qué servicios estaban expuestos.

Ejecuté el siguiente comando:

```bash
nmap -p- --min-rate 1000 -T4 10.10.48.44
```

<img width="774" height="194" alt="scaneo puertos" src="https://github.com/user-attachments/assets/e6eb9445-e641-439c-b202-cc446b81cdfd" />

📌 Explicación del comando:

-p- → escanea todos los puertos (1-65535).

--min-rate 1000 → acelera el escaneo enviando al menos 1000 paquetes por segundo.

-T4 → aumenta la agresividad para mayor rapidez.

10.10.48.44 → IP de la máquina objetivo.

📌 Resultados obtenidos:

22/tcp → SSH (abierto)

1337/tcp → Servicio desconocido identificado como waste

✅ Con esto confirmamos que el acceso web por 80/443 está cerrado, pero existe un puerto no estándar (1337) que probablemente contenga el servicio vulnerable a explotar.

## 🔎 Paso 4: Exploración del puerto 1337

Tras el escaneo de puertos, observamos que en el **1337/tcp** había un servicio abierto desconocido.  
Al acceder vía navegador a `http://10.10.48.44:1337`, se cargó una página con un **formulario de inicio de sesión**:

📌 **Características observadas:**
- Campos de **Email** y **Password**.  
- Opción de recuperación de contraseña (*Forgot your password?*).  
- No hay información de banners ni branding, lo que sugiere una aplicación web personalizada.  

<img width="899" height="645" alt="pagina web" src="https://github.com/user-attachments/assets/3464d25f-27a1-4546-8954-3c2da5e23c2f" />


✅ Este hallazgo indica que el **puerto 1337 aloja una aplicación web** y que probablemente sea el punto de entrada para el CTF.  
Los siguientes pasos deben enfocarse en:
- Probar credenciales por defecto o ataques de fuerza bruta.  
- Intentar **inyecciones SQL (SQLi)** en el formulario.  
- Revisar la funcionalidad de *"Forgot your password?"* como posible vector de ataque.

## 🔎 Paso 5: Revisión del código fuente y hallazgo de convención de directorios

Inspeccioné el **código fuente** de `http://10.10.48.44:1337/` y encontré una pista del desarrollador:

- Recurso cargado: `/hmr_css/bootstrap.min.css`
- Comentario: `<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->`

Esto revela que la app organiza rutas con el prefijo **`hmr_`**, por lo que podrían existir otros directorios interesantes (p.ej. `hmr_js`, `hmr_img`, `hmr_admin`, `hmr_backup`…).

Además, el formulario hace `POST` (acción vacía) con campos `email` y `password`, lo que invita a probar:
- **Enumeración de rutas** con el patrón `hmr_…`
- **SQLi** / **bypass** en el login
- Función de **“Forgot your password?”** (`reset_password.php`) como vector alternativo

<img width="890" height="724" alt="codigo fuente" src="https://github.com/user-attachments/assets/603846d7-8888-4875-989e-9eb5c6600c01" />

## 📌 Paso 6: Fuzzing dirigido y hallazgo de directorios sensibles

Tras identificar que la aplicación usaba prefijo hmr_, realicé un fuzzing con ffuf para descubrir rutas relacionadas:

<img width="771" height="511" alt="fuzzing" src="https://github.com/user-attachments/assets/b1c35fac-b42b-4c90-8cd1-bcac8168bc7d" />

📌 Resultados encontrados:

/hmr_css/

/hmr_img/

/hmr_js/

/hmr_logs/ ✅

## 📌 Paso 7: Exploración de /hmr_logs/

Al acceder a http://10.10.48.44:1337/hmr_logs/ encontré un índice abierto con un archivo de registros:

error.logs

<img width="899" height="399" alt="directrio hmr_logs" src="https://github.com/user-attachments/assets/0c0c9fe0-6d64-4ada-bee8-7fb5a79742a8" />

## 📌 Paso 8: Información sensible en error.logs

Dentro del archivo error.logs aparecieron trazas del sistema y errores de autenticación.
Lo más relevante: credenciales fallidas y nombres de usuario filtrados.

<img width="895" height="433" alt="entro al directorio logs" src="https://github.com/user-attachments/assets/db0d0910-866d-4795-8862-3953d9e84112" />

📌 Ejemplo extraído:

[Mon Aug 19 12:02:34.876543 2024] [auth_core:error] ...
user tester@hammer.thm: authentication failure for "/restricted-area": Password Mismatch

✅ De este log se extrae:

Usuario potencial: tester@hammer.thm

Posibles rutas restringidas: /restricted-area, /protected

Mención de archivos sensibles: /etc/shadow, /var/www/htm/Locked-down

## 🔎 Paso 9: Explotación de la función de reseteo de clave

En la aplicación existía una funcionalidad en `reset_password.php` que permitía **restablecer contraseñas**.  

Al acceder, se pedía un **nuevo password**, pero era necesario un **código de recuperación**.

<img width="881" height="534" alt="pagina reseteo clave" src="https://github.com/user-attachments/assets/29ada7f5-e46e-439f-a45b-da3f13d0ff12" />

## 🔎 Paso 10: Obtención de PHPSESSID y fuerza bruta del código de recuperación

Capturé mi **PHPSESSID** desde la sesión del navegador y lo utilicé en un script Python (`pin-brute.py`) que probaba códigos de recuperación de manera automática.

Ejecución:

```bash
python3 pin-brute.py
Enter the target IP address: 10.10.48.44
Enter the target port (1337): 1337
Enter your PHPSESSID cookie: 1en9f6fhm6dflfehuvn4orchbg
```

<img width="791" height="381" alt="script python para codigo" src="https://github.com/user-attachments/assets/6ccb9f41-b8f8-4b9f-93f6-835482bd0b08" />

📌 Resultado:

El script encontró el código de recuperación válido → 2935

## 🔎 Paso 11: Reset de la contraseña y acceso al Dashboard

Con el código válido, restablecí la clave de acceso del usuario.
Al iniciar sesión, accedí al Dashboard de la aplicación, donde apareció la primera flag:

📌 Flag 1 → THM{xxxxxxxxxxxx}

✅ Ahora ya contamos con un acceso válido al panel con ejecución de comandos disponible y la primera flag obtenida.

 <img width="896" height="584" alt="flag 1" src="https://github.com/user-attachments/assets/45c341ef-d4b9-424c-a167-7c8bdc4bc483" />

 ## 🔎 Paso 12: Command Injection en el Dashboard

Tras acceder al **Dashboard** con la contraseña reseteada, encontré una funcionalidad de "ejecutar comandos".  
Al introducir `ls`, el servidor devolvió un listado de ficheros y directorios, confirmando que el campo permite **ejecución remota de comandos** (RCE).

📌 **Salida obtenida**:

**188ade1.key
**composer.json
**config.php
**dashboard.php
**execute_command.php
**hmr_css
**hmr_images
**hmr_js
**hmr_logs
**index.php
**logout.php
**reset_password.php
**vendor

<img width="884" height="647" alt="listar directorios desde la web" src="https://github.com/user-attachments/assets/35ca3e3a-93dd-477a-8b80-75bfad8b1726" />

✅ Esto confirma que tenemos **control parcial sobre el sistema** y podemos ejecutar comandos arbitrarios desde la aplicación web.

 ## 🔎 Paso 13: Exposición de JWT Token en el cliente

Revisando el código fuente de `dashboard.php`, encontré un fragmento de JavaScript donde el desarrollador incluyó un **JWT Token** directamente en el cliente:

<img width="1171" height="677" alt="codigo fuente de la pagina principal" src="https://github.com/user-attachments/assets/322c73a5-b395-413d-8ec6-792f0ba4a348" />

```javascript
var jwtToken = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsInR…";
```
📌 Paso 14: Decodificación del JWT y descubrimiento de clave secreta

<img width="1237" height="780" alt="deodifcando token jwr" src="https://github.com/user-attachments/assets/6fc44898-9837-4fab-86a0-9aa8cd463f22" />

Revisando el token JWT en jwt.io, observé:
{
  "typ": "JWT",
  "alg": "HS256",
  "kid": "/var/www/mykey.key"
}

{
  "iss": "http://hammer.thm",
  "iat": 1756916300,
  "exp": 1756919900,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "user"
  }
}

📌 Hallazgos:

El campo kid apunta a un archivo local /var/www/mykey.key, indicando que la clave de firma JWT está guardada en texto plano en el servidor.

El token asigna al usuario el rol "user".

## 📌 Paso 15: Acceso al archivo de clave

Dentro del sistema, encontré un archivo llamado 188ade1.key.

<img width="763" height="145" alt="leyendo el archivo Key" src="https://github.com/user-attachments/assets/79f8e1cc-a843-4444-9966-a8f78ff46bb0" />

Al visualizarlo con cat, apareció lo siguiente:

cat 188ade1.key
56058354efb3daa97ebab00fabd7a7d7


📌 Esto corresponde a la clave secreta usada para firmar los JWT.
Con esta clave es posible:

Recrear un JWT válido.

Modificar el campo "role": "user" → "role": "admin".

Firmar el nuevo token con esta clave, logrando escalar privilegios en la aplicación.

✅ Con este descubrimiento, el próximo paso será generar un nuevo JWT con rol administrador y usarlo en las cabeceras Authorization: Bearer <token> para obtener acceso extendido.

## 🔎 Paso 16: Generación de un JWT con rol de administrador

Con la clave secreta encontrada (`188ade1.key`), generé un nuevo token JWT en el que modifiqué el payload para elevar privilegios:

```json
{
  "iss": "http://hammer.thm",
  "iat": 1756916300,
  "exp": 1756919900,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "admin"
  }
}
```

Luego utilicé un script en Python (generar_token.py) para firmar este payload con la clave secreta y producir un JWT válido:

<img width="1052" height="171" alt="genero nuevo token con admin" src="https://github.com/user-attachments/assets/881b8aa1-d809-4e06-abac-902f08b3eb06" />

📌 Resultado:

[+] JWT generado:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwOi...

✅ Con este token ahora es posible autenticarse como administrador en la aplicación, lo que permite acceso completo a funcionalidades restringidas y potencialmente a otras flags.

## 🔎 Paso 17: Uso del JWT admin en Burp Suite

Con el token JWT generado con privilegios de administrador, probé a **inyectarlo en las cabeceras de autorización** usando Burp Suite en el endpoint:

POST /execute_command.php HTTP/1.1
Host: 10.10.48.44:1337
Authorization: Bearer <JWT_ADMIN_GENERADO>
Content-Type: application/json

{
"command": "id"
}

📌 **Respuesta**:


Esto confirmó que tenía ejecución remota de comandos en el servidor, ahora autenticado como **admin**.

<img width="1143" height="613" alt="capturar solicitud de comando burpsuite token admin" src="https://github.com/user-attachments/assets/f91712aa-5f3f-454d-8a9b-b0ac779c5837" />

---

## 🔎 Paso 18: Exploración del sistema y búsqueda de flags

Posteriormente ejecuté:

<img width="1241" height="602" alt="capturar solicitud de comando burpsuite" src="https://github.com/user-attachments/assets/eaa88bf8-b4f5-4a90-bace-b18ee188533a" />

```json
{
  "command": "ls"
}

188ade1.key
composer.json
config.php
dashboard.php
execute_command.php
hmr_css
hmr_images
hmr_js
hmr_logs
index.php
logout.php
reset_password.php
vendor
```

## 🔎 Paso 19: Lectura de la segunda flag

Finalmente, encontré el archivo de flag en /home/ubuntu/flag.txt.
Ejecuté:

{
  "command": "cat /home/ubuntu/flag.txt"
}


📌 Resultado:

THM{RUNANYCOMMAND1337}

✅ Con esto, se obtuvo la segunda flag del CTF Hammer, demostrando el control total sobre el sistema a través de la vulnerabilidad de Command Injection y el mal manejo de JWT.

<img width="1099" height="626" alt="flag2" src="https://github.com/user-attachments/assets/8cbb03bf-cbe4-4d54-8edf-99d4027b5648" />


