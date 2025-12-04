# Conclusiones – TryHackMe 0day 🧨

## 🧠 Aprendizajes principales

- Reforcé la **metodología de enumeración**: empezar por conectividad, seguir con Nmap y luego profundizar en servicios concretos (en este caso, HTTP).
- Practiqué la **búsqueda de vulnerabilidades históricas**, en concreto **Shellshock (CVE-2014-6271)**, entendiendo tanto la teoría como su explotación práctica.
- Aprendí a combinar **herramientas automáticas** (Nikto, Metasploit) con técnicas más manuales (`curl`, análisis de cabeceras HTTP).

---

## 🐚 Shellshock y servicios CGI

- Confirmé por qué los **scripts CGI** son un vector típico para Shellshock: usan Bash y aceptan variables de entorno controladas desde peticiones HTTP.
- Pude explotar la vulnerabilidad:
  - Primero con **`curl`** (RCE básica ejecutando comandos como `www-data`).
  - Después con **Metasploit**, obteniendo una **reverse shell estable** y una sesión `meterpreter`.

Esta parte refuerza la idea de que entender el **funcionamiento interno** de la vulnerabilidad es tan importante como lanzar el exploit.

---

## 🔐 Gestión de claves y contraseñas

- Localicé una **clave privada RSA** expuesta en el directorio `/backup/`.
- Practiqué el proceso completo:
  - Guardar y proteger la clave (`chmod 600`).
  - Convertirla con `ssh2john`.
  - Romper la passphrase con `john` + `rockyou.txt`.

Este escenario recuerda la importancia de:
- No dejar copias de claves privadas en rutas accesibles desde la web.
- No reutilizar contraseñas débiles en claves y cuentas de usuario.

---

## 🧗‍♂️ Escalada de privilegios en Linux

- Enumeré la versión de kernel (`3.13.0-32-generic`) y utilicé `searchsploit` para encontrar un **exploit local adecuado**.
- Apliqué el exploit de **OverlayFS (CVE-2015-1328)**:
  - Transferencia del exploit con un servidor HTTP simple.
  - Compilación con `gcc`.
  - Ejecución y obtención de **root**.

Este proceso refuerza el flujo recomendado de escalada:
1. Recopilar información del sistema (kernel, distribución, SUID, cron, etc.).
2. Buscar exploits específicos y **verificar la compatibilidad** antes de ejecutarlos.

---

## 🔍 Buenas prácticas y errores útiles

- Documentar también los **intentos fallidos** (como la ruta incorrecta `/cga-bin`) ayuda a entender el proceso real de un pentest y no solo el “camino perfecto”.
- Usar herramientas como `Nikto`, `gobuster`, `searchsploit` y `john` de forma combinada permite una visión más completa del sistema objetivo.

---

## ✅ Valor global de la máquina

0day es una máquina muy útil para:

- Practicar vulnerabilidades históricas que aún aparecen en entornos reales mal mantenidos.
- Consolidar una **metodología completa**: reconocimiento → enumeración → explotación → escalada → loot.
- Trabajar tanto con **herramientas automáticas** como con enfoques manuales.

En resumen, 0day es una excelente práctica para perfiles **principiantes e intermedios** en pentesting Linux, especialmente para entender el impacto de una mala gestión de configuraciones, claves y versiones de software desactualizadas. 💻🔓

