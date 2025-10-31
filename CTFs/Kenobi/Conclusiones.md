# 📝 Conclusiones y Puntos Clave Aprendidos

---

## 💡 Resumen de Técnicas de Explotación

La sala **Kenobi** es un excelente ejemplo de cómo la enumeración exhaustiva y la combinación de vulnerabilidades pueden llevar al *root*. Las principales técnicas utilizadas en este *writeup* fueron:

| Fase | Técnica / Servicio | Descubrimiento Clave |
| :--- | :--- | :--- |
| **Enumeración Inicial** | Samba (SMB) en Puerto 445 | Existencia de la compartición `anonymous` con acceso de **sólo lectura**. |
| **Análisis de Información** | Contenido del `log.txt` | Ubicación de la clave privada SSH del usuario `kenobi`: `/home/kenobi/.ssh/id_rsa`. |
| **Vector de Acceso 1** | ProFTPD 1.3.5 | Vulnerabilidad **`mod_copy`** que permite mover archivos arbitrarios dentro del servidor. |
| **Vector de Acceso 2** | NFS en Puerto 111 | El directorio `/var` está **compartido** y puede ser montado localmente. |
| **Acceso al Usuario** | Explotación Combinada | Se utilizó `mod_copy` para mover `/home/kenobi/.ssh/id_rsa` a la ruta compartida por NFS (`/var/tmp/`). Luego, montamos NFS para extraer la clave y acceder vía SSH. |
| **Escalada de Privilegios** | Binario SUID `/usr/bin/menu` | El binario SUID ejecuta comandos del sistema (`ifconfig`) sin rutas absolutas, permitiendo la **manipulación de la variable `PATH`**. |

---

## 🧠 Lecciones Aprendidas

1.  **Enumeración de Servicios Múltiples:** No te centres solo en HTTP. Samba y NFS a menudo contienen configuraciones sensibles o permiten acceso no deseado.
2.  **El Poder del `mod_copy`:** La vulnerabilidad `mod_copy` es una excelente herramienta para mover archivos confidenciales a ubicaciones que son más fáciles de explotar (como un recurso compartido NFS o un servidor web).
3.  **Vulnerabilidad de `PATH` en Binarios SUID:** Cuando un binario SUID ejecuta comandos del sistema sin usar la ruta absoluta (`/bin/comando`), podemos engañarlo creando nuestro propio comando en un directorio y anteponiendo ese directorio a la variable `PATH`. Esto se usa para ejecutar una *shell* con privilegios elevados.

**Recomendación de Seguridad:**
Para prevenir la escalada de privilegios por manipulación de `PATH`, los desarrolladores deben siempre usar la **ruta absoluta** para ejecutar comandos dentro de *scripts* o binarios con privilegios elevados (e.g., usar `/sbin/ifconfig` en lugar de `ifconfig`).
