# 💻 Escribiendo el Writeup para el CTF: Smol (TryHackMe)

## Introducción

La sala **Smol** es una sala de dificultad media que contiene muchos detalles. En esta sala hackeamos un servidor Linux con **WordPress**. Tenemos una vulnerabilidad de **LFI (Local File Inclusion)** y una de **RCE (Remote Code Execution)** en 2 *plugins* diferentes de WordPress.

***

## ⚙️ Paso 1: Reconocimiento (Ping)

El primer paso en cualquier prueba de penetración o CTF es el reconocimiento para asegurarnos de que el objetivo está activo y para medir la conectividad de la red.

### 📝 Objetivo
Verificar la conectividad con la máquina objetivo y medir la latencia.

### 🛠 Herramienta Utilizada
**ping**

### 🧑‍💻 Comando Ejecutado
Utilizamos el comando `ping` con el flag `-c 4` para enviar exactamente 4 paquetes ICMP de solicitud de eco a la dirección IP objetivo, en este caso, **10.10.194.146**.

<img width="579" height="246" alt="Ping" src="https://github.com/user-attachments/assets/7d7a71b0-868e-4bc2-ac11-33bea6d4c877" />

```bash
$ ping -c 4 10.10.194.146
````
🖥 Salida y Análisis
La salida del comando ping confirma que la máquina objetivo está activa y es accesible:

Paquetes Transmitidos/Recibidos: Se transmitieron 4 paquetes y se recibieron 4, resultando en un 0% de pérdida de paquetes. Esto indica una conexión de red estable.

Tiempo (TTL/Latencia): El Time To Live (TTL) de 63 (un valor común para sistemas operativos Linux que tienen un TTL inicial de 64) y los tiempos de ida y vuelta (RTT) promedio de 54.965 ms confirman que la máquina está respondiendo. La latencia es aceptable para continuar con el escaneo de puertos.

Resultado: La máquina está online. Procedemos al escaneo de puertos.

## 🔍 Paso 2: Escaneo de Puertos (Nmap)

Una vez que hemos confirmado que la máquina está activa, procedemos a realizar un escaneo de puertos para identificar los servicios que se están ejecutando en la máquina objetivo y sus versiones.

### 📝 Objetivo
Identificar los puertos abiertos, los servicios que se ejecutan en ellos y la versión del software.

### 🛠 Herramienta Utilizada
**Nmap** (Network Mapper)

### 🧑‍💻 Comando Ejecutado
Utilizamos un comando de Nmap completo para un escaneo agresivo:

<img width="910" height="359" alt="Nmap" src="https://github.com/user-attachments/assets/bc011f87-81eb-49d4-986b-1ea21c7a02b2" />

* `-T4`: Configura la temporización para ser más rápida, pero aún confiable.
* `-n`: No resuelve DNS (más rápido).
* `-p-`: Escanea todos los 65535 puertos.
* `-sC`: Ejecuta los *scripts* por defecto de Nmap (para detección básica y vulnerabilidades).
* `-sV`: Realiza la detección de la versión del servicio.
* `-oA`: (Aunque no se ve en la captura, se recomienda para guardar la salida).

```bash
$ nmap -T4 -n -sC -sV -p- 10.10.194.146
````
Nota: La captura de pantalla muestra un escaneo con -Pn (tratar al host como activo), -sC, y -sV, lo cual también es un buen enfoque.

Detalles Clave del Puerto 80:

Título HTTP: Nmap detecta que la respuesta HTTP no redirige a http://www.smol.thm.

Encabezado del Servidor: Apache/2.4.41 (Ubuntu).

Resultado: El puerto 80 (HTTP) es el foco principal para la enumeración inicial, ya que el servicio SSH requiere credenciales. Procederemos a explorar el sitio web.

## 🌐 Paso 3: Configuración del Archivo Hosts y Primera Exploración Web

Aunque el escaneo de Nmap sugirió que el sitio podría funcionar sin un nombre de host específico, es una buena práctica y a menudo necesario configurar el archivo `/etc/hosts` para mapear la dirección IP a cualquier nombre de dominio asociado. Esto es crucial cuando el servidor web utiliza *Virtual Hosts*.

### 📝 Objetivo
1.  Mapear la IP objetivo (10.10.194.146) al nombre de dominio sugerido por la sala o deducido (`smol.thm`).
2.  Visitar el sitio web y determinar la tecnología de la plataforma.

