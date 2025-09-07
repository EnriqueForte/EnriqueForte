# 📌 Conclusiones – Bounty Hacker (TryHackMe)

Este CTF nos permitió repasar varias fases fundamentales del hacking ético y pentesting:

- **Reconocimiento y enumeración:** confirmamos conectividad, descubrimos puertos abiertos y servicios en ejecución (FTP, SSH, HTTP).  
- **Explotación inicial:** aprovechamos el acceso anónimo por FTP para descargar archivos que contenían un usuario y un listado de contraseñas.  
- **Fuerza bruta:** con Hydra, probamos las credenciales contra SSH y logramos obtener acceso como el usuario `lin`.  
- **Escalada de privilegios:** gracias a los permisos de `sudo` sobre `tar`, utilizamos una técnica documentada en GTFOBins para conseguir root.  
- **Captura de flags:** obtuvimos tanto la flag de usuario como la de root, completando con éxito el reto.  

---

## 💡 Aprendizajes clave

- La **enumeración exhaustiva** siempre es crítica: el acceso anónimo a FTP nos dio la pista principal.  
- El uso de **diccionarios personalizados** en ataques de fuerza bruta puede ser decisivo para encontrar credenciales.  
- Conocer y aplicar **GTFOBins** facilita identificar vectores de escalada de privilegios rápidamente.  
- La práctica en este tipo de CTF ayuda a reforzar un **flujo de trabajo metódico** en pentesting.

---

✅ **Bounty Hacker** es un reto ideal para principiantes, ya que combina acceso inicial mediante credenciales filtradas con una escalada de privilegios sencilla pero realista.
