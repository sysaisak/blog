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

- [Enumeración Web](#enumeración-web)
- [De vuelta al User-Agent](#de-vuelta-al-user-agent)
    - [Contenido de la página](#contenido-de-la-página)
- [Usuario Chris](#usuario-chris)
- [Enumeración FTP](#enumeración-ftp)
- [Análisis de Imágenes con Exiftool](#análisis-de-imágenes-con-exiftool)
- [Contenido del ZIP](#contenido-del-zip)
- [Login como James y Enumeración](#login-como-james-y-enumeración)
- [Escalada de Privilegios](#escalada-de-privilegios)

</div>
<!--endtoc-->

La página web dice:

```bash
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R
```

Con curl podemos cambiar el user-agent:

```bash
curl -A "user-agent" website
```

No encontré nada cambiando el user-agent con el comando curl, solo aparece un mensaje cuando el user-agent es el agente R.


## Enumeración Web {#enumeración-web}

Procedo con la enumeración de la web, en este caso subdirectorios y archivos ocultos. No se encontró nada enumerando los directorios ni los archivos con gobuster.

```bash
gobuster dir -u [http://10.66.130.189/](http://10.66.130.189/) -w /usr/share/wordlists/dirb/big.txt -x php,html,js,md -t 200 -o filesHidden

gobuster dir -u [http://10.66.130.189/](http://10.66.130.189/) -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -o subDirs
```


## De vuelta al User-Agent {#de-vuelta-al-user-agent}

La pista estaba en activar la lógica del servidor siguiendo el redirect con la flag \`-L\` de curl:

```bash
curl -A "C" -L [http://10.66.130.189](http://10.66.130.189)
```

Retorna:

```bash
┌──(aisak㉿yuanjiao)-[~/pracs/thm/agent-sudo/content]
└─$ curl -A "C" -L [http://10.66.130.189](http://10.66.130.189)

Attention chris,
```


### Contenido de la página {#contenido-de-la-página}

> Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!
>
> From, Agent R


## Usuario Chris {#usuario-chris}

Ahora tenemos dos nuevos agentes, el agente Chris y el agente J. Como se sabe que el agente Chris tiene una contraseña débil, se procede a usar hydra para atacar el puerto 22/ssh:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ssh://10.66.130.189
```

El ataque a ssh no es exitoso, no se encuentran las credenciales. Pienso en atacar el puerto 21/ftp:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.66.130.189
```


## Enumeración FTP {#enumeración-ftp}

Hydra encontró las siguientes credenciales:

**Host**: 10.66.130.189
**Login**: chris
**Password**: crystal

Por lo que ahora podremos loguearnos al servidor FTP con el usuario chris.

![](/blog/image/ftp-enum.png)
![](/blog/image/content-letter-ftp.png)

En la carta, el agente C le dice al agente J que su contraseña está escondida en las imágenes que encontramos en el servidor FTP.

{{< figure src="/blog/image/getting-all-files-ftp.png" >}}


## Análisis de Imágenes con Exiftool {#análisis-de-imágenes-con-exiftool}

Traté de usar exiftool en las dos imágenes. En la primera, `cutie`, encontré que al final del PNG hay archivos Binarios Tailored que interpreto como "atados".

Intenté usar steghide, pero dijo que el tipo de archivo era incompatible. La otra imagen, `cute-alien.jpg`, sí era compatible pero nos faltaba la passphrase.

{{< figure src="/blog/image/binaries-in-img.png" >}}

Luego de buscar en Google qué hacer, encontré binwalk. Antes de usarlo probé la utilidad `strings`, pero solo encontré `To_agentR.txt` al final del output.

{{< figure src="/blog/image/strings-img.png" >}}

La extracción de la imagen pudo hacerse:
![](/blog/image/content-of-cutieimg.png)

Se extrajeron 3 archivos, el más importante parece ser `8702.zip`.

```bash
$ file 8702.zip
8702.zip: Zip archive data, made by v6.3 UNIX, extract using at least v5.1, last modified Oct 29 2019 20:29:12, uncompressed size 86, method=AES Encrypted
```

Es un .zip encriptado, por lo que se necesita una passphrase para poder ver el contenido. Al pedirle una pista a TryHackMe nos sugiere "Mr. John"; entonces busco "¿Brute force passphrase with john the ripper?" y encuentro la utilidad `zip2john` para crear el hash.

{{< figure src="/blog/image/zip2john.png" >}}

Sigo el paso a paso y uso el diccionario `rockyou.txt`.

{{< figure src="/blog/image/john-in-action.png" >}}

Nos dice que la passphrase para el archivo zip es `alien`.


## Contenido del ZIP {#contenido-del-zip}

El contenido del archivo zip es una carta al Agente R, dice lo siguiente:

{{< figure src="/blog/image/to-agen-r-zip.png" >}}

Parece que hay que romper otro código: `QXJlYTUx`. Lo que siempre intento es comprobar si es [[[<https://es.wikipedia.org/wiki/Base64>](<https://es.wikipedia.org/wiki/Base64>)][Base64]].

{{< figure src="/blog/image/base65-decode.png" >}}

Después de decodificar con base64, nos queda `area51`. Esta debe ser la contraseña para la otra imagen que se descargó del FTP.

{{< figure src="/blog/image/agent-james.png" >}}

Encontramos una contraseña para el agente James; supongo que ya es tiempo de acceder al servicio ssh como `james`.


## Login como James y Enumeración {#login-como-james-y-enumeración}

Se encuentra en `/home/james/` lo siguiente:
![](/blog/image/enum-james.png)

Encontramos la primera flag llamada `user.txt` y una imagen que parece ser la autopsia de un alien. Con `rsync` podemos traernos el archivo a la máquina atacante.

{{< figure src="/blog/image/rsync-to-alien.png" >}}

La imagen encontrada es necesaria para una de las tareas de TryHackMe. Mientras escribo el writeup me he dado cuenta de que probablemente he pasado por alto algunas de las tareas.

La imagen en cuestión:
</blog/image/content>

La tarea es "¿Cómo se llama el incidente de la foto?". En uno de los hints nos dicen que se haga búsqueda inversa de la imagen y se use Fox News, algo que se puede hacer usando Google Search.

{{< figure src="/blog/image/google-image-search.png" >}}

Luego de adjuntar la imagen se encuentran las palabras clave **Roswell Incident**. Mi otra búsqueda es "Roswell Incident Fox News" y me encuentro con el titular: "Filmmaker reveals how he faked infamous 'Roswell alien autopsy' footage in a London apartment". La flag está ahí dentro.


## Escalada de Privilegios {#escalada-de-privilegios}

¿Qué nos queda? Nos hace falta escalar privilegios en la máquina. Antes de empezar con linpeas, algo que hago es usar `sudo -l` y después buscar los binarios SUID.

{{< figure src="/blog/image/sudo-dash-l.png" >}}

Esta es la parte donde las neuronas por alguna razón se activan; eso luce muy mal.

{{< figure src="/blog/image/meme.jpg" >}}

El exploit afortunadamente para nosotros es conocido y documentado, puede encontrarse como **CVE-2019-14287**.

`/etc/sudoers`, que es lo que se ve al hacer `sudo -l`, nos dice que podemos ejecutar `/bin/bash` como cualquier usuario menos root, por eso la negación `!root`. Se le pasa como UID el número -1; la versión vulnerable de sudo no hace las validaciones correctas y -1 se convierte en 0 (el UID del usuario root).

{{< figure src="/blog/image/ending.png" >}}
