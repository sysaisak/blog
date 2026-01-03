+++
title = "Pickle rick"
author = ["aisak"]
date = 2026-01-01
draft = false
+++

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [Pickle Rick CTF - Writeup](#pickle-rick-ctf-writeup)
    - [Fase 1: Enumeración](#fase-1-enumeración)
        - [Escaneo de Red con Nmap](#escaneo-de-red-con-nmap)
        - [Hallazgos Web](#hallazgos-web)
    - [Fase 2: Fuzzing de Directorios y Archivos](#fase-2-fuzzing-de-directorios-y-archivos)
        - [Intento 1: Diccionario mediano (Sin extensiones)](#intento-1-diccionario-mediano--sin-extensiones)
        - [Intento 2: Diccionario común (Con extensiones .php, .txt, .html)](#intento-2-diccionario-común--con-extensiones-dot-php-dot-txt-dot-html)
    - [Flags](#flags)
        - [Limitación del Entorno](#limitación-del-entorno)
        - [Recolección de Ingredientes](#recolección-de-ingredientes)

</div>
<!--endtoc-->



## Pickle Rick CTF - Writeup {#pickle-rick-ctf-writeup}


### Fase 1: Enumeración {#fase-1-enumeración}


#### Escaneo de Red con Nmap {#escaneo-de-red-con-nmap}

```bash
nmap -A 10.66.176.13
```

-   **Puerto 22 (SSH):** OpenSSH 8.2p1. Útil si encontramos llaves RSA o contraseñas.
-   **Puerto 80 (HTTP):** Apache 2.4.41. Es el vector de ataque principal.


#### Hallazgos Web {#hallazgos-web}

-   Título de la página: "Rick is sup4r cool"
-   Notas: Al revisar el código fuente (Ctrl+U), se encontró un comentario con un nombre de usuario: `R1ckRul3s`.


### Fase 2: Fuzzing de Directorios y Archivos {#fase-2-fuzzing-de-directorios-y-archivos}

Al principio, un escaneo genérico solo reveló la carpeta `/assets`. Fue necesario realizar un segundo escaneo buscando extensiones específicas para encontrar la lógica de la aplicación.


#### Intento 1: Diccionario mediano (Sin extensiones) {#intento-1-diccionario-mediano--sin-extensiones}

-   Resultado: Pobre. Solo se detectó `/assets`.


#### Intento 2: Diccionario común (Con extensiones .php, .txt, .html) {#intento-2-diccionario-común--con-extensiones-dot-php-dot-txt-dot-html}

```bash
gobuster dir -u http://10.66.176.13 -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,txt,html
```

Resultados clave:

-   `/login.php`: Panel de autenticación.
-   `/portal.php`: Redirige a login, probablemente el panel de control tras autenticarse.
-   `/robots.txt`: Contenido Wubbalubbadubdub.
-   `/denied.php`


### Flags {#flags}

Tras obtener las credenciales (`R1ckRul3s` : `Wubbalubbadubdub`), accedí al panel en `/portal.php`. El panel permite la ejecución de comandos de sistema.


#### Limitación del Entorno {#limitación-del-entorno}

Al intentar listar y leer archivos, noté que el comando `cat` estaba deshabilitado.


#### Recolección de Ingredientes {#recolección-de-ingredientes}

1.  **Primer Ingrediente:** Localizado en el directorio actual.
    -   Comando: `ls`
    -   Lectura: `less "Sup3rS3cretPickl3Ingred1ent.txt"`

2.  **Segundo Ingrediente:** Explorando el sistema de archivos.
    -   Comando: `ls /home/rick`
    -   Lectura: `less /home/rick/second\ ingredients`.

3.  **Tercer Ingrediente:** Requiere privilegios de root.
    -   Verificación de privilegios: `sudo -l` (Permite ejecutar cualquier comando como root sin clave).
    -   Localización: `sudo ls /root`
    -   Lectura: `sudo less /root/3rd.txt`
