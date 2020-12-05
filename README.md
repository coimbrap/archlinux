# Mon installation d'Archlinux

Après quelques temps sous Debian j'ai voulu passer sous Archlinux, voici mon installation.

## Choix technique

- Un volume principal crypté luks2 de type btrfs avec les partitions root et home séparés compatible avec snapper.
- Un swap à part du volume btrfs crypté avec une clé random à chaque démarrage
- Amorçage par systemd-boot sur une partition non crypté

Pour la partie utilisateur on aura gnome et tout les outils nécessaire à une son utilisation

## Partie "serveur"

On boot en mode UEFI sur la clé bootable contenant archiso. Dans mon cas j'ai lancé les commandes via ssh.

Sur la machine :

On passe le clavier en azerty, on change le mot de passe root et on démarre le serveur ssh.
```bash
loadkeys fr
passwd
systemctl start sshd
```

On peut maintenant s'y connecter via SSH.

### Partitionnement

Utilisez `lsblk` pour trouver le bon disque ici on prend `/dev/sdb`

#### Création de la nouvelle table de partition

On efface la table des partitions et on randomise le disque (1h pour 1To de mon côté)
```bash
DRIVE=/dev/sdb
sgdisk --zap-all $DRIVE
badblocks -c 10240 -s -w -t random -v $DRIVE
```

On défini les nouvelles partitions
```bash
sgdisk --clear \
       --new=1:0:+550MiB --typecode=1:ef00 --change-name=1:EFI \
       --new=2:0:+8GiB   --typecode=2:8200 --change-name=2:cryptswap \
       --new=3:0:0       --typecode=2:8200 --change-name=3:cryptsystem \
         $DRIVE
```
On aura une partition pour l'amorçage EFI, une parition de 8go pour le SWAP et une parition pour le volume btrfs

#### Partition EFI
```bash
mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI
```

#### Partition SWAP

```bash
cryptsetup open --type plain --key-file /dev/urandom /dev/disk/by-partlabel/cryptswap swap
mkswap -L swap /dev/mapper/swap
swapon -L swap
```

#### Partition volume BTRFS

On va segmenter le volume btrfs en 3 sous-volumes :
```
cryptsystem
  |
  |--root
  |--home
  `--snapshots
       |
       |--root_snapshot_datehere
       |--root_snapshot_datehere
       |--root_snapshot_datehere
       |--home_snapshot_datehere
       `--home_snapshot_datehere
```

Création du volume chiffré de type luks2 avec optimisation pour un SSD
```bash
cryptsetup luksFormat \
  --type luks2 \
  --hash sha256 \
  --align-payload 8192 \
  --iter-time 5000 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  --verify-passphrase /dev/disk/by-partlabel/cryptsystem
```

On ouvre le volume et on crée le volume BTRFS
```bash
cryptsetup open /dev/disk/by-partlabel/cryptsystem system
mkfs.btrfs --force --label system /dev/mapper/system
```


Montage de base
```bash
o=defaults,x-mount.mkdir
o_btrfs=$o,compress=lzo,ssd,noatime
mount -t btrfs LABEL=system /mnt
```

On crée nos sous-volumes BTRFS
```bash
btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/home
btrfs subvolume create /mnt/snapshots
```

Montage final en mode ssd
```bash
umount -R /mnt
mount -t btrfs -o subvol=root,$o_btrfs LABEL=system /mnt
mount -t btrfs -o subvol=home,$o_btrfs LABEL=system /mnt/home
mount -t btrfs -o subvol=snapshots,$o_btrfs LABEL=system /mnt/.snapshots

mkdir /mnt/boot
mount LABEL=EFI /mnt/boot
```

### Installation du système de base fstab

#### Installation des packets

On installe le système de base, le noyau linux les firmware, l'amorçage par UEFI, la partie réseau et des outils utile.

```bash
pacstrap /mnt base base-devel linux linux-firmware \
        btrfs-progs ntp dosfstools snapper ntfs-3g \
        efibootmgr intel-ucode os-prober mkinitcpio \
        dhcpcd dialog network-manager-applet networkmanager wireless_tools wpa_supplicant \
        git gvim nano zip unzip p7zip man-db man-pages reflector \
        exfat-utils alsa-utils texinfo wget curl zsh openssh sudo archlinux-keyring
```

