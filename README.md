# TryHackMe: Navigator — Writeup y Guía de Pruebas de Penetración

## 📝 Resumen Ejecutivo (Executive Summary)
Este repositorio contiene una documentación detallada y guiada por metodología del proceso de pruebas de penetración (Writeup) realizado para el laboratorio "Navigator" en la plataforma TryHackMe. El objetivo de esta evaluación fue identificar de manera sistemática los fallos de seguridad, lograr el acceso inicial mediante vulnerabilidades en aplicaciones web y realizar una escalada de privilegios para obtener el control administrativo total (`root`) sobre el sistema objetivo.

La evaluación se estructuró siguiendo estrictamente el Estándar de Ejecución de Pruebas de Penetración (**PTES**) y las técnicas ejecutadas fueron mapeadas directamente con el framework **MITRE ATT&CK®**.

---

## 🛡️ Hallazgos Clave y Vulnerabilidades Explotadas
* **Divulgación de Información mediante Código Fuente y Restricciones de Hosting Virtual:** Se filtraron direcciones de correo electrónico corporativas internas (`denisse@navigator.hm`) a través de comentarios HTML ocultos. Además, el servidor web estaba configurado con un *Virtual Hosting* estricto, descartando las peticiones directas realizadas a la dirección IP cruda.
* **Ejecución Remota de Código (RCE) No Autenticada en Navigate CMS:** El objetivo alojaba una instancia desactualizada de **Navigate CMS v2.8**, la cual contenía un fallo crítico que permitía a atacantes remotos no autenticados ejecutar comandos arbitrarios en el sistema mediante frameworks de explotación automatizados.
* **Configuración Débil de Permisos SUID (Escalada de Privilegios):** La cuenta de usuario local poseía capacidades excesivas de ejecución de binarios del sistema. El binario `php7.3` estaba configurado con el **bit SUID activo**, permitiendo a usuarios sin privilegios evadir los controles de seguridad locales y generar una shell como administrador.

---

## 🗺️ Ciclo de Vida del Ataque y Estructura del Repositorio
La evidencia técnica y los procedimientos paso a paso están documentados sistemáticamente en sus respectivos archivos dentro de este repositorio:

### 🔍 [Fase 1: Reconocimiento (Reconnaissance)](01-reconocimiento/reconocimiento-activo.md)
* **Objetivo:** Mapeo de la infraestructura objetivo y verificación inicial de los límites del sistema.
* **Herramientas:** `whatweb`, `curl`.
* **Resultado Clave:** Extracción del nombre de dominio interno de la organización (`navigator.hm`) y descubrimiento de la cuenta de usuario `denisse` a través de comentarios HTML ocultos.

### 📂 [Fase 2: Enumeración (Enumeration)](02-enumeracion/enumeracion-vhost-dns.md)
* **Objetivo:** Resolución de subdominios, fuerza bruta de directorios activos y firma/identificación de software.
* **Herramientas:** `gobuster`, `dnsrecon`, `dig`, `/etc/hosts`.
* **Resultado Clave:** Resolución de zonas DNS locales mediante búsquedas inversas (`PTR`), bypass del bloqueo de Host Virtual y localización del portal administrativo oculto de **Navigate CMS v2.8**.

### ⚡ [Fase 3: Explotación (Exploitation)](03-explotacion/movimiento-lateral-rce.md)
* **Objetivo:** Compromiso inicial del servidor y movimiento lateral.
* **Herramientas:** `msfconsole (Metasploit)`, `crackmapexec`, `ssh`.
* **Resultado Clave:** Obtención de una shell web con bajos privilegios (`www-data`) mediante un exploit RCE no autenticado, y ejecución de un ataque de *password spraying* por SSH para pivotar a una shell de usuario estable como `denisse`.

### 👑 [Fase 4: Post-Explotación (Post-Exploitation)](04-post-explotacion/elevacion-privilegios-suid.md)
* **Objetivo:** Escalada de privilegios locales y obtención de la bandera de root.
* **Herramientas:** `linpeas.sh`, `python3 (servidor HTTP)`, `GTFOBins`.
* **Resultado Clave:** Ejecución de enumeración automatizada en el host, localización de un binario mal configurado con permisos SUID en `php7.3` y abuso de sus parámetros interactivos para desplegar una shell de root.

---

## 🛠️ Buenas Prácticas de Ciberseguridad Demostradas
* **Documentación Técnica Modular:** División de cadenas de explotación complejas en módulos técnicos independientes y legibles, accesibles tanto para equipos de ingeniería como para áreas de gestión.
* **Evasión de Controles de Resolución de Nombres:** Demostración práctica de cómo modificar archivos locales de resolución de nombres (`/etc/hosts`) para evadir defensas industriales comunes como la validación de cabeceras de host (*host-header validation*).
* **Abuso de Binarios Nativos (Living-off-the-Land Binaries - LoLBins):** Utilización de programas legítimos del sistema operativo (PHP) para elevar privilegios sin necesidad de introducir malware externo compilado, evadiendo los mecanismos básicos de detección de firmas.

---
*Aviso legal: Este repositorio tiene fines estrictamente educativos y de demostración de portafolio profesional. Todas las pruebas fueron realizadas dentro de un entorno de laboratorio legal y controlado (TryHackMe).*
