## 🔓 Crack the Hash — Writeup 

### 🧭 Introducción

Este writeup documenta la resolución del CTF Crack the Hash paso a paso. El objetivo es obtener la contraseña asociada a un hash mediante ataques por diccionario y otras técnicas con Hashcat. El entorno utilizado es Windows, para ejecutar con mas velocidad las herramientas.

### 📦 Requisitos

🖥️ SO: Windows

⚙️ Herramienta: Hashcat (binarios para Windows)

📚 Diccionarios: rockyou.txt (u otros personalizados)

🗂️ Carpeta para almacenar el/los hashes a crakear

✍️ Editor de texto o PowerShell para crear/editar archivos de hashes

🗂️ Estructura de carpetas (configuración inicial)

Para mantener el entorno ordenado creamos una estructura simple dentro de la carpeta de Hashcat. En las capturas se marcaron en rojo dos carpetas importantes:

📁 Dict — carpeta donde colocaremos el archivo rockyou.txt u otros diccionarios.

📁 Hash — carpeta donde guardaremos un archivo de texto (hashes.txt) con el/los hash(es) objetivo.

✅ Consejo: coloca rockyou.txt en Dict y el/los hash(es) en Hash para referenciarlos fácilmente desde la línea de comandos.

### 🛠️ Paso 1 — Entorno Windows, instalación y configuración básica

1.1 Descargar e instalar Hashcat

🔽 Descargar la versión para Windows desde la web oficial de Hashcat (archivo ZIP con binarios).

🗃️ Extraer el contenido en una carpeta de trabajo, por ejemplo C:/tools/hashcat-7.1.2.

📂 Dentro de esa carpeta verás ficheros y subcarpetas como hashcat.exe, example.dict, masks, rules, etc.

1.2 Crear las carpetas Dict y Hash

En el directorio raíz de Hashcat crea:

C:/ruta/a/hashcat-7.1.2/Dict
C:/ruta/a/hashcat-7.1.2/Hash

<img width="667" height="930" alt="creamos directorios para guardar diccionario y hashes" src="https://github.com/user-attachments/assets/07e96b72-d0bf-4623-9870-068e00e842db" />


1.3 Colocar rockyou.txt en Dict

rockyou.txt es un diccionario muy usado en CTFs. Colócalo dentro de Dict.

Si el archivo está comprimido (.gz, .zip), descomprímelo antes de usarlo con Hashcat.

1.4 Crear el archivo de hashes en Hash

Crea un archivo de texto llamado, por ejemplo, hashes.txt dentro de la carpeta Hash.

En Windows puedes usar el Bloc de notas o desde PowerShell:

Set-Content -Path ./Hash/hashes.txt -Value 'HASH_AQUI'

Sustituye 'HASH_AQUI' por el hash objetivo (una línea por hash si hay varios).

Nota: asegura que el archivo no contenga BOM ni caracteres extra (usa codificación UTF‑8 sin BOM). A veces es útil comprobar con un editor que muestre caracteres invisibles.


## 🔬 Paso 2 — Ejecución del primer ataque y resultado

En este paso lanzamos Hashcat en modo **diccionario** contra el hash objetivo. La captura de la consola muestra que el hash es **MD5** (hashmode `0`) y que la contraseña se recuperó correctamente.

<img width="846" height="506" alt="Tarea 1 md5 -m0" src="https://github.com/user-attachments/assets/9b8a473d-47e9-473f-92b2-8b62ae6087e4" />


### ✅ Comando utilizado

```cmd
hashcat.exe -m 0 -a 0 Hash/hashes.txt Dict/rockyou.txt
```

🖥️ Salida relevante (resumen)

Hash.Target: 48bb6e862e54f2a795ffc4e541caed4d

Status: Cracked

Recovered: easy

Hash.Mode: 0 (MD5)

Diccionario usado: Dict/rockyou.txt

Velocidad reportada: ~45165.6 kH/s (según la captura)

Tiempo de ejecución (captura): ~1 s (ataque muy rápido porque la contraseña estaba en el diccionario)

En la consola Hashcat aparece 48bb6e862e54f2a795ffc4e541caed4d:easy, lo que indica que la contraseña para ese hash es easy.

### 🔎 Observaciones

Al tratarse de MD5 y de una contraseña común, rockyou.txt la contenía y Hashcat la encontró casi instantáneamente.

Hashcat guarda las contraseñas halladas en el potfile (por defecto hashcat.potfile) — útil para no volver a atacar hashes ya resueltos.

