# 🎯 Introducción: Investigating Windows (TryHackMe)

La sala Investigating Windows de TryHackMe es un desafío práctico de Forense Digital y Respuesta a Incidentes (DFIR) 🕵️ centrado en un entorno Windows comprometido.

Nuestro rol es el de un investigador de seguridad encargado de analizar un endpoint de Windows que ha sido hackeado. El objetivo es rastrear las huellas del atacante, analizando artefactos forenses clave como los registros de eventos, tareas programadas, el Registro de Windows y la actividad de archivos. 

Esto nos permitirá reconstruir la cronología del ataque ⏳, identificar las técnicas utilizadas y, finalmente, responder a las preguntas del CTF.

Es un ejercicio fundamental para desarrollar habilidades en la investigación de sistemas operativos Windows.

## 🛠️ Paso 1: Conexión Inicial por Escritorio Remoto (RDP)

El primer paso esencial es establecer la conexión con la máquina comprometida a través de la Conexión a Escritorio Remoto (RDP), tal como se muestra en tu captura.

<img width="402" height="489" alt="Conexion Remota" src="https://github.com/user-attachments/assets/4da403c0-05c9-44a5-a0e5-ea64d063387a" />


Acciones a realizar:

Obtener la IP: Inicia la máquina en TryHackMe para obtener su dirección IP. 📡

Configurar la conexión RDP:

En el campo Equipo: Introduce la dirección IP que te proporciona TryHackMe.

El campo Usuario: Ya está configurado como Administrator.

Conectar: Haz clic en el botón Conectar. 🔗

Credenciales: Cuando se te solicite, introduce la contraseña proporcionada en la sala para el usuario Administrator. 🔑

Una vez que las credenciales sean aceptadas, tendremos acceso al escritorio del servidor Windows comprometido y podremos empezar a buscar artefactos del ataque. ¡La investigación comienza ahora! 🔎



¡Fantástico! Ya estamos dentro de la máquina, como lo confirma tu segunda captura de pantalla, que muestra la ventana de Windows PowerShell abierta con la IP 10.10.123.8 en la barra de título de la sesión RDP.

Ahora, continuemos con el Paso 2: Reconocimiento del Sistema Operativo y, de paso, respondemos la primera pregunta del CTF.

## 💻 Paso 2: Reconocimiento del Sistema Operativo y Versión (Pregunta 1)

El objetivo es obtener la versión y el año de la máquina Windows comprometida.

1. 🔍 Análisis Inicial (Desde la Captura)
   
Observando la ventana de PowerShell en tu captura (Conectado dentro del escritorio.png):

<img width="1046" height="978" alt="Conectado dentro del escritorio" src="https://github.com/user-attachments/assets/e8375df8-1538-4ca9-9037-93906cb7b1fe" />


Vemos la línea de copyright: Copyright (C) 2016 Microsoft Corporation. All rights reserved.

Esto es un fuerte indicio de que estamos en una versión de Windows lanzada o actualizada en 2016, que suele ser Windows Server 2016.

2. 📝 Ejecución del Comando de Verificación
   
Aunque el copyright ya nos da una pista muy sólida, es una buena práctica forense confirmarlo con un comando del sistema. Usaremos systeminfo.

Consola	Comando a Ejecutar	Propósito

PowerShell	systeminfo | select-string "Nombre del SO"	Muestra información detallada del sistema y filtra la línea del Nombre del SO.

<img width="983" height="789" alt="Ejcutamos PowerShell comando GetComputerinfo" src="https://github.com/user-attachments/assets/c53fb7b5-7751-4ff7-8bd0-3c43f7179a70" />

## 🕵️ Paso 3: Identificar el Último Inicio de Sesión (Pregunta 2)

Hemos utilizado un método de PowerShell más preciso (Get-ComputerInfo -Property "Os*"), y la salida en tu captura Getinfo con Property OS para resumir la busqueda y obtener ver sis op.png es clara:

