# ✅ Conclusiones – TryHackMe Wgel CTF

Este CTF me ha servido para practicar un flujo completo de ataque: desde el reconocimiento inicial hasta la escalada de privilegios y la obtención de acceso como root.

## 🧠 Lo que he aprendido

1. Reconocimiento primero, siempre
   - Un simple `ping` y un buen escaneo con `nmap` bastan para descubrir la superficie de ataque.
   - Ver claramente los servicios (HTTP y SSH) ayudó a centrar la enumeración.

2. La web casi siempre esconde algo
   - Revisar el código fuente fue clave para obtener la pista del usuario `jessie`.
   - El uso de `gobuster` sobre `/` y luego sobre `/sitemap/` permitió encontrar rutas y archivos que no eran visibles desde la navegación normal.

3. Errores graves de gestión de claves
   - Exponer un directorio `.ssh` en un servidor web es un fallo crítico.
   - Una única clave privada fue suficiente para conseguir acceso SSH sin necesidad de ataques de fuerza bruta.

4. Importancia de revisar permisos locales
   - Después de conseguir acceso como usuario, ejecutar `sudo -l` fue decisivo.
   - Un binario aparentemente inofensivo como `wget`, mal configurado en `sudoers`, abrió la puerta a leer la `root_flag.txt`.

5. Conectar piezas pequeñas en una cadena de ataque
   - Comentario HTML + `.ssh` expuesto + `sudo wget` = camino completo hasta root.
   - Este CTF demuestra cómo varios descuidos pequeños pueden convertirse en una brecha total.

## 🚀 Mejora personal

Gracias a esta máquina he reforzado:

- Mi metodología de enumeración (especialmente web).
- Mi reflejo de revisar siempre `sudo -l` tras obtener acceso.
- Mi capacidad para documentar el proceso paso a paso para futuros reportes y para mi portafolio.

Aunque está catalogado como “fácil”, Wgel CTF es una buena máquina para afianzar hábitos de pentesting y buenas prácticas de documentación.
