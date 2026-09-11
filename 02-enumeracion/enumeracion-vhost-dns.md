# Fase 2: Enumeración de DNS y Hosts Virtuales (Virtual Hosts)

## 🎯 Objetivos de la Fase
El objetivo principal de esta fase fue descubrir endpoints ocultos interactuando directamente con la dirección IP del objetivo, analizar el servicio DNS expuesto para identificar mapeos de dominios internos (`navigator.hm`), evadir las restricciones de Virtual Hosting (VHost) mediante el mapeo de hosts locales y localizar el portal de gestión de la aplicación web principal.

---

## 2.1 Fuerza Bruta Inicial de Directorios vía IP en Bruto (`gobuster`)
Se ejecutó un ataque de fuerza bruta de directorios contra la dirección IP directa (`http://192.168.120.146`) utilizando **Gobuster** para enumerar recursos expuestos antes de realizar la resolución de dominio.

```bash
# Fuerza bruta de directorios en la IP raíz usando una lista mediana
gobuster dir -u http://192.168.120.146 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -re
```

### 🔍 Análisis de Hallazgos y Descubrimiento de Usuarios:

* **Endpoint Descubierto:** `http://192.168.120.146/navabout/`
* **Inspección del Archivo:** Al acceder a este endpoint se inició la descarga de un archivo plano llamado `navabout`. Se utilizaron utilidades de archivos del sistema para analizar su formato y contenido:

```bash
# Identificar el tipo de archivo e inspeccionar su contenido
file navabout
cat navabout
```

* **Resultado:** La inspección del contenido del archivo reveló un nombre de usuario de desarrollador interno adicional: **Alek**.
* **Actualización de la Lista de Palabras (Wordlist):**

```bash
# Agregar el nuevo usuario descubierto al diccionario local
echo "Alek" >> users
```

---

## 2.2 Interrogación DNS y Fuzzing Inverso PTR (Puerto 53)
Las consultas DNS directas para `navigator.hm` utilizando utilidades estándar fallaron con códigos de estado `NXDOMAIN`, lo que indicó que la resolución pública de registros tipo A no estaba configurada en el servidor DNS objetivo.

```bash
# Intento de resolución directa (Fallido: NXDOMAIN)
nslookup navigator.hm 192.168.120.146
dig a navigator.hm @192.168.120.146
```

Para auditar los registros de la zona local interna, se realizó un escaneo de búsqueda inversa PTR a través de la subred de loopback utilizando **dnsrecon**:

```bash
# Consulta de registros PTR locales a través del servidor DNS en el puerto 53
dnsrecon -n 192.168.120.146 -r 127.0.0.1/24
```

### 🔍 Análisis de la Salida (Output):
```text
[*] Performing Reverse Lookup from 127.0.0.0 to 127.0.0.255
[+] PTR navigator.hm 127.0.0.1
[+] 1 Records Found
```

Esto confirmó que el servidor web implementa enrutamiento por host virtual (Virtual Host Routing), lo que exige que las peticiones HTTP entrantes incluyan explícitamente la cabecera `Host: navigator.hm`.

---

## 2.3 Mapeo de Resolución de Nombres Local (/etc/hosts)
Para enrutar correctamente el tráfico del navegador y de las herramientas automatizadas, se añadió un mapeo estático de IP a dominio en el archivo `/etc/hosts` de la máquina atacante:

```bash
# Añadir mapeo estático para el Host Virtual
echo "192.168.120.146 navigator.hm" >> /etc/hosts

# Verificar la entrada en la configuración
cat /etc/hosts | grep navigator.hm
```

---

## 2.4 Fuzzing de Hosts Virtuales y Descubrimiento de Navigate CMS
Una vez establecida la resolución del dominio, se ejecutó **Gobuster** contra `http://navigator.hm` para localizar los endpoints internos de la aplicación web.

```bash
# Fuerza bruta de directorios utilizando el dominio del Host Virtual ya configurado
gobuster dir -u http://navigator.hm -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -re
```

### 🔍 Identificación de la Aplicación (Fingerprinting):
* **Endpoint Descubierto:** `http://navigator.hm/navigate`
* **Tecnología Identificada:** Navigate CMS v2.8 (Copyright 2024).
* **Impacto:** El acceso a la interfaz de gestión expuso un framework de aplicación web que es conocido por contener vulnerabilidades de ejecución remota de código (RCE) sin autenticación.

---

## 🛡️ Mapeo con MITRE ATT&CK
* **Táctica:** Discovery ([TA0007](https://mitre.org))
    * **Técnica:** Domain Trust Discovery ([T1482](https://mitre.org)) - Identificación y mapeo de nombres de dominio de la red interna mediante registros DNS PTR.
    * **Técnica:** Software Discovery: Web Application Discovery ([T1018](https://mitre.org) / [T1043](https://mitre.org)) - Enumeración de aplicaciones web activas y rutas ocultas mediante fuerza bruta de directorios.

---

## 💡 Mitigación y Robustecimiento Defensivo
1. **Restringir Consultas PTR y Transferencias DNS:** Configurar el servidor DNS para prevenir la enumeración mediante búsquedas inversas PTR y deshabilitar las transferencias de zona no autorizadas en las subredes internas.
2. **Implementar Bloques por Defecto (Catch-All) en el Servidor Web:** Configurar los *VirtualHosts* de Apache con bloques de respuesta por defecto que devuelvan un estado `403 Forbidden` o descarten directamente las peticiones que no incluyan cabeceras `Host` válidas.
