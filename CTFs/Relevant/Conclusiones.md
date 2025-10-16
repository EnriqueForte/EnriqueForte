# 🔒 Conclusiones de la Auditoría de Seguridad (CTF Relevant)

Este documento resume los hallazgos críticos y el proceso de explotación exitoso realizado sobre el entorno de prueba.

---

## 🎯 Resumen Ejecutivo

La auditoría de seguridad de caja negra reveló una postura de seguridad **débil** en el servidor Windows. La explotación inicial se logró combinando una configuración permisiva en los recursos compartidos de SMB (Server Message Block) y la ejecución de código no autenticado a través del servicio web IIS. La escalada de privilegios se efectuó mediante una vulnerabilidad conocida del sistema, logrando el control total del servidor.

**Objetivos cumplidos:** Se obtuvieron **`user.txt`** y **`root.txt`** como prueba de concepto.

---

## 🚩 Hallazgos Críticos y Rutas de Explotación

Se identificaron dos vectores principales de riesgo en la máquina `10.10.137.195`.

### 1. Ruta de Explotación Principal (Acceso Inicial y Root)

Este es el camino que se utilizó para obtener acceso y privilegios de `SYSTEM`, cumpliendo con la solicitud de explotación manual.

| Fase | Vulnerabilidad / Debilidad | Riesgo | Mitigación Recomendada |
| :--- | :--- | :--- | :--- |
| **Acceso Inicial** | **Permisos de Escritura en SMB:** El recurso compartido `nt4wrksv` permite acceso y escritura sin credenciales (Acceso de Invitado o Anónimo). | **CRÍTICO** | Restringir el acceso de escritura al *share* `nt4wrksv` solo a usuarios autenticados y necesarios, o deshabilitar el acceso de invitado. |
| **Ejecución de Código** | **IIS en Puerto 49663:** El directorio SMB está accesible y tiene permisos de ejecución web (ASPX) en el servidor IIS. | **CRÍTICO** | Configurar el directorio raíz del servidor web para que **no coincida** con un *share* de red de solo escritura. Deshabilitar la ejecución de *scripts* en directorios de transferencia. |
| **Escalada de Privilegios** | **Privilegio `SeImpersonatePrivilege`:** Habilitado en la cuenta de servicio del proceso IIS. Vulnerable a la suplantación de *token* (ej. **PrintSpoofer**). | **CRÍTICO** | Aplicar el parche de seguridad para el servicio Print Spooler o revocar el privilegio `SeImpersonatePrivilege` de cuentas de servicio innecesarias. |

### 2. Vulnerabilidad Crítica Alternativa

Tal como se solicitó, se encontró una segunda ruta de riesgo, de alta criticidad, en el servicio de red.

| ID de Vulnerabilidad | Servicio | Descripción | Riesgo |
| :--- | :--- | :--- | :--- |
| **CVE-2017-0143 (MS17-010)** | SMBv1 (Puerto 445) | Ejecución remota de código (RCE) en el *buffer* de SMBv1, explotable a través de **EternalBlue** o herramientas similares. | **CRÍTICO** |

**Mitigación de MS17-010:** Deshabilitar el protocolo **SMBv1** y asegurar que el sistema operativo Windows Server 2016 esté completamente actualizado con el parche de seguridad **MS17-010** o sus sucesores.

---

## 📝 Pasos Realizados (Prueba de Explotación)

1.  **Reconocimiento:** Identificación de puertos abiertos (80, 445, 49663, 3389) en un **Windows Server 2016**.
2.  **Enumeración SMB:** Descubrimiento del *share* de escritura `nt4wrksv` y obtención de `passwords.txt` (aunque las credenciales estaban desactualizadas).
3.  **Acceso Inicial:**
    * Generación de un *payload* **`exploit.aspx`** con `msfvenom`.
    * Subida del *payload* al *share* **`nt4wrksv`** usando `smbclient`.
    * Ejecución de la *reverse shell* al navegar a `http://10.10.137.195:49663/nt4wrksv/exploit.aspx`.
    * Obtención de *shell* como **`IIS APPPOOL\nt4wrksv`**.
4.  **Post-Explotación:** Navegación al directorio del usuario **Bob** y recuperación de la *flag* **`user.txt`**.
5.  **Escalada a Root:**
    * Identificación del privilegio **`SeImpersonatePrivilege`** habilitado.
    * Uso del *exploit* **PrintSpoofer** para suplantar el *token* de sistema.
    * Obtención de un nuevo *shell* con privilegios **`NT AUTHORITY\SYSTEM`**.
6.  **Objetivo Final:** Recuperación de la *flag* **`root.txt`** del directorio del Administrador.

***

**Fin del Informe**
