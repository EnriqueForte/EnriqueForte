### 🏁 Conclusiones y Aprendizaje: Flag Vault 2 (Format String)

#### **🎯 Objetivo del Desafío**

El reto **Flag Vault 2** fue un excelente ejercicio práctico sobre la explotación de fallos lógicos en la implementación de la función `printf()`, conocidos como vulnerabilidades de **Formato de Cadena** (*Format String Vulnerability*).

#### **📝 Resumen del Proceso de Explotación**

1.  **Vulnerabilidad Específica:** El fallo se localizó en la función `print_flag()` en la línea `printf(username);`. Al pasar la entrada del usuario (`username`) directamente como cadena de formato, el programa intentó interpretar especificadores como `%s` o `%x`.

2.  **Técnica de Filtrado (Stack Leak):** Se utilizó el especificador de formato `%s` en un *payload* iterativo (`%[posición]%s`) para buscar punteros válidos en la pila (stack).

3.  **Localización del Offset:** El *script* de Python encontró que en la **posición 5** de la pila, residía un puntero que apuntaba directamente a la variable `flag`.

4.  **Resultado:** Al utilizar el *payload* con el *offset* **5**, la función `printf` filtró el contenido de esa dirección de memoria como una cadena, revelando la *flag* que el desarrollador intentó ocultar al comentar la línea `printf("%s", flag);`.

#### **🔑 Lecciones Aprendidas (Key Takeaways)**

| Área | Lección Clave |
| :--- | :--- |
| **Programación Segura** | **Regla de Oro de `printf`:** La cadena de formato nunca debe ser controlada por la entrada del usuario. Siempre se debe usar `printf("%s", variable_usuario);` y nunca `printf(variable_usuario);`. |
| **Vulnerabilidad de `gets()`** | Aunque el vector principal fue *Format String*, el binario heredó la vulnerabilidad de **Buffer Overflow** de `gets(username)`, lo que es un doble fallo de seguridad. |
| **Explotación** | **Lectura de la Pila:** El desafío demostró la capacidad de las vulnerabilidades de formato de cadena para leer datos arbitrarios de la *stack* (`%x`, `%p`), incluyendo información sensible como *flags* o direcciones de memoria (útil para la evasión de ASLR). |

---
