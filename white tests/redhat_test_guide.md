# Guide Complet - Test Blanc Red Hat (WT-Aout 2022)

## Vue d'ensemble

Ce test comprend la configuration de deux machines Red Hat avec des tâches spécifiques de sécurité, gestion des utilisateurs, stockage, réseau et services.

---

## Machine 1 - Serveur Principal

### 1. Configuration SELinux
**Objectif :** Activer SELinux en mode enforcing

```bash
# Vérifier l'état actuel
getenforce

# Si nécessaire, modifier le fichier de configuration
sudo vi /etc/selinux/config
# Changer SELINUX=enforcing

# Redémarrer si nécessaire
sudo reboot
```

### 2. Target par défaut multi-user
**Objectif :** Configurer le système pour démarrer en mode multi-utilisateur

```bash
# Définir le target par défaut
sudo systemctl set-default multi-user.target

# Vérifier la configuration
systemctl get-default
```

### 3. Configuration httpd avec NFS et SELinux
**Objectif :** Corriger la politique SELinux pour permettre à httpd d'utiliser NFS

```bash
# Activer le boolean SELinux
sudo setsebool -P httpd_use_nfs on

# Vérifier l'activation
getsebool httpd_use_nfs
```

### 4. Création utilisateur eric sans shell interactif
**Objectif :** Créer un utilisateur sans possibilité de connexion interactive

```bash
# Créer l'utilisateur avec shell nologin
sudo useradd -s /sbin/nologin eric

# Vérifier la création
grep eric /etc/passwd
```

### 5. Création utilisateur alex avec UID et mot de passe spécifiques
**Objectif :** Créer alex avec UID 1234 et mot de passe alex111

```bash
# Créer l'utilisateur avec UID spécifique
sudo useradd -u 1234 alex

# Définir le mot de passe
echo "alex111" | sudo passwd --stdin alex
```

### 6. Configuration umask pour alex
**Objectif :** Fichiers créés par alex auront les permissions r--rw--rw- (466)

```bash
# Modifier le .bashrc d'alex
sudo su - alex
echo "umask 211" >> ~/.bashrc
source ~/.bashrc

# Tester la création d'un fichier
touch test_file
ls -l test_file
# Devrait afficher : -r--rw--rw-

# Explication du calcul :
# Permissions par défaut fichier : 666 (rw-rw-rw-)
# umask 211 : ----w---x--x
# Résultat : 666 - 211 = 455 mais avec ajustement = 466 (r--rw--rw-)
```

### 7. Politique de mots de passe
**Objectif :** Exiger au moins une majuscule et 9 caractères

```bash
# Installer pwquality si nécessaire
sudo dnf install libpwquality

# Configurer la politique
sudo vi /etc/security/pwquality.conf
# Ajouter :
# minlen = 9
# ucredit = -1

# Ou via authselect
sudo authselect current
sudo authselect select sssd --force
```

### 8. Création utilisateur fabrice
**Objectif :** Créer fabrice avec UID 5001 et groupe secondaire system

```bash
# Créer l'utilisateur
sudo useradd -u 5001 -G system fabrice

# Vérifier la création
id fabrice
```

### 9. Configuration umask pour fabrice
**Objectif :** Dossiers créés par fabrice auront permissions rwxrwxr-- (774)

```bash
# Se connecter en tant que fabrice
sudo su - fabrice

# Configurer umask pour les dossiers
echo "umask 001" >> ~/.bashrc
source ~/.bashrc

# Tester
mkdir test_dir
ls -ld test_dir
```

### 10. Création du dossier logs par fabrice
**Objectif :** Créer un dossier contenant tous les logs système

```bash
# Se connecter en tant que fabrice
sudo su - fabrice

# Créer le dossier logs
mkdir ~/logs

# Copier tous les fichiers de log
sudo cp -r /var/log/* ~/logs/
sudo chown -R fabrice:fabrice ~/logs
```

### 11. Compression du dossier logs
**Objectif :** Compresser le dossier avec bzip2

```bash
# En tant que fabrice
cd ~
tar -cjf logs.tar.bz2 logs/

# Vérifier la compression
ls -lh logs.tar.bz2
```

### 12. Préparation transfert vers machine 2
**Objectif :** Préparer le transfert du dossier logs

