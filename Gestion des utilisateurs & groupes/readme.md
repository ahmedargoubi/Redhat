# LAB3 : Gestion des utilisateurs & groupes

## Objectif

Le but de ce Lab est de maîtriser la gestion des utilisateurs et groupes ainsi que les droits d’accès des fichiers et des dossiers. Le changement d’utilisateur et de groupe propriétaires des fichiers/dossiers avec la ligne de commandes sous Linux sera aussi développé dans ce Lab.

---

## Exercice 1

1. Création le groupe `computestream`.

   ```bash
   groupadd computestream
   ```

2. Création un dossier `computestream` dans `/exam/`.

   ```bash
   mkdir -p /exam/computestream
   ```

3. Faites du groupe `computestream` le propriétaire du dossier `/exam/computestream`.

   ```bash
   chown :computestream /exam/computestream
   ```

4. Création un compte utilisateur `candidat` avec le mot de passe `cert456`avec la configuration sudo pour permettre au compte `candidat` d'accéder aux privilèges root sans invite de mot de passe.

   ```bash
   useradd candidat -G whell
   passwd candidat
   ```
   

5. Configuration le système afin qu'un fichier `NEWS` vide soit automatiquement créé dans le répertoire personnel de tout nouvel utilisateur.

   ```bash
   touch /etc/skel/NEWS
   ```

6. Création un groupe appelé `etudiants`.

   ```bash
   groupadd etudiants
   ```

7. Création un nouvel utilisateur `harry` avec les attributs suivants :
   - Mot de passe : `magique`
   - Commentaire : `student`
   - Membre du groupe `etudiants`

   ```bash
   useradd harry -c 'student' -G etudiants
   passwd harry
   ```

8. Créez un compte `sysadmin` avec les attributs suivants :
   - Mot de passe : `science`
   - Répertoire personnel : `/sysadmin/`
   - Privilèges sudo sans mot de passe
   - Shell par défaut : `zsh`

   ```bash
   mkdir /sysadmin
   useradd sysadmin -d /sysadmin -s /bin/zsh -G whell
   passwd sysadmin
   
  
   ```
   
   

9. Modifiez le compte `sysadmin` afin qu'il puisse utiliser `bash` comme shell par défaut.

   ```bash
   usermod sysadmin -s /bin/bash
   ```

---

## Exercice 3

1. Créez les utilisateurs `Rocky` et `Bullwinkle` avec des répertoires personnels.

   ```bash
   useradd -m Rocky
   useradd -m Bullwinkle
   ```

2. Créez les groupes `amis` et `boss` (GID=490).

   ```bash
   groupadd amis
   groupadd boss -g 490
   ```

3. Ajout `Rocky` aux groupes `amis` et `boss`, et `Bullwinkle` au groupe `amis`.

   ```bash
   usermod -aG amis,boss Rocky
   usermod -aG amis Bullwinkle
   ```

4. Création un dossier `somedir` appartenant au groupe `boss`.

   ```bash
   mkdir /home/Rocky/somedir
   chmod 770 /home/Rocky/somedir
   chgrp boss /home/Rocky/somedir
   
   ```

5. Ajout `Bullwinkle` au groupe `boss` .

   ```bash
   usermod -aG boss Bullwinkle
   su - Bullwinkle
   touch /home/Rocky/somedir/somefile
   ```