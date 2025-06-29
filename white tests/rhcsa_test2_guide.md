# Guide Complet - Test Blanc RHCSA (WT2-Aout)

## Vue d'ensemble

Ce test RHCSA couvre la configuration système, gestion des utilisateurs, stockage, réseau, conteneurs et services. Il est divisé en deux systèmes avec des tâches spécifiques.

---

## System 1 - Configuration Principal

### 1. Configuration des sources YUM
**Objectif :** Configurer les dépôts CentOS Vault

```bash
# Créer le fichier de configuration des dépôts
sudo vi /etc/yum.repos.d/centos-vault.repo
```

Contenu du fichier :
```ini
[BaseOS]
name=CentOS BaseOS
baseurl=http://vault.centos.org/$contentdir/$releasever/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial

[AppStream]
name=CentOS AppStream
baseurl=http://vault.centos.org/$contentdir/$releasever/AppStream/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```

```bash
# Nettoyer le cache YUM
sudo dnf clean all

# Vérifier les dépôts
dnf repolist
```

### 2. Débogage Serveur Web
**Objectif :** Installer httpd sur port 82 avec fichier exam.html

```bash
# Installer httpd
sudo dnf install httpd

# Configurer le port 82
sudo vi /etc/httpd/conf/httpd.conf
# Modifier : Listen 82

# Créer le fichier exam.html
sudo vi /var/www/html/exam.html
```

Contenu du fichier exam.html :
```html
<!DOCTYPE html>
<html>
<head>
    <title>WT RHCSA</title>
</head>
<body>
    <h1>WT RHCSA</h1>
</body>
</html>
```

```bash
# Configurer SELinux pour le port 82
sudo semanage port -a -t http_port_t -p tcp 82

# Configurer le firewall
sudo firewall-cmd --permanent --add-port=82/tcp
sudo firewall-cmd --reload

# Démarrer et activer httpd
sudo systemctl enable --now httpd

# Tester
curl http://localhost:82/exam.html
```

### 3. Gestion des utilisateurs et des groupes
**Objectif :** Créer le groupe managers et les utilisateurs associés

```bash
# a. Créer le groupe managers
sudo groupadd managers

# b. Créer l'utilisateur natasha avec managers comme groupe secondaire
sudo useradd -G managers natasha
sudo passwd natasha

# c. Créer l'utilisateur harry avec managers comme groupe secondaire
sudo useradd -G managers harry
sudo passwd harry

# d. Créer l'utilisateur sara sans shell interactif
sudo useradd -s /sbin/nologin sara
sudo passwd sara

# Vérifier les créations
id natasha
id harry
id sara
```

### 4. Planification des tâches
**Objectif :** Cron job pour natasha toutes les 5 minutes

```bash
# Se connecter en tant que natasha
sudo su - natasha

# Éditer le crontab
crontab -e

# Ajouter la ligne suivante :
# */5 * * * * logger "Examen en cours"

# Vérifier le crontab
crontab -l

# Tester en consultant les logs
sudo tail -f /var/log/messages
```

### 5. Manipulation d'un répertoire partagé
**Objectif :** Créer /home/managers avec permissions spéciales

```bash
# Créer le répertoire
sudo mkdir /home/managers

# Changer le propriétaire et le groupe
sudo chown root:managers /home/managers

# Définir les permissions (rwxrwx---)
sudo chmod 770 /home/managers

# Activer le sticky bit pour le groupe
sudo chmod g+s /home/managers

# Vérifier les permissions
ls -ld /home/managers
# Devrait afficher : drwxrws--- root managers

# Tester la création de fichiers
sudo su - natasha
cd /home/managers
touch test_file
ls -l test_file
# Le groupe devrait être managers
```

### 6. Configuration de autofs
**Objectif :** Monter automatiquement le home de remoteuser1

```bash
# Installer autofs et nfs-utils
sudo dnf install autofs nfs-utils

# Créer l'utilisateur remoteuser1
sudo useradd remoteuser1

# Créer le répertoire client
sudo mkdir -p /clienthome

# Configurer autofs
sudo vi /etc/auto.master
# Ajouter : /clienthome /etc/auto.clienthome

# Créer le fichier de mapping
sudo vi /etc/auto.clienthome
# Ajouter : remoteuser1 -fstype=nfs,rw server.example.com:/home/remoteuser1

# Démarrer et activer autofs
sudo systemctl enable --now autofs

# Tester l'automontage
ls /clienthome/remoteuser1
```

### 7. Gestion des permissions
**Objectif :** Configurer les ACL sur /var/tmp/fstab