### 🛠 Herramienta Utilizada
**GNU nano** (para editar `/etc/hosts`) y un **navegador web**.

### 🧑‍💻 Edición del Archivo Hosts
Se añade la siguiente línea al archivo `/etc/hosts`:

```bash
# Contenido añadido a /etc/hosts
10.10.194.146 smol.thm
````
<img width="442" height="225" alt="agrego el sitio a hosts" src="https://github.com/user-attachments/assets/03feb45c-5205-4c79-98c0-218836b348c4" />

🖥 Exploración del Sitio Web
Al acceder al sitio a través del navegador web usando el nombre de dominio configurado (http://smol.thm/), la página principal se carga correctamente.

<img width="1707" height="943" alt="visitamos sitio p80 wordpress" src="https://github.com/user-attachments/assets/78af3c1e-b17d-4c34-87c8-b1db1e902d85" />

Hallazgo Clave:
En la parte inferior de la página, se encuentra el indicio crucial:

"Proudly powered by WordPress"

Resultado: La máquina objetivo está ejecutando una instancia de WordPress. Este hallazgo cambia el enfoque de nuestra enumeración hacia la identificación de posibles vulnerabilidades en temas, plugins o en la configuración de esta plataforma de gestión de contenido (CMS). El siguiente paso será enumerar la instalación de WordPress.

## 🔎 Paso 4: Enumeración de WordPress con WPScan

Con la confirmación de que la máquina ejecuta **WordPress**, utilizamos **WPScan** para automatizar la detección de la versión de WordPress, temas y, lo más importante, *plugins* potencialmente vulnerables.

### 📝 Objetivo
Identificar *plugins* instalados y buscar vulnerabilidades conocidas asociadas a ellos.

### 🛠 Herramienta Utilizada
**WPScan**

### 🧑‍💻 Escaneo de Plugins y Detección
El escaneo reveló la presencia del *plugin* llamado **jsmol2wp**:


| Plugin | Versión | Ruta |
| :---: | :---: | :--- |
| **jsmol2wp** | **1.07** | `http://www.smol.thm/wp-content/plugins/jsmol2wp/` |

<img width="621" height="268" alt="scaneo con wpscan plugin vulnerable" src="https://github.com/user-attachments/assets/f84711e2-5805-4192-8020-ee93299f39bd" />

### 📚 Búsqueda de Vulnerabilidades

La versión **1.07** del *plugin* `jsmol2wp` fue el foco de la investigación. Al buscar vulnerabilidades conocidas (CVEs o entradas en la base de datos de WPScan) para esta versión, se encontraron al menos dos fallos de seguridad críticos:

#### 1. Server Side Request Forgery (SSRF) No Autenticada

* **Vulnerabilidad:** **SSRF No Autenticada** en la versión **<= 1.07** de **JSmol2WP**.
* **Impacto:** Permite a un atacante no autenticado forzar al servidor a hacer solicitudes a otros recursos de red, que pueden ser internos (Lectura de archivos locales o acceso a servicios internos) o externos.
* **Prueba de Concepto (PoC) Relevante:** El PoC sugiere la posibilidad de leer archivos locales (como el archivo de configuración de WordPress) a través de un *wrapper* de protocolo:
    ```
    .../wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
    ```
    
<img width="1409" height="448" alt="vulnerabilidad ssrf" src="https://github.com/user-attachments/assets/be60f5cb-d9b8-4cc5-b65d-ce7bd156b491" />

#### 2. Cross-Site Scripting (XSS) No Autenticada

* **Vulnerabilidad:** **XSS No Autenticada** en la versión **<= 1.07** de **JSmol2WP**.
* **Impacto:** Permite la inyección de código *script* malicioso en páginas web visualizadas por otros usuarios (por ejemplo, un administrador), lo que podría conducir al robo de *cookies* o a la ejecución de acciones.
* 
<img width="1322" height="402" alt="vulnerabilidad xss" src="https://github.com/user-attachments/assets/fd4d0011-70b4-4ca0-8bed-0e134f886354" />

**Decisión:** La vulnerabilidad de **SSRF/LFI** (Lectura de Archivos Locales a través del *wrapper* `php://filter`) es el camino más directo para obtener credenciales, ya que **`wp-config.php`** a menudo contiene las credenciales de la base de datos.

**Resultado:** Se ha identificado el *plugin* vulnerable **jsmol2wp** y se priorizará la explotación de la vulnerabilidad **SSRF/LFI** para la lectura de archivos.
