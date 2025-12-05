# Conclusiones – ChillHack 🧊

En esta máquina hemos encadenado varias técnicas clásicas de pentesting, muy útiles para aprender metodología de principio a fin.

---

## 🕵️‍♂️ 1. Importancia de la enumeración

- Empezamos con reconocimiento básico (`ping`, `nmap`) para identificar servicios expuestos.
- Detectar **FTP anónimo**, **HTTP** y posteriormente servicios internos en localhost (puerto `9001`) fue clave.
- Lección: **sin buena enumeración, no hay buen exploit**. No te saltes `nmap`, gobuster/ffuf ni la revisión manual de directorios.

---

## 💻 2. De RCE a shell inversa

- El panel `/secret/` permitía ejecutar comandos, pero con filtrado de palabras.
- Mediante bypass con barra invertida (`\ls`, `\id`, `ba\sh`) conseguimos **RCE** como `www-data`.
- Desde ahí montamos un servidor HTTP y lanzamos una **reverse shell** usando `curl` + script bash.
- Lección: cuando veas ejecución de comandos limitada, prueba siempre **bypasses simples** (barra invertida, comillas, variables, encoding…).

---

## 👥 3. Movimiento lateral y abuso de sudo

- Enumerando `/home` y permisos con `sudo -l`, vimos que `www-data` podía ejecutar `.helpline.sh` como `apaar`.
- El script ejecutaba directamente el contenido de una variable: `msg`, lo que permitió lanzar `/bin/sh` y obtener shell como `apaar`.
- Lección: revisa siempre scripts que se ejecutan con más privilegios; si ejecutan **input del usuario**, suelen ser un gran vector de escalada.

---

## 🧩 4. Enumeración avanzada: linpeas, servicios internos y base de datos

- Usamos **LinPEAS** para detectar servicios sólo en `127.0.0.1`, como el portal del puerto `9001`.
- Buscando el código fuente de ese portal en `/var/www/files`, encontramos `index.php` con credenciales de **MySQL**.
- Con esas credenciales, accedimos a la base de datos, listamos la tabla `users` y extraímos hashes de contraseñas.
- Lección: los **servicios internos** y las **credenciales en código** son una mina de oro. Siempre revisa `/var/www`, configs y bases de datos.

---

## 🖼️ 5. Esteganografía y análisis de archivos

- El archivo `hacker.php` nos dio una pista (“look in the dark”) que nos llevó a una imagen.
- Con `steghide` extraímos `backup.zip`, que estaba protegido con contraseña.
- Con `zip2john` + John the Ripper y `rockyou.txt` rompimos la contraseña y obtuvimos `source_code.php`.
- Lección: las máquinas CTF suelen combinar técnicas (web, sistema, crypto, stego). Mantén la mente abierta y sigue las pistas.

---

## 🔐 6. De código a credenciales: base64 y lógica de login

- En `source_code.php` encontramos una condición con `base64_encode($password) == "<CADENA_BASE64>"`.
- Decodificando esa cadena con `base64 -d` obtuvimos la contraseña de **Anurodh**.
- Con ella conseguimos acceso al usuario `anurodh` en el sistema.
- Lección: entender la **lógica de autenticación** y las funciones comunes (MD5, Base64, etc.) es esencial para extraer credenciales.

---

## 🐳 7. Escalada final con Docker

- El usuario `anurodh` pertenecía al grupo `docker`, lo que permite controlar contenedores.
- Ejecutando:
````bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
````

montamos el sistema host dentro de un contenedor y obtuvimos una shell como **root**.
- Lección: pertenecer al grupo `docker` es prácticamente equivalente a ser **root**. En auditorías reales, esto es un problema grave de seguridad.

---

## ✅ 8. Resumen de la cadena de ataque

1. Enumeración de servicios (FTP, HTTP, SSH).
2. RCE en `/secret` y reverse shell como `www-data`.
3. Abuso de `sudo` con `.helpline.sh` para moverse a `apaar`.
4. Enumeración con LinPEAS y descubrimiento de servicio interno en `127.0.0.1:9001`.
5. Análisis de código web, credenciales MySQL y extracción de hashes.
6. Esteganografía en imagen, crack de ZIP y análisis de `source_code.php`.
7. Obtención de credenciales de Anurodh.
8. Escalada a `root` mediante Docker.

---

## 🧠 Lección global

ChillHack muestra muy bien cómo un ataque real raramente se basa en **un solo fallo**: es una **cadena** de pequeñas debilidades (RCE filtrado, mala configuración de sudo, credenciales en código, grupo docker con demasiados permisos) que, combinadas, llevan al control total del sistema.

---


