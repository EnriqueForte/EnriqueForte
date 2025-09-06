# 🥒 Pickle Rick - Walkthrough

## 📌 Introducción
Este CTF se encuentra en la plataforma **TryHackMe** y está basado en la serie *Rick and Morty*.  
El objetivo es conseguir las credenciales y acceder al sistema para capturar las flags.

---

## 🛠️ Herramientas utilizadas

Durante la resolución de este CTF utilicé las siguientes herramientas y recursos:

- **Ping** → para verificar la conectividad con la máquina víctima.  
- **Nmap** → para el escaneo de puertos y detección de servicios.  
- **Gobuster** → para la enumeración de directorios y archivos ocultos.  
- **Navegador web** → inspección de código fuente y análisis de recursos accesibles.  
- **Command Panel (portal web)** → aprovechado para ejecutar comandos en el sistema (RCE).  
- **Comandos básicos de Linux** → `ls`, `whoami`, `sudo -l`, etc., para enumeración y escalada de privilegios.  
- **Técnicas alternativas de lectura de archivos** → uso de `tac`, `head`, `tail`, `less` en lugar de `cat` (que estaba bloqueado).  

---

## 🚀 Desarrollo del CTF

### 🔹 Paso 1: Verificación de conectividad
Lo primero es comprobar si la máquina víctima está activa:

```bash
ping 10.10.160.159
```
📸 Evidencia:


<img width="750" height="377" alt="Ping" src="https://github.com/user-attachments/assets/008f8fb1-724b-4b19-8c74-f08f8591d8a2" />

✅ La máquina responde correctamente, lo que confirma que está activa y lista para continuar con la fase de reconocimiento.

### 🔹 Paso 2: Exploración del servicio web (puerto 80)

Accedemos a la dirección http://10.10.160.159 y encontramos una página relacionada con la temática de Rick and Morty.

📸 Evidencia:

<img width="1218" height="664" alt="pagina abierta puerto 80" src="https://github.com/user-attachments/assets/dc659daf-7ce7-4901-af26-38767fac3a43" />

En el contenido se nos pide ayudar a Rick a encontrar los ingredientes secretos para su poción.
Este mensaje sugiere que habrá pistas escondidas en el sistema que debemos ir descubriendo.

### 🔹 Paso 3: Revisión del código fuente

Al inspeccionar el código fuente de la página principal, encontramos un comentario oculto con un nombre de usuario:

📸 Evidencia:

<img width="1255" height="694" alt="codigo fuente con usuario" src="https://github.com/user-attachments/assets/8b2524df-61fb-46c0-b9ec-e40c9dc9b792" />

Note to self, remember username!
Username: R1ckRul3s

✅ Con este hallazgo ya tenemos un usuario válido que seguramente nos servirá para autenticarnos más adelante.

### 🔹 Paso 4: Escaneo de puertos con Nmap
Realizamos un escaneo de la máquina víctima para identificar los servicios disponibles:

```bash
nmap -sC -sV 10.10.160.159
```
📸 Evidencia:

<img width="832" height="393" alt="nmap puertos" src="https://github.com/user-attachments/assets/58eff51a-8770-414a-9fa6-2c7e382b12fc" />

Resultados relevantes:

22/tcp (SSH): OpenSSH 8.2p1 (Ubuntu)

80/tcp (HTTP): Apache 2.4.41 (Ubuntu)

Métodos soportados: GET, POST, OPTIONS, HEAD

Título de la web: Rick is sup4r cool

✅ Ahora sabemos que el sistema expone un servicio web (puerto 80) y un servicio SSH (puerto 22).
Con el usuario encontrado en el código fuente (R1ckRul3s), el acceso por SSH podría ser un objetivo importante a intentar más adelante.

### 🔹 Paso 5: Enumeración de directorios con Gobuster
Para descubrir directorios y archivos ocultos en el servidor web, utilizamos **Gobuster** con la wordlist *directory-list-2.3-medium.txt*:

```bash
gobuster dir -u http://10.10.160.159/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,css,js,txt,pdf,php
```

📸 Evidencia:

<img width="863" height="312" alt="enumeracion gobuster" src="https://github.com/user-attachments/assets/d1fcc3d2-7606-4c6d-be86-ed6ecfb8b12c" />

Resultados obtenidos:

/index.html → Página principal

/login.php → Panel de autenticación (muy interesante)

/assets/ → Archivos estáticos

/portal.php → Redirige hacia /login.php

/robots.txt → Archivo de control de indexación que podría contener pistas

✅ Con esto ya sabemos que existe un formulario de login en /login.php y un archivo /robots.txt que puede darnos información útil.

