# ✅ Conclusiones – Injectics (TryHackMe)

Durante la resolución de este CTF se han puesto en práctica diferentes técnicas de **enumeración, explotación y escalada de privilegios**, lo cual permitió obtener las flags planteadas.

---

## 🔎 Puntos clave del aprendizaje

1. **Enumeración inicial**
   - El escaneo con `nmap` reveló puertos críticos: **22 (SSH)** y **80 (HTTP)**.
   - Con `gobuster` descubrimos rutas sensibles (`/login.php`, `/flags/`, `/phpmyadmin/`, `composer.json`,`mail.log`).

2. **Exposición de información sensible**
   - El archivo `composer.json` reveló la versión de **Twig (2.14.0)**, vulnerable a **CVE-2024-45411**.
   - El archivo `mail.log` contenía **credenciales por defecto**, lo que permitió acceder con usuario `superadmin`.

3. **Explotación de vulnerabilidades**
   - El login principal aplicaba filtros en **JavaScript**, fácilmente saltados con **Burp Suite**, confirmando SQLi en backend.
   - Con el rol `dev` en el dashboard, fue posible inyectar SQL en la edición de datos.
   - Con el rol `superadmin`, se accedió a la sección **Profile**, vulnerable a **SSTI en Twig**, obteniendo ejecución remota de comandos.

4. **Obtención de flags**
   - Flag 1: obtenida al acceder con la cuenta de **superadmin**.
   - Flag 2: localizada con un `find` desde la RCE y leída mediante `cat`.

---

## 📚 Lecciones aprendidas

- Los **filtros en cliente (JavaScript)** no son una medida de seguridad válida: siempre deben implementarse validaciones en el backend.
- La exposición de archivos sensibles (`composer.json`, `mail.log`) otorga información crítica a un atacante.
- El uso de dependencias desactualizadas (**Twig 2.14.0**) introduce vulnerabilidades graves que permiten desde **SSTI** hasta **RCE**.
- La implementación de un servicio de restauración automática de la base de datos (InjecticsService) fomenta la práctica segura del atacante, pero en un entorno real sería una puerta abierta a abusos.
- La combinación de **SQLi + SSTI** demuestra lo devastador que resulta un sistema sin controles de seguridad adecuados.

---

## 🛡️ Medidas de mitigación recomendadas

1. **Actualizar dependencias**: mantener Twig y phpMyAdmin en sus últimas versiones.
2. **Eliminar archivos sensibles** (`composer.json`, logs accesibles desde web).
3. **Implementar validación de entrada y consultas preparadas** para evitar SQL Injection.
4. **Aplicar WAF o filtrado en servidor**, no solo en el cliente.
5. **Restringir phpMyAdmin** a direcciones internas o autenticación fuerte.

---

🎯 **Conclusión general**:  
Este CTF demuestra cómo una mala configuración, sumada a la falta de controles de seguridad en backend y dependencias obsoletas, permite a un atacante escalar desde un simple escaneo inicial hasta obtener control total del sistema.
