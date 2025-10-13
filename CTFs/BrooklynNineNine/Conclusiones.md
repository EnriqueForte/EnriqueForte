### 💡 Conclusiones del CTF Brooklyn Nine Nine 👮‍♂️

Este *writeup* detalla la metodología completa empleada para resolver la sala **Brooklyn Nine Nine** de TryHackMe, cubriendo desde el reconocimiento inicial hasta la obtención de privilegios de *root*.

---

#### 🗺️ Resumen de la Ruta de Ataque

1.  **Reconocimiento:** Se identificaron los puertos abiertos 21 (FTP), 22 (SSH) y 80 (HTTP) mediante Nmap.
2.  **Acceso Inicial:**
    * Se utilizó FTP anónimo para obtener el archivo `note_to_jake.txt` (pista sobre credenciales y web directory).
    * Se analizó la página web (Puerto 80) y su código fuente, encontrando una pista sobre **Esteganografía**.
    * Se descargó la imagen `brooklyn99.jpg` y se crackeó la contraseña de `steghide` (que resultó ser `admin`).
    * Se extrajo el archivo `note.txt` con las credenciales: **`holt:fluffydog12@ninenine`**.
3.  **Obtención de User Flag:** Se obtuvo acceso SSH con las credenciales de `holt` y se leyó `user.txt`.
4.  **Escalada de Privilegios:**
    * Se verificaron los permisos de `sudo` mediante `sudo -l`, encontrando un binario vulnerable: **`(ALL) NOPASSWD: /bin/nano`**.
    * Se utilizó la técnica de GTFOBins para escapar de `nano` y obtener una *shell* de `root`.
5.  **Obtención de Root Flag:** Se leyó `root.txt` en el directorio `/root`.

---

#### 📚 Lecciones Aprendidas (Key Takeaways)

| Concepto | Detalle |
| :---: | :---: |
| **Reconocimiento Detallado** | Un escaneo Nmap exhaustivo (`-sC -sV`) reveló la configuración crítica del FTP anónimo, la puerta de entrada inicial. |
| **Búsqueda de Pistas** | La información valiosa no solo estaba en archivos descargables, sino también oculta en comentarios HTML y mediante técnicas de **Esteganografía**. |
| **Seguridad de Binarios (SUDO)** | La principal vulnerabilidad de la máquina fue la configuración de `sudo` que permitía al usuario de bajo privilegio ejecutar un binario (nano) como *root* sin contraseña, un fallo común de **Privilege Escalation**. |
| **Herramientas Clave** | **Nmap** para escaneo, **FTP** para acceso inicial, **Stegcracker** / **Steghide** para esteganografía y **GTFOBins** como recurso para *exploits* de escalada. |
