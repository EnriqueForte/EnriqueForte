# 🐇 TryHackMe - Wonderland CTF Write-up 🗝️

## 🎩 Introducción: Cayendo por la Madriguera del Conejo

¡Bienvenidos a un viaje alucinante a través de Wonderland! Esta sala de TryHackMe (THM) es un desafío de nivel intermedio con una encantadora temática inspirada en la obra clásica de Lewis Carroll, Alicia en el País de las Maravillas.

El objetivo, como en todo buen CTF, es caer por la madriguera, encontrar la bandera del usuario (user.txt) y, finalmente, escalar privilegios para obtener la bandera raíz (root.txt). A lo largo del camino, pondremos a prueba nuestras habilidades de enumeración web, manejo del sistema de archivos Linux y escalada de privilegios mediante binarios.

¿Serás capaz de encontrar la salida o te perderás para siempre en el País de las Maravillas? ¡Es hora de empezar!

## 1️⃣ Paso 1: Verificación de Conectividad (Ping) 📡

El primer paso fundamental en cualquier CTF es confirmar que la máquina objetivo está activa y que tenemos conectividad de red con ella.

Para este desafío, la dirección IP de la máquina objetivo es 10.10.177.118. Utilizamos la herramienta ping con el argumento -c 4 para enviar un total de 4 paquetes ICMP y verificar la respuesta.

<img width="615" height="382" alt="Ping" src="https://github.com/user-attachments/assets/2a4aa65d-18a7-4153-bfd3-20bc0c09e9b0" />


💻 Comando Utilizado
````
Bash
ping -c 4 10.10.177.118
````

✅ Resultado

Tal como se muestra en la imagen proporcionada, la máquina objetivo está activa y accesible:

Se recibieron 4 paquetes de vuelta.

El porcentaje de pérdida de paquetes es 0%.

Esto confirma que podemos proceder con la fase de reconocimiento de puertos y servicios.


## 2️⃣ Paso 2: Escaneo de Puertos y Servicios (Nmap) 🗺️

Con la conectividad confirmada, el siguiente paso lógico es descubrir qué puertos están abiertos y qué servicios se están ejecutando en la máquina objetivo (10.10.177.118). Esto nos dará los vectores de ataque iniciales.

Utilizamos un escaneo agresivo para obtener información detallada de versiones y scripts por defecto (-sC, -sV), utilizando la opción -T4 para una ejecución rápida.

<img width="918" height="350" alt="Nmap" src="https://github.com/user-attachments/assets/2dee050d-0c23-4843-b725-56d671b9bb13" />

💻 Comando Utilizado
````
Bash
nmap -sV -sC -T4 -oN nmap_scan 10.10.177.118
````

🔑 Análisis de Servicios

Puerto 22 (SSH): El servicio SSH está activo. Aunque proporciona un método de acceso remoto, no podremos usarlo por ahora ya que no tenemos credenciales. Lo mantendremos en mente para una posible escalada de privilegios o post-explotación.

Puerto 80 (HTTP): Este es nuestro punto de entrada más prometedor. Un servidor web está corriendo en el puerto HTTP estándar. La versión identificada es un servidor Golang net/http, y la cabecera HTTP (mostrada en el snippet de Nmap) incluye un título revelador:

http-title: Follow the white rabbit. 🐇

Esto nos confirma la temática de Alicia en el País de las Maravillas y nos da una clara pista para el siguiente paso: seguir al conejo blanco, lo que se traduce en enumerar y explorar la aplicación web.


## 3️⃣ Paso 3: Exploración del Servidor Web (HTTP - Puerto 80) 🐇

Al navegar a la dirección 10.10.177.118 en el navegador, nos encontramos con una página de aterrizaje sencilla que refuerza el tema de Alicia en el País de las Maravillas:

<img width="1082" height="894" alt="pagina puerto 80 imagen" src="https://github.com/user-attachments/assets/ac76ea62-f8af-4de6-ba98-d2207da0c74d" />


Título: "Follow the White Rabbit." (Sigue al Conejo Blanco.)

Cita: Una cita de la historia: "Curiouser and curiouser!" cried Alice (she was so much surprised, that for the moment she quite forgot how to speak good English).

Imagen: Una ilustración del Conejo Blanco mirando su reloj de bolsillo.