<img width="770" height="229" alt="Getinfo con Property OS para resumir la busqueda y obtener ver sis op" src="https://github.com/user-attachments/assets/f9c59a80-60f5-40be-ac45-485c666d7e81" />


OsName: Microsoft Windows Server 2016 Datacenter

OsVersion: 10.0.14393 (Esta es la versión del kernel de Server 2016)

El copyright de 2016 y la información del OsName confirman la versión.


## 👥 Paso 4: Listar Usuarios Locales y Preparar el Rastreo de Sesiones

La captura , obtener usuarios locales.png, muestra el resultado del comando de PowerShell Get-LocalUser. 

<img width="605" height="142" alt="obtener usuarios locales" src="https://github.com/user-attachments/assets/5a9289ae-a0f4-453a-bdcf-d0ce056c1a66" />


Este comando es excelente para el reconocimiento, ya que lista todas las cuentas de usuario en el sistema.

🔍 Análisis de la Salida
   
La lista de usuarios activos es la siguiente:

Administrator (Cuenta predefinida)

Guest (Cuenta de invitado)

Jenny

John

## Paso 5: Identificar el Último Inicio de Sesión

Para determinar el usuario que inició sesión por última vez en la máquina, analizamos la información de "Last Logon" de las cuentas activas (Administrator, John, Jenny, Guest).

<img width="518" height="197" alt="conexiones al sistema quien se conecto" src="https://github.com/user-attachments/assets/4363727d-ce25-434b-b3da-1bc4685ed71a" />


🔍 Análisis de la Salida (net user [usuario] \| findstr "Last")

Tu captura muestra los siguientes datos relevantes:

Usuario	Last Logon (Último Inicio de Sesión)	Observaciones

Jenny	Never	Nunca ha iniciado sesión.

John	3/2/2019 5:48:32 PM	Inicio de sesión en el pasado (marzo de 2019).

Administrator	10/1/2025 7:42:15 AM	El más reciente. (Este es tu propio inicio de sesión de RDP para la investigación).

DefaultAccount	Never	Cuenta del sistema, nunca se usa para inicio de sesión interactivo.

Guest	Never	Nunca ha iniciado sesión.

Para propósitos del CTF, la pregunta se refiere al último usuario  interactuó con el sistema (Administrator).

John el 03/02/2019 a las 5:48:32 PM.


## Paso 6: Identificar la Conexión Maliciosa de Inicio (Pregunta 4)

La pregunta es: "¿Qué dirección IP intenta contactar el sistema cuando arranca por primera vez?"

Esto implica revisar el contenido del archivo hosts para buscar entradas inusuales que redirijan dominios legítimos a IPs de atacantes.

<img width="1112" height="285" alt="Buscaremos a que ip se conecta el sist cuando incio dentro del hosts" src="https://github.com/user-attachments/assets/8c0665a7-7a3a-4eb6-b1e1-0f95ff87247c" />

⌨️ Acceder al Contenido del Archivo hosts

Para ver el contenido utilizamos el bloc de notas:

<img width="790" height="776" alt="Leemos el archivo con blocdenotas, parece que se esta produciendo envenanmietno" src="https://github.com/user-attachments/assets/2ff65720-a61b-4fb3-8fc3-3b7d955da20f" />

Ya con el tipo de conexiones nos podemos dar cuenta de que hubo algun tipo de envenenamiento.


## ⚙️ Paso 7: Búsqueda de Persistencia en el Registro (Pregunta 5 y Posteriores)

Accederemos a regedit para ver el registro.png, indica que ahora buscaremos en el Registro de Windows (regedit). 

<img width="392" height="643" alt="accederemos a regedit para ver el registro" src="https://github.com/user-attachments/assets/76ce71e8-506f-4c21-ab3a-a78dd4090d22" />

📂 Abrir el Editor del Registro
   
🔍 Rutas Clave de Persistencia