```bash
# Le fichier logs.tar.bz2 est prêt pour le transfert
# Cette étape sera complétée avec la machine 2
```

### 13. Configuration serveur NTP
**Objectif :** Configurer cette machine comme serveur NTP

```bash
# Installer chrony
sudo dnf install chrony

# Configurer le serveur NTP
sudo vi /etc/chrony.conf
# Ajouter :
# allow 10.10.1.0/24

# Démarrer et activer le service
sudo systemctl enable --now chronyd

# Ouvrir le firewall
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload
```

### 14. Configuration timezone
**Objectif :** Définir Europe/Paris comme timezone

```bash
# Configurer la timezone
sudo timedatectl set-timezone Europe/Paris

# Vérifier
timedatectl
```

### 15. Configuration autofs pour mathias
**Objectif :** Configurer l'automontage du home directory

```bash
# Installer autofs
sudo dnf install autofs

# Créer l'utilisateur mathias
sudo useradd -d /server/mathias mathias

# Créer les répertoires
sudo mkdir -p /server/mathias
sudo mkdir -p /client

# Configurer autofs
sudo vi /etc/auto.master
# Ajouter : /client /etc/auto.client

sudo vi /etc/auto.client
# Ajouter : mathias -fstype=bind :/server/mathias

# Démarrer autofs
sudo systemctl enable --now autofs
```

---

## Machine 2 - Client et Services

### 1. Configuration client NTP
**Objectif :** Synchroniser avec la machine 1

```bash
# Installer chrony
sudo dnf install chrony

# Configurer le client
sudo vi /etc/chrony.conf
# Modifier : server 10.10.1.X iburst

# Redémarrer le service
sudo systemctl restart chronyd

# Vérifier la synchronisation
chrony sources -v
```

### 2. Création container logserver
**Objectif :** Créer un conteneur rsyslog

```bash
# Installer podman
sudo dnf install podman

# Créer l'utilisateur fabrice
sudo useradd fabrice

# Créer le conteneur en tant que fabrice
sudo su - fabrice

# Solution 1 : Utiliser une image publique alternative
podman pull docker.io/rsyslog/rsyslog_base_centos8
podman run -d --name logserver docker.io/rsyslog/rsyslog_base_centos8

# Solution 2 : Utiliser CentOS/RHEL UBI sans authentification
podman pull registry.access.redhat.com/ubi8/ubi
podman run -d --name logserver registry.access.redhat.com/ubi8/ubi tail -f /dev/null

# Solution 3 : Si vous avez accès au registre Red Hat
# podman login registry.redhat.io
# podman pull registry.redhat.io/ubi8/rsyslog
# podman run -d --name logserver registry.redhat.io/ubi8/rsyslog

# Solution 4 : Créer une image personnalisée avec rsyslog
podman run -d --name logserver registry.access.redhat.com/ubi8/ubi \
  /bin/bash -c "dnf install -y rsyslog && rsyslogd -n"
```

### 3. Configuration service systemd pour le conteneur
**Objectif :** Créer un service systemd géré par fabrice

```bash
# En tant que fabrice
mkdir -p ~/.config/systemd/user

# Générer le service
podman generate systemd --name logserver --files --new

# Déplacer le fichier service
mv container-logserver.service ~/.config/systemd/user/

# Activer le service utilisateur
systemctl --user daemon-reload
systemctl --user enable container-logserver.service

# Activer le linger pour fabrice
sudo loginctl enable-linger fabrice
```

### 4. Démarrage automatique du service
**Objectif :** Le service démarre au boot

```bash
# Déjà configuré avec enable-linger dans l'étape précédente
# Vérifier :
systemctl --user status container-logserver.service
```

### 5. Configuration journal persistant
**Objectif :** Stocker les journaux de manière persistante

```bash
# Créer le répertoire journal
sudo mkdir -p /var/log/journal

# Configurer systemd-journald
sudo vi /etc/systemd/journald.conf
# Décommenter : Storage=persistent

# Redémarrer journald
sudo systemctl restart systemd-journald
```

### 6. Copie des journaux
**Objectif :** Copier tous les fichiers .journal

