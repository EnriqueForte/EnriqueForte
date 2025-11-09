# 🏁 Conclusiones — UltraTech (TryHackMe)

---

## 📝 Resumen ejecutivo
En este laboratorio hemos realizado una prueba completa de pentesting sobre la máquina *UltraTech*. El flujo principal fue:

1. Comprobación de conectividad y descubrimiento de servicios (ping, nmap).  
2. Enumeración web y de APIs (dirb/gobuster + revisión manual de frontend).  
3. Identificación y explotación de una **vulnerabilidad de Command Injection** en `/:8081/ping?ip=...`.  
4. Exfiltración de la base de datos SQLite (`utech.db.sqlite`) vía servidor HTTP lanzado desde el objetivo.  
5. Análisis local de la base de datos: extracción de usuarios y hashes, crackeo de hashes.  
6. Ataques de fuerza bruta dirigidos (Hydra) contra servicios FTP/SSH y obtención de credenciales válidas.  
7. Acceso inicial al sistema con usuario obtenido; descubrimiento de pertenencia al grupo `docker`.  
8. Escalada a root aprovechando acceso al daemon Docker (montar `/` y `chroot` / ejecutar contenedor privilegiado).  
9. Recolección de evidencias finales (`/root/private.txt`, presencia de `id_rsa`, etc.).

---

## 🔎 Vulnerabilidades explotadas (resumen)
- **Command Injection (alta severidad)** — Endpoint `/ping` concatenaba directamente el parámetro `ip` en un comando del sistema, permitiendo ejecución arbitraria.  
- **Almacenamiento inseguro de credenciales (media/alta)** — Hashes MD5 en base de datos sin salt, crackeables con ataques rápidos.  
- **Exposición de servicios innecesarios / configuraciones por defecto (media)** — FTP/SSH accesibles con credenciales recuperadas y nginx/apache + Node expuestos sin hardening.  
- **Mala segregación de privilegios en Docker (alta)** — Usuario del sistema perteneciente al grupo `docker`, lo que permitió escalar a root mediante control del daemon.

---

## ✅ Qué aprendimos / buenas prácticas demostradas
- Revisar el **frontend** puede revelar endpoints de API y parámetros críticos (JS público muchas veces “filtra” la API).  
- Las APIs que ejecutan utilidades del sistema con parámetros del usuario deben **sanitizar y validar** estrictamente la entrada.  
- **No usar MD5** sin salt para almacenar contraseñas—usar algoritmos de hashing adaptativos (bcrypt, Argon2) y sal.  
- Pertenecer al grupo `docker` es prácticamente equivalente a tener un escalado total si el socket de Docker es accesible: **evitar** añadir usuarios no-trust al grupo `docker`.  
- Exfiltración de ficheros binarios mediante un servidor http temporal (o `base64`) es una técnica práctica en entornos CTF para recuperar artefactos completos.

---

## 🛠 Herramientas utilizadas
- Recon / enumeración: `ping`, `nmap`, `dirb`, `gobuster`, `curl`, navegador (inspección fuente).  
- Testing & exploitation: Burp Suite (intercept), payloads de command injection, `curl` para pruebas.  
- Exfiltración: `python3 -m http.server`, `wget`/`curl`.  
- Análisis local: `sqlite3`, DB Browser for SQLite, servicios de cracking / John / Hashcat (según necesidad).  
- Brute force: `hydra`.  
- Post-explotación: Docker CLI (`docker run`, `docker ps`, `docker exec`), chroot, comandos estándar Linux.

---

## 🔧 Recomendaciones de mitigación (priorizadas)
1. **Corregir Command Injection**  
   - Nunca interpolar directamente parámetros de usuario en comandos del sistema.  
   - Usar llamadas seguras (APIs nativas) o escapado riguroso, y preferiblemente eliminar la ejecución de comandos del sistema desde la web.  
   - Añadir validaciones whitelisting para valores permitidos y límites (ej. validar IP con regex y no permitir otros caracteres).

2. **Mejorar almacenamiento de credenciales**  
   - Sustituir MD5 por algoritmos modernos: **bcrypt**, **scrypt** o **Argon2** con salt único por usuario.  
   - Implementar políticas de contraseñas robustas y verificación de fuerza al crear cuentas.

3. **Hardenizar servicios expuestos**  
   - Deshabilitar FTP si no es necesario o forzar FTPS/SFTP.  
   - Restringir acceso SSH, usar autenticación por clave y deshabilitar login por contraseñas para cuentas críticas.  
   - Limitar información revelada por `robots.txt` o sitemaps en entornos productivos.

4. **Segregar y proteger Docker**  
   - Evitar añadir usuarios a `docker` si no es imprescindible.  
   - No exponer el socket `/var/run/docker.sock` a usuarios no confiables.  
   - Considerar el uso de herramientas que implementen control de acceso (RBAC) para operaciones Docker y políticas de seguridad (AppArmor/SELinux).  
   - Revisar contenedores para no ejecutarlos con `--privileged` ni montar `/` del host.

5. **Monitorización y alertas**  
   - Registrar y monitorizar operaciones sensibles (ej. ejecución de comandos, creación de servidores, accesos a /root).  
   - Alertas en caso de subida de nuevos binarios o arranque de servidor HTTP desde procesos web.

6. **Pruebas continuas**  
   - Añadir pruebas de seguridad en CI para endpoints sensibles (fuzzing, análisis estático dinámico).  
   - Auditar dependencias (npm, paquetes OS) y aplicar actualizaciones regulares.

---

## 📚 Pasos siguientes / recomendaciones para el equipo
- Remediar las vulnerabilidades listadas en entorno de staging y volver a ejecutar un pentest de verificación.  
- Rotar credenciales expuestas (si aplica) y revisar usuarios con privilegios innecesarios.  
- Implementar políticas de seguridad en despliegue de Docker y revisar imágenes/volúmenes.  
- Formar al equipo de desarrollo sobre sanitización de entradas y gestión segura de secretos.

---

## 🔐 nota final sobre evidencias
En el repositorio público se ha omitido información sensible (contraseñas en texto claro, claves privadas completas).

---