Las rutas más comunes para que el malware se ejecute al inicio son:

Rama (Hive)	Ruta de Persistencia (Run Keys)

HKEY_LOCAL_MACHINE (Para todos los usuarios)	SOFTWARE\Microsoft\Windows\CurrentVersion\Run

HKEY_CURRENT_USER (Para el usuario actual)	SOFTWARE\Microsoft\Windows\CurrentVersion\Run

<img width="876" height="944" alt="ip a la que se conecta, HKEY_LOCAL_MACHINE  SOFTWARE  Microsoft Windows  CurrentVersion  Run" src="https://github.com/user-attachments/assets/dd31db6e-40a4-483a-9d41-be36a23a0d61" />



## 👥 Paso 8: Identificar Cuentas con Privilegios Elevados (Pregunta 5)

La pregunta del CTF es: "¿Qué dos cuentas tenían privilegios administrativos (aparte del usuario Administrator)? (Listarlas en orden alfabético)."

🔍 Análisis de la Salida (Get-LocalGroupMember)
   
Ejecutamos el comando de PowerShell: Get-LocalGroupMember -Group "Administrators". Este comando lista todos los usuarios que pertenecen al grupo de administradores locales.

<img width="533" height="116" alt="Usuarios Administradores" src="https://github.com/user-attachments/assets/92983981-ccaf-4576-bd74-2453ed7d65d8" />


La lista completa de miembros del grupo Administrators es:

Administrator

Guest

Jenny


## ⚙️ Paso 9: Búsqueda de Persistencia

Hemos identificado el usuario (John), la fecha de la intrusión y la IP del atacante (76.32.97.132). El siguiente paso es encontrar el mecanismo de persistencia que el atacante utilizó para mantener el acceso.

Listamos las tareas programadas en busca de la que esta afectado, muestra la lista de tareas programadas.

<img width="745" height="796" alt="Lsitamos las tareas programadas en busca de la que esta afectado" src="https://github.com/user-attachments/assets/b58298ab-01a6-482e-aa6c-34438899ebd8" />


Acotamos la busqueda:

<img width="709" height="152" alt="Acotamos la busqueda de las tareas" src="https://github.com/user-attachments/assets/04bdef5b-1a8c-4850-9530-dc782575d9de" />


🔍 Análisis de Tareas

Al revisar la columna TaskName, la mayoría de las tareas son procesos legítimos del sistema operativo (.NET Framework, AppID, Bluetooth, etc.).

Sin embargo, hay una tarea que destaca por su nombre genérico e inusual, típicamente utilizado para camuflar actividades maliciosas:

Nombre de la Tarea Maliciosa: ocultada 


## 👂 Paso 10: Identificar el Puerto de Escucha y la tarea

Hemos establecido que el atacante usa la tarea programada Clean file system para la persistencia. Ahora, necesitamos saber a qué puerto está escuchando el script malicioso que se ejecuta.

<img width="693" height="183" alt="buscamos la tarea que se ejcuta dirariamente y puerto de escucha" src="https://github.com/user-attachments/assets/61261649-9867-43cd-b368-99f862bdfd8f" />


🔍 Análisis de la Salida de Actions

Lacaptura muestra el análisis de la variable $task.Actions, que revela los detalles de ejecución del malware:

Arguments: -l 1348

Execute: C:\TMP\nc.ps1 (asumiendo que el campo Execute apunta al script de netcat, aunque se ve parcialmente oculto, la lógica forense lo confirma).

El argumento clave aquí es -l 1348. En la sintaxis de Netcat (nc), la opción -l significa "modo de escucha" (listen mode), y el número que le sigue es el puerto local que se abre para recibir conexiones.


## 🔎 Paso 11: Confirmación de Actividad de la Cuenta Jenny

<img width="430" height="77" alt="cuando se conecto jenny por ultima vez" src="https://github.com/user-attachments/assets/dc90ef9e-5f34-4fc6-a84d-6ac4e0fc5299" />


