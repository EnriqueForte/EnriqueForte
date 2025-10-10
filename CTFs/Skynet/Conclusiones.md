# 🏁 Conclusiones del CTF: Skynet (TryHackMe)

## Resumen del Ataque 🎯

El CTF Skynet fue un ejercicio completo que puso a prueba una amplia gama de habilidades de pentesting, desde la enumeración pasiva hasta la escalada de privilegios a nivel de sistema.

El camino a la explotación se desarrolló en las siguientes fases:

1.  **Enumeración de SMB:** Se obtuvo una lista de contraseñas de forma anónima (`log1.txt`).
2.  **Webmail y Pivote:** Se forzó la entrada a **SquirrelMail** usando una contraseña del *log* con el usuario `milesdyson`, obteniendo una **nueva contraseña de Samba** (del correo de "Password Reset").
3.  **Descubrimiento de Endpoints:** Se usó la nueva credencial de Samba para acceder al recurso compartido de `milesdyson`, encontrando un **endpoint web oculto** (`/45kra24zxs28v3yd`).
4.  **Initial Foothold (LFI/RFI):** Se identificó y explotó una vulnerabilidad de **LFI/RFI** en **Cuppa CMS** para leer archivos de configuración sensibles (obteniendo credenciales de la DB) y, finalmente, lograr una **shell inversa** como el usuario `www-data`.
5.  **Escalada de Privilegios:** Se identificó y explotó un **Cron Job** que se ejecutaba como `root` con un comando `tar` vulnerable a la inyección de comodines (**Wildcard Injection**), obteniendo así la *shell* de `root`.

---

## Lecciones Aprendidas 🧠

Este CTF ofrece varias lecciones importantes en seguridad ofensiva:

| Área de Seguridad | Lección Aprendida |
| :--- | :--- |
| **Enumeración (SMB)** | Nunca subestimes los recursos compartidos anónimos; a menudo contienen *wordlists* o claves para el siguiente paso. |
| **Servicios Web** | Las rutas sensibles deben ser aleatorias y protegidas. Usar un CMS desactualizado (**Cuppa CMS**) con una vulnerabilidad LFI/RFI conocida es una falla crítica de configuración. |
| **Reutilización de Credenciales** | La contraseña débil del webmail condujo a una contraseña fuerte del servicio Samba. La **información sensible** nunca debe enviarse por correo electrónico sin cifrar. |
| **Escalada de Privilegios** | Los **trabajos cron** y los *scripts* de *backup* que se ejecutan como `root` son un vector común. El uso de comodines (`*`) o comandos sin rutas absolutas (`/bin/tar`) en *scripts* de `root` puede ser explotado mediante **Path Hijacking** o **Wildcard Injection**. |
| **Higiene de Scripting** | Los *scripts* que se ejecutan como `root` nunca deben usar el comodín `*` en directorios modificables por usuarios con menos privilegios. |

---

## 🏆 Banderas Capturadas

| **Flag** | **Ubicación Final** |
| :---: | :--- |
| **user.txt** | `/home/milesdyson/user.txt` |
| **root.txt** | `/root/root.txt` |

*¡Ha sido un placer documentar este recorrido por Skynet!*
