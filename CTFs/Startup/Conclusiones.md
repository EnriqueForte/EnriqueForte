## Conclusiones del CTF: Startup (TryHackMe)

Esta máquina virtual de CTF es un excelente ejemplo de un flujo de ataque clásico que combina una configuración inicial deficiente con fallos en la administración de permisos, culminando en dos vectores de escalada de privilegios distintos.

### 1. Resumen del Flujo de Ataque

| Fase | Vulnerabilidad Explotada | Usuario Obtenido | Flag Obtenida |
| :--- | :--- | :--- | :--- |
| **Acceso Inicial** | FTP anónimo con directorio escribible, usado para cargar una *reverse shell* PHP. | `www-data` | **Ninguna** |
| **Escalada Usuario** | Archivo PCAPNG (`suspicious.pcapng`) expuesto, revelando una contraseña en texto claro. | `lennie` | `user.txt` |
| **Escalada Root** | Falla de permisos en un *cronjob*: El script `/etc/print.sh` es ejecutado por `root` pero es escribible por `lennie`. | `root` | `root.txt` |

### 2. Puntos Clave de Seguridad (Lecciones Aprendidas)

Las siguientes vulnerabilidades fueron críticas para el compromiso total de la máquina:

#### A. Configuraciones Iniciales Deficientes (www-data)
* **FTP Anónimo Escribible:** Permitir el acceso anónimo al FTP es un riesgo, pero permitir la escritura en un directorio que es públicamente accesible por el servidor web (`/var/www/html/files/ftp`) es una configuración de riesgo crítico. Esto permitió la carga y ejecución de la *reverse shell* de PHP.

#### B. Exposición de Información Sensible (Escalada a Usuario)
* **Gestión de Logs:** El archivo `suspicious.pcapng` se dejó expuesto en el directorio `/incidents`, el cual reveló una contraseña en texto claro utilizada en un intento previo de `sudo`. La exposición de credenciales en *dumps* de tráfico es una falla de seguridad mayor.

#### C. Mala Administración de Permisos (Escalada a Root)
* **Vulnerabilidad de Cronjob:** El vector de escalada de `lennie` a `root` se basó en la asignación incorrecta de permisos para el script `/etc/print.sh`. Un script en el directorio `/etc` **nunca** debe ser escribible por un usuario estándar.
* **Principio del Mínimo Privilegio:** Al poder modificar un script que es ejecutado por `root` (a través del *cronjob*), se violó el principio de mínimo privilegio, lo que permitió la creación de un *binary* SUID y el acceso completo de `root`.

En resumen, la máquina fue comprometida debido a una cadena de configuraciones laxas que proporcionaron un acceso inicial fácil y la exposición de credenciales, culminando en un error crítico de permisos a nivel de sistema.
