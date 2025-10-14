## 🎉 Conclusiones Finales: Soupedecode01 (TryHackMe)

La sala **Soupedecode01** de TryHackMe fue un desafío altamente educativo que se centró intensamente en el ciclo de vida de un ataque en un entorno de **Active Directory (AD)**. La resolución requirió una comprensión sólida de las herramientas y metodologías específicas para la enumeración y explotación de credenciales en dominios Windows.

### 🔑 Puntos Clave del Ataque

| Etapa | Técnica/Vulnerabilidad | Herramienta Clave |
| :--- | :--- | :--- |
| **1. Enumeración Inicial** | RID Bruteforce (SID Cycling) | `nxc` (crackmapexec) |
| **2. Acceso Inicial** | Password Spraying (Usuario=Contraseña) | `nxc` |
| **3. Pivot de Credenciales** | Kerberoasting (Explotación de SPNs) | `GetUserSPNs.py` (Impacket) |
| **4. Acceso a Recursos** | Permisos de la Cuenta de Servicio (`file_svc`) | `smbclient` |
| **5. Escalada de Privilegios** | Hash Spraying de NTLM (Datos de Backup) | `nxc` y `smbexec.py` (Impacket) |

### 💡 Lecciones Aprendidas

1.  **Valor de la Enumeración de AD:** Herramientas como `nxc` y las *scripts* de **Impacket** son indispensables para extraer información valiosa (usuarios, SPNs) que de otra manera sería invisible.
2.  **Seguridad de las Contraseñas de Servicio:** La explotación fue posible gracias a la debilidad en la contraseña de la cuenta de servicio Kerberoastable (Paso 9) y la reutilización de contraseñas/almacenamiento inseguro en el *backup* (Paso 11).
3.  **Encadenamiento de Vulnerabilidades:** El éxito se basó en encadenar un conjunto de vulnerabilidades de baja prioridad (contraseña débil de `ybob317`) para llegar a un impacto alto (compromiso del Domain Controller).

En resumen, esta sala refuerza la importancia de las configuraciones seguras en Active Directory, especialmente en cuentas de servicio y la gestión de copias de seguridad.

---
[Volver al Índice del Write-up](LINK_AL_INDICE)
