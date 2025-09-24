# Conclusiones — SimpleCTF

## Resumen ejecutivo

Durante la resolución de la máquina **SimpleCTF** se identificaron y explotaron vulnerabilidades en varios componentes que permitieron el acceso inicial, la extracción de credenciales y la escalada completa a root. El ejercicio siguió un flujo clásico: reconocimiento → enumeración → explotación → post‑explotación.

## Hallazgos principales

1. **CMS Made Simple 2.2.8** en `/simple/` (versión vulnerable).

   * Vulnerabilidad: **SQL Injection** (CVE‑2019‑9053 / Exploit-DB EDB‑46635).
   * Impacto: extracción de datos sensibles (usuarios, hashes, salts) desde la base de datos.

2. **Credenciales extraídas** mediante PoC adaptado (exploit.py):

   * Usuario: `mitch`
   * Contraseña (descifrada): `secret`
   * Email: `admin@admin.com`

3. **Acceso SSH** en puerto no estándar `2222` con las credenciales anteriores — acceso de usuario `mitch` conseguido.

4. **Permiso sudo inseguro** para el usuario `mitch`: NOPASSWD para `/usr/bin/vim`.

   * Impacto: `vim` permite ejecutar comandos del sistema cuando se ejecuta con privilegios; esto facilitó la escalada inmediata a root (GTFOBins: `:!/bin/sh`).

5. **Flags obtenidas**: `user.txt` (usuario) y `root.txt` (root) — objetivo cumplido para el CTF.

## Evidencia y artefactos

* Capturas de pantalla de: `nmap`, `gobuster`, página `/simple/` (CMS), Exploit‑DB, ejecución del `exploit.py`, salida con credenciales (salt, user, hash), intento de crack y contraseña final, conexión SSH (puerto 2222), `sudo -l` mostrando `vim` NOPASSWD, shell root y `cat /root/root.txt`.
* Scripts/tools usados: `exploit.py` (PoC adaptado de Exploit‑DB), `gobuster`, `nmap`, `curl`, `sqlmap` (opcional), `ssh`.

(Ver carpeta `images/` y `tools/exploit.py` en el repositorio para las evidencias exactas y los logs.)

## Comandos clave y PoC reproducible

Resumen de los comandos más relevantes usados durante el test (ejemplos):

```bash
# Recon y enumeración
ping -c 4 10.10.197.170
nmap -sC -sV -T4 -oN nmap_scan 10.10.197.170
gobuster dir -u http://10.10.197.170:80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100

# Exploit (PoC adaptado - time-based SQLi)
python3 tools/exploit.py -u http://10.10.197.170/simple --crack -w /usr/share/wordlists/rockyou.txt

# Acceso SSH con credenciales extraídas
ssh -p 2222 mitch@10.10.197.170

# Revisar sudoers y escalada (en la máquina)</nsudo -l
sudo /usr/bin/vim -c ':!/bin/sh'
whoami
cat /root/root.txt
```

> Nota: ajustar rutas y tiempos en el PoC según latencia/timeout de la red (el exploit es time‑based).

## Evaluación de riesgo

* **Confidencialidad:** Alta — extracción de credenciales/usuarios.
* **Integridad:** Media — acceso root permite modificación de datos.
* **Disponibilidad:** Baja‑Media — con root se puede afectar disponibilidad, pero la explotación realizada no fue destructiva.

## Recomendaciones de mitigación

1. **Actualizar CMS**: actualizar CMS Made Simple a la versión ≥ 2.2.10 o aplicar parches oficiales.
2. **Revisar políticas `sudo`**: evitar `NOPASSWD` para binarios que permiten ejecución de comandos (vim, less, find, python, perl, ...) o restringir a comandos específicos y necesarios.
3. **Restringir y auditar servicios**: desactivar FTP anónimo o limitarlo a directorios concretos; asegurar puertos SSH y forzar autenticación por clave pública.
4. **Rotación de credenciales y hardening**: forzar contraseñas robustas, habilitar autenticación multifactor donde sea posible y revisar backups/archivos de configuración por contraseñas en claro.
5. **Revisión de permisos en /home**: asegurar que los directorios de usuarios no son world‑readable si contienen información sensible.
6. **Monitoreo y alertas**: habilitar detección de accesos SSH inusuales y registrar uso de `sudo` con auditoría.

## Lecciones aprendidas y notas finales

* Un flujo de enumeración metódico (nmap → gobuster → revisión de CMS → PoC) permitió pasar de reconocimiento a control total de la máquina.
* El problema de mayor impacto en esta máquina fue la combinación de una aplicación web vulnerable (permitiendo extracción de credenciales) y una mala configuración de `sudo` que permitió escalada inmediata.

## Referencias

* Exploit‑DB: *CMS Made Simple < 2.2.10 - SQL Injection* (EDB‑46635 / CVE‑2019‑9053).
* GTFOBins: *vim — shell vector*.
* Herramientas: `nmap`, `gobuster`, `sqlmap`, `ssh`.

---
