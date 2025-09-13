# 📌 Conclusiones – CTF Include (TryHackMe)

## 🔑 Vulnerabilidades explotadas
Durante la resolución del CTF se identificaron y explotaron varias vulnerabilidades críticas:

1. **Credenciales por defecto**  
   - El portal en el puerto 4000 permitía autenticarse con `guest/guest`.

2. **Manipulación de parámetros**  
   - Fue posible modificar campos del perfil (ej. `isAdmin`) para escalar privilegios dentro de la aplicación.

3. **Server-Side Request Forgery (SSRF)**  
   - Desde el panel de administrador se pudo redirigir la carga de recursos a endpoints internos (`127.0.0.1:5000`), accediendo a **APIs restringidas**.

4. **Local File Inclusion (LFI)**  
   - El parámetro `img` en `profile.php` permitió incluir archivos locales del sistema (`/etc/passwd`, `/var/log/mail.log`, etc.).

5. **Log Poisoning → Remote Code Execution (RCE)**  
   - Mediante la inyección de PHP en los logs del servicio de correo y posterior inclusión vía LFI, se logró ejecutar comandos en el servidor.

---

## 🚀 Técnicas empleadas
- **Enumeración de puertos y servicios** con `nmap`.
- **Descubrimiento de rutas** con `gobuster`.
- **Acceso inicial** con credenciales por defecto.
- **Privilege escalation lógico** manipulando parámetros internos (`isAdmin`).
- **Explotación SSRF** para consultar APIs internas.
- **Fuzzing con BurpSuite Intruder** para detectar LFI.
- **File Inclusion + Log Poisoning** para ejecutar código remoto.
- **Búsqueda y lectura de flags** en el sistema (`/var/www/html/...`).

---

## 📚 Aprendizajes clave
- La importancia de **no usar credenciales por defecto**.
- El riesgo de **no validar parámetros del cliente**, permitiendo alterar roles o permisos.
- Cómo un **panel mal diseñado** puede dar pie a **SSRF** y exponer servicios internos.
- La combinación de **LFI + Log Poisoning** como técnica potente para escalar a **RCE**.
- Lo esencial de la **defensa en profundidad**: incluso si un servicio cae, otro debe proteger al sistema.

---

## 🏁 Resultado final
El laboratorio fue completado con éxito, obteniendo todas las **flags** y comprendiendo la cadena de ataque:  

**Credenciales por defecto → Escalada de privilegios → SSRF → LFI → Log Poisoning → RCE → Lectura de flags** ✅
