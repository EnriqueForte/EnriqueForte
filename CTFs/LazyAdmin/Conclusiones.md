# ✅ Conclusiones del CTF: Lazy Admin (TryHackMe)


## 🎯 Objetivo General
La máquina Lazy Admin (clasificada como fácil) fue resuelta con éxito mediante una cadena de explotación que incluye enumeración exhaustiva de directorios, compromiso de credenciales y un vector de escalada de privilegios simple basado en permisos incorrectos de sudo.

## ⚙️ Metodología Aplicada

Fase	Técnica Clave	Resultado

I. Reconocimiento	Escaneo Nmap y Ping	Puertos abiertos: 22 (SSH) y 80 (HTTP).

II. Enumeración Web	Fuzzing con Gobuster	Descubrimiento del CMS SweetRice v1.5.1 en /content/ y la ruta de administración /content/as/.

III. Acceso Inicial (Foothold)	Dirb + Descarga de Backup SQL	Acceso a un backup de la base de datos MySQL en /content/inc/mysql_backup/, lo que nos proporcionó credenciales válidas (admin:admin).

IV. Ejecución Remota (RCE)	File Upload Bypass	Se bypassó el filtro de extensiones de archivos del CMS (que bloqueaba .php) subiendo un reverse shell con la extensión .phtml. Se obtuvo una shell inicial como usuario www-data.

V. Escalada de Privilegios (PrivEsc)	Abuso de Permisos sudo	Se encontró que el usuario www-data podía ejecutar el script de Perl /home/itguy/backup.pl sin contraseña. Este script ejecutaba un archivo modificable (/etc/copy.sh), lo que permitió la inyección de código y la obtención de una shell de root.


## 💡 Lecciones de Seguridad

Este CTF destaca varios errores comunes de configuración que los administradores "perezosos" suelen cometer, de ahí el nombre de la sala:

Credenciales Hardcodeadas/Poco Seguras:

El uso de contraseñas por defecto o fáciles de descifrar (como admin:admin) es un fallo grave.

Mitigación: Usar contraseñas fuertes y únicas, y evitar hashes MD5 sin salt.

Archivos de Configuración y Backup Expuestos:

Permitir que el archivo de backup de la base de datos (.sql) sea accesible públicamente a través del navegador (especialmente en un directorio común como /inc/) es un riesgo de seguridad catastrófico.

Mitigación: Asegurar que los directorios sensibles y los archivos de backup se almacenen fuera del directorio raíz web (/var/www/html/) o estén protegidos con control de acceso estricto (.htaccess o reglas del servidor).

Filtrado de Subida de Archivos Insuficiente:

El CMS solo filtraba la extensión principal (.php) pero fallaba en bloquear extensiones alternativas como .phtml o .php5.

Mitigación: Implementar una política de lista blanca (solo permitir extensiones conocidas y seguras como .jpg, .png, .pdf) en lugar de una lista negra (bloquear extensiones peligrosas).

Permisos de Sudo Mal Configurados (Elevación de Privilegios):

Permitir que un usuario de bajo privilegio (www-data) ejecute un script con permisos de root sin contraseña, donde el script llama a otro archivo (/etc/copy.sh) que es escribible por el mismo usuario, es una vulnerabilidad de escalada trivial.

Mitigación: Seguir el Principio de Mínimo Privilegio. El usuario www-data no debería tener permisos sudo y, si los tiene, el binario o script ejecutado debe ser de solo lectura para evitar inyección de código.

**Este write-up fue creado con fines educativos para documentar la resolución del Capture The Flag (CTF) Lazy Admin de TryHackMe.**
