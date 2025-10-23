# 🎉 Conclusiones y Lecciones Aprendidas (Smol - TryHackMe)

La sala **Smol** fue una máquina de dificultad media que ofreció un recorrido exhaustivo a través de múltiples vulnerabilidades, destacando la importancia de la enumeración minuciosa y la reutilización de la información obtenida.

## 📝 Resumen de la Cadena de Explotación

1.  **Reconocimiento:** Se identificó un servidor web (Apache) con **WordPress** en el puerto 80.
2.  **Fuga de Información (LFI/SSRF):** Se explotó una vulnerabilidad de **LFI/SSRF no autenticada** en el *plugin* `jsmol2wp` para leer el archivo `wp-config.php`.
3.  **Acceso Inicial:** Se obtuvieron las credenciales de la base de datos (`wpuser`) y se reutilizaron para acceder al panel de administración de WordPress.
4.  **Ejecución de Código Remoto (RCE):** La enumeración interna del CMS reveló un indicio de una **backdoor ofuscada** en el *plugin* `Hello Dolly`. Se utilizó LFI para leer el código, decodificar el *webshell* y obtener RCE como `www-data`.
5.  **Escalada Horizontal (diego):** Se extrajeron los *hashes* de las contraseñas de la base de datos de WordPress y se crackeó la contraseña del usuario `diego`.
6.  **Escalada Horizontal (think y gege):** Se encontraron claves SSH privadas de `think` a las que `diego` tenía acceso. Se escaló a `think` vía SSH y luego a `gege` (al parecer, sin contraseña).
7.  **Escalada a Root (xavi):** El directorio *home* de `gege` contenía un archivo `.zip` cifrado. Tras crackear su contraseña, se encontraron credenciales de un usuario de alto nivel, **`xavi`**.
8.  **Obtención de Root:** El usuario `xavi` tenía permisos de **`sudo` sin restricción (`(ALL : ALL) ALL`)**, lo que permitió ejecutar `sudo su -` y obtener acceso de *root* para leer el *root flag*.

## 💡 Lecciones Clave Aprendidas

### 1. Peligros de la Reutilización de Credenciales
La máquina ilustra perfectamente el riesgo de la **reutilización de credenciales**. Las credenciales de la base de datos de WordPress se reutilizaron para el acceso inicial al panel y, más tarde, se descubrieron *hashes* que revelaron contraseñas (como la de `diego`) usadas para el acceso al sistema operativo.

### 2. Importancia de Parchear Plugins
La intrusión inicial se basó completamente en una vulnerabilidad de un *plugin* obsoleto. Las organizaciones deben auditar y actualizar regularmente todos los componentes de su CMS para prevenir accesos no autenticados (LFI/SSRF/XSS).

### 3. Enumeración Recursiva y Contextual
El camino hacia el RCE no fue obvio (no se trataba de subir un *shell*). Requirió:
* Acceder al panel.
* Encontrar una pista en un post de WordPress (`Webmaster Tasks`).
* Usar la vulnerabilidad de LFI/SSRF para leer un archivo diferente (`hello.php`).

Esta secuencia subraya que la enumeración debe ser constante y guiada por cualquier información que se encuentre.

### 4. Errores de Configuración (Permisos y Contraseñas)
El fallo de seguridad final fue un error de configuración simple pero crítico:
* **Permisos de Sudo:** El usuario `xavi` tenía permisos de `sudo` ilimitados, lo que equivale a tener acceso de `root` directo.
* **Contraseñas Débiles/Fugas:** El *hashing* de contraseñas de WordPress es débil y el archivo ZIP cifrado con una contraseña simple (crackeable con `rockyou`) permitió el acceso a credenciales de alto valor.
