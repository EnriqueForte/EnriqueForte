# 🎉 Conclusiones y Resumen del CTF Overpass 🔐

## 📝 Resumen del Recorrido

El CTF **Overpass** de TryHackMe fue un excelente ejercicio para consolidar varias técnicas de ciberseguridad, demostrando cómo una cadena de fallos de seguridad (desde el desarrollo hasta la administración del sistema) puede llevar al compromiso total.

| Fase | Técnica Principal | Vulnerabilidad Explotada | Usuario Obtenido |
| :--- | :--- | :--- | :--- |
| **I. Acceso Inicial** | Enumeración Web y Consola JS | **Authentication Bypass** por Cookies. | `james` |
| **II. Pivote** | *Password Cracking* | Clave privada SSH cifrada con *passphrase* débil. | `james` |
| **III. Escalada** | Enumeración de Sistema (Crontab) | **Ejecución remota de código** como `root` a través de un Cron Job. | `root` |

## 🧠 Lecciones Aprendidas

1.  **No Confiar en la Seguridad del Cliente:** La primera etapa se basó en el fallo de la autenticación implementada en JavaScript, la cual fue fácilmente eludida manipulando una *cookie* de sesión.
2.  **Riesgos de Credenciales Débiles:** La clave privada SSH estaba cifrada con una *passphrase* tan simple (`james13`) que fue crackeada en segundos con un *wordlist* básico.
3.  **Configuraciones Críticas (Cron Jobs):** El fallo más grave fue la ejecución de un *script* descargado de un dominio externo (`overpass.thm`) como el usuario **`root`** cada minuto. Este es un error de configuración de sistemas de altísimo riesgo que permitió el control total de la máquina.

---
*¡Gracias por acompañarme en este recorrido! Espero que este write-up sirva como una guía útil.*