```bash
# Copier le fichier
sudo cp /etc/fstab /var/tmp/fstab

# Définir les ACL
sudo setfacl -m u:natasha:rw /var/tmp/fstab
sudo setfacl -m u:harry:--- /var/tmp/fstab

# Vérifier les ACL
getfacl /var/tmp/fstab

# Tester les permissions
sudo su - natasha
cat /var/tmp/fstab  # Devrait fonctionner
echo "test" >> /var/tmp/fstab  # Devrait fonctionner

sudo su - harry
cat /var/tmp/fstab  # Devrait échouer
```

### 8. Configuration de NTP
**Objectif :** Synchroniser avec server.example.com

```bash
# Installer chrony
sudo dnf install chrony

# Configurer le serveur NTP
sudo vi /etc/chrony.conf
# Ajouter ou modifier : server server.example.com iburst

# Redémarrer chronyd
sudo systemctl restart chronyd

# Vérifier la synchronisation
chrony sources -v
timedatectl
```

### 9. Configuration des comptes utilisateurs
**Objectif :** Créer manel et jaques, gérer les fichiers

```bash
# Créer l'utilisateur manel avec UID 3800
sudo useradd -u 3800 manel
echo "tekup" | sudo passwd --stdin manel

# Créer l'utilisateur jaques avec UID 2800
sudo useradd -u 2800 jaques
sudo passwd jaques

# Se connecter en tant que jaques et créer les fichiers
sudo su - jaques
for i in {1..100}; do touch fich$i; done
ls fich*

# Créer le répertoire de recherche
sudo mkdir -p /home/recherche

# Copier tous les fichiers appartenant à jaques
sudo find /home/jaques -user jaques -exec cp {} /home/recherche/ \;

# Ou plus simplement :
sudo cp -r /home/jaques/* /home/recherche/

# Trouver les lignes contenant "ro" dans /etc/passwd
grep "ro" /etc/passwd > /lignes
sudo mv /lignes /lignes
```

### 10. Compression
**Objectif :** Compresser /usr en back.tar.bz2

```bash
# Créer l'archive compressée
sudo tar -cjf back.tar.bz2 /usr

# Vérifier l'archive
ls -lh back.tar.bz2
tar -tjf back.tar.bz2 | head -10
```

### 11. Page HTML et compression (utilisateur student)
**Objectif :** Créer une page HTML et la compresser

```bash
# Se connecter en tant que student
sudo su - student

# Créer le fichier HTML
cat > exemple.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Exam RHCSA</title>
</head>
<body>
    <h1>exam rhcsa</h1>
</body>
</html>
EOF

# Compresser avec gzip
gzip exemple.html

# Vérifier
ls -l exemple.html.gz
```

### 12. Conteneur Apache avec montage
**Objectif :** Créer un conteneur Apache avec ~/storage monté

```bash
# En tant que student
# Créer le répertoire de stockage
mkdir ~/storage

# Extraire le contenu de exemple.tar.gz dans ~/storage
tar -xzf exemple.tar.gz -C ~/storage/

# Installer podman si nécessaire
sudo dnf install podman

# Créer et lancer le conteneur Apache
podman run -d --name apache-container \
  -p 2000:80 \
  -v ~/storage:/var/www/html:Z \
  registry.access.redhat.com/ubi8/httpd-24

# Vérifier le conteneur
podman ps
curl http://localhost:2000
```

### 13. Service systemd pour le conteneur
**Objectif :** Configurer le conteneur comme service

```bash
# En tant que student
mkdir -p ~/.config/systemd/user

# Générer le fichier de service
podman generate systemd --name apache-container --files --new

# Déplacer le fichier de service
mv container-apache-container.service ~/.config/systemd/user/

# Recharger systemd utilisateur
systemctl --user daemon-reload

# Activer le service
systemctl --user enable container-apache-container.service

# Activer le linger pour démarrage automatique
sudo loginctl enable-linger student

# Tester le service
systemctl --user status container-apache-container.service
```

### 14. Recherche de lignes nologin
**Objectif :** Trouver les lignes avec nologin dans /etc/passwd

```bash
# Rechercher et écrire dans /tmp/testfile
grep "nologin" /etc/passwd > /tmp/testfile

# Vérifier le résultat
cat /tmp/testfile
```

### 15. Configuration client NTP
**Objectif :** Se synchroniser avec la deuxième machine

```bash
# Configurer chrony pour pointer vers la machine 2
sudo vi /etc/chrony.conf
# Modifier : server 172.25.250.X iburst (remplacer X par l'IP de la machine 2)

# Redémarrer chronyd
sudo systemctl restart chronyd

# Vérifier la synchronisation
chrony sources -v
```

### 16. Ajout du groupe admin
**Objectif :** Créer le groupe admin avec GID 6000

```bash
# Créer le groupe admin avec GID spécifique
sudo groupadd -g 6000 admin

# Vérifier la création
getent group admin
```

