# 📌 Conclusiones - Pickle Rick CTF

Durante la resolución de este CTF se aplicaron técnicas clásicas de pentesting web y escalada de privilegios.  

### 🔑 Puntos clave del aprendizaje
- La **inspección del código fuente** de páginas web puede revelar información sensible, como usuarios ocultos.  
- Los archivos como **robots.txt** muchas veces contienen contraseñas o cadenas útiles.  
- La **enumeración de directorios** con herramientas como *Gobuster* permite descubrir paneles de login y otros archivos críticos.  
- Los **paneles de comandos** integrados en aplicaciones web pueden ser vulnerables a **RCE (Remote Command Execution)**.  
- Incluso con restricciones (bloqueo de `cat`), es posible evadirlas con **comandos alternativos** (`tac`, `head`, `less`).  
- La verificación de permisos (`sudo -l`) reveló que el usuario `www-data` podía ejecutar comandos como root sin contraseña, lo que facilitó una escalada de privilegios inmediata.  

### 🏆 Resultado
- Se obtuvieron los **tres ingredientes secretos** solicitados:  
  1. `mr. meeseek hair`  
  2. `1 jerry tear`  
  3. `fleeb juice`  

👉 Esto permitió completar con éxito el reto **Pickle Rick** en TryHackMe, reforzando conocimientos en **enumeración web, RCE y escalada de privilegios**.  
