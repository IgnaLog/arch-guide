# Guía de Instalación de Arch Linux

![Arch Linux logo](https://raw.githubusercontent.com/archlinux/.github/6b33d3e9e2a522f73c8a65dc68d504e5deff6ad2/profile/archlinux-logo-dark-scalable.svg)

## Comprobaciones previas

Cambiar el teclado a español:

```shell
loadkeys es
```

Comprobar el estado de la red:

```shell
ping -c 1 google.es
```

Comprobar que el reloj del hardware está correcto:

```shell
timedatectl status
```

Si no es correcto actualizarlo:

```shell
timedatectl set-ntp true
```

## Creamos particiones

Listar nuestros discos duros:

```shell
lsblk
```

Abrir el programa de particionamiento especial para discos GPT:

```shell
cgdisk /dev/nvme0n1
```

Sustituye `nvme0n1` por el disco duro que vas a particionar.
Particionamiento que yo uso:

| #   | Partition           | Description     | Hex Code |
| --- | ------------------- | --------------- | -------- |
| 1   | 512MB EFI partition | /boot partition | ef00     |
| 2   | 100% size partition | /root partition | 8300     |

Posteriormente escribe con `write` y salde cgdisk en `quit`.

## Formateamos particiones y hacemos el montaje de las unidades

Ejecutamos `lsblk -l` para ver que está todo bien definido.

<details>
<summary><big>Formato ext4:</big></summary>

Primero formateamos las particiones:

```shell
mkfs.vfat -F 32 /dev/nvme0n1p5
mkfs.ext4 /dev/nvme0n1p6
```

Sustituye `nvme0n1p5` por tu particion EFI.
Sustituye `nvme0n1p6` por tu particion raíz.

Después montamos las unidades:

```shell
mount /dev/nvme0n1p6 /mnt
mkdir /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot
```

lsblk para ver que todo está montado correctamente.

</details>
</br>
<details>
<summary><big>Formato btrfs:</big></summary>

Primero formateamos las particiones:

```shell
mkfs.vfat -F 32 /dev/nvme0n1p5
mkfs.btrfs /dev/nvme0n1p6
```

Sustituye `nvme0n1p5` por tu particion EFI.
Sustituye `nvme0n1p6` por tu particion raíz.

Después montamos las unidades:

```shell
mount /dev/nvme0n1p6 /mnt/
btrfs su cr /mnt/@
mount -o compress=xstd,subvol=@ /dev/nvme0n1p6 /mnt
btrfs su cr /mnt/@home
btrfs su cr /mnt/@pkg
btrfs su cr /mnt/@log
btrfs su cr /mnt/@snapshots
umount /mnt
mkdir /mnt
mkdir -p /mnt/home
mkdir -p /mnt/var/cache/pacman/pkg
mkdir -p /mnt/var/log
mkdir -p /mnt/.snapshots
mount -o compress=xstd,subvol=@home /dev/nvme0n1p6 /mnt/home
mount -o compress=xstd,subvol=@pkg /dev/nvme0n1p6 /mnt/var/cache/pacman/pkg
mount -o compress=xstd,subvol=@log /dev/nvme0n1p6 /mnt/var/log
mount -o compress=xstd,subvol=@snapshots /dev/nvme0n1p6 /mnt/.snapshots
lsblk
mkdir -p /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot
lsblk
```

</details>
</br>

Montamos ahora la partición EFI en /boot:

```shell
mkdir -p /mnt/boot
mount /dev/nvme0n1p5 /mnt/boot
```

Ejecutamos `lsblk` para ver que esta todo bien montado.

## Creamos la tabla de particiones

```shell
genfstab -U /mnt
genfstab -U /mnt > /mnt/etc/fstab
```

## Instalamos el kernel y algunos paquetes necesarios

```shell
pacstrap /mnt base base-devel linux=6.7.9.arch1-1 linux-firmware networkmanager grub efibootmgr os-prober pulseaudio man git nano vim neofetch
```

## Accedemos a al montaje con chroot

```shell
arch-chroot /mnt
```

## Definimos zona horaria

```shell
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```

Sincroniza la hora del sistema operativo con la del hardware en UTC:

```shell
hwclock --systohc --utc
```

## Configuración de idioma y teclado

```shell
nano /etc/locale.gen
```

- `ctrl + w` para filtrar por `es_ES`.
- Descomentamos `es_ES.UTF-8 UTF-8` y guardamos.

```shell
locale-gen
```

```shell
echo LANG=es_ES.UTF-8 > /etc/locale.conf
export LANG=es_ES.UTF-8
```

Ahora escribimos y guardamos `KEYMAP=es` en:

```shell
nano /etc/vconsole.conf
```

## Comprobar que el kernel se instaló correctamente

```shell
mkinitcpio -p linux
```

## Asignación de usuarios

```shell
passwd
useradd -m nacho
passwd nacho
usermod -aG wheel nacho
```

Ahora ejecutamos el siguiente comando y deberemos observar la salida `wheel nacho`

```shell
groups nacho
```

Instalamos sudo por si no lo tenemos:

```shell
pacman -S sudo
```

Ahora modificamos el archivo de sudo y descomentamos `%wheel ALL=(ALL:ALL) ALL`:

```shell
nano /etc/sudoers
```

### Comprobaciones:

Comprobamos que pasamos al user nacho:

```shell
su nacho
```

Ver que pertenecemos al grupo wheel:

```shell
id
```

Probar si al hacer `sudo su` y pones tu contraseña te conviertes en root.

## Configuración de Grub:

Para una instalación en un sistema 64bits, UEFI y con disco duro GPT es necesario ejecutar este comando para que tu firmware detecte Grub:

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Grub --recheck
```

La ubicación de `--efi-directory=/boot` será aquella donde este montada nuestra ESP. En este caso fue montada en `/boot`.

Para detectar estos cambios y configurar Grub debemos ejecutar:

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

Si queremos que Grub detecte otro sistema operativo como Windows tenemos que hacer lo siguiente.

Primero debemos modificar el fichero grub:

```shell
sudo nano /etc/default/grub
```

Estableciendo y descomentando los siguientes parámetros:

- GRUB_TIMEOUT=8
- GRUB_DISABLE_OS_PROBER=false

Ahora crearemos un punto de montaje para la EFI System Partition de Windows y una vez montado allí copiaremos el contenido de Microsoft en nuestra EFI de Grub:

```shell
mkdir /mnt/win
mount /dev/nvme0n1p1 /mnt/win/
cp -r /mnt/win/EFI/Microsoft /boot/EFI
```

Volvemos a ejecutar grub-mkconfig para que se apliquen los cambios

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

## Configuración de Host

Le asignamos un nombre a la máquina.

```shell
echo ArchLinux > /etc/hostname
```

Modificamos nuestro fichero hosts con el nombre de la máquina que asignamos.

```shell
sudo nano /etc/hosts
```

Añadimos:

```
127.0.0.1	localhost
::1			localhost
127.0.0.1	ArchLinux.localhost ArchLinux
```

## Añadimos el Multilib repo para librerías x32

Descomentamos `multilib` y NO `multilib-testing`.

```shell
nano /etc/pacman.conf
```

Actualizamos pacman:

```shell
pacman -Syy
```

## Reiniciamos

```shell
exit
umount -R /mnt
reboot now
```

## Comprobaciones

- Ver si el grub se monto correctamente y si accedemos a nuestro sistema sin interfaz gráfica.
- Tecleamos nuestro usuario "nacho" y la contraseña creada anteriormente.
- Comprobar si el idioma de nuestro teclado es el correcto escribiendo un guión `-`.
- Comprobamos conexión a Internet con `ping -c 1 google.es`.

Si no tenemos Internet en el siguiente paso lo resolvemos.

## Habilitando algunos servicios importantes

Habilitamos el servicio de NetworkManager para poder tener conexión a Internet.

```shell
sudo su
systemctl start NetworkManager.service
systemctl enable NetworkManager
```

Comprobamos que tenemos Internet con `ping -c 1 google.es`.

Habilitamos el servicio de audio

```shell
systemctl start pulseaudio
systemctl --user enable pulseaudio
```

## Añadimos el repositorio AUR de yay

Primero comprobar que estamos con el usuario nacho y no como root.

Cremos un direcorio repos donde clonar el repo de yay. Desde allí descargamos el repo oficial de yay o de paru como preferáis y hacemos makepkg.

```shell
mkdir repos
cd !$
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay
```

## Instalación de controladores gráficos open source de AMD

DRIVER AMDGPU:

Este es el principal que debemos instalar.

```shell
sudo pacman -S xf86-video-amdgpu mesa
```

VULKAN:

```shell
sudo pacman -S vulkan-radeon lib32-vulkan-radeon
```

OPENCL:

```shell
sudo pacman -S opencl-mesa
```

VDPU:

```shell
sudo pacman -S libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau
```

## Instalación de un entorno de escritorio

En este caso instalaré Gnome con wayland y el gestor de pantalla por defecto gdm pero podéis elegir cualquier otro como Hyprland y sdmm por ejemplo.

```shell
sudo pacman -S gnome gdm
sudo systemctl enable gdm.service
```

## Otros paquetes interesantes

Aquí hay algunos paquetes interesantes que podemos instalar si queremos.

```shell
sudo pacman -S kitty dolphin dunst unrar unzip firefox htop feh gnome-screenshot gimp
```

## Como hacer snapshots en un sistema btrfs

### Crear un snapshot:

Lo crearemos de todo el sistema root / en nuestro directorio que creamos de snapshots.

```shell
sudo btrfs subvolume snapshot / /.snapshots/<nombre del snapshot>
```

Comprobar que está creado.

### Restaurar desde un snapshot:

Montamos nuestro partición root en /mnt.

```shell
mount /dev/nvme0n1p6 /mnt
```

`nvme0n1p6` es donde se encuentra mi sistema btrfs root.

Comprobar que se montó correctamente:

```shell
ls -l /mnt/
```

Movemos nuestro volumen raíz @ a un volumen @latest_root. Después movemos nuestra snapshot creada al volumen raíz @. Por último reiniciamos.

```shell
mv /mnt/@ /mnt/@latest_root
mv /mnt/@.snapshots/Base /mnt/@
reboot
```

### Limpiar snapshots:

Este paso es importante para eliminar el anterior root y no ocupar espacio.

Montamos de nuevo nuestro sistema raíz en mi caso está en la partición `nvme0n1p6` en /mnt.

```shell
mount /dev/nvme0n1p6 /mnt/
```

Ahora tenemos dos opciones para borrar el volumen antiguo con todo el contenido root antiguo que ya no usaremos:

1. Con btrfs:

   ```shell
   btrfs subvolume delete /mnt/@latest_root
   ```

2. Con rm:

   ```shell
   sudo rm -r /mnt/@latest_root
   ```

Finalmente desmontamos /mnt.

```shell
umount /mnt
```
