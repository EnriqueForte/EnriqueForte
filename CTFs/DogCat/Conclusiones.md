# Conclusiones del Writeup CTF Dogcat (TryHackMe)

## Resumen de la Ruta de Explotación

El CTF Dogcat presentó una ruta de explotación de tres etapas, desde la ejecución de código inicial hasta el escape final del contenedor.

| Fase | Pasos Clave Realizados | Vulnerabilidad Explotada | Usuario Obtenido |
| :--- | :--- | :--- | :--- |
| **I. Acceso Inicial** | **1. Reconocimiento:** Se identificó un parámetro `view` sospechoso en la URL (`/?view=dog`). | **LFI** (Local File Inclusion) | `www-data` |
| | **2. Análisis de Código:** Se descubrió el parámetro `ext` sin filtrar y el filtro de las cadenas 'dog'/'cat'. | **PHP Code Inclusion** (a través de LFI) | |
| | **3. RCE:** Se inyectó una *webshell* PHP en el *access log* de Apache a través del `User-Agent` (Envenenamiento de Logs). | **Envenenamiento de Logs** | |
| | **4. Shell Inversa:** Se transfirió y ejecutó una *reverse shell* PHP (tras fallar Bash/Netcat) para obtener una shell estable. | **PHP Reverse Shell** | |
| **II. Escalada de Privilegios** | **5. Enumeración:** Se encontró que el usuario `www-data` podía ejecutar `/usr/bin/env` como `root` sin contraseña. | **Sudo Misconfiguration** (GTFOBins) | `root` (en contenedor) |
| | **6. PrivEsc:** Se utilizó `sudo /usr/bin/env /bin/bash` para obtener *root* dentro del contenedor Docker. | **Vulnerabilidad de env** | |
| **III. Escape de Contenedor** | **7. Enumeración Root:** Se buscaron tareas *cron* o *scripts* sensibles, encontrando `/opt/backups/backup.sh`. | **Cron Job Misconfiguration** (Container Escape) | `root` (en host) |
| | **8. Escape Final:** Se inyectó un *payload* de *reverse shell* en `backup.sh` y se esperó a que la tarea *cron* del sistema *host* lo ejecutara, garantizando el control total. | **Inyección en Script** | |

---

## Análisis Final y Lecciones Aprendidas

El CTF Dogcat es una máquina excelente para entender el encadenamiento de vulnerabilidades de alto impacto, centrándose en configuraciones erróneas y lógica de aplicación.

### 1. La Importancia del Código Fuente y el Filtrado

El error más significativo del servidor fue la implementación de un filtro de lista negra ineficaz:

* Se utilizó `containsStr()` para asegurar que la URL contuviera **'dog'** o **'cat'**. Este filtro se eludió fácilmente con `dog/../../../.../etc/passwd`.
* La inclusión de un segundo parámetro (`ext`) que no tenía ninguna validación fue la clave para convertir la LFI en RCE, permitiendo anular la extensión `.php` y forzar la inclusión de archivos de log.
* **Lección:** La validación de entrada nunca debe depender solo de una lista negra (o *denylist*); siempre debe usarse una **lista blanca** (*allowlist*) de archivos permitidos y desinfectarse toda la entrada del usuario.

### 2. Contenedorización y Riesgos de Sudo

La escalada de privilegios dentro del contenedor fue sencilla una vez que se identificó la configuración errónea de `sudo` en `/usr/bin/env`.

* **Lección:** Las configuraciones de **sudoers** deben ser mínimas. Permitir que cualquier binario flexible como `env`, `less`, `awk` o `vi` se ejecute como `root` permite una escalada trivial a través de técnicas de GTFOBins.

### 3. El Peligro del Escape de Contenedores

La etapa final enfatiza que obtener *root* en un contenedor **no siempre significa** *root* en el *host*.

* El escape se logró explotando un error de configuración del sistema *host*: permitir que una tarea programada (`cron job`) se ejecutara como `root` y que esta, a su vez, operara sobre archivos dentro del contenedor (`backup.sh`).
* **Lección:** Los administradores de sistemas deben ser extremadamente cautelosos al configurar tareas *cron* que interactúan con volúmenes de contenedores, ya que una cuenta de bajo privilegio en el contenedor puede modificar un *script* que luego es ejecutado con privilegios máximos en el *host* subyacente.
