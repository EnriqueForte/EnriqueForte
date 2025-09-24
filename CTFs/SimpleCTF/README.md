# SimpleCTF — Walkthrough

> **Introducción**

SimpleCTF es una máquina de TryHackMe orientada a principiantes que sirve para practicar las fases básicas de un test de intrusión: reconocimiento, enumeración, explotación y post-explotación. En este walkthrough iré documentando paso a paso las pruebas realizadas, las herramientas y los comandos usados, capturas de pantalla y las conclusiones finales.

**Objetivo:** Obtener la(s) bandera(s) contenidas en la máquina objetivo y documentar el proceso de manera reproducible.

---

## 📋 Requisitos

* Máquina atacante con Kali Linux (o similar).
* Conexión a la red de TryHackMe (VPN o red proporcionada por la plataforma).
* Herramientas: `ping`, `nmap`, `gobuster`/`dirb`, `nikto`, `curl`, `ssh`, `netcat` (según necesidad).

---

## 🛠️ Herramientas utilizadas

* Terminal (bash) — comandos básicos de red.
* `ping` — comprobación de conectividad y latencia.

---

## Paso 1 — Comprobación de conectividad (Recon)

Primera comprobación: verificar que la máquina objetivo responde a ICMP y está en línea.

**Comando ejecutado:**

```bash
ping -c 4 10.10.197.170
```

**Resultado observado (captura):**

<img width="601" height="285" alt="Ping" src="https://github.com/user-attachments/assets/b8aa4d89-ec5e-4981-b62a-2e2af1feb5a7" />



**Interpretación:**

* La máquina responde 4 paquetes ICMP con tiempos aproximados de 43-45 ms.
* No hay pérdida de paquetes (`0% packet loss`) — el host está accesible.

**Propósito de este paso:**

Confirmar que la IP objetivo está activa antes de proseguir con la enumeración (escaneo de puertos, enumeración de servicios y directorios, etc.).

---

## Paso 2 — Escaneo de puertos y detección de servicios (Enumeración)

Ejecutamos un escaneo más completo para identificar puertos abiertos, servicios y versiones.

Comando ejecutado:

nmap -sC -sV -T4 -oN nmap_scan 10.10.197.170

Resultado observado (captura):

<img width="882" height="549" alt="Nmap para los puertos" src="https://github.com/user-attachments/assets/941debb6-94b4-40b1-ba21-1c881bf290a8" />


Salida relevante detectada:

21/tcp ftp — vsftpd 3.0.3 (Anonymous FTP login allowed).
Nota: ftp-anon: Anonymous FTP login allowed (FTP code 230) indica que podemos conectar sin credenciales; el listado de directorios puede dar timeout, pero es necesario explorar con un cliente FTP.

80/tcp http — Apache httpd 2.4.18 (Ubuntu) — Página por defecto de Apache.

2222/tcp ssh — OpenSSH 7.2p2 (puerto SSH no estándar).

Interpretación y prioridades:

FTP anónimo es un objetivo prioritario: revisar archivos descargables que pudieran contener credenciales o información útil para el siguiente paso.

Servicio web en puerto 80: comprobar el contenido de la web (ficheros, directorios, aplicaciones) con curl, gobuster o navegación manual para buscar posibles vectores (ficheros de backup, paneles, parámetros inseguros).

SSH en puerto 2222: tener en cuenta si encontramos credenciales (por ejemplo en FTP) para intentar acceso. También puede ser útil realizar fuerza bruta si está permitido en el CTF y no hay bloqueo.

## Paso 3 — Visita pagina en puerto 80

Accedimos al puerto 80 mediante un navegador y observamos la página por defecto de Apache en Ubuntu. Esto indica que el servidor web está funcionando pero no contiene (a simple vista) una aplicación personalizada o contenido interesante — sin embargo, puede haber ficheros ocultos, páginas en subdirectorios o backups accesibles.

Captura del resultado (navegador):

<img width="1145" height="878" alt="Puerto 80 pagina" src="https://github.com/user-attachments/assets/fce18d57-0005-433c-82f0-26abaf4a1b03" />


Observaciones:

Página por defecto de Ubuntu: sugiere que el contenido en /var/www/html/index.html no ha sido reemplazado por una aplicación.

Aunque la página sea la default, es buena práctica realizar una enumeración de directorios y ficheros (brute-force de rutas) y comprobar posibles endpoints comunes (/backup, /admin, /uploads, /robots.txt, /server-status, etc.).

## Paso 4 — Enumeración de directorios (Brute‑force web)

Realizamos una enumeración de directorios con gobuster para descubrir rutas ocultas en el servidor web.

Comando ejecutado:

gobuster dir -u http://10.10.197.170:80 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 100

Resultado observado (captura):

<img width="1025" height="359" alt="gobuster" src="https://github.com/user-attachments/assets/4c0daa71-66c5-46e5-a172-fc6b259af3e2" />


