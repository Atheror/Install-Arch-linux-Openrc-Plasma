# Install-Arch-linux-Openrc-Plasma
Instalación Plasma 5 en Arch Linux openrc
Si desea usar BTRFS y LUKS(Home encriptado), puede revisar https://gist.github.com/ansulev/86240de242fa2dfd710b744247da6df3#file-inst-arch-openrc-btrfs-enc-home

## Descargar el ISO desde
Arch OpenRC: https://sourceforge.net/projects/archopenrc/

## Copiar en USB
dd if=archlinux-openrc-xxx-xx-xx.iso of=/dev/sdX bs=16M status=progress && sync

## Iniciar sesion desde la USB

`loadkeys es`

### Conexiòn wifi por wpa_supplicant
Crear el archivo `RED.conf` con el siguiente contenido

```
network={
ssid="MYSSID"
psk="MYPSK"
priority=3
}
```

Aplicar la configuración:

```
wpa_supplicant -B -i wlo1 -c WIFI.conf && dhcpcd wlo1
```

### Crear particiones
Crear particiones con fstab o cfdisk según la necesidad. En ésta guía se asume boot independiente en sda3, DATOS independientes en sda7 y root en sda6

### Crear sistema de archivos, sin swap

```
mkfs.ext2 /dev/sda3 # boot
mkfs.ext4 /dev/sda6 # root
```

### Montar las particiones

```
mkdir /mnt/
mount /dev/sda6 /mnt/
mkdir /mnt/boot
mount /dev/sda3 /mnt/boot
```

### Instalación del sistema base y coenxión wifi

```
pacstrap /mnt base base-devel grub vim wpa_supplicant terminus-font os-prober
```

### Generar el fstab

```
genfstab -p /mnt >> /mnt/etc/fstab
vi /etc/fstab
```
Agregar la particion en RAM para el directorio temporal y DATOS, En éste caso queda así:

```
/dev/sda6               /               ext4            rw,relatime,data=ordered        0 1
/dev/sda3               /boot           ext2            rw,relatime,block_validity,barrier,user_xattr,acl       0 2
/dev/sda7    /mnt/DATOS    xfs    defaults     0 0
tmpfs    /tmp    tmpfs    nodev,nosuid    0 0 
```

### ingresar al nuevo sistema

```
arch-chroot /mnt /bin/bash
```

#### Ajustar la hora y fecha

```
ln -s /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc --utc
```
#### Ajustar el nombre del equipo

```
echo hostname="MYHOSTNAME" > /etc/conf.d/hostname
```

#### Ajustar locales
En éste caso se usará manualmente ingles en lugar de generarlos pero se puede usar español como es_ES.UTF-8(España) es_CO.UTF-8(Colombia), etc segun aparecen en `/etc/locale.gen`.
El método recomendado es descomentar la línea de idioma(s) respectivo(s) en ``/etc/locale.gen`` y usar el comando `localegen`

```
echo LANG=en_US.UTF-8 >> /etc/locale.conf
echo LANGUAGE=en_US >> /etc/locale.conf
echo LC_ALL=C >> /etc/locale.conf
```
#### Ajustar disposición del teclado y fuente de consola
En caso de usar un teclado ne español latinamericano, usar `la-latin1` en lugar de `es` de España.

```
echo keymap=es >> /etc/conf.d/keymaps
echo consolefont=Lat2-Terminus16 >> /etc/conf.d/consolefont
```

#### Ajustar contraseña de root

```
passwd
```
#### Agregar un usuario real

```
useradd -m -g users -G lp,wheel,storage,optical,power,scanner,input,audio,video,network -s /bin/bash MYUSERNAME
passwd MYUSERNAME
```
#### Habilitar sudo para el grupo wheel descomentanto la respectiva línea
revisar el archivo y editar con:

```
visudo
```
Se puede descomentar ésta línea para que los usuarios del grupo wheel puedan escalar permisos con sudo:
`# %wheel ALL=(ALL) ALL` O se puede descomentar ésta línea para que además de escalar permisos con sudo no se pregunte por confirmación de contraseña: `# %wheel ALL=(ALL) NOPASSWD: ALL`