🔍 Análisis de la Salida (net user Jenny)

Tu captura de pantalla muestra el resultado del comando net user Jenny \| findstr "Last":

Last logon: El campo está oculto.

Conclusión Forense: El atacante creó o elevó la cuenta Jenny para la persistencia, pero nunca la usó para un inicio de sesión interactivo, lo que ayuda a evitar dejar rastros directos.


## 🚨 Paso 12: Determinando la Hora de la Elevación de Privilegios

Hemos establecido la fecha del compromiso como 03/02/2019. Ahora debemos encontrar el momento exacto en que el atacante ganó acceso con derechos elevados, lo que significa que el ataque pasó de ser una intrusión inicial a un compromiso total.

<img width="884" height="552" alt="cuando tuvo fecha el compromiso entramos en C" src="https://github.com/user-attachments/assets/51731830-fbb5-4c3f-baa2-b99e9785c002" />

Buscando en el disco local C: podemos ver la modificacion de varias carpetas en la misma fecha , lo que confirma la intrusión a la fecha nombrada con anterioridad.


## 🚨 Paso 13: Determinando la Hora de la Elevación de Privilegios (Pregunta 11)

La pregunta es: "Durante el compromiso, ¿a qué hora Windows asignó por primera vez privilegios especiales a un nuevo inicio de sesión?" (Formato: MM/DD/YYYY HH:MM:SS AM/PM)

🔍 Estrategia de Búsqueda: Event ID 4672

Estrategia correcta: filtrar el Registro de Seguridad por la fecha del compromiso (03/02/2019) y buscar el ID de Evento, que marca cuando un usuario obtiene privilegios especiales al iniciar sesión.

<img width="763" height="793" alt="a que hora se asignaron privilegios por 1ra vez en event viewer" src="https://github.com/user-attachments/assets/fa1c4023-daeb-46bf-949b-82948ab924c4" />


<img width="878" height="764" alt="creamos una custom view personalizada" src="https://github.com/user-attachments/assets/d9e68531-9f0f-448a-92af-3636e45dea9f" />


<img width="540" height="547" alt="eventos logs security" src="https://github.com/user-attachments/assets/2880bd6f-0bb8-4674-8e15-4e9eda9db7a7" />


<img width="1212" height="942" alt="encontramos la fecha y hora de asignaciond e privelegios" src="https://github.com/user-attachments/assets/87d1bec1-6594-4c3e-8a0a-3500a0f2370e" />


⏳ Análisis de la Captura
   
En la captura encontramos la fecha y hora de asignacion de privelegios muestra los eventos de ese día (3/2/2019) ordenados por hora. 

Al escanear los eventos de la parte inferior, que representan los más antiguos en ese rango de tiempo, vemos el siguiente evento 4735 (Special Logon).

Fecha y Hora del Primer Evento 4672: 3/2/2019 4:04:49 PM (justo antes del evento 4674).

Este es el momento exacto en que el sistema operativo concedió privilegios de administrador al inicio de sesión del atacante, marcando el inicio formal de la fase de elevación de privilegios.


## 🛠️ Paso 14: Búsqueda de Herramientas de Ataque

🔍 Análisis de la Evidencia

La captura muestra el contenido del directorio C:\TMP, el cual ha sido utilizado por el atacante. 

<img width="339" height="103" alt="sigue apareciendo TMP mim exe posible herramienta que se este utilizando" src="https://github.com/user-attachments/assets/9e80e7ce-1d70-4c00-85d1-5d3574df4881" />

<img width="1364" height="788" alt="nombre de la herramienta para obtner password" src="https://github.com/user-attachments/assets/8c5b55eb-f58f-466d-aaea-772591cfb626" />

Dentro, vemos un archivo llamado mim-out cuyo contenido se muestra en el Bloc de Notas.