```bash
# Créer le répertoire de destination
sudo mkdir -p /home/fabrice/container_logserver
sudo chown fabrice:fabrice /home/fabrice/container_logserver

# Copier les journaux
sudo cp -r /var/log/journal/* /home/fabrice/container_logserver/
sudo chown -R fabrice:fabrice /home/fabrice/container_logserver
```

### 7. Configuration automount pour le conteneur
**Objectif :** Monter automatiquement les journaux dans le conteneur

```bash
# Modifier la configuration du conteneur pour inclure le bind mount
sudo su - fabrice
podman run -d --name logserver \
  -v /home/fabrice/container_logserver:/var/log/journal:Z \
  registry.redhat.io/ubi8/rsyslog
```

### 8. Création volume STRATIS
**Objectif :** Créer un volume Stratis selon les spécifications

```bash
# Installer stratis
sudo dnf install stratisd stratis-cli

# Démarrer le service
sudo systemctl enable --now stratisd

# Créer le pool (remplacer /dev/sdX par le disque approprié)
sudo stratis pool create stratpool /dev/sdX

# Créer le volume
sudo stratis filesystem create stratpool stratfs

# Créer le point de montage
sudo mkdir /stratvol

# Monter de manière permanente
echo "/stratis/stratpool/stratfs /stratvol xfs defaults 0 0" | sudo tee -a /etc/fstab

# Monter
sudo mount /stratvol

# Créer un snapshot
sudo stratis filesystem snapshot stratpool stratfs stratissnap
```

### 9. Création Volume Logique
**Objectif :** Créer un LV avec 30 PE de 32MB chacun

```bash
# Créer le volume logique (960MB = 30 × 32MB)
sudo lvcreate -l 30 -n lv group

# Formater en vfat
sudo mkfs.vfat /dev/group/lv

# Créer le point de montage
sudo mkdir /lv

# Monter de manière permanente
echo "/dev/group/lv /lv vfat defaults 0 0" | sudo tee -a /etc/fstab
sudo mount /lv
```

### 10. Redimensionnement Volume Logique
**Objectif :** Redimensionner entre 1270M et 1290M

```bash
# Calculer la nouvelle taille (1280M = environ 40 PE de 32MB)
sudo lvextend -L 1280M /dev/group/lv

# Note: vfat ne supporte pas le redimensionnement à chaud
# Il faudra reformater ou utiliser des outils spécialisés
```

### 11. Création partition VDO
**Objectif :** Créer une partition VDO de 50GB

```bash
# Installer VDO
sudo dnf install vdo kmod-kvdo

# Créer le volume VDO (remplacer /dev/sdY par le disque approprié)
sudo vdo create --name=Vdo1 --device=/dev/sdY --vdoLogicalSize=50G

# Formater en ext4
sudo mkfs.ext4 /dev/mapper/Vdo1

# Créer le point de montage
sudo mkdir /vdo

# Monter de manière permanente
echo "/dev/mapper/Vdo1 /vdo ext4 defaults 0 0" | sudo tee -a /etc/fstab
sudo mount /vdo
```

### 12. Ajout partition swap
**Objectif :** Ajouter 1GB de swap

```bash
# Créer la partition swap (remplacer /dev/sdZ par la partition appropriée)
sudo mkswap /dev/sdZ

# Activer le swap
sudo swapon /dev/sdZ

# Rendre permanent
echo "/dev/sdZ none swap defaults 0 0" | sudo tee -a /etc/fstab
```

### 13. Configuration profil tuned
**Objectif :** Appliquer le profil recommandé

```bash
# Installer tuned
sudo dnf install tuned

# Voir les profils disponibles
tuned-adm list

# Appliquer le profil recommandé
sudo tuned-adm recommend
sudo tuned-adm profile $(tuned-adm recommend)

# Vérifier
tuned-adm active
```

### 14. Configuration dépôts YUM
**Objectif :** Configurer les dépôts sous /etc/yum.repos2

```bash
# Créer le répertoire
sudo mkdir -p /etc/yum.repos2

# Créer le fichier de configuration
sudo vi /etc/yum.repos2/custom.repo
```

Contenu du fichier :
```ini
[BaseOS]
name=BaseOS Repository
baseurl=http://content.example.com/rhel8.0/x86_64/dvd/BaseOS
enabled=1
gpgcheck=0

[AppStream]
name=AppStream Repository
baseurl=http://content.example.com/rhel8.0/x86_64/dvd/AppStream
enabled=1
gpgcheck=0
```

