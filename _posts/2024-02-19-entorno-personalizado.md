---
layout: single
title: Personalización de Entorno en Kali Linux para Pentesting
excerpt: "¿Te gustaría aprender sobre Cómo crear un entorno de trabajo personalizado?, pues vamos a ello."
date: 2024-02-19
classes: wide
header:
  teaser: /assets/images/entorno-personalizado/entorno-personalizado.png
  teaser_home_page: true
categories:
  - kali linux
  - bspwm
  - polybar
  - kitty
  - p10k
  - zsh
  - picom
  - nvim

tags:
  - entorno personalizado
  - kali linux personalizado
---

<p align="center">
<img src="/assets/images/entorno-personalizado/entorno-personalizado.png">
</p>

# Personalización de Entorno en Kali Linux para Pentesting

## Resumen

Este tutorial te guiará paso a paso a través de la personalización de tu entorno en Kali Linux, utilizando herramientas como bspwm, sxhkd, polybar, Powerlevel10k, zsh, y Kitty, etc. Estas personalizaciones están diseñadas para mejorar la eficiencia y hacer más cómodas las tareas de pentesting. Créditos para [s4vitar](https://twitter.com/s4vitar) ya que es el creador del entorno y quien nos aporta mucho a la comunidad de ciberseguridad.

## Dependencias Necesarias

- **Kali Linux instalado**: Asegúrate de tener Kali Linux instalado en tu máquina.
- **Conexión a internet**: Necesitarás acceso a Internet para descargar e instalar paquetes.
- **Archivos necesarios para la configuración**: Estos archivos están en un comprimido y organizados por carpetas, cada una con sus respectivos nombres, para que se puedan guiar y no se confundan a la hora de copiarlos. [Descargar](https://ewor01.github.io/assets/files/entornoPersonalizadoKaliLinux.rar)

## Pasos

## Actualización del sistema operativo

Actualizaremos nuestro sistema operativo mediante los siguientes comandos:

```bash
sudo apt update
sudo apt upgrade
```

instalamos paquetes necesarios:

```bash
apt install build-essential git vim libxcb-util0-dev libxcb-ewmh-dev libxcb-randr0-dev libxcb-icccm4-dev libxcb-keysyms1-dev libxcb-xinerama0-dev libasound2-dev libxcb-xtest0-dev libxcb-shape0-dev
```

Luego si todo esta bien, nos colocamos como usuario sin privilegios y clonamos los siguientes repositorios en nuestra carpeta de descargas.

```bash
git clone https://github.com/baskerville/bspwm.git
git clone https://github.com/baskerville/sxhkd.git
```

Entramos en cada directorio y hacemos los siguientes pasos para ambos:

```bash
cd bspwm
make
sudo make install

cd sxhkd
make
sudo make install
```

Por si hay algún problema con el paquete libxinerama instalarlo aparte:

```bash
sudo apt install libxinerama1 libxinerama-dev
```

Creamos los directorios del bspwm y sxhkd dentro del directorio config:

```bash
mkdir ~/.config/{bspwm,sxhkd}
```

Ahora nos copiamos el archivo bspwmrc que esta dentro del directorio bspwm/examples:

```bash
cp bspwmrc ~/.config/bspwm
```

Hacemos lo mismo para el sxhkdrc que esta en la misma ruta:

```bash
cp sxhkdrc ~/.config/sxhkd
```

## Instalamos y Configuramos kitty

Procedemos a instalar la kitty la cual usaremos como terminal

```bash
sudo apt install kitty
```

## Modificando el sxhkdrc

Luego en el sxhkdrc sustituimos la terminal por la kitty usando la ruta absoluta:

```bash
which kitty
/usr/bin/kitty
```

Sustituimos esto en el archivo como lo mencionamos arriba y debería quedar así:

```bash
# terminal emulator
super + Return
        /usr/bin/kitty
```

Modificaremos a nuestro gusto el sxhkd solo agregaremos una configuración y una carpeta que crearemos dentro de ~/.config/bspwm que se llame scripts, el nombre del script sera bspwm_resize le daremos permisos de ejecución y lo llamaremos desde el sxhkdrc:

```bash
# Custom move/resize
alt + super + {Left,Down,Up,Right}
        /home/tu_usuario/.config/bspwm/scripts/bspwm_resize {west,south,north,east}
```

El contenido del script es el siguiente, este script nos permitirá redimensionar los tamaños de las ventanas:

```bash
#!/usr/bin/env dash

if bspc query -N -n focused.floating > /dev/null; then
        step=20
else
        step=100
fi

case "$1" in
        west) dir=right; falldir=left; x="-$step"; y=0;;
        east) dir=right; falldir=left; x="$step"; y=0;;
        north) dir=top; falldir=bottom; x=0; y="-$step";;
        south) dir=top; falldir=bottom; x=0; y="$step";;
esac

bspc node -z "$dir" "$x" "$y" || bspc node -z "$falldir" "$x" "$y"
```

Si nos da un problema cuando estamos como root hacemos control c cuando tire el error y luego ponemos compaudit y la ruta que nos de se la asignamos a root:

```bash
chown root:root y la ruta
```

Instalamos otros paquetes:

```bash
sudo apt install cmake cmake-data pkg-config python3-sphinx libcairo2-dev libxcb1-dev libxcb-util0-dev libxcb-randr0-dev libxcb-composite0-dev python3-xcbgen xcb-proto libxcb-image0-dev libxcb-ewmh-dev libxcb-icccm4-dev libxcb-xkb-dev libxcb-xrm-dev libxcb-cursor-dev libasound2-dev libpulse-dev libjsoncpp-dev libmpdclient-dev libuv1-dev libnl-genl-3-dev
```

## Instalación de polybar:

Luego nos vamos a nuestro directorio de descargas y nos clonamos el siguiente repositorio para instalar la polybar:

```bash
git clone --recursive https://github.com/polybar/polybar](https://github.com/polybar/polybar
```

Entramos al directorio de la polybar y creamos un directorio llamado build, entramos al directorio y hacemos cmake:

```bash
mkdir build
cd build
cmake ..
```

Hacemos lo siguiente:

```bash
make -j$(nproc)
```

Ahora hacemos un sudo make install y queda lista la polybar:

```bash
sudo make install
```

Instalamos otros paquetes necesarios:

```bash
apt install meson libxext-dev libxcb1-dev libxcb-damage0-dev libxcb-xfixes0-dev libxcb-shape0-dev libxcb-render0-dev libxcb-composite0-dev libxcb-image0-dev libxcb-present-dev libxcb-xinerama0-dev libpixman-ls1-dev libdbus-1-dev libconfig-dev libgl1-mesa-dev libpcre2-dev libevdev-dev uthash-dev libev-dev libx11-xcb-dev libxcb-glx0-dev
```

Clonamos el siguiente repositorio para instalar picom y jugar con las transparencias:

```bash
git clone https://github.com/ibhagwan/picom.git
```

Entramos al directorio y escribimos los siguientes comandos:

```bash
git submodule update --init --recursive
meson --buildtype=release . build
```

Si nos da algún error de deprecated instalamos el siguiente paquete:

```bash
sudo apt install libpcre3 libpcre3-dev
```

luego ya todo tiene que estar correcto, hay un error de libev pero no afecta.

Continuamos ejecutando los siguientes comandos:

```bash
ninja -C build
sudo ninja -C build install
```

Instalamos rofi y bspwm:

```bash
sudo apt install rofi
sudo apt install bspwm
```

Probamos el entorno que todo vaya bien, hacemos un kill -9 -1 y entramos pero antes de iniciar sesión debemos seleccionar bspwm como entorno. Ya probado que todo nos funciona bien, procedemos con la instalación de la fuente.

## Instalación de la fuente

Nos dirigimos a la pagina para descargar la fuente llamada (hack nerd font):
[Ver pagina](https://www.nerdfonts.com/font-downloads)

Nos situamos en el directorio /usr/local/share/fonts y nos ponemos como super usuario para mover el archivo descargado a ese directorio:

```bash
mv /home/su nombre de usuario/Downloads/Hack.zip .
```

luego hacemos unzip para descomprimir el contenido y borramos el comprimido:

```bash
unzip Hack.zip
rm Hack.zip
```

Hacemos que nos funcione la clipboard bidireccional agregando este comando al final del archivo bspwmrc:

```bash
vmware-user-suid-wrapper &
```

## Configuración de la kitty

Copiaremos 2 archivos que meteremos dentro de ~/.config/kitty
uno es color.ini y el otro es kitty.conf (los cuales estan en los archivos que les deje al inicio).

Nos ponemos como root y como el tiene otra kitty nos pasamos los archivos de configuración

```bash
cp /home/su usuario/.config/kitty/* .
```

Instalamos feh:

```bash
sudo apt install feh
```

Nos creamos una carpeta dentro del escritorio con nuestro nombre de usuario luego otra que se llame Fondos y ahi dentro colocamos el fondo que queremos tener
ahora agregamos este comando al bspwmrc:

```bash
feh --bg-fill /home/ewor01/Desktop/ewor01/Fondos/fondo.png &
```

Instalamos un paquete para visualizar imágenes desde la kitty:

```bash
sudo apt install imagemagick
```

Para visualizar una imagen desde la kitty utilizamos el siguiente comando:

```bash
kitty +kitten icat imagen.jpg
```

Instalamos scub para eliminar archivos y hacer mas difícil el rescate de estos:

```bash
sudo apt install scrub
```

## Configuración de la polybar

Ahora nos dirigimos al directorio Downloads y clonamos el siguiente repositorio:

```bash
git clone https://github.com/VaughnValle/blue-sky.git
```

Creamos el directorio de polybar y copiamos todo dentro del directorio situados dentro de /blue-sky/polybar:

```bash
mkdir ~/.config/polybar
cp -r * ~/.config/polybar
```

Ponemos este comando en el bspwmrc de la siguiente manera: 

```bash
echo '~/.config/polybar/./launch.sh' >> ~/.config/bspwm/bspwmrc
```

Nos dirigimos dentro del directorio blue-sky/polybar/fonts y copiaremos las fuentes en /usr/share/fonts/truetype como super usuario:

```bash
sudo cp * /usr/share/fonts/truetype
```

Aplicamos este comando:

```bash
fc-cache -v
```

Configuramos la polybar y todos sus módulos.
creamos un directorio llamado picom dentro de ~/.config como usuario normal:

```bash
mkdir picom
```

Nos pasamos el archivo picom.conf y ahora configuramos picom a nuestro gusto si queremos blur o no y luego en kitty.conf agregamos lo siguiente dependiendo la opacidad que queramos:

```bash
background_opacity 0.95
```

## Modificando la polybar

Buscamos font-6 en current.ini de los archivos de la polybar de nuestra configuración y agregamos estas líneas para que nos salga el icono que queramos en la parte izquierda:

```bash
"font-7 = "Hack Nerd Font Mono:size=35;8.0"
```

Estos valores ya dependerá del gusto y icono que coloquen, pero para que funcione agregar esas líneas debajo de font-6.

## Privacidad de firefox y algunos plugins

Si queremos vamos a hacer que firefox no recuerde nuestro historial mas que todo es depende las opciones de cada persona si gustan lo hacen o no.
Nos dirigimos a Settings > Privacy & Security > Luego buscamos donde dice History y colocamos never remember history, reinician y listo.

Instalamos un plugin llamado darkreader en firefox para colocar modo oscuro en las paginas que queramos.
Instalamos otro plugin llamado wappalyzer que nos ayudara en pentesting web a identificar las tecnologías que utilizan las webs.

## Instalación de powerlevel10k

Instalamos la Powerlevel10k
nos dirigimos al repositorio:
[ver repositorio](https://github.com/romkatv/powerlevel10k))
y buscamos las instrucciones donde dice Manual y hay un comando que se los dejo igual:

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
echo 'source ~/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```

Esto hay que hacerlo tanto como usuario normal y como root. Una vez clonada procedemos a instalarla:
colocamos zsh y comenzamos a configurar a nuestro gusto.

## Configurando p10k

Ahora retocamos alguna configuracion del archivo .p10k.zsh

Si queremos poner un icono diferente en la izquierda buscamos por esta linea que les dejo y colocamos el icono que queramos:

```bash
typeset -g POWERLEVEL9K_OS_ICON_CONTENT_EXPANSION=''
```

Hacemos la instalacion de la powerlevel10k para root:

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
echo 'source ~/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```

Modificamos el .p10k.zsh de root después de configurar la power level y hay una opción para que no nos ponga tanto texto cuando somos root, buscamos esta linea y borramos lo que esta entre comillas  simples:

```bash
typeset -g POWERLEVEL9K_CONTEXT_PREFIX=''
```

Modificamos esta para poner un icono y saber cuando somos root en lugar de texto:

```bash
typeset -g POWERLEVEL9K_CONTEXT_ROOT_TEMPLATE='icono'
```

## Enlace simbólico zshrc

Crearemos un enlace simbólico de la .zshrc para que sea la misma en root y usuario normal,, lo hacemos estando como root:

```bash
ln -s -f /home/su usuario/.zshrc .zshrc
```

Hacemos un cambio en la .zshrc para que nos cargue la del usuario normal y no de problemas,, agregamos esto:

```bash
source /home/su usuario/powerlevel10k/powerlevel10k.zsh-theme
```

Modificamos algunas cosas del .zshrc en la opcion history configuration:
dejamos solo las primeras 3 opciones y aumentamos a 10000 los valores del histórico y agregamos estas líneas al final:
setopt histignorealldups sharehistory
Para que cuando mandemos una cadena vacía ejemplo: 

```bash
function echo '' > ~/.zsh_history 
```

se borren los comandos que hayamos ejecutado antes.

Ahora quitaremos la negrita que nos aparece cuando navegamos por directorios en el lado izquierdo, lo haremos para root y usuario normal:
Modificamos el .p10k.zsh y filtramos por ANCHOR_BOLD y donde pone true colocamos false. Hacemos lo mismo para el usuario root

## Instalando batcat y lsd

Descargamos batcat que es como un cat pero con esteroides, desde este repo y bajamos la versión más reciente:
[Ver repositorio](https://github.com/sharkdp/bat/releases)
bat_0.24.0_amd64.deb

Descargamos lsd que es com un ls pero con esteroides, desde este repo y bajamos la versión más reciente:
[Ver repositorio](https://github.com/lsd-rs/lsd/releases)
lsd_1.0.0_amd64.deb

Para instalarlos solo vamos a la carpeta donde decargamos y ponemos el siguiente comando:

```bash
dpkg -i nombre_del_paquete
```

Instalamos locate:

```bash
sudo apt install locate
```

Vamos a sincronizar todos los archivos del sistema para buscar lo que queremos:

```bash
updatedb
```

## Instalando fzf

Ahora instalamos la fzf para hacer busquedas inteligentes en el sistema, desde el siguiente repo:

[Ver repositorio](https://github.com/junegunn/fzf)

Para instalar lo hacemos como root y usuario normal en el home de cada uno:

```bash
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

## Alias para bat y lsd

Ahora copiamos unos alias para nuestra .zshrc de bat y lsd:
[Ver link](https://pastebin.com/QGvVx3wG)

Tambien copiamos este archivo para que no nos salga en negrita a la hora de listar con lsd lo colocamos casi al final de la .zshrc:
[Ver link](https://pastes.io/xsqk9oi1hz)

Ahora vamos a arreglar un problema que da con burpsuite al estar en bspwm, agregamos estas líneas casi al principio de nuestro archivo, pero para que nos funcione burpsuite debemos correrlo desde la terminal no desde el shortcut del sxhkdrc ya que siempre presentara el problema:

```bash
# Fix the Java Problem
export _JAVA_AWT_WM_NONREPARENTING=1
```

Tambien en el bspwmrc agregamos lo siguiente al final para arreglar el mismo problema:

```bash
wmname LG3D &
```

## Creando módulos para la polybar

Creamos unos módulos para la polybar para lo cual tenemos que crear una carpeta llamada bin en ~/.config:

```bash
mkdir bin
```

Una vez creada aquí dentro pondremos los scripts que ocuparemos para cargar en los módulos de la polybar.

Para agregar un modulo tenemos que modificar el archivo current.ini y también el launch.sh, en el current.ini agregamos los módulos y en el launch.sh agregamos la ruta del modulo para que aparezca en la polybar.

Ejemplo de un modulo, este va casi al principio del current.ini donde esten los otros modulos:

```bash
[bar/ethernet_bar]
inherit = bar/main
width = 10%
height = 40
offset-x = 11.1%
offset-y = 15
bottom = false
font-7 = "Hack Nerd Font Mono:size=22;5"
foreground = ${color.white}
background = ${[color.bg](http://color.bg/)}
padding = 1
modules-center = ethernet_status
wm-restack = bspwm
```

Este otro casi al final donde están los demás módulos:

```bash
[module/ethernet_status]
type = custom/script
interval = 2
exec = ~/.config/bin/ethernet_status.sh
```

Este seria el contenido de nuestro script para el ethernet_status en ~/.config/bin de la carpeta que creamos anteriormente:

```bash
#!/bin/sh

echo "%{F#2495e7}󰈀 %{F#ffffff}$(/usr/sbin/ifconfig eth0 | grep "inet " | awk '{print $2}')%{u-}"
```

Luego en el launch.sh agregaremos esto para que nos cargue el modulo, El archivo tendrá los otros módulos que trae por default si no queremos que muestre uno solo comentamos:
Ejemplo launch.sh:

```bash
## Left bar

#polybar log -c ~/.config/polybar/current.ini &
polybar secondary -c ~/.config/polybar/current.ini &
polybar ethernet_bar -c ~/.config/polybar/current.ini &
```

## Sudo plugin

Ahora instalaremos el sudo plugin, para hacer doble ESC nos ponga sudo antes.
para agregarlo nos dirigimos a /usr/share/ y aquí adentro crearemos una carpeta todo lo hacemos como root:

```bash
mkdir zsh-sudo-plugin
```

nos metemos al directorio y luego descargamos el raw:

```bash
cd /zsh-sudo-plugin/
wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/plugins/sudo/sudo.plugin.zsh
```

Una vez descargado le asignamos permisos de ejecución:

```bash
chmod +x nombre de plugin
```

Y le asignan como propietario su usuario:

```bash
chown usuario:usuario plugin
```

Agregamos este código a nuestra .zshrc:

```bash
if [ -f /usr/share/zsh-sudo-plugin/sudo.plugin.zsh ]; then
source /usr/share/zsh-sudo-plugin/sudo.plugin.zsh
fi
```

Ahora modificamos la polybar para identificar que ventana esta activa, inactiva y vacia.
Modificaremos el workspace.ini de la polybar. Buscamos las siguientes líneas:

```bash
label-active-foreground = ${color.green}
```

sustituimos el color de active y mas abajo esta la de ocupada, colocamos el color que mejor nos parezca.

## Auto completado inteligente

Instalamos un autocompletado inteligente en nuestro archivo de .zshrc, que esta en este link:
[Ver link](https://pastebin.com/H87J3nMj)
Copiamos y pegamos lo del archivo en nuestra .zshrc, filtramos por completion features y borramos las que ya trae y pegamos las nuevas.

Ahora instalaremos npm para instalar neovim de neochad:

```bash
sudo apt install npm
```

Procedemos a clonar el repositorio para tener nvim:

```bash
git clone https://github.com/NvChad/NvChad ~/.config/nvim --depth 1
```

Ahora entramos en esta url y buscamos el que pone nvim-linux64.tar.gz en esta url y lo descargamos:
[Ver link](https://github.com/neovim/neovim/releases)

Nos movemos al directorio opt, nos ponemos como root movemos y descomprimimos el archivo en ese directorio:

```bash
cd /opt
sudo su
mv /home/usuario/Downloads/nvim-linux64.tar.gz .
```

Borramos el comprimido y entramos en la carpeta de nvim, luego en bin y ejecutamos como usuario normal el nvim:

```bash
cd nvim
cd bin
su usuario
./nvim
```

 y se instalara todo correctamente.

Ahora modificamos nuestro archivo .zshrc y agregamos estas líneas en custom aliases:

```bash
#nvim
alias nvim='/opt/nvim-linux64/bin/nvim'
```

 
y con esto ya tendríamos nvim instalado, solo cerramos la ventana y abrimos una nueva para probar.

Ahora hacemos lo mismo para el usuario root, volvemos a hacer el git clone en el home de root:

```bash
git clone https://github.com/NvChad/NvChad ~/.config/nvim --depth 1
```

Ejecutamos nvim y se instalara todo correctamente.

## Instalando flameshot

Como opcional podemos instalar flameshot que sirve para hacer capturas de pantalla personalizadas.

```bash
sudo apt install flameshot
```

## Instalando i3lock y i3lock-fancy

```bash
sudo apt install i3lock

```

Luego vamos a este github y vemos las instrucciones para instalar:
[Ver link](https://github.com/meskarune/i3lock-fancy)
Primero nos dirigimos a Downloads y clonamos el repo:

```bash
cd /home/usuario/Downloads
git clone https://github.com/meskarune/i3lock-fancy
```

Luego entramos al directorio y hacemos un sudo make install y listo. 

Ejecutamos este comando para darle un texto y un tipo de fuente:

```bash
i3lock-fancy -t "Introduce tu contraseña" -f /usr/local/share/fonts/HackNerdFont-Regular.ttf
```

Con esto ya lo tendríamos.

Creamos un shortcut en en el sxhkdrc para activarlo con un atajo haciendo uso de la ruta absoluta:

```bash
/usr/bin/i3lock-fancy -t "Introduce tu contraseña" -f /usr/local/share/fonts/HackNerdFont-Regular.ttf
```

Después de esto reiniciamos el entorno para aplicar los cambios.

Listo con esto tenemos nuestro entorno listo para trabajar.

## Happy hacking.