#### fstab

On génère la table de montage puis on modifier le montage de la partition swap de manière à ce qu'elle soit montée avec une clé généré aléatoirement à chaque démarrage. De cette manière on sort le swap du système chiffré principal (cela évite une couche lvm) tout en gardant un swap chiffré.

```bash
genfstab -L -p /mnt >> /mnt/etc/fstab
sed -i "s+LABEL=swap+/dev/mapper/swap+" /mnt/etc/fstab
echo "swap /dev/disk/by-partlabel/cryptswap /dev/urandom swap,cipher=aes-cbc-essiv:sha256,size=256" >> /mnt/etc/crypttab
```


### Configuration du système

On entre dans le système : `arch-chroot /mnt /bin/bash`

#### Clavier & Langage

Rien de bien spécial

##### locale.gen
```bash
cat << EOF > /etc/locale.gen
en_US.UTF-8 UTF-8
fr_FR.UTF-8 UTF-8
EOF
```

On génère

```bash
locale-gen
```

##### locale.conf
```bash
cat << EOF > /etc/locale.conf
LANG=fr_FR.UTF-8
LC_COLLATE=C
EOF
```

##### vconsole.conf
```bash
cat << EOF > /etc/vconsole.conf
KEYMAP=fr
EOF
```

#### Configuration date, hostname et hosts

Encore une fois, rien de bien spécial

##### Date & Heure
```bash
hwclock --systohc --utc
timedatectl set-ntp true
timedatectl set-timezone Europe/Paris
```

##### Hostname
```bash
hostnamectl set-hostname archbox
```

##### Fichier Hosts
```bash
cat << EOF > /etc/hosts
127.0.0.1 localhost
::1 localhost
127.0.1.1 archbox.local archbox
EOF
```

#### Création de l'utilisateur

##### Utilisateur principal
```bash
useradd -g users -G coimbrap,users,wheel,storage,power,network,audio -c "Pierre Coimbra" -m -s /bin/zsh coimbrap
passwd coimbrap
```

##### Sudo
```bash
sed -i 's/# %wheel ALL=(ALL) ALL/%wheel ALL=(ALL) ALL/g' /etc/sudoers
```

##### Mot de passe root
```bash
passwd root
```

#### Installation des packets

##### Mise à jours de la liste des dépôts

On prend les dix plus rapide de france
```bash
reflector --country France --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```

##### Activation de dhcp
```bash
systemctl enable dhcpcd
```

##### Optimisation pour le SSD

Activation TRIM pour soulager les blocs du SSD
```bash
systemctl enable fstrim.timer
```

On utilise le swap à partir de 90% d'utilisation de la RAM
```bash
echo "vm.swappiness=10" >> /etc/sysctl.d/99-sysctl.conf
```

#### Configuration et génération du noyau

##### /etc/mkinitcpio.conf

Remplacer les lignes par
```
BINARIES=(btrfsck)
HOOKS=(base udev autodetect modconf block keyboard keymap encrypt filesystems btrfs)
```

Génération
```bash
mkinitcpio -p linux
```

Il ne faut pas d'erreur

#### Mise en place de systemd-boot

##### Installation du système d'amorçage
```bash
bootctl --esp-path=/boot install
```

##### Configuration du loader
```bash
cat << EOF > /boot/loader/loader.conf
default arch.conf
editor no
timeout 4
console-mode max
EOF
```

##### Configuration de l'entrée archlinux

Pour obtenir l'UUID on utilise la commande `blkid | grep /dev/sda3`

```bash
cat << EOF > /boot/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID>:cryptsystem:allow-discards root=/dev/mapper/cryptsystem rootflags=subvol=root rw rootflags=subvol=root rw
EOF
```

### Fin de l'installation de base et redémarrage
```bash
exit
umount -R /mnt
cryptsetup luksClose system
reboot
```

### En cas d'échec

Relancer archiso et :

