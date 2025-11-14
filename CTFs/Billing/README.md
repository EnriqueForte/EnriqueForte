# TryHackMe: Billing - Writeup

## 🏁 Introducción

La sala de facturación era sencilla; allí explotamos una vulnerabilidad de inyección de comandos en la aplicación web **MagnusBilling** para obtener acceso inicial. Posteriormente, utilizando nuestros privilegios de sudo, que nos permitieron interactuar con el servidor **fail2ban** y configurarlo, escalamos privilegios hasta convertirnos en el usuario **root** y completamos la sala.

---

## 📝 Paso 1: Ping al objetivo

🚀 **Identificamos la IP objetivo**  
   En este primer paso, comprobamos la conectividad con la máquina utilizando el comando `ping`. La dirección IP del objetivo es `10.10.223.153`.

   ````bash
   ping -c 4 10.10.223.153
   ````

<img width="645" height="265" alt="Ping" src="https://github.com/user-attachments/assets/7c6e8a10-3be2-486a-a75f-1f81318658ba" />

- El resultado muestra que los paquetes ICMP son recibidos correctamente, lo que indica que la máquina está activa y responde en la red.
- Estadísticas principales:
  - **0% packet loss** (sin pérdida de paquetes)
  - **Promedio de latencia ~47ms**

---

## 🔎 Paso 2: Escaneo con Nmap

🧭 **Escaneo de puertos con Nmap**  
   El siguiente paso fue escanear la máquina objetivo utilizando Nmap para identificar los servicios y puertos abiertos:

````bash
nmap -p- -sV -sC --open --min-rate 5000 -vvv -n -Pn -oN escaneo 10.10.223.153
````

- **Puertos abiertos encontrados:**
  - **22/tcp (SSH):** OpenSSH 9.2p1 Debian
  - **80/tcp (HTTP):** Apache/2.4.62 (Debian), con el título *MagnusBilling*
  - **3306/tcp (MySQL):** MariaDB (no autorizado)
- Observamos que el sitio web en el puerto 80 corresponde a MagnusBilling, lo cual será clave para identificar vectores de ataque.
- El escaneo revela también métodos HTTP soportados y pequeños detalles sobre la configuración del servidor.

<img width="1782" height="636" alt="Nmap" src="https://github.com/user-attachments/assets/3b4aaa10-7502-4451-9494-354a096819a8" />

---

## 🌐 Paso 3: Inspección de la web y escaneo con ffuf

🧐 **Exploración del puerto 80 y enumeración de ficheros/directorios**  
   Tras identificar el servicio web MagnusBilling en el puerto 80, accedimos a la interfaz de login a través de nuestro navegador para observar posibles vectores de ataque y vulnerabilidades.

   <img width="895" height="985" alt="Pagina p80" src="https://github.com/user-attachments/assets/48a6652d-d2cd-4900-9c8a-078fc4e58f23" />

   - Detectamos aspectos importantes del portal de ingreso y la estructura del sitio, lo que nos motivó a buscar rutas y archivos ocultos.

   🚩 **Descubrimiento de archivos y directorios con ffuf**  
   Utilizamos la herramienta `ffuf` para enumerar rutas y archivos accesibles en el servidor web. Esto nos permitió identificar varios recursos interesantes como `README.md`, `cron.php`, `LICENSE`, y otros.

   <img width="470" height="185" alt="ffuf" src="https://github.com/user-attachments/assets/556be5af-8293-456b-895b-11ea602deba3" />

   - Algunos archivos como `cron.php` y `README.md` pueden ser valiosos para análisis posteriores, ya sea por información sensible o posibles puntos de entrada.
   - El directorio `/mbilling/` es confirmado como relevante en la aplicación.

---

## 🛡️ Paso 4: Identificación y explotación de vulnerabilidad (CVE-2023-30258)

🕵️ **Investigación en la aplicación, revisión de vulnerabilidades y PoC**  
   Tras explorar los archivos encontrados, accedimos a `README.md` y confirmamos que la aplicación corre la versión **MagnusBilling 7.x.x**, la cual, según distintas fuentes, es vulnerable a inyección de comandos sin autenticación (CVE-2023-30258).

   <img width="831" height="569" alt="entroa README y veo version de la app" src="https://github.com/user-attachments/assets/4a804d2d-4014-4a62-9fe2-d3c0a3769f77" />

   - Consultando distintas fuentes de seguridad, comprobamos la existencia de la vulnerabilidad CVE-2023-30258 que permite a un atacante ejecutar comandos arbitrarios sobre el sistema a través de peticiones HTTP especialmente diseñadas, sin necesidad de estar autenticado.

   <img width="811" height="726" alt="vulnearbilidad encontrada" src="https://github.com/user-attachments/assets/25c41145-2700-4a19-b7ab-4e4cc8747b3d" />

   - Localizamos un PoC funcional y una ruta vulnerable (`/mbilling/lib/icepay/icepay.php?democ=`) para explotar la inyección de comando utilizando la siguiente carga (como el ejemplo sleep;ls):

   ````bash
   /mbilling/lib/icepay/icepay.php?democ=/dev/null;ls%20a
   ````
   
  <img width="843" height="598" alt="encontramos poc" src="https://github.com/user-attachments/assets/96e0b8b6-7f65-4042-8ee5-4a3212960326" />