La salida muestra métricas útiles (velocidad, progreso, hardware); revisa esas métricas si quieres optimizar ataques futuros.


## 🔧 Paso 3 — Segundo hash (SHA1) y resultado

Segunda captura ejecutamos Hashcat contra un hash SHA1 (hashmode 100) usando el mismo diccionario rockyou.txt.

<img width="773" height="482" alt="Tarea 2 sha1 -m100" src="https://github.com/user-attachments/assets/baf90507-36d0-46be-9d57-1d5427e304e6" />


✅ Comando utilizado

hashcat.exe -m 100 -a 0 Hash/hashes.txt Dict/rockyou.txt

Nota: -m 100 corresponde a SHA1 en la tabla de modos de Hashcat.

🖥️ Salida relevante (resumen)

Hash.Target: cbfdac6008f9cab4083784cbd1874f76618d2a97

Status: Cracked

Recovered: (mostrado en la captura — contraseña encontrada con rockyou.txt)

Hash.Mode: 100 (SHA1)

Diccionario usado: Dict/rockyou.txt

Velocidad reportada: ~41236.9 kH/s

Tiempo de ejecución (captura): ~1 s

En la consola aparece cbfdac6008f9cab4083784cbd1874f76618d2a97:<password> indicando que el hash SHA1 fue resuelto con una contraseña presente en rockyou.txt.

### 🔎 Observaciones

SHA1 es más lento que MD5 en algunas configuraciones, pero si la contraseña está en el diccionario sigue siendo muy rápido.


## 🔬 Paso 4 — SHA-256 (hashmode 1400) — Ejecución y resultado

En este paso atacamos un nuevo hash (SHA-256, `-m 1400`) con `rockyou.txt` y obtuvimos la contraseña. 

<img width="874" height="497" alt="Tarea 3 SHA2-256 -m1400" src="https://github.com/user-attachments/assets/5cb2feff-ba9a-4229-b939-f86e17f4e28b" />

---

### ✅ Comando utilizado

```cmd
# Ataque por diccionario contra SHA-256 (modo 1400)
hashcat.exe -m 1400 -a 0 Hash\hashes.txt Dict\rockyou.txt
```

🖥️ Salida relevante (resumen)

Hash.Target: 1c8bfe8f801d79745c4631d09fff36c82aa37fc4cce4fc946683d7b336b63032

Status: Cracked

Recovered: <PASSWORD_REDACTED> ← (reemplaza por la contraseña en claro si quieres incluirla en el informe)

Hash.Mode: 1400 (SHA-256)

Diccionario usado: Dict/rockyou.txt

Velocidad reportada: ~42206.6 kH/s (según la captura)

Tiempo de ejecución (captura): ~1 s

En la consola Hashcat aparece una línea con el formato hash:password indicando que el hash SHA-256 fue resuelto con una contraseña presente en rockyou.txt.

### 🔎 Observaciones

SHA-256 (modo 1400) es más costoso que MD5/SHA1, pero si la contraseña está en el diccionario la recuperación sigue siendo rápida.


## 🔐 Paso 5 — Bcrypt / Blowfish (uso de herramienta online por limitaciones de Hashcat)

En este paso me encontré con un hash de tipo **bcrypt** (Blowfish) y, dado que Hashcat no obtuvo resultado práctico por tiempo/arquitectura/encodificación, 

tuve que utilizar una herramienta online (`hashes.com`) para resolverlo rápidamente.

<img width="994" height="296" alt="Tarea 4bcrypt-blowfish online porque demora mucho con la herramienta" src="https://github.com/user-attachments/assets/00b2d32e-5dec-40d3-a338-68e1492c00d5" />


---

### 🧾 Qué intenté localmente (resumen)

Intenté atacar el hash con Hashcat usando un ataque por diccionario (rockyou), el comando típico sería:

```cmd
# bcrypt (Blowfish) = hashmode 3200
hashcat.exe -m 3200 -a 0 Hash\hashes.txt Dict\rockyou.txt
```
Problemas comunes con bcrypt:

Bcrypt es deliberadamente lento (cost factor), por eso los ataques por diccionario requieren mucho tiempo incluso en hardware potente.

Dependiendo del work factor (cost), Hashcat puede tardar demasiado para ser práctico en una máquina normal.

En algunos entornos Windows/Drivers, la aceleración por GPU no ofrece grandes beneficios con bcrypt (es muy CPU-bound).