### 17. Création des utilisateurs avec groupe admin
**Objectif :** Créer user1, user2, user3 avec spécifications

```bash
# Créer user1
sudo useradd user1
echo "redhat" | sudo passwd --stdin user1

# Créer user2 avec admin comme groupe secondaire
sudo useradd -G admin user2
echo "redhat" | sudo passwd --stdin user2

# Créer user3 avec admin comme groupe secondaire
sudo useradd -G admin user3
echo "redhat" | sudo passwd --stdin user3

# Vérifier les créations
id user1
id user2
id user3
```

### 18. Répertoire /home/admins
**Objectif :** Créer un répertoire partagé pour le groupe admin

```bash
# Créer le répertoire
sudo mkdir /home/admins

# Changer le propriétaire et le groupe
sudo chown root:admin /home/admins

# Définir les permissions avec sticky bit groupe
sudo chmod 2770 /home/admins

# Vérifier
ls -ld /home/admins
# Devrait afficher : drwxrws--- root admin

# Tester
sudo su - user2
cd /home/admins
touch test_file
ls -l test_file
# Le groupe devrait être admin
```

### 19. Configuration ACL sur /var/tmp/fstab
**Objectif :** Permissions spécifiques pour user1 et user2

```bash
# Copier le fichier
sudo cp /etc/fstab /var/tmp/fstab

# Définir le propriétaire
sudo chown root:root /var/tmp/fstab

# Retirer les permissions d'exécution pour tous
sudo chmod 644 /var/tmp/fstab

# Configurer les ACL
sudo setfacl -m u:user1:rw /var/tmp/fstab
sudo setfacl -m u:user2:--- /var/tmp/fstab

# Vérifier
getfacl /var/tmp/fstab
ls -l /var/tmp/fstab
```

### 20. Script replace.sh
**Objectif :** Script qui compte les fichiers dans un répertoire

```bash
# Créer le script
cat > replace.sh << 'EOF'
#!/bin/bash

# Vérifier si un paramètre est fourni
if [ $# -eq 0 ]; then
    echo "Usage: $0 <directory>"
    exit 1
fi

DIR="$1"

# Vérifier si le répertoire existe
if [ ! -d "$DIR" ]; then
    echo "Erreur: $DIR n'est pas un répertoire valide"
    exit 1
fi

# Compter les fichiers (pas les répertoires)
COUNT=$(find "$DIR" -maxdepth 1 -type f | wc -l)

echo "Nombre de fichiers dans $DIR: $COUNT"
EOF

# Rendre le script exécutable
chmod +x replace.sh

# Tester le script
./replace.sh /home
./replace.sh /etc
```

---

## System 2 - Configuration Stockage et Réseau

### 21. Partition de 512M avec ext4
**Objectif :** Créer une partition sur /dev/sdb et la monter sur /mnt/data

```bash
# Créer la partition
sudo fdisk /dev/sdb
# Dans fdisk :
# n (nouvelle partition)
# p (primaire)
# 1 (numéro de partition)
# Entrée (premier secteur par défaut)
# +512M (taille)
# w (écrire les modifications)

# Formater en ext4
sudo mkfs.ext4 /dev/sdb1

# Créer le point de montage
sudo mkdir -p /mnt/data

# Monter temporairement
sudo mount /dev/sdb1 /mnt/data

# Ajouter à /etc/fstab pour montage permanent
echo "/dev/sdb1 /mnt/data ext4 defaults 0 2" | sudo tee -a /etc/fstab

# Vérifier
df -h /mnt/data
```

### 22. Partition swap de 100MB
**Objectif :** Créer une partition swap sur /dev/sdb

```bash
# Créer la partition swap
sudo fdisk /dev/sdb
# Dans fdisk :
# n (nouvelle partition)
# p (primaire)
# 2 (numéro de partition)
# Entrée (premier secteur par défaut)
# +100M (taille)
# t (changer le type)
# 2 (partition 2)
# 82 (type swap)
# w (écrire)

# Formater en swap
sudo mkswap /dev/sdb2

# Activer le swap
sudo swapon /dev/sdb2

# Ajouter à /etc/fstab
echo "/dev/sdb2 none swap defaults 0 0" | sudo tee -a /etc/fstab

# Vérifier
swapon --show
free -h
```

### 23-25. Configuration LVM
**Objectif :** Créer VG avec 50 PE, PE=16M, et deux LV

