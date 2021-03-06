# Changement de serveur "à la barbare"

Voici le descriptif d'une méthode barbare (mais parfaitement fonctionnelle !) pour migrer un Se3 d'un serveur physique à un autre.

![Oh ! Un barbare :-)](../images/barbare.png)

Cette méthode a été utilisée pour migrer un serveur SambaÉdu3 d'un serveur physique vers une machine virtuelle sous Proxmox, avec réorganisation des disques et partitions, mais sans réinstallation du système.

Elle est largement inspirée de la méthode décrite ici : https://wiki.debian-fr.xyz/Restauration_ou_Reconstruction_de_sa_Debian_-_Racine_et_Home_s%C3%A9par%C3%A9s

* Sur le serveur de départ :
```
sda      8:0    0 465,8G  0 disk 
├─sda1   8:1    0  18,6G  0 part [SWAP]
├─sda2   8:2    0  46,6G  0 part /
└─sda3   8:3    0  93,1G  0 part /var
sdb      8:16   0   2,7T  0 disk 
├─sdb1   8:17   0   1,4T  0 part /home
└─sdb2   8:18   0   1,4T  0 part /var/se3
```

* Sur le serveur migré :
```
sda      8:0    0    32G  0 disk 
└─sda1   8:1    0    32G  0 part /
sdb      8:16   0     1T  0 disk 
└─sdb1   8:17   0  1024G  0 part /home
sdc      8:32   0     1T  0 disk 
└─sdc1   8:33   0  1024G  0 part /var/se3
```

Note : dans les deux cas, `/var/lib/backuppc` et `/sauvese3/` sont montés sur des NAS externes depuis le `fstab`.

## Préparer la machine cible
Première chose, récupérer un LiveCD de Debian Wheezy (la même version que le Se3 en prod) [http://cdimage.debian.org/mirror/cdimage/archive/7.11.0-live/amd64/iso-hybrid/debian-live-7.11.0-amd64-gnome-desktop.iso](URL)

* Créer la VM sous Proxmox, avec les 3 disques sda, sdb et sdc

* Booter sur le LiveCD

* Passer le clavier en fr

* Désactiver la mise en veille (pour info, utilisateur : `user`, mot de passe : `live`)

* Ouvrir une console root

* Formater les disques
```
mkfs.ext4 /dev/sda
mkfs.xfs /dev/sdb
mkfs.xfs /dev/sdc
```

* Monter les disques
```
mkdir /mnt/racine
mkdir /mnt/home
mkdir /mnt/varse3
mount /dev/sda1 /mnt/racine
mount /dev/sdb1 /mnt/home
mount /dev/sdc1 /mnt/varse3
```

* Créer un fichier d'exclusion `/root/exclure` contenant :
```
/home
/var/se3
/var/lib/backuppc
/sauvese3
/proc
/sys
/dev
/mnt
/media
/lost+found
```
## Récupérer les données
* Récupérer les données des home :

`rsync -av --del root@IP_SE3_EN_PROD:/home/ /mnt/home` (c'est trèèès long : on pourra lancer cette commande une première fois bien en amont de la migration, et la relancer une dernière fois juste avant la migration.)

* Récupérer les données de /var/se3 :

`rsync -av --del root@IP_SE3_EN_PROD:/var/se3/ /mnt/varse3` (c'est trèèès long : on pourra lancer cette commande une première fois bien en amont de la migration, et la relancer une dernière fois juste avant la migration.)

* Récupérer le système :

`rsync -av --del --exclude-from=/root/exclure root@IP_SE3_EN_PROD:/ /mnt/racine`

* Créer dans /mnt/racine/ les répertoires manquants listés dans `/root/exclure`
```
mkdir /mnt/racine/home
mkdir /mnt/racine/var/se3
mkdir /mnt/racine/var/lib/backuppc
mkdir /mnt/racine/sauvese3
mkdir /mnt/racine/proc
mkdir /mnt/racine/sys
mkdir /mnt/racine/dev
mkdir /mnt/racine/mnt
mkdir /mnt/racine/media
mkdir /mnt/racine/lost+found
```

## Finaliser

* Modifier le fstab

    * Ouvrir une seconde console pour afficher les UUID des disques avec `blkid`

    * Modifier le fichier `/mnt/racine/etc/fstab` en conséquence

* Démonter home et varse3
```
umount /mnt/home
umount /mnt/varse3
```

* Monter home et varse3 à la bone place
```
mount /dev/sdb1 /mnt/racine/home
mount /dev/sdc1 /mnt/racine/var/se3
```

* Monter le bazar pour chrooter :
```
mount --bind /dev /mnt/racine/dev
mount --bind /sys /mnt/racine/sys
mount -t proc /proc /mnt/racine/proc
```

* Chrooter :
```
chroot /mnt/racine
```

* Mettre à jour Grub :
```
update-grub
```

* Installer Grub sur sda
```
grub-install /dev/sda
```

* Quitter le chroot :
```
exit
```

* Démonter tout proprement :
```
umount /mnt/racine/proc
umount /mnt/racine/sys
umount /mnt/racine/dev
umount /mnt/racine/home
umount /mnt/racine/var/se3
umount /mnt/racine
```

## (Re)Démarrage

* Débrancher (la prise réseau) ou éteindre le serveur physique. Attention : les deux serveurs ont le même nom, la même adresse IP, etc.

* Redémarrer la VM contenant le nouveau serveur

* Boire une bonne petite bière bien fraîche.

## Problèmes rencontrés
Pour le moment, le seul effet de bord constaté a été la difficulté de démarrer le service freeradius (authentification wifi du paquet se3-radius). Problème de propriétaire sur un dossier et un fichier (sans doute parce que le groupe ssl-cert n'existe pas sur le livecd, rsync a utilisé mysql à la place).

Constat :
```
# ls -alh /etc/ssl/private/ssl-cert-snakeoil.key 
-rw-r----- 1 root mysql 1,7K nov.  13  2017 /etc/ssl/private/ssl-cert-snakeoil.key

# ls -alh /etc/ssl/ | grep mysql
drwx--x---   2 root mysql 4,0K nov.  13  2017 private
```

Résolution :
```
chown root:ssl-cert /etc/ssl/private
chown root:ssl-cert /etc/ssl/private/ssl-cert-snakeoil.key 
service freeradius start
```