El archivo mim-out es un log o volcado de una herramienta de recolección de credenciales.

La cabecera del archivo mim-out contiene información


## 🌐 Paso 15: Identificar el Servidor de Comando y Control (C2)

🔍 Análisis del Archivo hosts

<img width="1274" height="744" alt="vemos dentro de hosts google sospechoso" src="https://github.com/user-attachments/assets/6fde17f1-8e11-4c6b-9464-05c0ce9b23d9" />


En la captura vemos dentro de hosts google sospechosos muestra el contenido del archivo C:\Windows\System32\drivers\etc\hosts. Hemos identificado entradas de DNS Poisoning.

La evidencia clave es la redirección de dominios populares a una IP externa específica:

76.32.97.132 google.com

76.32.97.132 www.google.com

El atacante modificó este archivo para redirigir el tráfico del sistema, probablemente para evitar que la máquina contacte a Google y, en su lugar, se comunique con el servidor del atacante, que es el Servidor de Comando y Control (C2).

🔌 Confirmación del Efecto (Ping)

<img width="956" height="488" alt="ping a google no hay respuesta" src="https://github.com/user-attachments/assets/35588bb4-a63f-42e7-87b3-49af504d5e3c" />


La captura ping a google no hay respuesta.png confirma el resultado de esta manipulación:

Al hacer ping google.com, el sistema resuelve el nombre al host manipulado: 76.32.97.132.

El ping a esta IP falla (Request timed out, 100% loss).


## 🕸️ Paso 16: Identificar el Web Shell

🔍 Análisis del Directorio Web (C:\inetpub\wwwroot)

<img width="1106" height="458" alt="extension del shell cargado" src="https://github.com/user-attachments/assets/5d426c9c-dc67-41db-8d1b-3d7f0ba6bb8e" />

La captura extension del shell cargado.png muestra el contenido de la carpeta raíz del servidor web IIS, que se encuentra en C:\inetpub\wwwroot.

Al revisar los archivos, encontramos dos archivos sospechosos con una extensión inusual para este directorio:

shell.jps

test.jsp

Un archivo con extensión .jsp (Java Server Pages) se utiliza comúnmente como un web shell para mantener el acceso al servidor después de la explotación inicial.


## 🚪 Paso 17: Búsqueda del Último Puerto Abierto

🔍 Análisis de la Evidencia de Firewall

Hemos realizado un filtro en el Firewall de Windows con Seguridad Avanzada para revisar las Reglas de Entrada (Inbound Rules).

<img width="1436" height="935" alt="buscamos cual fue el ultimo puerto que abrio el atacante" src="https://github.com/user-attachments/assets/356b65ae-c9b6-4d31-930e-234a141d0308" />

En la captura ahi encontramos el puerto muestra la regla sospechosa que el atacante creó:

<img width="1035" height="784" alt="ahi encontramos el puerto" src="https://github.com/user-attachments/assets/ddea7823-4e80-4872-888f-a97d3b6d5719" />


Nombre de la Regla: Allow outside connections for development (Un nombre genérico para camuflar la actividad maliciosa).

Propiedades: Al examinar las propiedades de Protocolo y Puertos:

Protocolo: TCP

Puerto Local: Se observa un campo que está oculto, pero que, basado en la lógica de este CTF (y el formato que se espera), es el último puerto que el atacante abrió para mantener una puerta trasera persistente.


## 🎯 Paso 18: Identificar el Sitio Web Objetivo

<img width="783" height="781" alt="que sitio fue el objetivo" src="https://github.com/user-attachments/assets/7001070e-0ba5-4ea3-8c65-b1039eb7be09" />

🔍 Análisis del Archivo hosts

La captura muestra cual sitio fue el objetivo y muestra las entradas maliciosas:

Las líneas inferiores muestran la IP del atacante (76.32.97.132) seguida del sitio web.

El sitio web redirigido es google.com (y www.google.com).