### 15. Reset mot de passe root
**Objectif :** Définir le mot de passe root à "redhat"

```bash
# Méthode 1 : Si vous avez accès sudo
echo "redhat" | sudo passwd --stdin root

# Méthode 2 : Boot en mode rescue si nécessaire
# Au boot, éditer GRUB, ajouter rd.break à la ligne kernel
# mount -o remount,rw /sysroot
# chroot /sysroot
# passwd root
# touch /.autorelabel
# exit && reboot
```

### 16. Configuration Apache sur port 93
**Objectif :** Configurer httpd sur port 93 avec DocumentRoot /tekup

```bash
# Installer Apache
sudo dnf install httpd

# Créer le DocumentRoot
sudo mkdir -p /tekup

# Configurer Apache
sudo vi /etc/httpd/conf/httpd.conf
# Modifier :
# Listen 93
# DocumentRoot "/tekup"

# Configurer SELinux pour le nouveau port
sudo semanage port -a -t http_port_t -p tcp 93

# Configurer le firewall
sudo firewall-cmd --permanent --add-port=93/tcp
sudo firewall-cmd --reload

# Démarrer Apache
sudo systemctl enable --now httpd
```

### 17. Création fichier shells
**Objectif :** Créer /tekup/shells avec la liste des shells

```bash
# Extraire les shells de /etc/passwd
sudo awk -F: '{print $7}' /etc/passwd | sort -u > /tmp/shells_list
sudo mv /tmp/shells_list /tekup/shells

# Ajuster les permissions
sudo chown apache:apache /tekup/shells
sudo chmod 644 /tekup/shells

# Tester l'accès web
curl http://localhost:93/shells
```

### 18. Configuration accès depuis machine 2
**Objectif :** Permettre l'accès au fichier depuis la seconde machine

```bash
# S'assurer que le firewall autorise les connexions
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.1.0/24" port protocol="tcp" port="93" accept'
sudo firewall-cmd --reload

# Configurer SELinux si nécessaire
sudo setsebool -P httpd_can_network_connect on
```

### 19. Configuration réseau statique
**Objectif :** Configurer l'adresse IP statique

```bash
# Configurer l'interface réseau (remplacer ens33 par l'interface appropriée)
sudo nmcli con mod ens33 ipv4.addresses 10.10.1.10/8
sudo nmcli con mod ens33 ipv4.gateway 10.10.1.255
sudo nmcli con mod ens33 ipv4.dns 8.8.8.8
sudo nmcli con mod ens33 ipv4.method manual

# Redémarrer la connexion
sudo nmcli con down ens33 && sudo nmcli con up ens33

# Définir le hostname
sudo hostnamectl set-hostname machine2.example.com

# Vérifier la configuration
ip addr show
hostnamectl
```

---

## Points de Vérification Importants

### Machine 1
- [ ] SELinux en mode enforcing
- [ ] Target multi-user par défaut
- [ ] Boolean httpd_use_nfs activé
- [ ] Utilisateurs eric, alex, fabrice créés correctement
- [ ] Politiques umask appliquées
- [ ] Serveur NTP fonctionnel
- [ ] Autofs configuré pour mathias

### Machine 2
- [ ] Client NTP synchronisé
- [ ] Conteneur logserver opérationnel
- [ ] Service systemd utilisateur configuré
- [ ] Volumes Stratis, LVM, VDO créés
- [ ] Swap activé et permanent
- [ ] Profil tuned appliqué
- [ ] Apache sur port 93 accessible
- [ ] Configuration réseau statique

## Commandes de Diagnostic

```bash
# Vérification générale du système
systemctl status
timedatectl
hostnamectl
selinux status

# Vérification stockage
df -h
lsblk
lvdisplay
stratis pool list
vdo status

# Vérification réseau
ip addr show
ss -tlnp
firewall-cmd --list-all

# Vérification services
systemctl list-unit-files --state=enabled
```

---

**Note :** Ce guide suppose l'utilisation de Red Hat Enterprise Linux 8 ou CentOS 8. Certaines commandes peuvent nécessiter des ajustements selon la version exacte du système.
