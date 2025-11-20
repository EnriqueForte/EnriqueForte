# 🏁 Conclusiones - Brute It (TryHackMe)

## Aprendizajes Clave

Durante la resolución del reto **Brute It**, pude reforzar y aplicar varias técnicas fundamentales de hacking ético:

- **Enumeración y Metodología:** Se demostró la importancia de validar la conectividad, escanear todos los puertos y buscar rutas ocultas antes de intentar explotar vulnerabilidades.
- **Fuerza bruta controlada:** Apliqué ataques de fuerza bruta tanto para descifrar formularios web como para claves protegidas, aprendiendo a identificar cadenas de error que aseguren que el ataque sea efectivo y eficiente.
- **Análisis de código fuente:** Revisar a fondo el HTML y los comentarios de páginas puede revelar información sensible, como usuarios predeterminados u otras pistas discretas.
- **Escalada de privilegios:** Aprovechar permisos sudo mal configurados sobre binarios aparentemente inofensivos (como `cat`) permite leer archivos críticos y obtener acceso root.
- **Crackeo de hashes:** El uso de herramientas como John the Ripper para descifrar contraseñas demuestra lo esencial que es almacenar los hashes de forma segura y con contraseñas robustas.

## Herramientas Utilizadas

- `ping`, `nmap`, `gobuster`, `hydra`, `john`, `ssh`, `sudo`
- Recursos de ayuda: [rockyou.txt], [GTFOBins], navegadores con opciones para inspección de elementos y código fuente.

## Superficie de Ataque y Defensa

- La presencia de archivos y rutas ocultas (`/admin`, claves RSA) refuerza la importancia de proteger recursos críticos y eliminar comentarios o credenciales del lado cliente.
- Los permisos de sudo sin contraseña sobre binarios como `cat` transforman una máquina estándar en un objetivo vulnerable a la escalada de privilegios.
- El uso de claves protegidas solo es efectivo si el passphrase es robusto; de lo contrario, puede ser rota fácilmente con diccionarios comunes.

## Consejos para próximos retos

- **Analiza siempre los permisos** de usuario y sudo tan pronto tengas acceso inicial.
- Utiliza diccionarios actualizados y revisa cuidadosamente los mensajes de error durante la fuerza bruta.
- Diversifica tu aproximación: las vulnerabilidades suelen ser combinaciones de pequeños errores en varias capas.
- Documenta cada paso y captura las salidas críticas para poder analizar o presentar el proceso luego.

## Reflexión final

Brute It es un laboratorio excelente para practicar técnicas esenciales de pentesting y CTF, desde la fase de reconocimiento y ataque de fuerza bruta, hasta la post-explotación y la obtención del control total de la máquina. Este ejercicio recalca la importancia de asegurar los servicios básicos y revisar la configuración de privilegios y credenciales en cualquier sistema.

---