Esta página no proporciona información directamente explotable, pero la cita y el título sugieren que debemos ser "más curiosos" y "seguir" buscando. Es una indicación clara de que necesitamos una enumeración de directorios más profunda.

El siguiente paso es usar una herramienta como Gobuster o Dirb para forzar la búsqueda de directorios y archivos ocultos en el servidor web.


## 4️⃣ Paso 4: Enumeración de Directorios con GoBuster 🔎

Dado que la página principal no ofrecía pistas obvias, el siguiente paso es la fuerza bruta para descubrir directorios o archivos ocultos que podrían ser nuestro punto de entrada. Para ello, hemos utilizado la herramienta GoBuster.

<img width="1345" height="304" alt="gobuster" src="https://github.com/user-attachments/assets/659d7076-92e8-4037-8994-a4371bcbb0e8" />

💻 Comando Utilizado
````
Bash
gobuster dir -u http://10.10.177.118 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
````

📁 Resultados Encontrados

GoBuster arrojó tres resultados con el código de estado 301 (Movido Permanentemente), indicando que el contenido ha sido redirigido a una nueva ubicación. Estos son nuestros nuevos caminos:

/img

/r

/poem

🗺️ Exploración de Directorios

El directorio /img probablemente contiene la imagen del Conejo Blanco que ya vimos. Los directorios /r y /poem son mucho más interesantes.



## 5️⃣ Paso 5: Esteganografía en la Imagen del Conejo Blanco 🖼️

En muchos CTFs, si una imagen prominente está presente y el texto apunta a ser "curioso" o "seguir al conejo", la esteganografía es una técnica común.

5.1 Descarga de la Imagen 📥

Primero, localizamos la imagen del conejo blanco, probablemente en el directorio /img que descubrió GoBuster, y la descargamos utilizando wget.

<img width="1402" height="410" alt="descargamos la imagen y usamos steghide para buscar otros archivos" src="https://github.com/user-attachments/assets/b9c43512-2043-4da4-95be-80b8feaa5376" />