---

## 🏹 Paso 5: Explotando la vulnerabilidad y obteniendo acceso inicial

🐚 **Explotación de Command Injection para ejecución remota y obtención de Shell**  
   Utilizando el parámetro vulnerable de MagnusBilling, realizamos pruebas con el valor del parámetro `democ` en `icepay.php` inyectando comandos `sleep` para validar el tiempo de respuesta y la capacidad de ejecutar comandos arbitrarios tanto desde el navegador como desde consola con `curl`.

   - Prueba desde el navegador (3 y 10 segundos usando sleep):   
  
  <img width="840" height="185" alt="inserto el poc espero 3s" src="https://github.com/user-attachments/assets/9556311a-71e5-44fd-af64-5b88f7b2d697" />

  <img width="851" height="185" alt="inserto el poc espero 10s" src="https://github.com/user-attachments/assets/f66e57fc-7505-4981-9ae5-890b4eba46b5" />

   - Prueba desde la terminal con `curl` para medir el tiempo y automatizar la ejecución:
   
````bash
time curl -s 'http://10.10.151.3/mbilling/lib/icepay/icepay.php?democ=;sleep+2;'
time curl -s 'http://10.10.151.3/mbilling/lib/icepay/icepay.php?democ=;sleep+5;'
````

<img width="873" height="190" alt="probando time curl en consola y ejecuta comandos" src="https://github.com/user-attachments/assets/ef445be2-63ce-439f-9e20-bda2ffa41e32" />

- A continuación, procedimos a conseguir una reverse shell mediante una petición especialmente formada que invoca una conexión hacia nuestro equipo atacante. Al recibir la conexión entrante podemos interactuar como usuario limitado dentro del sistema objetivo.

<img width="853" height="739" alt="obtengo reverse shell y flag user" src="https://github.com/user-attachments/assets/81b72863-0a93-4382-bf9e-d87779abd479" />

- Se mostró el contenido de la flag de usuario (`THM{4a683...}`) y el acceso inicial quedó establecido.

---

## ⚙️ Paso 6: Enumeración de privilegios y configuración de fail2ban

🔒 **Enumerando permisos e investigando fail2ban para escalar privilegios**
   Tras obtener acceso como usuario limitado (asterisk), comenzamos la enumeración de privilegios y servicios críticos. Descubrimos que la máquina utiliza **fail2ban** y que el usuario `asterisk` tiene configuración especial en `sudo` para ejecutar el binario `/usr/bin/fail2ban-client` sin contraseña.

   <img width="743" height="162" alt="revisamos el archivo jail local" src="https://github.com/user-attachments/assets/787bc949-49d5-4cf8-8794-8437800cd245" />

   - Localizamos la ruta de logs gestionados por fail2ban y observamos el filtro para `asterisk`.

   <img width="867" height="267" alt="vemos que el bin se ejcuta como root" src="https://github.com/user-attachments/assets/e07bb499-087b-43d1-a962-58d9a4ab5aba" />

   - Analizando procesos comprobamos que el servidor fail2ban corre bajo privilegios de root, abriendo posibilidades para explotación mediante manipulación de logs o comandos.

   <img width="843" height="251" alt="permisos de asterisk puede ejecutar un bin" src="https://github.com/user-attachments/assets/887e021f-8959-4cc0-906c-7f3ff93f7255" />

   - Usando `sudo -l`, validamos que el usuario `asterisk` puede lanzar `/usr/bin/fail2ban-client` como root con permisos completos, lo que ofrece un vector para escalar privilegios mediante la interacción con este servicio.

---

## 🏁 Paso 7: Escalada de privilegios y captura de la flag root

🚀 **Escalando privilegios mediante manipulación de fail2ban y obteniendo root**
   Gracias a los permisos de sudo y la ejecución de fail2ban-client como root, explotamos su funcionalidad para configurar una acción personalizada. Usamos el comando `actionban` para modificar los permisos de `/bin/bash`, habilitando el SUID bit y permitiendo la ejecución de bash como root.

   <img width="877" height="348" alt="modificamos el comando para la actionban" src="https://github.com/user-attachments/assets/8220915b-2114-4493-994e-2abfe30ded61" />

   - Verificamos el cambio de permisos y confirmamos el bit SUID sobre `/bin/bash`:

   <img width="792" height="65" alt="permisos de root" src="https://github.com/user-attachments/assets/9d612381-4831-4c3d-a285-5d0fd8929658" />

   <img width="872" height="438" alt="configuramos manualmente las direccion IP" src="https://github.com/user-attachments/assets/d2160064-dc14-43cb-a2ae-cb4d193b5729" />

   - Finalmente accedimos a una shell privilegiada ejecutando `bash -p`, comprobando que somos root (`uid=0(root) gid=0(root)`). Accedimos al directorio `/root` y leímos la flag del reto:

   <img width="868" height="362" alt="elevamos priveleggios y la bandera root es nuestra" src="https://github.com/user-attachments/assets/f07502d9-61c4-4446-bd9b-dcb78e2e2739" />


   - Flag obtenida: `THM{33ac...}`

---











