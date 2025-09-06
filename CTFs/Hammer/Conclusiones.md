# 🏁 Conclusiones – CTF Hammer

## 📋 Resumen de la explotación
Durante la resolución del CTF se identificaron y explotaron varias vulnerabilidades encadenadas que llevaron al control completo del sistema y a la obtención de las flags:

1. **Enumeración inicial** → descubrimiento de puertos (22 y 1337).  
2. **Exploración web** → formulario de login y código fuente con pistas (`hmr_`).  
3. **Fuzzing** → hallazgo del directorio `/hmr_logs/` con usuarios y rutas sensibles.  
4. **Reset de contraseña inseguro** → explotación de la función con un código PIN de 4 dígitos.  
5. **Exposición de JWT en el cliente** → descubrimiento de `kid` apuntando a `/var/www/mykey.key`.  
6. **Compromiso de la clave secreta** (`188ade1.key`) → generación de un JWT con rol `admin`.  
7. **Command Injection** en `execute_command.php` → ejecución de comandos arbitrarios y acceso a flags.

---

## ⚠️ Vulnerabilidades detectadas
- **Directory Traversal / Directory Listing** → directorios accesibles públicamente (`/hmr_logs`).  
- **Information Disclosure** → logs expuestos con usuarios y rutas internas.  
- **Weak Password Reset** → uso de un PIN corto, vulnerable a fuerza bruta.  
- **Insecure JWT Implementation** → token hardcodeado + clave secreta accesible en el servidor.  
- **Improper Access Control** → manipulación del JWT permitió escalar privilegios.  
- **Remote Command Execution (RCE)** → ejecución de comandos arbitrarios a través del dashboard.

---

## 📚 Lecciones aprendidas
- **No exponer comentarios ni convenciones internas** en el código fuente.  
- **Restringir el acceso a logs y archivos sensibles**.  
- **Implementar contraseñas seguras y tokens robustos** para recuperación de cuentas.  
- **Manejar JWT de forma segura**, evitando exponer claves y validando correctamente firmas.  
- **Aplicar validaciones estrictas** a cualquier input que interactúe con el sistema operativo.  
- **Defensa en profundidad**: un fallo no debería comprometer todo el sistema.

---

✅ Este CTF demuestra cómo una serie de malas prácticas en desarrollo y configuración pueden ser **encadenadas** para comprometer completamente un sistema.  
El aprendizaje clave es que la **seguridad debe aplicarse en todas las capas**: desde la gestión de usuarios y autenticación, hasta la protección del backend y la infraestructura.

