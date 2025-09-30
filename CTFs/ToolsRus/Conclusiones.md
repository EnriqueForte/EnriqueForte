# 💡 Conclusiones y Aprendizajes del CTF Toolsrus

El CTF **Toolsrus** fue un excelente ejercicio para practicar la metodología de un *penetration tester* estándar, demostrando cómo una serie de pequeñas pistas y hallazgos pueden converger en una ruta de explotación clara.

## 🔑 Puntos Clave del Recorrido

### 1. La Importancia de la Enumeración Exhaustiva

La fase de reconocimiento fue la más crítica. No solo el escaneo de Nmap reveló puertos no estándar (1234, 8009), sino que el escaneo de directorios con `dirb` nos dio las pistas esenciales que redirigieron el ataque:

* **Pista de Servicios:** El mensaje de la página principal (**"Other parts of the website is still functional..."**) nos obligó a pivotar del puerto 80 a los otros servicios.
* **Pista de Credenciales:** El directorio `/guidelines/` reveló el nombre de usuario **`bob`** y confirmó el vector de ataque en **Tomcat**.

### 2. Aprovechamiento de la Información Obtenida

Una vez que obtuvimos el nombre de usuario (`bob`), se convirtió en un recurso invaluable que utilizamos inmediatamente:

* **Fuerza Bruta Dirigida:** El uso de `hydra` contra el directorio `/protected` (código 401) con un nombre de usuario válido (`bob`) nos ahorró tiempo al no tener que buscar *exploits* ciegamente.
* **Acceso de Manager:** Las credenciales de `bob` resultaron tener permisos de **`manager`** en el servidor Tomcat, lo que simplificó la fase de explotación al permitir la carga de *webshells* de forma trivial (a través de Metasploit).

### 3. Explotación de Configuraciones Antiguas

El éxito final en el acceso inicial se basó en la explotación de un servidor **Apache Tomcat/7.0.88** mal configurado/desactualizado.

* **Nikto** confirmó que los peligrosos métodos **`PUT` y `DELETE`** estaban habilitados, y la interfaz **Manager Application** estaba expuesta.
* El módulo de Metasploit, `exploit/multi/http/tomcat_mgr_upload`, demostró ser la forma más profesional y eficiente de automatizar el proceso de **carga de un WAR malicioso** con credenciales de `manager`, resultando en la obtención de la *Meterpreter shell*.

## 🛠️ Herramientas Fundamentales Practicadas

Este CTF sirvió como un excelente campo de pruebas para el uso coordinado de las siguientes herramientas de ciberseguridad:

* **`ping` & `nmap`**: Reconocimiento básico y mapeo de servicios.
* **`dirb`**: Enumeración de contenido web oculto.
* **`hydra`**: Ataque de fuerza bruta dirigido a autenticación HTTP.
* **`Nikto`**: Escaneo de vulnerabilidades web para confirmar *exploits* de configuración (ej. métodos PUT/DELETE).
* **`Metasploit`**: Automatización del proceso de explotación y post-explotación para obtener la *reverse shell*.

---
