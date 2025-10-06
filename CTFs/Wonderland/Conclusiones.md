# 👑 Conclusiones y Resumen del CTF Wonderland 🐇

## 📜 Resumen del Recorrido

El CTF Wonderland de TryHackMe fue un ejercicio excelente que cubrió una amplia gama de habilidades, desde la enumeración web hasta la escalada de privilegios en Linux.

### Acceso Inicial (Alice) 🔎

Vector: Enumeración Web Profunda y Esteganografía.

Vulnerabilidad Explotada: Pista de la ruta del directorio (/r/a/b/b/i/t/) y credenciales (alice:H0tt3rHasD1gitalK3y) ocultas en el código fuente de la página final.

### Pivote (Alice a Rabbit) ➡️

Vector: Escalada Horizontal con sudo.

Vulnerabilidad Explotada: El usuario alice tenía permisos para ejecutar un script de Python (walrus_and_the_carpenter.py) con los privilegios del usuario rabbit a través de sudo -u, permitiendo la inyección de un shell.

### Escalada Final (Rabbit a Hatter y Root) 🔑

Vector: Secuestro de PATH y Binario SUID.

Vulnerabilidad Explotada: El binario SUID teaParty (propiedad de root) llamaba al comando date sin una ruta absoluta. Se explotó esta vulnerabilidad con un Path Hijacking para obtener un shell como hatter.

### Root: La clave final de root se encontró en el archivo /home/hatter/password.txt.


## 💡 Lecciones Aprendidas

🔍 No Confiar en lo Visible: La clave para el acceso inicial fue ser "curioso". Se demostró la necesidad de revisar el código fuente (Paso 8) y utilizar herramientas como Steghide para la esteganografía (Paso 5) en elementos temáticos como la imagen del conejo.

🛡️ Riesgos de Sudoers Inseguros: La escalada de alice a rabbit ilustró cómo una configuración permisiva en sudo permite que un usuario de bajo privilegio ejecute scripts como otro usuario, facilitando la inyección de shells con privilegios.

🛣️ Maestría en Path Hijacking (SUID): La escalada de rabbit a hatter fue un ejercicio práctico de secuestro de rutas. Al identificar el binario SUID teaParty y descubrir que llamaba comandos (date) sin ruta absoluta, se pudo manipular la variable $PATH para ejecutar un shell malicioso con los permisos de root.

⚡ Enumeración Avanzada (getcap): Se exploró la importancia de buscar Linux Capabilities como vector de escalada alternativo, identificando que el binario perl podía ser utilizado para obtener un shell de `root* (Paso 15).