#### Regenerar imágen initrd

```
mkinitcpio -p linux
```
#### Reconfiguración de GRUB

```
grub-mkconfig -o /boot/grub/grub.cfg
grub-install /dev/sda
```

#### Salir y reiniciar en el sistema Arch Linux recién instalado

```
exit   # sale del chroot
umount -R /mnt  # Desmonta las particiones
reboot
```

## Configurando el Nuevo sistema Arch Linux OpenRC

### Instalando Servicios necesarios y modo gráfico base
```
pacman -S --needed acpid-openrc alsa-utils-openrc autofs-openrc syslog-ng-openrc cronie-openrc procps-ng-nosystemd avahi-openrc avahi-nosystemd cups-openrc hdparm-openrc autofs-openrc fuse-openrc haveged-openrc netifrc consolekit-openrc polkit-consolekit cgmanager-openrc udisks2-nosystemd samba-openrc displaymanager-openrc device-mapper-openrc lvm2-openrc desktop-privileges lxsession xorg-server xf86-video-intel xorg-utils xorg-xbacklight xorg-xinput xorg-xinit openbox xarchiver bash_completion unzip unrar p7zip mlocate intel-ucode ttf-dejavu htop xorg-server-utils xorg-utils xorg-xinit xorg-server mesa-nosystemd mesa-demos xorg-twm xorg-xclock xterm
```

### Se configuran los servicios
el comando para cada servicio sería `rc-update add SERVICIO default` para ahorrar tiempo se configuran todos los recién instalados así:

```
for daemon in acpid alsasound autofs dbus consolekit cronie cupsd xdm fuse haveged hdparm smb syslog-ng; do rc-update add $daemon default; done
```

### Probar el modo gráfico base
con `startx` se ha de iniciar twm con consolas xterm visibles, para terminar ejecutar `pkill X`

### Instalación de plasma5

```
pacman --noconfirm --needed -S plasma kde-applications ttf-dejavu ttf-liberation networkmanager-openrc networkmanager-consolekit sddm-consolekit libpulse-nosystemd
```

Luego de instalar plasma se habilita el servicio de red y el display manager:
```
rc-update add NetworkManager default
rc-update add sddm default
```

También es necesario editar `/etc/conf.d/xdm` y cambiar el DISPLAYMANAGER por sddm modificando la línea así:
`DISPLAYMANAGER="sddm"`

También es necesario editar /etc/
### instalación de pacaur

```
# Directorio para descarga del instalador y prerequisitos
mkdir -p /tmp/pacaur_install
cd /tmp/pacaur_install
sudo pacman --noconfirm --needed -S binutils make gcc fakeroot expac yajl git

# Instalación de "cower" desde AUR
curl -o PKGBUILD https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=cower
makepkg PKGBUILD --skippgpcheck
sudo pacman -U cower*.tar.xz --noconfirm

# Instalación de "pacaur" desde AUR
curl -o PKGBUILD https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=pacaur
makepkg PKGBUILD
sudo pacman -U pacaur*.tar.xz --noconfirm

# Limpieza del directorio del instalador...
cd ~
rm -r /tmp/pacaur_install
```
### Aplicaciones, elección personal

```
pacaur -S --noconfirm --needed --noedit bash-completion zsh zsh-completions \
mlocate keepassx2 bind-tools flashplugin pepper-flash google-chrome palemoon-bin uget \
uget-chrome-wrapper qupzilla gvfs-mtp libmtp mtpfs clamtk gimp inkscape \
blender dvdauthor cdrkit kdenlive scribus qownnotes ghostwriter pandoc \
libreoffice-fresh libreoffice-fresh-es geogebra klavaro virtualbox \
testdisk kodi sublime-text-dev openssh mpd cantata transmission-qt \
 apachedirectorystudio jdk8-openjdk gtk-recordmydesktop python \
vlc-nightly smplayer smplayer-skins smplayer-themes smtube mpv youtube-dl
```
