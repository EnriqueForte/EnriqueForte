## Conclusiones del Compromiso de la Máquina 🛡️

Este análisis de penetración, dividido en diez pasos, ilustra un flujo de ataque clásico en entornos con configuraciones inseguras de servicios y una vulnerabilidad local no parcheada.

### Resumen del Flujo de Ataque

El compromiso de la máquina se logró a través de un enfoque de tres etapas:

1.  **Reconocimiento Inicial y Vector de Acceso (FTP Anónimo):**
    * Se descubrió que el servicio FTP (puerto 21, VSFTPD 3.0.3) permitía el **acceso anónimo**.
    * Se identificó que el directorio `/scripts` era **escribible** (`Writable`).
    * El análisis de los archivos descargados (`to_do.txt`, `removed_files.log`, `clean.sh`) confirmó la existencia de un **Cron Job recurrente** ejecutando `clean.sh`.

2.  **Obtención de Acceso Inicial (Shell de Usuario):**
    * Se inyectó un *payload* de *reverse shell* (`/bin/sh`) en el script `clean.sh` utilizando el comando `append` de FTP para evitar la sobrescritura total del archivo.
    * La tarea cron ejecutó el *payload*, resultando en una *shell* de bajo privilegio como el usuario **`namelessone`**.
    * Se obtuvo la bandera de usuario (`user.txt`).

3.  **Escalada de Privilegios (Exploit de SUID en `env`):**
    * La enumeración de binarios SUID reveló que `/usr/bin/env` tenía el *bit* SUID activado.
    * Se utilizó el exploit SUID del binario `env` documentado en **GTFOBins** para ejecutar `/bin/sh -p`.
    * Esto escaló los privilegios a **`root`** y permitió la lectura de la bandera final (`root.txt`).

### Vulnerabilidades Críticas

Las siguientes configuraciones de seguridad fueron los puntos de fallo que permitieron el compromiso total:

| Vulnerabilidad | Impacto | Recomendación de Mitigación |
| :--- | :--- | :--- |
| **Acceso FTP Anónimo** | Permitió el acceso sin credenciales a un directorio clave que contenía un *script* de ejecución periódica. | **Deshabilitar** el acceso anónimo si no es estrictamente necesario. Restringir el directorio raíz del usuario FTP. |
| **Permisos de Escritura en Cron Script** | Permitía que un usuario no autenticado modificara un script ejecutado por el sistema. | **Eliminar** permisos de escritura al grupo y otros para directorios que contienen *scripts* de cron. |
| **Binario SUID Vulnerable** | El binario `/usr/bin/env` con el *bit* SUID activado permitió a un usuario de bajo privilegio obtener la *shell* de `root`. | **Eliminar** el *bit* SUID de binarios que no lo requieran o que sean conocidos vectores de escalada (revisar listas como GTFOBins). Mantener el sistema operativo **actualizado**. |

Este análisis subraya la importancia de una configuración de permisos estricta y de la gestión de parches para prevenir la escalada de privilegios a través de vectores conocidos.