```bash
# Installer LVM si nécessaire
sudo dnf install lvm2

# Créer le Physical Volume
sudo pvcreate /dev/sdc

# Créer le Volume Group avec PE=16M
sudo vgcreate -s 16M vg /dev/sdc

# Vérifier que nous avons au moins 50 PE
sudo vgdisplay vg

# 24. Créer lvo1 avec 13 LE (13 × 16M = 208M)
sudo lvcreate -l 13 -n lvo1 vg

# Formater en ext4
sudo mkfs.ext4 /dev/vg/lvo1

# Créer le point de montage
sudo mkdir /lvo1

# Monter et ajouter à fstab
sudo mount /dev/vg/lvo1 /lvo1
echo "/dev/vg/lvo1 /lvo1 ext4 defaults 0 2" | sudo tee -a /etc/fstab

# 25. Créer lvo2 avec 200M
sudo lvcreate -L 200M -n lvo2 vg

# Formater en xfs
sudo mkfs.xfs /dev/vg/lvo2

# Créer le point de montage
sudo mkdir /lvo2

# Monter et ajouter à fstab
sudo mount /dev/vg/lvo2 /lvo2
echo "/dev/vg/lvo2 /lvo2 xfs defaults 0 2" | sudo tee -a /etc/fstab
```

### 26. Extension de lvo1
**Objectif :** Étendre lvo1 entre 225M et 230M

```bash
# Calculer la nouvelle taille (228M ≈ 14.25 LE, donc 15 LE)
sudo lvextend -l 15 /dev/vg/lvo1

# Étendre le système de fichiers ext4
sudo resize2fs /dev/vg/lvo1

# Vérifier la nouvelle taille
df -h /lvo1
# Devrait afficher environ 225-230M
```

### 27. Configuration VDO
**Objectif :** Créer un volume VDO de 30GB logique

```bash
# Installer VDO
sudo dnf install vdo kmod-kvdo

# Charger le module kernel
sudo modprobe kvdo

# Créer le volume VDO (supposant /dev/sdd de 10GB physique)
sudo vdo create --name=class1_vdo --device=/dev/sdd --vdoLogicalSize=30G

# Formater en XFS
sudo mkfs.xfs -K /dev/mapper/class1_vdo

# Créer le point de montage
sudo mkdir /class1_mnt

# Monter temporairement
sudo mount /dev/mapper/class1_vdo /class1_mnt

# Ajouter à fstab avec options VDO
echo "/dev/mapper/class1_vdo /class1_mnt xfs defaults,x-systemd.requires=vdo.service 0 0" | sudo tee -a /etc/fstab

# Vérifier
df -h /class1_mnt
vdo status --name=class1_vdo
```

### 28. Configuration réseau statique
**Objectif :** Configurer l'adresse IP statique

```bash
# Identifier l'interface réseau
ip link show

# Configurer l'interface (supposant ens33)
sudo nmcli con mod ens33 ipv4.addresses 172.25.250.10/24
sudo nmcli con mod ens33 ipv4.gateway 172.25.250.254
sudo nmcli con mod ens33 ipv4.dns 172.25.250.254
sudo nmcli con mod ens33 ipv4.method manual

# Appliquer la configuration
sudo nmcli con down ens33 && sudo nmcli con up ens33

# Vérifier la configuration
ip addr show ens33
ping -c 3 172.25.250.254
```

---

## Commandes de Vérification Finales

### Vérifications System 1
```bash
# Services et configuration
systemctl status httpd
systemctl status autofs
systemctl status chronyd
curl http://localhost:82/exam.html

# Utilisateurs et groupes
id natasha harry sara manel jaques user1 user2 user3
getent group managers admin

# Permissions et ACL
ls -ld /home/managers /home/admins
getfacl /var/tmp/fstab

# Conteneurs
podman ps
systemctl --user status container-apache-container
curl http://localhost:2000
```

### Vérifications System 2
```bash
# Stockage
df -h
swapon --show
lvdisplay
vdo status
lsblk

# Réseau
ip addr show
ping -c 3 172.25.250.254

# Montages permanents
cat /etc/fstab
mount | grep -E "(mnt|lvo|class1)"
```

### Tests Fonctionnels
```bash
# Test des permissions de groupe
sudo su - natasha
cd /home/managers && touch test_natasha
ls -l test_natasha

# Test des ACL
sudo su - user1
cat /var/tmp/fstab
echo "test" >> /var/tmp/fstab

# Test du script
./replace.sh /home
./replace.sh /etc
```

---

## Points Critiques à Retenir

1. **SELinux** : Toujours configurer les ports personnalisés
2. **Firewall** : Ouvrir les ports nécessaires
3. **Permissions** : Utiliser chmod, ACL et sticky bits correctement
4. **Montages** : Vérifier /etc/fstab pour la persistance
5. **Services** : Activer avec `systemctl enable --now`
6. **LVM/VDO** : Respecter les tailles exactes demandées
7. **Réseau** : Tester la connectivité après configuration

Ce guide couvre tous les aspects du test RHCSA avec des instructions détaillées et des commandes de vérification pour chaque étape.