Salida relevante detectada:

/simple (Status: 301) redirige a http://10.10.197.170/simple/.

Interpretación:

Se ha descubierto un directorio llamado /simple que devuelve una redirección a /simple/. Esto sugiere que existe contenido o una aplicación en ese subdirectorio que debemos investigar a continuación.

## Paso 5 — Inspección del contenido en `/simple/` (Aplicación detectada)


Accedimos a `http://10.10.197.170/simple/` y revisamos el contenido cargado en el directorio descubierto por `gobuster`.


**Captura (navegador):**

<img width="1125" height="949" alt="pagina puerto80 simple" src="https://github.com/user-attachments/assets/8272eb35-154b-4f06-9d1f-45165a879b24" />


**Observaciones relevantes:**


- La página muestra un CMS que parece ser **CMS Made Simple** (en el pie de página aparece "This site is powered by CMS Made Simple version 2.2.8").
- Esto indica que el sitio está ejecutando una versión concreta (2.2.8) — conocer la versión nos permite buscar vulnerabilidades públicas o rutas comunes como el panel de administración.


**Interpretación y prioridades:**

1. Buscar el panel de administración del CMS. En muchos CMS la ruta de administración suele ser `/admin`, `/cmsmspath/admin`, `/cmsms/admin`, `/simple/admin` o similar. El propio contenido de la página ya sugiere `cmsmspath/admin` en texto genérico.
2. Enumerar subdirectorios dentro de `/simple/` usando `gobuster` para localizar rutas como `/admin`, `/login`, `/uploads`, `/modules`, `/install` o backups.
3. Revisar la posibilidad de exploits conocidos contra **CMS Made Simple 2.2.8** (CVE/Exploit-DB) que pudieran permitir RCE, SQLi, o bypass de autenticación. (Siempre comprobar PoC y adaptarlo al entorno del CTF).

## Paso 6 — Investigación de vulnerabilidades conocidas (CMS Made Simple)

Al identificar que el sitio ejecuta CMS Made Simple v2.2.8, buscamos vulnerabilidades públicas que afecten a esa versión.

Referencia encontrada: Exploit-DB muestra una vulnerabilidad de SQL Injection que afecta a versiones de CMS Made Simple anteriores a la 2.2.10 (CVE-2019-9053). La entrada en Exploit-DB aporta detalles y PoC que pueden ser útiles para una prueba controlada en el CTF.

Captura (Exploit-DB):

<img width="1421" height="546" alt="ataque por inyeccion SQL" src="https://github.com/user-attachments/assets/5dd30182-4480-4fec-8891-a4764530c530" />

Interpretación:

La vulnerabilidad reportada sugiere la posibilidad de inyección SQL en ciertos endpoints del CMS. 

Dado que el servidor ejecuta la versión vulnerable, merece la pena intentar una prueba controlada para confirmar la presencia de la vulnerabilidad en la instancia del CTF.

## Paso 7 — Preparar PoC exploit (script Python) y prueba controlada

Hemos tomado el PoC publicado en Exploit‑DB y lo hemos convertido en un script exploit.py adaptado al objetivo. El exploit realiza una inyección SQL time‑based contra el endpoint vulnerable usado por CMS Made Simple (en la PoC original suele apuntar a /moduleinterface.php?mact=News,m1,,default,0 o similar).


<img width="1073" height="768" alt="creamos script en python sacado de exploitDB" src="https://github.com/user-attachments/assets/7cc06876-9b53-43ec-80c4-a46631bc41db" />


Resumen del comportamiento del script:

Construye payloads de inyección SQL basados en tiempos para extraer datos (por ejemplo nombres de usuario y hashes de contraseña) cuando la inyección no muestra errores SQL claros.

Incorpora una opción para hacer "crack" del hash con un wordlist (modo --crack o -c).

Parámetros principales:

