# 🛡️ CTF Writeup: Flag Vault (TryHackMe) 🚩

### Descripción del Desafío

"Cipher me pidió que creara la bóveda más segura para las banderas, así que creé una bóveda inaccesible. ¿No me crees? Pues aquí tienes el código con la contraseña codificada. Aunque ya no sirve de mucho."

El reto proporciona el código fuente de un sistema de inicio de sesión basado en C. El objetivo es bypassear o explotar este sistema.

## 1. 🔍 Detección y Conexión (Ping)
Confirmamos que la máquina objetivo está activa y accesible.

Máquina Objetivo: 10.10.162.176

Herramienta: ping

Comando Ejecutado
````bash
ping -c 4 10.10.162.176
````

<img width="596" height="285" alt="Ping" src="https://github.com/user-attachments/assets/a8710d2f-42dd-4561-9f6c-e02cdde766c2" />

Resultado
````
Métrica,Valor
Paquetes Transmitidos,4
Paquetes Recibidos,4
Pérdida de Paquetes,0%
TTL,63
````
✅ Conclusión: La máquina está activa y la conexión es estable.

## 2. 📖 Análisis del Código Fuente C (FlagVault.c) 🧐
El desafío nos proporciona el código fuente del binario que se ejecuta en la máquina objetivo. El análisis se centra en la función login() para identificar la vulnerabilidad o la lógica de autenticación.

Función login() Detallada
Al examinar la función login(), encontramos la siguiente estructura:

````C
void login(){
    char password[100] = "";
    char username[100] = "";

    printf("Username: ");
    gets(username); // <-- ⚠️ Primer Punto Crítico

    // IF I disable the password, nobody will get in.
    // printf("Password: ");
    // gets(password); // <-- Línea de entrada de contraseña COMENTADA

    if(!strcmp(username, "bytereaper") && !strcmp(password, "5up3rP4zz123Byte")){
        print_flag();
    }
    else {
        printf("Wrong password! No flag for you.");
    }
}
````

### Puntos Clave y Vulnerabilidades

Uso de gets(): La función gets() es extremadamente insegura porque no verifica el límite del buffer, lo que permite un Buffer Overflow (desbordamiento de buffer) si la entrada del usuario excede los 100 bytes asignados a username.

Lógica de Autenticación Rota:

El código para solicitar la contraseña (gets(password);) está comentado.

Sin embargo, la verificación de la contraseña aún se realiza en la línea !strcmp(password, "5up3rP4zz123Byte").

El Buffer de la Contraseña:

La variable password se inicializa a una cadena vacía ("").

Como no se le pide al usuario que ingrese la contraseña, la variable password mantiene su valor inicial: una cadena vacía.

### Identificación de la Explotación

La condición de autenticación es: if(!strcmp(username, "bytereaper") && !strcmp(password, "5up3rP4zz123Byte"))

Donde:

username debe ser "bytereaper".

password debe ser "5up3rP4zz123Byte".

¡Pero espera! Debido al bug de la lógica:

La variable password realmente contiene una cadena vacía ("").

La variable username la controlamos nosotros.

La verdadera vulnerabilidad que debemos considerar es que la función login() utiliza gets(username) y luego llama a la función print_flag() si se cumplen las condiciones. El buffer de password está en la stack justo después del buffer de username.
````
Variable,Tamaño,Contenido Inicial
password,100 bytes,""""" (vacío)"
username,100 bytes,""""" (vacío)"
````
Si explotamos el Buffer Overflow en username, podemos sobrescribir el buffer de password con la contraseña correcta: "5up3rP4zz123Byte".

## 3. 🐍 Creación del Exploit (Python)

Se utilizó la librería pwntools en Python para automatizar el proceso de Buffer Overflow y la búsqueda del padding exacto.

### Lógica del Payload
El payload está diseñado para cumplir la condición de autenticación al forzar los valores de las variables en memoria:

1. bytereaper + \x00: Esto se escribe en el buffer de username. El \x00 (byte nulo) es esencial porque truncará la función de comparación de cadenas (strcmp) justo después de bytereaper, haciendo que strcmp(username, "bytereaper") sea verdadero.

2. 0 * padding: Se utilizan bytes de relleno para llegar al inicio de la variable password en el stack. El exploit itera este valor desde 80 a 130 para encontrar el offset correcto.

3. 5up3rP4zz123Byte: Esta es la contraseña real. Sobrescribe el valor de la variable password en la memoria stack, haciendo que strcmp(password, "Sup3rP4zz123Byte") sea verdadero.

Código del Exploit (Fragmento)
````Python
# === BÚSQUEDA ITERATIVA ===
for padding in range(PADDING_MIN, PADDING_MAX + 1):
    # ... conexión remota ...

    payload = (
        USERNAME_PART +  # bytereaper
        b'\x00' +        # Byte nulo para truncar strcmp
        b'0' * padding + # PADDING (ej. 80-130 bytes)
        PASSWORD_REAL    # Sobrescribe el buffer 'password'
    )
    
    # ... envío y chequeo de la respuesta ...
````

## 4. 🚀 Ejecución del Exploit y Obtención de la Flag

Ejecutamos el script exploit.py para automatizar la búsqueda del padding y la explotación del Buffer Overflow.

<img width="449" height="518" alt="Flag" src="https://github.com/user-attachments/assets/b6eee258-88bf-476a-a220-f22f905c5f6c" />

Comando Ejecutado
````Bash
python exploit.py
````
Resultado y Éxito 

El script comenzó a probar padding desde $80$ hasta $100$ sin éxito. Al probar conpadding = 101, la respuesta del binario remoto cambió de "Wrong password" a una "Respuesta rara". Esta "respuesta rara" es la ejecución de la función print_flag(), que devuelve la flag.
````
Métrica,Valor
Padding Exitoso,101
Resultado,THM{...}
````

🥳 Conclusión: El offset de $101$ bytes es el padding exacto requerido entre el final del buffer de username (incluyendo \x00) y el inicio del buffer de password. Al sobrescribir password con el valor correcto, la condición lógica se cumplió y se ejecutó la función print_flag().