### 🔹 Paso 6: Revisión de robots.txt
Accedemos al archivo `/robots.txt` para comprobar si contiene información sensible:

📸 *Evidencia:*

<img width="885" height="229" alt="directorio Robots txt" src="https://github.com/user-attachments/assets/8812e69d-14d5-43d0-8709-892e09ea4d4a" />


El archivo revela la siguiente cadena: Wubbalubbadubdub


✅ Es muy probable que esta cadena sea una **contraseña** asociada al usuario descubierto en el **Paso 3** (`R1ckRul3s`).  
El siguiente paso lógico será probar estas credenciales en el formulario de login.

### 🔹 Paso 7: Acceso al panel de comandos
Probamos las credenciales obtenidas en los pasos anteriores:

- **Usuario:** `R1ckRul3s`  
- **Contraseña:** `Wubbalubbadubdub`

Con ellas logramos autenticarnos en el portal web y accedemos al siguiente panel:

📸 *Evidencia:*

<img width="785" height="409" alt="panel principal" src="https://github.com/user-attachments/assets/47a43178-4e96-45af-a803-9300eba5b4bd" />

Este panel nos permite introducir y ejecutar comandos directamente en el servidor.  
✅ Hemos conseguido acceso inicial al sistema, lo que abre la puerta a la enumeración y posterior escalada de privilegios.


### 🔹 Paso 8: Exploración del portal
Tras autenticarnos, intentamos navegar por las diferentes secciones del portal:

📸 *Evidencia:*

<img width="912" height="546" alt="no funcionan el resto de pestañas" src="https://github.com/user-attachments/assets/0b966933-086e-47b6-b1af-7070628fd166" />

Observaciones:
- `Potions`, `Creatures` y `Beth Clone Notes` muestran mensajes de acceso restringido:  
  *“Only the REAL Rick can view this page..”*  
- La única sección funcional es **Commands**, donde podemos ejecutar comandos en el servidor.

✅ Esto confirma que el **Command Panel** es el punto de entrada principal para continuar la explotación.

### 🔹 Paso 9: Ejecución de comandos en el servidor
Desde el **Command Panel** probamos la ejecución de comandos básicos para verificar el acceso al sistema.  
Ejecutamos:

```bash
ls -a
```
📸 Evidencia:

<img width="841" height="464" alt="comando ls-a" src="https://github.com/user-attachments/assets/e9cdf797-008f-4a53-8eb6-60c4690bc0c0" />

Archivos encontrados:

Sup3rS3cretPickl3Ingred.txt → posible flag/ingrediente

clue.txt → puede contener información adicional

Archivos del portal web (index.html, login.php, portal.php, robots.txt, etc.)

✅ Esto confirma que tenemos ejecución remota de comandos (RCE) en el sistema.

### 🔹 Paso 10: Restricciones en los comandos
Intentamos leer el archivo `Sup3rS3cretPickl3Ingred.txt` utilizando el comando `cat`:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

📸 Evidencia:

<img width="636" height="424" alt="comando cat no funciona" src="https://github.com/user-attachments/assets/21ce08b1-ed87-492a-925c-457313d9794a" />

El sistema devuelve un mensaje indicando que el comando está deshabilitado:
Command disabled to make it hard for future PICKLEEEE RICCCKKKK.

✅ Esto demuestra que algunos comandos, como cat, están restringidos en el portal.
👉 Para seguir avanzando, debemos probar alternativas como less, more, head,tac, tail o incluso comandos de copia (cp, mv) para visualizar el contenido de los archivos.

### 🔹 Paso 11: Obtención del primer ingrediente
Como `cat` estaba deshabilitado, probamos un comando alternativo para leer el archivo `Sup3rS3cretPickl3Ingred.txt`:

```bash
tac Sup3rS3cretPickl3Ingred.txt
```
📸 Evidencia:

<img width="605" height="308" alt="primer ingrediente" src="https://github.com/user-attachments/assets/fd2a0310-fdfc-43bb-b9cc-13a8d4dfc417" />

Resultado:

mr. meeseek hair

### 🔹 Paso 12: Pista en clue.txt
Después de encontrar el primer ingrediente, exploramos el archivo `clue.txt` para buscar más información:

```bash
tac clue.txt
```

📸 Evidencia:

<img width="686" height="309" alt="archivo clue txt" src="https://github.com/user-attachments/assets/c3345723-87c1-4ec4-9bff-52918bd1c957" />

Contenido:

Look around the file system for the other ingredient.


✅ La pista indica que debemos buscar en el sistema de archivos para encontrar el siguiente ingrediente.

### 🔹 Paso 13: Explorando el directorio /home
Siguiendo la pista de `clue.txt`, comenzamos a explorar el sistema de archivos.  
Listamos el contenido de `/home`:

