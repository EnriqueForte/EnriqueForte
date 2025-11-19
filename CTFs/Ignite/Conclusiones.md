# 🎓 Conclusiones - TryHackMe Ignite CTF

## 📌 Resumen

El CTF **Ignite** de TryHackMe representa un escenario realista de compromiso de un servidor web mal configurado que ejecuta FUEL CMS 1.4.1, una versión conocida por contener la vulnerabilidad crítica **CVE-2018-16763** (Remote Code Execution). A través de este ejercicio, se demostró cómo la combinación de software desactualizado y malas prácticas de gestión de credenciales puede llevar a un compromiso total del sistema.

---

## 🎯 Objetivos Alcanzados

### ✅ Objetivos Técnicos

1. **Reconocimiento y Enumeración**
   - Identificación exitosa de servicios expuestos mediante Nmap
   - Enumeración completa de directorios web con Gobuster
   - Análisis de archivos de configuración accesibles públicamente

2. **Explotación de Vulnerabilidades**
   - Identificación y explotación de CVE-2018-16763 (RCE en FUEL CMS 1.4.1)
   - Establecimiento de reverse shell para acceso interactivo
   - Obtención de acceso inicial como usuario `www-data`

3. **Escalada de Privilegios**
   - Enumeración efectiva de archivos de configuración sensibles
   - Extracción de credenciales desde `database.php`
   - Explotación de reutilización de contraseñas para obtener acceso root

4. **Captura de Flags**
   - User Flag: ✅ Obtenida desde `/home/www-data/flag.txt`
   - Root Flag: ✅ Obtenida desde `/root/root.txt`

---

## 🔍 Análisis de Vulnerabilidades

### Vulnerabilidades Críticas Identificadas

| Vulnerabilidad | CVE | Severidad | Impacto | Mitigación |
|----------------|-----|-----------|---------|------------|
| RCE en FUEL CMS 1.4.1 | CVE-2018-16763 | 🔴 Crítica | Ejecución remota de código sin autenticación | Actualizar a versión parcheada |
| Credenciales en texto plano | N/A | 🟠 Alta | Exposición de contraseñas de base de datos | Usar variables de entorno o vault |
| Reutilización de contraseñas | N/A | 🟠 Alta | Escalada de privilegios trivial | Implementar contraseñas únicas por servicio |
| Contraseña débil | N/A | 🟡 Media | Facilita ataques de fuerza bruta | Política de contraseñas robustas |
| Usuario root para MySQL | N/A | 🟡 Media | Violación del principio de mínimo privilegio | Usar usuario dedicado con permisos limitados |

---

## 💡 Lecciones Aprendidas

### 1. **La Importancia de las Actualizaciones**

El vector de ataque inicial fue posible debido al uso de **FUEL CMS 1.4.1**, una versión con una vulnerabilidad RCE conocida y públicamente documentada. Este escenario subraya la importancia crítica de:

- Mantener todo el software actualizado con los últimos parches de seguridad
- Suscribirse a boletines de seguridad de los productos utilizados
- Implementar un proceso de gestión de vulnerabilidades proactivo
- Realizar auditorías regulares del software desplegado

### 2. **Gestión Segura de Credenciales**

La escalada de privilegios fue trivial debido a credenciales almacenadas en texto plano dentro del archivo `database.php`. Las mejores prácticas incluyen:

- **Nunca** almacenar contraseñas en texto plano en archivos de configuración
- Utilizar gestores de secretos (Vault, AWS Secrets Manager, Azure Key Vault)
- Implementar variables de entorno para credenciales sensibles
- Aplicar cifrado en reposo para datos sensibles
- Rotar credenciales periódicamente

### 3. **El Peligro de la Reutilización de Contraseñas**

La contraseña `mememe` de la base de datos era idéntica a la del usuario root del sistema. Esta práctica representa un riesgo de seguridad masivo:

- Una única credencial comprometida puede escalar privilegios instantáneamente
- Implementar contraseñas únicas por servicio y usuario
- Utilizar gestores de contraseñas para generar y almacenar credenciales robustas
- Establecer políticas organizacionales que prohíban la reutilización

### 4. **Principio de Mínimo Privilegio**

El uso del usuario `root` de MySQL para la aplicación web viola el principio de mínimo privilegio:

- Crear usuarios de base de datos con permisos específicos y limitados
- El usuario web solo debería tener permisos sobre su propia base de datos
- Evitar usar cuentas administrativas para servicios de aplicación
- Implementar segregación de privilegios en todos los niveles

### 5. **Enumeración como Fase Crítica**

El éxito en este CTF dependió de una **enumeración exhaustiva**:

- Los archivos de configuración son objetivos de alto valor
- La información pública (README.md, robots.txt) puede revelar rutas críticas
- Cada archivo descubierto debe ser analizado cuidadosamente
- La paciencia y metodología sistemática son fundamentales

---

