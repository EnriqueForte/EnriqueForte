# 🎯 Conclusiones y Lecciones Aprendidas (Cyborg - TryHackMe)

## Resumen del Reporte 🤖

El CTF Cyborg es un excelente ejercicio que se centró en la exposición de información y la explotación de servicios no convencionales y malas configuraciones del sistema. 

La ruta de acceso combinó la enumeración web, la obtención de credenciales a través de hashes débiles y, finalmente, la explotación de permisos de sudo incorrectos para la escalada de privilegios.

**Hallazgos Críticos**

La fase de Enumeración Web fue crítica, ya que nos dirigió al directorio sensible /etc y reveló mensajes internos que mencionaban el share music_archive. 

La exposición del directorio /etc nos permitió acceder directamente a /etc/squid/passwd, donde se encontró el hash de music_archive.

Este hash fue crackeado con éxito, revelando la Passphrase de Borg, que fue utilizada para desencriptar el repositorio de backup descargado. 

Dentro de los archivos de backup de Alex, encontramos un archivo note.txt que contenía la clave del sistema del usuario alex. Al utilizarla, obtuvimos acceso al sistema vía SSH y aseguramos la flag de usuario.

Finalmente, la Escalada de Privilegios fue posible al descubrir que el usuario alex podía ejecutar el script /etc/mp3backups/backup.sh como root sin contraseña (NOPASSWD). 

Al inyectar un comando de reverse shell en el script y ejecutarlo con sudo, obtuvimos una shell de root y la flag final.

## Lecciones Aprendidas y Recomendaciones 🛡️

Esta máquina destaca fallos de seguridad comunes que pueden llevar a la escalada de privilegios si se exponen datos sensibles y se descuidan los permisos de ejecución.

*1. Control de Acceso a Archivos y Directorios*
Riesgo: La exposición del directorio /etc permitió el acceso directo a archivos de configuración, un fallo de seguridad fundamental.

Recomendación: Implementar una configuración estricta en el servidor web que deniegue por defecto el acceso a directorios del sistema y que asegure que ningún archivo sensible esté fuera del web root.

*2. Seguridad de Credenciales y Hashes*
Riesgo: El uso de un hash Apache MD5 (APR1) para la Passphrase de Borg es débil y se crackeó con rapidez. Además, se encontraron las credenciales de Alex en un archivo de texto simple en su escritorio.

Recomendación: Emplear algoritmos de hashing modernos y lentos (como bcrypt o Argon2). Nunca almacenar contraseñas o passphrases en archivos de texto plano o fácilmente accesibles, incluso en directorios de backup.

*3. Principio del Mínimo Privilegio (Sudo)*
Riesgo: Permitir a un usuario de bajo privilegio (alex) ejecutar un script como root sin contraseña (NOPASSWD) constituye una vulnerabilidad de escalada trivial.

Recomendación: Limitar estrictamente los permisos de sudo. Si un script debe ejecutarse como root, debe ser propiedad de root y no debe ser modificable por el usuario que lo ejecuta. 

El script también debe ser revisado para evitar la inyección de comandos a través de argumentos o variables de entorno.