![white_rabbit_1](https://github.com/user-attachments/assets/2d37b847-646a-4e48-a939-7f6ef8e7bec9)


````
Bash
wget http://10.10.177.118/img/white_rabbit_1.jpg
````

5.2 Análisis con Steghide 🕵️

Una vez con la imagen en tu sistema, utilizamos la popular herramienta de esteganografía Steghide para intentar extraer datos ocultos.

````
Bash
steghide extract -sf white_rabbit_1.jpg
````

Steghide preguntó por una contraseña ("Anote el salvoconducto", o passphrase). le dimos al enter directamente:

[...datos extraídos...] y los datos extraídos son "hint.txt".

📄 Archivo Desvelado: hint.txt

El archivo extraído, llamado hint.txt, es nuestra nueva pista, con ls, confirmamos su presencia.


## 6️⃣ Paso 6: Análisis de la Pista (hint.txt) 🤔

El archivo hint.txt que extrajimos de la imagen del Conejo Blanco con Steghide contiene un mensaje muy corto y peculiar:

<img width="425" height="82" alt="leyendo el archivo hint" src="https://github.com/user-attachments/assets/85bdd8f9-e846-4ede-a919-212990a76b5d" />


📄 Contenido de hint.txt

follow the r a b b i t

Este mensaje parece una repetición del tema ("follow the rabbit"), pero con un espacio adicional entre las letras r, a, b, b, i, t.

En ciberseguridad, un espacio o un carácter mal colocado a menudo indica un nombre de archivo o directorio oculto en el servidor.


## 7️⃣ Paso 7: Navegando el Directorio Críptico /r/a/b/b/i/t 🚪

La pista follow the r a b b i t no era solo un nombre de directorio, sino una ruta completa que debía explorarse letra por letra, capitalizando la importancia de las dos 'b's.

Al navegar a http://10.10.177.118/r/a/b/b/i/t/, nos encuentramos con una nueva página que refuerza la narrativa:

<img width="1227" height="991" alt="directorio rabbit, si navegamos por cada letra encontraremos paginas distintas" src="https://github.com/user-attachments/assets/9496de9e-d221-4ef9-aa01-2a83c444e6de" />

Título: "Open the door and enter wonderland" (Abre la puerta y entra en el País de las Maravillas).

Cita: Una conversación entre Alicia y el Gato de Cheshire que menciona al Sombrerero Loco (Mad Hatter) y la Liebre de Marzo (March Hare).

“In that direction,” the Cat said, waving its right paw round, “lives a March Hare: and in that direction,” waving the other paw, “lives a Mad Hatter. Visit either you like: they’re both mad.”

🗺️ Nuevas Pistas para la Enumeración

El texto no solo te dice que "abras la puerta", sino que te da dos posibles nombres de usuario/pistas para continuar la exploración:

March Hare

Mad Hatter

Es probable que uno de estos nombres sea el nombre de usuario que necesitamos para la primera bandera, o un directorio oculto adicional.

Si navegas por cada letra encontrarás una pagina distinta, prueba si quieres.


## 8️⃣ Paso 8: Descubriendo la Clave de Alice en el Código Fuente 🗝️

Al ver la página en la ruta /r/a/b/b/i/t/ que invitaba a "abrir la puerta", revisamos su código fuente (view-source:http://10.10.177.118/r/a/b/b/i/t/).

<img width="1062" height="417" alt="codigo fuente rabbit encotramos clave de alice" src="https://github.com/user-attachments/assets/e8aa2c58-9383-437e-bafa-674c014a895d" />

Justo debajo de la cita del Gato de Cheshire, encontramos una línea de código HTML que estaba oculta (style="display: none"):

📄 Código Fuente Revelador

HTML

<p style="display: none;">alice:H0tt3rHasD1gitalK3y</p>

Esta línea contiene un par de credenciales en el formato usuario:contraseña o una clave crítica:

Usuario (Probable): alice

Contraseña/Clave: H0tt3rHasD1gitalK3y


## 9️⃣ Paso 9: Acceso Inicial por SSH y Enumeración del Directorio Home 🏠

Con el par de credenciales descubierto en el código fuente, es hora de utilizar el servicio SSH que Nmap identificó en el puerto 22.

<img width="718" height="667" alt="iniciamos ssh con alice y la contraseña" src="https://github.com/user-attachments/assets/4fc600dc-fc0c-4ff4-bef0-9cee5961cd79" />

💻 Conexión SSH Exitosa

Utilizamos las credenciales alice:H0tt3rHasD1gitalK3y para iniciar sesión con éxito:

````
Bash
ssh alice@10.10.177.118
````

Una vez conectado, entramos al sistema como el usuario alice.

🔎 Enumeración del Directorio Home

Lo primero que hicimos, y muy acertadamente, fue listar los archivos del directorio home (/home/alice) con el comando ls -lah.

Los archivos destacados son:

root.txt: La bandera de root. Aún no tenemos permisos para leerla (pertenece al usuario root).

<img width="280" height="85" alt="no tenemos permisos para leer root txt" src="https://github.com/user-attachments/assets/aed22bd5-6772-4e1e-9068-9a3321a7bbd5" />

walrus_and_the_carpenter.py: Un archivo Python. Su nombre hace referencia a otro famoso poema de Lewis Carroll.

<img width="632" height="788" alt="leemos el archivo walrus_and" src="https://github.com/user-attachments/assets/3f386d1b-7ada-4f4a-a4c7-d222d303d13a" />

Este archivo es nuestro objetivo inmediato, ya que a menudo contienen pistas o funciones críticas para el movimiento lateral o la escalada de privilegios.

user.txt: ¡La bandera de usuario! Todavia no tenemos acceso.


## 🔟 Paso 10: Ejecutando el Script walrus_and_the_carpenter.py 📜

El archivo Python walrus_and_the_carpenter.py tiene un nombre que evoca el poema del Gato de Cheshire. Al ejecutarlo con python3, el script parece imprimir varias líneas de texto, probablemente partes del famoso poema.

<img width="513" height="193" alt="ejecuto el script de python" src="https://github.com/user-attachments/assets/40cc8045-e96f-4023-8760-2f0a476bc972" />


💻 Ejecución del Script
````
Bash
python3 walrus_and_the_carpenter.py
````

🧐 Análisis del Resultado

El script imprime líneas de texto sin una salida obvia de la bandera. Esto significa que hay dos posibilidades:

La Bandera está en el Código: El script lee la bandera de user.txt y la utiliza de alguna manera (por ejemplo, como una cadena dentro de las líneas de texto impresas).

La Bandera es la Pista: El script nos da la pista para acceder a la bandera o al siguiente usuario/nivel.

Dado que la ejecución solo muestra texto del poema, el código fuente del script es lo que realmente importa. Necesitamos saber qué hace exactamente el programa.


## Paso 11: Escalada de Privilegios Horizontal (Alice a Rabbit) 🐇

En los CTFs, después de obtener el acceso de un usuario, siempre es crucial verificar si ese usuario tiene la capacidad de ejecutar comandos con privilegios de otro usuario o de root.

11.1 Descubrimiento de Sudoers (sudo -l) 🕵️

<img width="997" height="125" alt="sudo l podemos ejecutar scirpt con usuario rabbit" src="https://github.com/user-attachments/assets/1e154f92-dffc-4dcf-8693-ca8bf4776765" />

Utilizamos el comando sudo -l para verificar los permisos del usuario alice y encontramos una entrada muy importante:

User alice may run the following commands on wonderland:

(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

Esto significa que alice puede ejecutar el script de Python /home/alice/walrus_and_the_carpenter.py con los privilegios del usuario rabbit sin necesidad de su contraseña.

## 11.2 Explotación de la Vulnerabilidad 🛡️

Esta es una vulnerabilidad clásica de sudo mal configurado. Podemos explotarla creando un nuevo script de Python que spawnee un shell y luego pidiendo a sudo que ejecute nuestro script en lugar del original.

El plan era el siguiente:

Reemplazar el Script: Modificar el archivo /home/alice/walrus_and_the_carpenter.py con una carga útil (payload) de shell inversa.

Ejecutar con rabbit: Usar sudo -u rabbit para ejecutar el script modificado con los privilegios del usuario rabbit.

Sin embargo, dado que estamos usando la versión original, la técnica más limpia y que usamos es crear un nuevo script de Python en un archivo llamado random.py (o similar) y hacer que el script original lo importe o ejecute.

En el caso de Wonderland, el script original probablemente permitía al usuario cargar módulos de Python, lo que permitió inyectar código. Si el script original estaba configurado para importar módulos, simplemente creamos un archivo malicioso, por ejemplo:

<img width="859" height="91" alt="escalamos privilegios con nuestro script" src="https://github.com/user-attachments/assets/e49a046a-f2df-4be7-bf90-9149274add6f" />


1. Creación de un Shell

Creamos un script de Python simple que ejecuta un /bin/sh local:
````
Bash
echo "import subprocess;subprocess.call('/bin/sh')" > /tmp/shell.py 
# Nota: La captura usa una sintaxis de shell más limpia:
# echo "import subprocess;subprocess.call(('/bin/sh',))" > random.py
````

2. Ejecución como rabbit
   
Luego, aprovechamos la configuración de sudo para escalar al usuario rabbit:

````
Bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
````

Al ejecutar el comando, se spawneó un nuevo shell con los privilegios del usuario rabbit.

3. Confirmación de Shell

La confirmación con el comando whoami demuestra el éxito:
````
Bash
$ whoami
rabbit
````

## Paso 12: Captura de la Bandera de Usuario (user.txt) 🚩

Tras escalar privilegios al usuario rabbit, nuestro primer objetivo es obtener la bandera del usuario (user.txt).

En esta sala, el archivo se encontraba en un directorio accesible por el usuario rabbit (o en /root, aunque es poco común para la bandera de usuario).

💻 Localización y Lectura

Al encontrar el archivo, se utiliza el comando cat para mostrar su contenido:
````
Bash
# cat /root/user.txt
````
<img width="314" height="102" alt="badnera user" src="https://github.com/user-attachments/assets/f3c3b1c4-673b-445f-853d-ff6962e6ae3d" />


🎉 ¡Primera Bandera Capturada!

Con esto, hemos completado la primera mitad del desafío y hemos obtenido la bandera del usuario.

Bandera	Valor

user.txt	thm{...} (Bandera Oculta)


## Paso 13: Enumeración para Escalada a Root (SUID) 👑

Ahora, como usuario rabbit, el objetivo es convertirnos en el Sombrerero Loco (root). El método más común después de un shell es buscar binarios con el bit SUID (Set User ID) activado, lo que permite a un usuario ejecutar el archivo con los permisos de su propietario.

13.1 Descubrimiento del Binario SUID 🕵️

Al enumerar el directorio home de rabbit (/home/rabbit), has encontrado un binario crítico:

<img width="646" height="186" alt="binario teaparty en home" src="https://github.com/user-attachments/assets/930f8a63-2405-48e8-9155-9eaa7f21101d" />

-rwsr-sr-x 1 root root 17K May 25 2020 teaParty

El bit s en la posición de usuario y grupo (rwsr-sr-x) indica que este archivo se ejecutará con los permisos de su propietario, que es root.

Binario: teaParty (Fiesta del Té)

Propietario: root

Vector de Escalada: Ejecución con privilegios de root.

13.2 Análisis del Binario

Al ejecutar el binario, se muestra un mensaje temático y luego se produce un Segmentation fault (core dumped):

<img width="623" height="163" alt="ejcutamos el binario podria ser un desbordamiento de buffer" src="https://github.com/user-attachments/assets/0dd77a0b-52d6-47fa-ba90-f6f0e589592d" />

````
Bash
$./teaParty
Welcome to the tea party!
The Mad Hatter will be here soon.
...
Ask very nicely, and I will give you some tea while you wait for him
Segmentation fault (core dumped)
````

Un "fallo de segmentación" es a menudo la señal de una mala gestión de la memoria, lo que sugiere una vulnerabilidad de Desbordamiento de Búfer (Buffer Overflow) o un problema de manejo de rutas de archivo (path hijacking).

Dada la temática del CTF y la presencia del binario SUID, la vulnerabilidad aquí es el manejo inseguro de rutas de archivo. El mensaje "Ask very nicely, and I will give you some tea" probablemente implica que el programa intenta ejecutar un comando externo (como echo o cat) para darte el té, y ese comando no está siendo llamado con una ruta absoluta (/bin/comando).

13.3 Explotación de Path Hijacking 🛠️

Para explotar esta vulnerabilidad, debemos engañar al binario teaParty para que ejecute nuestro propio shell en lugar del comando que busca.

Crear un Shell Malicioso: Creamos un script simple que se llame igual que el comando que el binario teaParty intenta ejecutar (por ejemplo, si intenta ejecutar cat, nombramos nuestro script cat). Este script contendrá un payload para ejecutar un shell (/bin/sh).

Modificar la Variable $PATH: Modificamos la variable de entorno $PATH para que el sistema busque en nuestro directorio actual (donde está nuestro script malicioso) antes de buscar en las rutas estándar (/bin, /usr/bin, etc.).

Ejecutar el Binario: Al ejecutar ./teaParty, este buscará el comando en nuestro directorio primero, ejecutará nuestro shell con permisos de root (gracias al SUID) y nos dará un shell de root.


## Paso 14: Path Hijacking y Acceso a Hatter 🎩

El binario teaParty (con SUID) no utiliza rutas absolutas para uno de los comandos que ejecuta, haciéndolo vulnerable a un secuestro de la variable de entorno $PATH. En este CTF, el comando vulnerable que el binario teaParty intentaba ejecutar era date.

14.1 Preparación de la Carga Útil (Payload) 🛠️

Creación del Shell Malicioso: Creamos un script ejecutable en nuestro directorio actual (/home/rabbit) llamado date. Este script contendrá un payload simple para ejecutar un shell (/bin/bash).

<img width="818" height="417" alt="modificamos valor de PATH, ubicacion al inicio del directorio para leer arhcivos, crecion de date con comando, entramso al usuario hatter" src="https://github.com/user-attachments/assets/5a801aa5-2c71-4013-8873-daf7ab4828b0" />

````
Bash
# Crear el archivo "date" con el payload
$ cat > date
#!/bin/bash
/bin/bash

# Dar permisos de ejecución
$ chmod a+x date
````

**Modificación del $PATH:** Modificamos la variable $PATH  para que el sistema busque en nuestro directorio actual ( /home/rabbit) antes que en las ubicaciones estándar (/bin ,  /usr/bin , etc.). 

Esto garantiza que, cuando  teaPartyllame adate\, ejecute nuestro script en lugar del binario legítimo.

````
Bash
$ export PATH=/home/rabbit:$PATH
# (Verificación)
$ echo $PATH
/home/rabbit:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
````

14.2 Ejecución y Escalada de Privilegios

Finalmente, ejecutamos el binario teaParty. El programa intenta ejecutar date con permisos de root (debido al bit SUID), encuentra nuestro script malicioso primero y, en su lugar, nos spawnea un shell como el siguiente usuario:

````
Bash
$./teaParty
````
El resultado es un nuevo shell con privilegios elevados, tal como lo confirma whoami:

hatter@wonderland:/home/rabbit$ whoami

hatter

Hemos escalado exitosamente de rabbit a hatter (Sombrerero Loco), lo que constituye la última etapa antes de root. Ahora, necesitamos explorar el directorio home del nuevo usuario.


## Paso 15: Enumeración del Directorio de Hatter y Clave Final 🔑

Una vez dentro, el primer paso es siempre enumerar el directorio home del nuevo usuario (/home/hatter).

<img width="521" height="108" alt="archivo con password" src="https://github.com/user-attachments/assets/ebb08c10-6ec8-412a-a920-ba45b2a381f9" />


15.1 Descubrimiento del Archivo Secreto 📁

Al listar el contenido del directorio, encontramos un único archivo muy prometedor:

````
Bash
$ cd /home/hatter
$ ls -l
total 4
-rw------- 1 hatter hatter 29 May 25 2020 password.txt
````

El archivo password.txt es nuestra última pista. Es de solo lectura para el usuario hatter (-rw-------), lo cual es una buena señal de que contiene algo valioso.

15.2 Lectura de la Clave

Al leer el contenido del archivo, obtienes una cadena de texto que, por su ubicación y naturaleza, solo puede ser la contraseña de root o la clave para la escalada final.

````
Bash
$ cat password.txt
Wh[...]
````

Paso 16: Escalada Final y Bandera Root 👑

Una vez que somos el usuario hatter (logrado a través de la explotación del binario SUID teaParty), el objetivo final es obtener la bandera root.txt.

<img width="399" height="59" alt="uso la clave obtenido para entrar" src="https://github.com/user-attachments/assets/3a1c58db-07d8-47cf-8e66-9704edddeda4" />

16.1 Búsqueda de Vectores de Escalada Adicionales (Capacidades)

 De aqui sacamos los binarios.
 
<img width="1229" height="498" alt="de esta pagina saco la busqueda de binarios" src="https://github.com/user-attachments/assets/708138e1-126a-46e2-b8d6-3702257b9560" />


Verificado si el usuario hatter tiene alguna Capacidad de Linux (Capabilities) configurada, un método moderno para elevar privilegios sin SUID. Utilizamos el comando getcap:

<img width="545" height="129" alt="se podira usar perl para obtener el shell" src="https://github.com/user-attachments/assets/1dace506-bd90-4a61-ba6d-feae95a01e49" />

````
Bash
$ getcap -r / 2>/dev/null
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
````

El binario perl tiene la capacidad cap_setuid+ep. Esto permite a perl cambiar su ID de usuario efectivo a cualquier ID, incluido root (UID 0), lo que ofrece otra ruta para obtener el shell de root.

🛠️ Explotación de Capabilities (Perl)

<img width="929" height="214" alt="badnera dentro root txt" src="https://github.com/user-attachments/assets/4c375ae7-51d2-4176-a3f7-b5017147491c" />


El payload para explotar esta capacidad de perl y obtener un shell de root es:

````
Bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
````

16.2 El Camino Directo: La Clave de Root en password.txt 🔑

Aunque la escalada con Capabilities era posible, el CTF a menudo deja la clave final en el directorio del último usuario, tal como encontramos:

Archivo Encontrado: En /home/hatter, encontramos el archivo password.txt.

Contraseña Revelada: El archivo contenía la contraseña del usuario root.

16.3 Conversión a Root (su) 🏆

Utilizamos el comando su (Switch User) para cambiar al usuario root, proporcionando la contraseña que encontramos en password.txt:

````
Bash
# Cambiamos al usuario root
$ su root
Password: <PASSWORD_DE_password.txt>
# whoami
root
````

Tras la autenticación exitosa, obtenemos un shell como root.

16.4 Captura de la Bandera Root (root.txt) 🎉

Finalmente, navegamos al directorio /root y leemos el archivo final para completar el desafío.

````
Bash
cat /root/root.txt
````
