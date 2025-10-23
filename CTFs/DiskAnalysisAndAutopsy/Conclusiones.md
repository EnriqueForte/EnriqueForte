# Conclusiones del Análisis Forense (Disk Analysis and Autopsy)

El análisis forense de la imagen de disco **`HASAN2.E01`** revela evidencia concluyente de actividad maliciosa, instalación de herramientas de hacking y compromiso del sistema por parte de múltiples cuentas de usuario.

---

## 🛑 Hallazgos Clave de la Investigación

La investigación se centró en la actividad de varios usuarios, principalmente **`HASAN`** y **`shreya`**, y arrojó la siguiente evidencia crítica:

### 1. Intrusión y Uso de Exploits (Zerologon)

* **Exploit Identificado:** Se encontró evidencia de la descarga de un archivo relacionado con un exploit conocido para vulnerar controladores de dominio de Windows: **`2.2.0 20200918 Zerologon encrypted.zip`** (Paso 14 y 15).
* **Actividad Sospechosa:** El script de PowerShell **`exploit.ps1`** fue encontrado en el escritorio del usuario `HASAN`, conteniendo código de *reverse shell* y un mensaje de compromiso: `Flag{I-hacked-you}` (Paso 11).

### 2. Herramientas de Hacking de Credenciales

Se identificó la presencia y/o la detección por parte del antivirus de herramientas de hacking de contraseñas de alto riesgo en la carpeta de descargas de `HASAN`:

* **Mimikatz:** Archivos comprimidos (`mimikatz.zip`, `mimikatz_trunk.zip`) encontrados en la carpeta de descargas (Paso 12).
* **LaZagne:** Detectado y clasificado por Windows Defender como `HackTool:Win32/LaZagne` (Paso 12).
* **Confirmación YARA:** La presencia del archivo de regla **`kiwi_passwords.yar`** confirmó que se estaba utilizando una regla específica para identificar el binario de Mimikatz (Paso 13).

### 3. Actividad de Usuarios y Artefactos

* **Usuarios Implicados:** Se identificaron múltiples cuentas, siendo **`HASAN`**, **`shreya`**, **`sandhya`** y **`joshuva`** los usuarios que mostraron actividad relevante o almacenaron evidencia crítica (Paso 5 y 10).
* **Cambio de Bandera:** La usuaria **`shreya`** utilizó PowerShell para manipular un archivo de texto, dejando una bandera modificada: `flag{HarleyQuinnForQueen}` (Paso 11).
* **Monitoreo de Red:** El programa **`Look@LAN 2.50 Build 35`** fue instalado y se encontró información de configuración de red, incluyendo la IP `192.168.100.21` y la MAC `000072CE4B89` (Paso 6 y 7).

---

## 📌 Datos de Identificación y Metadatos

| Artefacto | Dato Encontrado | Paso |
| :--- | :--- | :--- |
| **Hash MD5 de la Imagen** | `39b8c51bd0b3b5c1359d949657d8e2079` | Paso 3 |
| **Nombre del Ordenador** | `DESKTOP-0R59O3J` | Paso 4 |
| **Adaptador de Red (NIC)** | `Intel(R) PRO/1000k NDIS 6 Adapter Driver` | Paso 8 |
| **Ubicación Geográfica** | `12°52'23.0"N 80°13'25.0"E` (Marcador Web) | Paso 9 |

---

## 📝 Resumen

La imagen de disco presenta una clara intención maliciosa, evidenciada por la instalación de software de hacking de contraseñas (Mimikatz, LaZagne) y la descarga de un exploit de elevación de privilegios (Zerologon). La actividad fue distribuida entre varias cuentas, siendo **`HASAN`** la principal fuente de las herramientas de ataque.
