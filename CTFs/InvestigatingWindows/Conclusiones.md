## 📝 Resumen

Esta investigación de Respuesta a Incidentes (IR) se centró en un servidor Windows Server 2016 Datacenter comprometido. El análisis forense de artefactos del sistema (registros de eventos, registro, tareas programadas y archivos) ha permitido establecer la cronología, identificar las herramientas y rastrear la persistencia del atacante.

Conclusión Principal: El compromiso ocurrió el 03/02/2019 y el atacante utilizó técnicas de persistencia (tareas programadas y reglas de firewall), robo de credenciales y web shells para mantener el acceso y control total de la máquina.

## 📅 Cronología del Incidente

La secuencia de eventos se establece a partir de las marcas de tiempo de los artefactos manipulados:

Hora (03/02/2019)	Artefacto	Descripción del Evento
4:04:49 PM	Evento de Seguridad 4735	El primer inicio de sesión logra la asignación de privilegios especiales (elevación de privilegios).
5:31 PM	Archivo hosts	El atacante modifica el archivo hosts para implementar DNS Poisoning.
5:48:32 PM	Registro de Sesión	Último Logon interactivo registrado para el usuario John.


## 🚨 Artefactos de Persistencia y Control

1. Manipulación de DNS (C2)
IP de Comando y Control (C2): El atacante redirigió el tráfico web a su propia IP externa: 76.32.97.132.

Sitio Objetivo: El envenenamiento DNS afectó directamente a la resolución de google.com.

2. Tarea Programada Maliciosa
Nombre de la Tarea: Clean file system.

Archivo Ejecutado: Un script de Netcat de PowerShell llamado nc.ps1.

Puerto de Escucha: Este script estaba configurado en modo de escucha (-l) en el puerto 1348.

3. Puerta Trasera de Firewall
Regla Abierta: El atacante creó una regla de firewall genérica (ej. Allow outside connections for development).

Último Puerto Abierto: El puerto expuesto para el acceso remoto final fue el 1337.

4. Web Shell y Robo de Credenciales
Web Shell: Se subieron archivos de shell con la extensión .jsp (ej. shell.jsp) al directorio raíz del servidor web IIS (C:\inetpub\wwwroot), asegurando una puerta trasera basada en web.

Herramienta de Robo de Credenciales: La herramienta Mimikatz fue ejecutada en el sistema, con archivos de log de volcado de credenciales (mim-out) encontrados en la carpeta temporal (C:\TMP).

## 👥 Cuentas de Usuario Comprometidas
Usuario de Acceso: John fue el usuario que inició sesión por última vez antes de la actividad maliciosa.

Cuentas Elevadas: El atacante elevó (o creó) cuentas con privilegios administrativos (además del Administrator): Guest y Jenny. La cuenta Jenny nunca fue utilizada para un inicio de sesión (Last Logon: Never).

## 🛡️ Recomendaciones de Mitigación
Eliminación de Persistencia: Eliminar la tarea programada Clean file system y las reglas de firewall maliciosas (ej. Allow outside connections for development).

Limpieza de Cuentas: Eliminar o degradar las cuentas Guest y Jenny si no son requeridas. Cambiar las contraseñas de todos los usuarios comprometidos (incluyendo John).

Restauración del Sistema: Restaurar los archivos manipulados (hosts) y eliminar todos los artefactos de malware (C:\TMP\mim-out, C:\inetpub\wwwroot\*.jsp, nc.ps1, etc.).

Monitoreo: Implementar monitoreo de logs para Event ID 4735 (Asignación de Privilegios Especiales) y el uso de Get-ScheduledTask para detectar nuevas tareas de persistencia.
