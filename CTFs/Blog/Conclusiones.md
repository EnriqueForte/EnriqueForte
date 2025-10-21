# 📝 Conclusiones del Write-up CTF "Blog" - TryHackMe

## Resumen de la Metodología

El CTF "Blog" de TryHackMe fue una excelente caja que combinó la enumeración de servicios comunes (Web y SMB), la explotación de una vulnerabilidad de software conocido (WordPress) y una técnica de escalada de privilegios basada en una configuración incorrecta de variables de entorno.

La cadena de explotación se desarrolló en tres fases principales: **Reconocimiento**, **Acceso Inicial** y **Escalada de Privilegios**.

---

## Cadena de Explotación Detallada (Kill Chain)

### 1. Reconocimiento (Nmap, WPSCAN, Gobuster)

* **Identificación de Servicios:** Se descubrieron los puertos **22 (SSH)**, **80 (HTTP)** y **139/445 (SMB/Samba)**.
* **Identificación de Software:** El puerto 80 corría un sitio de **WordPress 5.0**, una versión obsoleta.
* **Recolección de Pistas:** El post del blog "A Note From Mom" reveló el posible nombre de usuario **`Karen Wheeler`** y una pista sobre el dueño de la máquina, **`Billy Joel`**.

### 2. Acceso Inicial (Fuerza Bruta y RCE)

* **Obtención de Credenciales:** Utilizando `wpscan` para enumerar usuarios y realizar fuerza bruta, se encontraron los usuarios **`kwheel`** y **`bjoel`**. La credencial obtenida fue **`kwheel:cutiepie1`**.
* **Explotación:** Se utilizó el módulo de Metasploit **`exploit/multi/http/wp_crop_rce`** (CVE-2019-8943), que explota una vulnerabilidad RCE/LFI en la función de recorte de imágenes de WordPress 5.0.
* **Resultado:** Se obtuvo acceso inicial a la máquina como el usuario de bajo privilegio **`www-data`**.

### 3. Escalada de Privilegios (Variable de Entorno)

* **Enumeración Interna:** La primera `user.txt` fue un señuelo. La enumeración reveló un binario inusual, `/usr/sbin/checker`.
* **Análisis del Binario:** El análisis con Ghidra demostró que el binario ejecutaba una *shell* de root (`/bin/bash` después de `setuid(0)`) **si y solo si** la variable de entorno **`admin`** estaba definida.
* **Escalada:** Se estableció la variable de entorno (`export admin=valor`) y se ejecutó el binario, lo que resultó en una **shell de `root`**.

---

## Lecciones Aprendidas

1.  **Parcheo de Software:** La clave del acceso inicial fue la ejecución de **WordPress 5.0**. Mantener el software actualizado es fundamental para prevenir la explotación de vulnerabilidades conocidas (CVEs).
2.  **Configuración de Servicios:** La escalada de privilegios se basó completamente en una **configuración errónea** del binario `checker`, que confió en una variable de entorno para otorgar privilegios de root. Los desarrolladores deben evitar el uso de `setuid(0)` basado en datos controlados por el usuario o variables de entorno.
3.  **Búsqueda exhaustiva de Flags:** La *flag* del usuario (`user.txt`) estaba oculta en un punto de montaje no estándar (`/media/usb/`), lo que subraya la importancia de usar comandos de búsqueda potentes (`find`) una vez que se obtienen privilegios elevados.
