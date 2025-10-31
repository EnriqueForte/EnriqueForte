### 🏁 Conclusiones y Aprendizaje: Flag Vault (TryHackMe)

#### **🎯 Objetivo del Desafío**

El reto **Flag Vault** se centró en la explotación de vulnerabilidades de bajo nivel en código C, demostrando cómo la combinación de un fallo de seguridad en la gestión de memoria y un error en la lógica de autenticación puede llevar a la ejecución de código no autorizado.

#### **📝 Resumen del Proceso de Explotación**

1.  **Vulnerabilidad Dual:** El éxito se basó en dos fallos en el código fuente:
    * **Buffer Overflow (BOF):** El uso de la función insegura `gets(username)` permitió al atacante controlar el flujo de datos que se escribían en el *stack*.
    * **Lógica de Autenticación Deficiente:** Al estar comentada la entrada de la contraseña, el binario requería una contraseña dura (`Sup3rP4zz123Byte`) para una variable (`password`) que estaba inicializada como vacía (`""`).

2.  **Payload Estratégico:** La clave fue usar el BOF en `username` para **sobrescribir** la variable adyacente `password` en el *stack* con el valor requerido.

3.  **Ajuste del Padding:** Mediante la iteración controlada por el *exploit* de Python, se determinó que un **padding de $101$ bytes** era el *offset* exacto para alinear la contraseña correcta en la posición de memoria de la variable `password`.

4.  **Resultado:** Al lograr que tanto `username` (`bytereaper`) como `password` (`Sup3rP4zz123Byte`) coincidieran, el *exploit* satisfizo la condición lógica y activó la función `print_flag()`.

#### **🔑 Lecciones Aprendidas (Key Takeaways)**

| Área | Vulnerabilidad y Lección |
| :--- | :--- |
| **Programación Segura** | **Eliminar `gets()`:** El uso de funciones de biblioteca inseguras (como `gets()`, `strcpy()`, `sprintf()`) es la causa raíz de la mayoría de los *Buffer Overflows*. Siempre se deben usar funciones con comprobación de límites (e.g., `fgets()`, `strncpy()`). |
| **Lógica de Aplicación** | **Validación Rigurosa:** Un simple error de lógica (dejar un fragmento de código comentado que rompe la inicialización de variables) fue tan explotable como el fallo de memoria. La seguridad requiere revisión completa del código. |
| **Explotación** | **Ingeniería del Stack:** El desafío es un ejemplo perfecto de cómo un atacante puede manipular el diseño de la *stack* para sobrescribir variables adyacentes a través de un *buffer* vulnerable, sin necesidad de sobrescribir directamente la dirección de retorno. |
