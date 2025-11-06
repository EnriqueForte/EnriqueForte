# ✅ Conclusiones — VulnNet Roasted (TryHackMe)

**Resumen rápido**  
VulnNet Roasted es una máquina de nivel *Fácil* orientada a Active Directory/Windows. Con un reconocimiento metódico se pudo progresar desde la enumeración inicial hasta la obtención de credenciales privilegiadas y la lectura de las flags `user` y `system`.

---

## 🔎 Hallazgos principales
- 🟢 La máquina expone servicios relevantes de AD: LDAP, Kerberos, SMB, RPC y WinRM.  
- 📂 Existen shares SMB accesibles anónimamente (`VulnNet-Business-Anonymous`, `VulnNet-Enterprise-Anonymous`) que filtraron ficheros con información útil.  
- 🔐 Se detectaron cuentas vulnerables a **AS-REP roasting** (usuarios sin preauth), permitiendo extraer `krb5asrep` para cracking offline.  
- 🔑 Se obtuvieron SPNs kerberoastables y sus hashes TGS (`krb5tgs`), crackeados para recuperar contraseñas de servicio.  
- 📎 En `NETLOGON` había un script (`ResetPassword.vbs`) con credenciales en claro (`a-whitehat`), lo que facilitó acceso inicial.  
- 🧰 Con `impacket-secretsdump` se extrajeron hashes (incluido `Administrator`) y, usando `wmiexec` (pass-the-hash), se consiguió shell administrativo y lectura de flags.

---

## 🧠 Lecciones aprendidas
1. La información presente en ficheros compartidos, incluso en shares read-only, puede ser crítica para la escalada.  
2. Kerberos/AD es un vector prioritario: AS-REP y Kerberoasting siguen siendo técnicas efectivas contra configuraciones débiles.  
3. Scripts en NETLOGON o SYSVOL con credenciales hardcodeadas ofrecen un camino directo a la compromisión.  
4. La suite Impacket (`GetNPUsers`, `GetUserSPNs`, `secretsdump`, `wmiexec`) y cracking offline (`john`/`hashcat`) son piezas clave en ejercicios AD.  
5. Mantener listas de usuarios limpias y wordlists relevantes aumenta la eficiencia en cracking.

---

## 🛡️ Recomendaciones (prioritarias)
- 🔒 Eliminar o restringir shares anónimos; aplicar principio de menor privilegio.  
- ✅ Exigir pre-autenticación para todas las cuentas (evitar AS-REP roasting).  
- 🔐 Revisar y minimizar privilegios de cuentas con SPNs; rotar contraseñas de cuentas de servicio.  
- 🧰 Evitar credenciales hardcodeadas en scripts; usar managed accounts o vaults para secrets.  
- 📜 Habilitar auditoría y alertas para actividades Kerberos/SMB anómalas (GetNPUsers, GetUserSPNs, intentos de dump).  
- 🔁 Rotación y fortificación de contraseñas; considerar MFA para accesos administrativos.

---

## ▶️ Pasos recomendados para mitigación inmediata
1. Inventario y cierre/aseguramiento urgente de shares públicos (NETLOGON/SYSVOL).  
2. Forzar cambio de contraseñas para cuentas expuestas y cuentas con SPN detectadas.  
3. Aplicar la política de preauth en Active Directory y revisar GPOs/scripts de inicio.  
4. Implementar detección (SIEM) para patrones de ataque a Kerberos y dumping de credenciales.  
5. Formar a admins en buenas prácticas: no hardcodear credenciales, usar vaults y revisar scripts.

---

## 📝 Nota final
Este CTF muestra cómo errores de configuración típicos en entornos Windows/AD (shares abiertos, cuentas sin preauth, credenciales en scripts) permiten una cadena de ataque completa desde reconocimiento hasta obtención de privilegios. La defensa efectiva combina reducción de superficie, gestión segura de secretos y monitorización activa.