```bash
ls /home
```
📸 Evidencia:

<img width="531" height="373" alt="comando ls home para ver directorio home" src="https://github.com/user-attachments/assets/6284a4a0-41f4-419c-8388-2519ef932dff" />

Resultado:

rick
ubuntu

✅ Identificamos dos usuarios en el sistema: rick y ubuntu.
👉 El siguiente paso será revisar el contenido del directorio de rick, ya que probablemente allí se encuentre otro ingrediente.

### 🔹 Paso 14: Localización del segundo ingrediente
Exploramos el directorio de `rick` para buscar más pistas:

```bash
ls -l /home/rick
```

📸 Evidencia:

<img width="891" height="514" alt="rick segundo ingrediente" src="https://github.com/user-attachments/assets/6e9787cd-2fde-401b-be54-88911278c690" />

Resultado:

-rwxrwxrwx 1 root root ... second ingredients

✅ Encontramos el archivo second ingredients, que contiene el segundo ingrediente de la poción.


### 🔹 Paso 15: Restricción al intentar leer el segundo ingrediente
Intentamos visualizar el contenido del archivo `second ingredients`:

```bash
cat /home/rick/second\ ingredients
```
📸 Evidencia:

<img width="753" height="448" alt="directorio second ingredient" src="https://github.com/user-attachments/assets/8daa8fe3-e3cf-4b40-b823-0a5259fd29ad" />

El sistema devuelve:

Command disabled to make it hard for future PICKLEEEE RICCCKKKK.

✅ El comando cat también está deshabilitado en este caso.
👉 Necesitamos usar una alternativa como tac, less, head o tail para poder leer el contenido del archivo y obtener el segundo ingrediente.


### 🔹 Paso 16: Obtención del segundo ingrediente
Probamos un comando alternativo para leer el archivo `second ingredients`:

```bash
tac /home/rick/second\ ingredients
```
📸 Evidencia:

<img width="695" height="345" alt="segundo ingrediente" src="https://github.com/user-attachments/assets/af58d84c-37cd-4eaf-9e3e-3450432b6503" />

Resultado:

1 jerry tear

### 🔹 Paso 17: Verificación de usuario actual
Ejecutamos el comando `whoami` para identificar con qué usuario estamos ejecutando las órdenes en el sistema:

```bash
whoami
```
📸 Evidencia:

<img width="608" height="333" alt="comando whoami" src="https://github.com/user-attachments/assets/5a4ff199-6a95-4182-9499-3863ff48e656" />

Resultado:

www-data

✅ Confirmamos que tenemos acceso como usuario www-data, el usuario por defecto del servicio web.
👉 Para obtener el tercer ingrediente (probablemente en /root), será necesario realizar una escalada de privilegios.

### 🔹 Paso 18: Escalada de privilegios con sudo
Comprobamos qué permisos tiene el usuario `www-data` utilizando `sudo -l`:

```bash
sudo -l
```
📸 Evidencia:

<img width="1178" height="354" alt="comando sudo l" src="https://github.com/user-attachments/assets/12c80a7a-f936-4930-af26-0f814033decc" />

Resultado:

User www-data may run the following commands on ip-10-10-160-159:
    (ALL) NOPASSWD: ALL


✅ Esto significa que el usuario www-data puede ejecutar cualquier comando como root sin necesidad de contraseña.
👉 Con esta configuración, tenemos acceso completo al sistema como root, lo que nos permitirá obtener el tercer ingrediente.

### 🔹 Paso 19: Exploración del directorio /root
Tras conseguir privilegios de root, exploramos el contenido del directorio `/root`:

```bash
sudo ls -al /root/
```
📸 Evidencia:

<img width="863" height="443" alt="listar directorios como root" src="https://github.com/user-attachments/assets/c11a40cb-7a01-4722-8d85-e6172c00ffcc" />

Archivos destacados:

.bash_history

.bashrc

.profile

.ssh/

.viminfo

3rd.txt → contiene el tercer ingrediente

✅ Confirmamos la presencia del archivo 3rd.txt, que contiene el tercer ingrediente de la poción.

### 🔹 Paso 20: Obtención del tercer ingrediente
Finalmente, con privilegios de root, leemos el archivo `3rd.txt` ubicado en `/root`:

```bash
sudo tac /root/3rd.txt
```
📸 Evidencia:

<img width="687" height="347" alt="tercer ingrediente" src="https://github.com/user-attachments/assets/be13bbf5-9ce5-49c7-b772-0de86fc13bf8" />

Resultado:

3rd ingredients: fleeb juice

✅ Con esto conseguimos el tercer y último ingrediente de la poción de Rick.



