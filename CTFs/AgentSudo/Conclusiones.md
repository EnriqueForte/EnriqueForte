# Conclusiones — Agent Sudo

Este documento resume las lecciones aprendidas, las mitigaciones recomendadas y las acciones sugeridas tras la resolución del CTF **Agent Sudo**.

---

## 1. Resumen rápido

Durante la resolución se siguieron los pasos clásicos de pentesting: reconocimiento, escaneo (nmap), enumeración web condicionada por `User‑Agent`, extracción de información (FTP, stegano en imágenes), obtención de credenciales, acceso por SSH con `james`, y escalado a root mediante un bypass de `sudo` (CVE‑2019‑14287). Se encontraron dos flags (user y root) como evidencia del escalado.

---

## 2. Lecciones técnicas

* **Validación y control de cabeceras (User‑Agent).** El servidor cambió su comportamiento según el `User‑Agent`. No confiar en cadenas del cliente para controlar acceso crítico: la seguridad no debe depender solo de un header fácilmente manipulable.

* **Exposición de ficheros en FTP.** El servicio FTP exponía ficheros útiles (backups, imágenes) que contenían datos sensibles. Evitar servicios de transferencia de archivos sin controles de acceso y registro.

* **Esteganografía y datos embebidos.** Las imágenes contenían artefactos útiles (ZIP embedido y archivo oculto con `steghide`). Nunca descartar recursos binarios en los que pueda esconderse información sensible.

* **Vulnerabilidad conocida en `sudo`.** La versión instalada (`sudo 1.8.21p2`) era susceptible al bypass numérico (CVE‑2019‑14287). Las reglas de `sudoers` que usan negaciones (`!root`) pueden producir escenarios explotables si el binario `sudo` es vulnerable.

---

## 3. Mitigaciones y recomendaciones inmediatas

1. **Actualizar `sudo`.** Parchear a una versión segura (>= 1.8.28 o la versión estable más reciente disponible para la distribución):

```bash
# en Debian/Ubuntu
sudo apt update && sudo apt install --only-upgrade sudo

# comprobar versión después:
sudo --version
```

2. **Revisar y endurecer `sudoers`.** Evitar reglas ambiguas y negaciones innecesarias. Preferir definiciones explícitas de comandos permitidos y limitar la lista de usuarios/hosts. Ejemplo: evitar `(ALL, !root) /bin/bash` y, en su lugar, usar comandos concretos con argumentos controlados.

3. **Eliminar servicios inseguros o restringir acceso.** Si FTP no es necesario, deshabilitarlo (`systemctl disable --now vsftpd`). Si es necesario, forzar SFTP/SSH con chroot y autenticación fuerte.

4. **Auditoría de ficheros expuestos.** Revisar directorios públicos, backups y ficheros subidos. Implementar detección de patrones (hashing, DLP) para identificar credenciales en texto plano.

5. **Registro y monitoreo.** Habilitar logging detallado para FTP/HTTP/SSH. Monitorizar intentos de subida/descarga y accesos con User‑Agent atípicos.

6. **Política de contraseñas.** El CTF mostró contraseñas débiles en archivos. Forzar contraseñas robustas y MFA donde sea posible.

7. **Hardening de servicios web.** No confiar en cabeceras para controlar acceso. Implementar autenticación real (tokens, sesiones), validación del lado servidor y controles de subida de ficheros (extensiones, escaneo, sandboxing).

---

## 4. Comandos útiles para comprobación y respuesta

* Comprobar versión de sudo:

```bash
sudo --version
```

* Listar reglas de sudo disponibles para un usuario:

```bash
sudo -l -U <usuario>
```

* Buscar ficheros con permisos SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

* Buscar ficheros modificados recientemente (ej. durante despliegue o subida):

```bash
find /var/www -type f -mtime -30
```

* Deshabilitar vsftpd (si no es necesario):

```bash
sudo systemctl stop vsftpd
sudo systemctl disable vsftpd
```

---

## 5. Buenas prácticas a largo plazo

* Mantener inventario de software y aplicar actualizaciones regulares (gestión de parches).
* Revisiones periódicas de `sudoers` y auditoría de permisos de ejecución.
* Minimizar la superficie de ataque: deshabilitar servicios innecesarios y limitar acceso por IP/ACL.
* Políticas de subida de archivos seguras (escaneo automático, limitación de tipos y tamaños).
* Formación para administradores y desarrolladores sobre peligros de confiar en inputs del cliente (headers, parámetros) para seguridad.

---

## 6. Conclusión

Agent Sudo es un buen ejercicio que combina técnicas de enumeración (web, FTP, stegano) con una vulnerabilidad clásica de escalado local. La conclusión principal: las micro‑mala configuraciones (ficheros públicos, reglas sudo confusas) y software sin parchear proporcionan vectores sencillos para comprometer un sistema. La defensa efectiva es actualización, control de acceso y auditoría continua.

---