```bash
o=defaults,x-mount.mkdir
o_btrfs=$o,compress=lzo,ssd,noatime
cryptsetup open /dev/disk/by-partlabel/cryptsystem system
mount -t btrfs -o subvol=root,$o_btrfs LABEL=system /mnt
mount -t btrfs -o subvol=home,$o_btrfs LABEL=system /mnt/home
mount -t btrfs -o subvol=snapshots,$o_btrfs LABEL=system /mnt/.snapshots
mount LABEL=EFI /mnt/boot
cryptsetup open --type plain --key-file /dev/urandom /dev/disk/by-partlabel/cryptswap swap
mkswap -L swap /dev/mapper/swap
swapon -L swap
arch-chroot /mnt
```

On fait une snapshot avant de continuer

## Partie "Utilisateur"

### Mise en place de BTRFS

#### Activation de scrub
```bash
systemctl enable btrfs-scrub@-.timer && systemctl start btrfs-scrub@-.timer
```

#### Mise en place de snapper
```bash
umount /.snapshots/
rm -rf /.snapshots/
snapper -c root create-config /

systemctl start snapper-cleanup.timer
systemctl enable snapper-cleanup.timer

snapper -c root create -c timeline --description ServerInstall
```

### Installation de la partie utilisateur

On passe en root `sudo su -`

#### Ajout des dépots multilib

**/etc/pacman.conf**
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

#### OhMyZsh
```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### Mise à jours des dépôts et du système
```bash
pacman -Syu
```

#### Installation de la base
```bash
pacman -S hdparm util-linux archlinux-keyring
```

#### Serveur X est autres outils
```bash
pacman -S irqbalance cpupower x86_energy_perf_policy \
        xorg-server xf86-video-intel xf86-input-synaptics \
        xorg-xbacklight xorg-xinit rxvt-unicode compton openbox tint2 \
        conky  dmenu  volumeicon slock feh nitrogen scrot xarchiver \
        unrar rfkill terminus-font powertop whois ethtool archey3
```

#### VirtualBox
```bash
pacman -S xf86-video-vesa virtualbox virtualbox-host-dkms
```

#### Outils, thème, gnome, imprimantes/scanner et audio
```bash
pacman -S arc-gtk-theme arc-icon-theme elementary-icon-theme \
        cups gimp gimp-help-fr hplip python-pyqt5 xsane unoconv \
        foomatic-{db,db-ppds,db-gutenprint-ppds,db-nonfree,db-nonfree-ppds} gutenprint \
        gnome gnome-extra system-config-printer telepathy shotwell rhythmbox \
        pulseaudio pulseaudio-equalizer pulseaudio-bluetooth pulseaudio-jack \
        pulseaudio-equalizer-ladspa pulseaudio-zeroconf pulseaudio-alsa \
        audacity gimp gimp-plugin-gmic darktable atom transmission-gtk \
        firefox-i18n-fr firefox-ublock-origin thunderbird-i18n-fr \
        ttf-{bitstream-vera,droid,liberation,freefont,dejavu} libreoffice-fresh-fr hunspell-fr \
        networkmanager network-manager-applet gnome-keyring gnome-bluetooth bluez bluez-utils \
        vlc chromium keepassxc
```

Activation et demarage de certains outils, il peut y avoir des erreurs c'est pas grave

```bash
systemctl enable syslog-ng@default
systemctl enable cronie
systemctl enable avahi-daemon
systemctl enable avahi-dnsconfd
systemctl enable org.cups.cupsd
systemctl enable bluetooth
systemctl enable networkmanager
systemctl enable ntpd
systemctl enable gdm
```

On lance gnome
```bash
systemctl start networkmanager
systemctl start bluetooth
systemctl start gdm
```

Il faut passer le clavier en azerty dans Paramètres/Langues/Saisie

### AUR
```bash
git clone https://aur.archlinux.org/yay
cd yay
makepkg -sri
```

Installation de certain packages utilise disponible sur AUR
```bash
yay -Sy ttf-ms-fonts ttf-vista-fonts mattermost discord typora
```

Un petit redémarrage s'impose : `reboot`

Voilà c'est bon, l'installation est fini, il reste jutste à faire une snapshot au cas où.