# Conclusiones del Proceso de Hacking

Este documento resume la metodología empleada para comprometer la máquina, destacando los puntos de entrada, los vectores de ataque y la ruta de escalada de privilegios utilizada.

## 1. Fase de Enumeración y Obtención de Credenciales (Foothold)

La fase inicial se centró en la identificación y explotación de servicios UDP que a menudo contienen información sensible.

* **Descubrimiento de Servicios Clave:** Se identificaron los puertos UDP abiertos `500/isakmp` y `4500/nat-t-ike` (VPN IKE/IPsec), junto con el puerto `69/tftp`.
* **Vector TFTP:** El servicio TFTP (Trivial File Transfer Protocol) permitió la descarga del archivo de configuración del *router* (`ciscortr.cfg`), revelando el nombre de usuario **`ike`**.
* **Vector IKE (Paso 5):** Se utilizó `ike-scan` en modo agresivo para capturar parámetros de la clave precompartida (PSK) asociados al ID `ike@expressway.htb`.
* **Crackeo de la PSK (Paso 5):** La herramienta `psk-crack` descifró la PSK utilizando un ataque de diccionario, revelando la clave: **`freakingrockstarontheroad`**.

## 2. Acceso de Usuario (User Flag)

Con las credenciales obtenidas, se procedió a obtener acceso interactivo a la máquina.

* **Conexión SSH (Paso 6):** El servicio SSH (TCP 22) estaba abierto. Se logró iniciar sesión utilizando el usuario `ike` y la PSK descifrada como contraseña.
* **Bandera del Usuario:** Una vez dentro, se obtuvo la bandera del usuario leyendo el archivo `user.txt`.

## 3. Fase de Escalada de Privilegios (Privilege Escalation)

La fase final se enfocó en la enumeración local para encontrar una vulnerabilidad que permitiera elevar privilegios a *root*.

* **Enumeración Local (Paso 7):** Se utilizó el *script* `linpeas.sh` (descargado a través de `wget`) para identificar binarios con permisos especiales.
* **Descubrimiento SUID (Paso 7):** La enumeración destacó la presencia del binario `/usr/bin/sudo` con el *bit* SUID activado, sugiriendo una posible vulnerabilidad en el programa.
* **Identificación de Vulnerabilidad (Paso 8):** Se confirmó la versión de Sudo como **`1.9.17`** mediante `sudo -V`.
* **Exploit Seleccionado (Paso 8):** Se identificó un *exploit* específico en Exploit Database (EBD-ID **52354**, CVE-2025-32462) para la versión 1.9.17 de Sudo.
* **Explotación y Root (Paso 9):** Se creó y ejecutó un *script* basado en el PoC (`exploit.sh`). El *exploit* fue exitoso, otorgando una *shell* con privilegios de **`root`**.
* **Bandera Root:** Se obtuvo la bandera final leyendo el archivo `/root/root.txt`.

---

### Resumen de Vulnerabilidades Clave

1.  **Mala Configuración de Servicio (TFTP):** El servicio TFTP estaba mal configurado, permitiendo la descarga no autenticada de archivos sensibles (`ciscortr.cfg`) y el descubrimiento de credenciales.
2.  **Protocolo VPN (IKEv1 Aggressive Mode):** La configuración permitía el uso del Modo Agresivo sin protección, lo que expuso los parámetros de la PSK a ataques de *crackeo* fuera de línea.
3.  **Vulnerabilidad en Sudo:** La versión instalada (`1.9.17`) contenía una vulnerabilidad conocida (CVE-2025-32462) que permitió la escalada de privilegios de usuario a *root*.
