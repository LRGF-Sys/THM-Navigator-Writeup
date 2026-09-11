# Fase 1: Reconocimiento Activo (Active Reconnaissance)

## 🎯 Objetivo de la Fase
El objetivo principal de esta fase fue interactuar directamente con la dirección IP del objetivo (`192.168.120.146`) para identificar los servicios web expuestos, analizar las cabeceras de respuesta HTTP del servidor y extraer cualquier información residual o metadatos que pudieran revelar vectores de ataque iniciales, nombres de dominio o identidades de usuarios.

---

## 1.1 Enumeración de Puertos y Servicios
Se realizó un escaneo inicial utilizando **Nmap** para identificar los servicios de red activos y las firmas de sus versiones en el objetivo `192.168.120.146`.

```bash
# Escaneo detallado de puertos específicos con scripts por defecto y detección de versiones
nmap -sV -sC -p 22,53,80 192.168.120.146 -vv -oA nmap/servicios
```

### Servicios Activos Identificados:
* **Puerto 22/TCP:** OpenSSH 7.9p1 (Debian 10)
* **Puerto 53/TCP:** Domain Name System (Servidor DNS)
* **Puerto 80/TCP:** Apache httpd 2.4.38

---

## 1.2 Reconocimiento Web y Filtración de Metadatos HTML (Puerto 80)
Se ejecutó **whatweb** contra el servicio web raíz en el puerto 80 para identificar las tecnologías utilizadas y examinar las cabeceras de respuesta HTTP.

```bash
# Huella digital (fingerprinting) del sitio web
whatweb http://192.168.120.146
```

### Análisis de Divulgación de Información:
Al inspeccionar los comentarios del código fuente HTML se detectó la filtración de un correo electrónico sensible:

* **Identidad Filtrada:** `denisse@navigator.hm`

Este hallazgo proporcionó dos vectores de ataque críticos:
1. **Candidato a nombre de usuario objetivo:** `denisse`
2. **Dominio interno de la organización:** `navigator.hm` (lo que indica una configuración de Virtual Hosting)

Adicionalmente, se inspeccionó el renderizado de la página web directamente desde la CLI:

```bash
# Extracción y conversión de HTML a texto plano
curl --url http://192.168.120.146 -s | html2text
```

---

## 1.3 Enumeración de Directorios y Recolección de Usuarios
Se lanzó un ataque de fuerza bruta de directorios contra `http://192.168.120.146` utilizando **Gobuster**.

```bash
# Fuerza bruta de directorios ocultando códigos de estado no deseados si es necesario
gobuster dir -u http://192.168.120.146 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -re
```

### Análisis del Endpoint Expuesto:
* **Ruta Descubierta:** `http://192.168.120.146/navabout/`

Al descargar e inspeccionar el archivo `navabout` mediante utilidades del sistema (`file navabout` & `cat navabout`), se reveló un nombre de usuario interno adicional: **Alek**.

---

## 1.4 Búsqueda Inversa DNS y Resolución de Nombres (Puerto 53)
Las consultas DNS directas iniciales para `navigator.hm` utilizando `nslookup` y `dig` devolvieron un estado de error `NXDOMAIN`. Para auditar los registros PTR de la zona DNS interna, se realizó una búsqueda inversa de la subred utilizando **dnsrecon**:

```bash
# Consulta de registros PTR locales a través del servidor DNS en el puerto 53
dnsrecon -n 192.168.120.146 -r 127.0.0.1/24
```

```text
Output:
[*] Performing Reverse Lookup from 127.0.0.0 to 127.0.0.255
[+] PTR navigator.hm 127.0.0.1
[+] 1 Records Found
```

### Mapeo de Host Virtual Local
Para asegurar que las peticiones HTTP llegaran al host virtual (Virtual Host) correcto, se añadió el dominio al archivo `/etc/hosts`:

```bash
# Redirección local del dominio hacia la IP objetivo
echo "192.168.120.146 navigator.hm" >> /etc/hosts
```

---

## 1.5 Compilación de Diccionarios Locales
Los nombres de usuario y dominios recolectados se guardaron en diccionarios locales para utilizarlos en las fases automatizadas del ataque:

```bash
# Creación de listas de palabras (wordlists) personalizadas
echo "denisse" >> users
echo "Alek" >> users
echo "navigator.hm" >> dominios
```

---

## 🛡️ Mapeo con MITRE ATT&CK
* **Táctica:** Reconnaissance ([TA0043](https://mitre.org))
* **Técnica:** Gather Victim Network Information - DNS internal record enumeration ([T1590.002](https://mitre.org)).
* **Técnica:** Gather Victim Identity Information - Email address harvesting via web comments and endpoint files ([T1589.002](https://mitre.org)).

---

## 💡 Mitigación y Robustecimiento Defensivo
1. **Sanitización del Código Fuente:** Eliminar los comentarios de los desarrolladores, direcciones de correo electrónico internas y nombres propios (`denisse`, `Alek`) del código HTML en producción.
2. **Restricción de Endpoints Expuestos:** Deshabilitar el acceso público a archivos internos o administrativos como `/navabout/`.
3. **Robustecimiento de DNS (DNS Hardening):** Restringir las búsquedas inversas PTR internas y las transferencias de zona (`AXFR`) únicamente a los rangos de direcciones IP internas autorizadas.