Si el hash tenía un encoding/formatación no estándar (prefijos, salt inlined o formato con $2y$… vs $2a$…) puede requerir normalización antes de atacar con Hashcat.

### 🌐 Uso de la herramienta online.

Por rendimiento/tiempo/compatibilidad decidí usar Hashes.com (u otro servicio similar) — pasos generales:

Abrí e la web (Hashes.com → Desencriptar hashes → subir/pegar el hash).

Inicié la búsqueda en su base de datos / motores de cracking.

El servicio devolvió un resultado y mostró la contraseña en claro (captura: muestra Found con el hash y la contraseña).

En la captura se ve la interfaz con el mensaje “✅ Encontrado” y el hash resuelto por el servicio.


## 🔎 Paso 6 — MD4 (intento local fallido → uso de servicio online)

En este paso nos enfrentamos a un hash que, por su formato y tras varias pruebas, parecía **MD4**. Intenté descifrarlo localmente con Hashcat (varios `-m`), 

pero no fue posible (probablemente por formato/encoding o limitaciones del entorno). Finalmente recurri a un servicio online (Hashes.com) que devolvió el resultado.

<img width="994" height="298" alt="Tarea 5 md4 online porque  la herramienta no lo descifraba intentando de todas las maneras" src="https://github.com/user-attachments/assets/a1512e4b-f3c4-475f-ad69-192d1e599cff" />


---

### 🧪 Comandos que probaste localmente (ejemplos)

Probé varios modos con Hashcat sin éxito. Algunos de los comandos típicos que se intentan para MD4/NTLM/otros fueron:

```cmd
# Intento MD4 (hashmode 900)
.\hashcat.exe -m 900 -a 0 .\Hash\hashes.txt .\Dict\rockyou.txt --potfile-path .\hashcat.potfile
```

### ❓ Posibles razones del fallo local

Formato / encoding distinto: el hash podía incluir salt, estar codificado (Base64, hexadecimal con prefijo) o contener caracteres extra (espacios, CRLF, BOM) que impedían que Hashcat lo reconociera tal cual.

Tipo de hash incorrecto: a veces un hash que parece MD4 es en realidad otro formato cercano; es importante identificar correctamente con hash-identifier, hashid o comprobaciones manuales.

Limitaciones de HW / Kernel: en algunos casos el driver/ OpenCL en Windows no acelera bien ciertos modos, o Hashcat necesita parámetros adicionales (-D, --opencl-device-types, -w) para forzar CPU/GPU.

Salt embebido / formato usuario:hash: si el hash tiene formato user:hash o hash:extra hay que limpiarlo antes.

### 🌐 Uso del servicio online (qué hiciste)

Ante la imposibilidad práctica de resolverlo localmente, pegué el hash en Hashes.com y el servicio devolvió el resultado. En la captura aparece el sitio indicando Encontrado y mostrando el hash:password.

Herramienta: Hashes.com

Acción: Pegar el hash → procesar → resultado Found


## 🔬 Paso 7 — SHA-256 (hashmode 1400) — Ejecución y resultado

En este paso atacamos otro hash **SHA-256** (modo `1400`) usando `rockyou.txt` y Hashcat. A continuación se documenta el comando usado, la salida relevante.

---

### ✅ Comando utilizado

```cmd
# Ataque por diccionario contra SHA-256 (modo 1400)

hashcat.exe -m 1400 -a 0 Hash\hashes.txt Dict\rockyou.txt
```

🖥️ Salida relevante (resumen)

<img width="830" height="483" alt="Tarea 2 1 SHA2 256 -m1400" src="https://github.com/user-attachments/assets/10c76ab8-2ffb-45a8-ade3-a27352fb61b0" />


Hash.Target: f09edcb1fcefc6dfb23dc3505a882655ff77375ed8aa2d1c13f640fccc2d0c85

Status: Cracked

Recovered: <PASSWORD_REDACTED> ← (la contraseña aparece en la captura; indícame si la incluyo en claro)

Hash.Mode: 1400 (SHA-256)

Diccionario usado: Dict/rockyou.txt

Velocidad reportada: ~42736.5 kH/s (según la captura)

Tiempo de ejecución (captura): ~1 s

En la consola Hashcat aparece una línea con el formato hash:password, que indica que el hash SHA-256 fue resuelto con una entrada presente en rockyou.txt.

### 🔎 Observaciones

Este hash SHA-256 se resolvió rápidamente porque la contraseña estaba presente en el diccionario.

