# 🏁 Conclusiones - TryHackMe Billing

El laboratorio de "Billing" en TryHackMe ha sido una excelente oportunidad para poner en práctica técnicas fundamentales de pentesting, abarcando desde la enumeración de servicios hasta la explotación avanzada y escalada de privilegios. A continuación, se destacan los aprendizajes y observaciones más importantes de la sala.

---

## 📚 Lecciones Clave

- **Enumeración efectiva**: La correcta identificación de los servicios activos mediante Nmap y la enumeración de rutas y archivos sensibles con ffuf fueron cruciales para encontrar la superficie de ataque inicial.
- **Análisis de versiones**: Revisar los archivos disponibles, como README.md, permitió asegurarse de la versión del software y correlacionar vulnerabilidades públicas relevantes.
- **Investigación de vulnerabilidades**: La detección de la CVE-2023-30258 y la comprensión de su funcionamiento permitieron planificar y ejecutar la explotación con éxito.
- **Escalada de privilegios creativa**: La explotación de una configuración permitida de sudo sobre fail2ban-client, aun en escenarios restringidos, resaltó la importancia de revisar permisos y rutas de ataques post-explotación.

---

## 🚩 Puntos Críticos y Recomendaciones

- **Exposición de archivos sensibles**: Se identificaron archivos accesibles como README.md y scripts que podrían dar pistas para un atacante; se recomienda restringir el acceso a documentación interna y eliminar archivos innecesarios en producción.
- **Gestión de privilegios de sudo**: Permitir comandos administrativos como fail2ban-client sin password amplía la superficie de escalada. Es fundamental revisar y reducir estos permisos al mínimo indispensable.
- **Actualización de software**: MagnusBilling vulnerable fue clave para comprometer el sistema. Mantener los sistemas y aplicaciones al día es la mejor defensa ante exploits públicos.
- **Monitorización de logs y procesos privilegiados**: Un control más estricto sobre logs, procesado y segregación de funciones hubiera dificultado la explotación vertical.

---

## 💡 Reflexión final

Superar la sala no solo permitió afianzar conocimientos técnicos y habilidades de explotación, sino también interiorizar la importancia de una defensa en profundidad y la actualización continua de sistemas. CTFs de este tipo resultan ideales para adquirir práctica realista, aprender de los errores y documentar cada fase del análisis de un objetivo.

---

⭐ ¡Continúa entrenando, aprendiendo y documentando tus retos!