-u, --url : URL base objetivo (ej.: http://10.10.197.170/simple)

-w, --wordlist : ruta a un wordlist local para intentar crackear hashes

-c, --crack : activar modo cracking

## Paso 8 — Ejecución del exploit y obtención de credenciales (Post‑explotación)

Ejecutamos exploit.py contra http://10.10.197.170/simple/ y utilizamos un wordlist (rockyou.txt) para intentar crackear el hash de la contraseña extraído mediante la inyección SQL.

Comando ejecutado:

python3 exploit.py -u http://10.10.197.170/simple --crack -w /home/quiqu3h4ck/rockyou.txt

Resultados observados (capturas):

<img width="475" height="204" alt="intentnado descifrar la contraseña" src="https://github.com/user-attachments/assets/45727747-db80-49ff-b96d-e63934f98b52" />


<img width="922" height="311" alt="usuario y clave encontradas" src="https://github.com/user-attachments/assets/3e6e4560-6d4f-419f-a57b-dd8e1fc5feb1" />


Salida relevante extraída:

Salt para la contraseña encontrada: 1dac0d92e9fa6bb2.

Username found: mitch.

Email found: admin@admin.com.

Password hash encontrado: 0c01f4468bd75d7a84c7eb73846e8d96 (ejemplo en la salida).

Password cracked: secret.

Interpretación:

Hemos conseguido credenciales válidas del CMS (usuario mitch con contraseña secret). 

Estas credenciales pueden usarse para intentar acceso al panel de administración del CMS y/o acceso SSH (puerto 2222) si el mismo usuario se mapea a una cuenta del sistema.

## Paso 9 — Acceso por SSH (Acceso inicial)

Probamos acceso SSH al puerto 2222 con las credenciales obtenidas (mitch:secret) y conseguimos un shell remoto.

Comando ejecutado:

ssh -p 2222 mitch@10.10.197.170

Captura (SSH conectado):


<img width="897" height="441" alt="accedemos mediante ssh a la maquina" src="https://github.com/user-attachments/assets/7fa7a734-9e5f-48fe-91b2-16e818323307" />



Salida observada:

Inicio de sesión correcto en Ubuntu 16.04.6 LTS (kernel 4.15.x).

Shell interactivo disponible como el usuario mitch.

## Paso 10 — Obtención de la bandera de usuario (user.txt)

Tras enumerar el sistema como mitch, localizamos y leímos el fichero user.txt en el home del usuario. Esto confirma que hemos completado la fase de acceso inicial y obtenido la flag de usuario.

Comando ejecutado:

cat /home/mitch/user.txt

Captura (flag de usuario):

<img width="819" height="445" alt="primera bandera dentro del archivo user" src="https://github.com/user-attachments/assets/1fabb697-ec44-4b21-9b38-b5f7e17bbb52" />


Interpretación:

La bandera de usuario se ha encontrado y leído correctamente — la prueba de acceso inicial y extracción de información sensible ha sido exitosa.

## Paso 11 — Descubrimiento de otro usuario en /home

Mientras inspeccionábamos /home desde la sesión de mitch, encontramos otro directorio de usuario llamado sunbath.

<img width="259" height="107" alt="otro usuario dentro del directorio home" src="https://github.com/user-attachments/assets/88055a55-8276-41ce-bb87-5885096c6472" />


Comando observado:

ls /home
# salida: mitch  sunbath

Interpretación:

La existencia de /home/sunbath indica otro usuario en la máquina. Esto abre vías adicionales:

Buscar archivos accesibles en el home de sunbath (posibles flags, claves SSH, ficheros de configuración).

Comprobar si mitch tiene permisos para leer ficheros en ese directorio (p. ej. por permisos mal configurados).

Si encontramos credenciales o una llave privada, podríamos pivotar o iniciar sesión como sunbath y continuar la escalada.

##Paso 12 — Escalada de privilegios: sudo permite ejecutar vim como root

Al ejecutar sudo -l descubrimos que el usuario mitch puede ejecutar /usr/bin/vim como root sin contraseña:

<img width="495" height="80" alt="que puede ejecutar mitch sin password" src="https://github.com/user-attachments/assets/03f7d104-de46-4c27-8670-19f398246faa" />


(root) NOPASSWD: /usr/bin/vim

Esto es un vector clásico de escalada: vim permite ejecutar comandos del sistema cuando se ejecuta con privilegios. 

## Paso 13 — Escalada a root usando vim (GTFObins) y obtención de root.txt

Confirmamos que vim está disponible y que la versión instalada soporta la ejecución de comandos. Siguiendo la técnica documentada en GTFOBins para vim, ejecutamos vim como root mediante sudo y abrimos un shell con :! /bin/sh.

Referencia: https://gtfobins.github.io/gtfobins/vim/#shell

Comando usado:

sudo /usr/bin/vim -c ':!/bin/sh'

Capturas:

página de GTFOBins que muestra el payload para vim.

<img width="935" height="523" alt="gtfobins " src="https://github.com/user-attachments/assets/c7a51991-407b-4449-b0c6-afbff4d98352" />


captura de la consola una vez obtenido shell root y mostrando cat /root/root.txt.

<img width="931" height="481" alt="escalada de privilegios y segunda flag" src="https://github.com/user-attachments/assets/393d0a55-7e25-4fa7-a657-24ebccb1cd41" />

Salida observada:

vim se abrió y desde su modo de comandos ejecutamos /bin/sh, obteniendo shell con UID 0.

Leímos root.txt y obtuvimos la bandera de root.

Comandos de verificación post‑explotación:

whoami
id
cat /root/root.txt

Interpretación y conclusiones:

La capacidad de ejecutar /usr/bin/vim con sudo sin contraseña es una mala configuración que permite la escalada de privilegios.

Utilizando una técnica documentada en GTFOBins se puede obtener un shell root de forma rápida y elegante.

Con esto, completamos las fases del CTF: reconocimiento, enumeración, explotación y post‑explotación (flags de usuario y root obtenidas).