Hashcat almacena los resultados en el potfile por defecto (o en la ruta indicada con --potfile-path), por lo que posteriores ejecuciones evitarán volver a intentar hashes ya resueltos.


## 🔬 Paso 8 — NTLM (hashmode 1000) — Ejecución y resultado

En este paso atacamos un hash **NTLM** (modo `1000`) usando `rockyou.txt` y Hashcat. A continuación se documenta el comando usado, la salida relevante.

<img width="830" height="480" alt="Tarea 2 2 NTLM -m1000" src="https://github.com/user-attachments/assets/e34cc050-e21c-4c6e-a051-cf3fc22d2e97" />


---

### ✅ Comando utilizado

```cmd
# Ataque por diccionario contra NTLM (modo 1000)

hashcat.exe -m 1000 -a 0 Hash\hashes.txt .\Dict\rockyou.txt
```

🖥️ Salida relevante (resumen)

Hash.Target: 1dfeca0c002ae40b8619ecf94819cc1b

Status: Cracked

Recovered: <PASSWORD_REDACTED> ← (la contraseña aparece en la captura; indícame si la incluyo en claro)

Hash.Mode: 1000 (NTLM)

Diccionario usado: Dict/rockyou.txt

Velocidad reportada: ~17534.9 kH/s (según la captura)

Tiempo de ejecución (captura): ~1–2 s

En la consola Hashcat aparece la línea hash:password, indicando que el hash NTLM fue resuelto con una entrada presente en rockyou.txt.

### 🔎 Observaciones

NTLM suele ser más rápido en GPU/CPU que hashes costosos (bcrypt/argon2). En la captura se resolvió por diccionario de forma muy rápida.

NTLM no usa sal por defecto, por eso los ataques por diccionario son directos cuando la contraseña está en la lista.

## 🔐 Paso 9 — SHA512-crypt (hashmode 1800) — Intento local fallido → uso de servicio online

En este paso nos enfrentamos a un hash en formato **SHA512-crypt** (a menudo representado con `$6$…`), un esquema diseñado para ser lento y resistente a ataques por GPU/ASIC. 

Al intentar descifrarlo localmente Hashcat no fue práctico (mi GPU no era compatible o la configuración no aceleraba este modo),

por lo que tuve que recurrir a el servicio online (Hashes.com) para obtener el resultado. Aquí se documenta la metodología.

<img width="984" height="375" alt="Tarea 2 3 SHA512crypt, hash online ya que mi gpu no era comptible o no funcionaba" src="https://github.com/user-attachments/assets/55587267-258d-4daf-ad6e-c80f61b33358" />


---

### 🧾 Identificación del hash

- Formato visible en la captura: comienza con `$6$...` → **SHA512-crypt** (Hashcat mode `1800`).

- Este scheme incluye salt embebido (tras `$6$`) y está pensado para ser computacionalmente costoso (work factor).

---

### ✅ Comandos típicos intentados con Hashcat

```powershell
# Modo SHA512-crypt = 1800

hashcat.exe -m 1800 -a 0 Hash\hashes.txt Dict\rockyou.txt
```

🌐 Uso del servicio online (qué hiciste)

Dado que el cracking local no era práctico, pegué el hash en Hashes.com y el servicio devolvió el resultado (captura con ✅ Encontrado).

Fuente: Hashes.com

Resultado: Encontrado → contraseña mostrada en la interfaz


## 🔎 Paso 10 — Resultado obtenido mediante servicio online (GPU no funcional)

En este paso te encontraste con otro hash que no pude descifrar localmente porque mi GPU no funcionaba o no era compatible con el modo necesario. 

Para avanzar use nuevamente el servicio online (Hashes.com) y obtuve el resultado. Aquí queda documentado el procedimiento y las recomendaciones.

---

### 🖥️ Intentos locales (resumen)

Probaste atacar el hash con Hashcat (modo común para SHA1 u otros modos), por ejemplo:

```powershell
# Ejemplo para SHA1 (hashmode 100)

hashcat.exe -m 100 -a 0 Hash\hashes.txt Dict\rockyou.txt
```

### 🌐 Paso online — Hashes.com

Servicio usado: Hashes.com

Acción: pegué el hash en la web → el servicio procesó la consulta → resultado Encontrado (captura incluida).

Línea mostrada en la captura:

<img width="930" height="352" alt="Tarea 2 4 SHA1 gpu no era comptible o no funcionaba" src="https://github.com/user-attachments/assets/6b195479-e5e0-4bc0-97cc-0f2f86b441b5" />

