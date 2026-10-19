# HTB — Crocodile (Starting Point)

**Difficulté :** Very Easy
**OS :** Linux
**Catégorie :** Starting Point

## Résumé

Crocodile combine deux vecteurs vus précédemment : un accès **FTP anonyme** mal configuré exposant des identifiants, puis un panneau d'administration web (`/login.php`) découvert par bruteforce de répertoires et accessible avec ces mêmes identifiants.

## 1. Connexion au VPN

Connexion au lab HTB via OpenVPN (pool *Starting Point*), puis vérification de la connectivité :

```bash
ping -c 5 <IP_cible>
```

## 2. Reconnaissance / Scan de ports

```bash
sudo nmap -sC -sV <IP_cible>
```

Résultat :

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
|_http-title: Smash - Bootstrap Business Template
```

Deux ports ouverts : **21 (FTP, anonyme autorisé)** et **80 (HTTP/Apache)**.

## 3. Récupération d'identifiants via FTP anonyme

```bash
ftp <IP_cible>
```

Connexion avec le login `anonymous` (mot de passe vide ou libre), puis listage du répertoire :

```
ftp> dir
-rw-r--r--   1 ftp   ftp    33 allowed.userlist
-rw-r--r--   1 ftp   ftp    62 allowed.userlist.passwd
```

Téléchargement des deux fichiers :

```
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
ftp> exit
```

Lecture en local :

```bash
cat allowed.userlist
cat allowed.userlist.passwd
```

Ces fichiers contiennent respectivement une liste d'utilisateurs potentiels et une liste de mots de passe associés (sans correspondance ligne par ligne garantie).

Tentative de réutilisation de ces identifiants directement sur le service FTP : refusée (`530 This FTP server is anonymous only`) — il faut donc chercher une autre cible pour ces identifiants.

## 4. Énumération web avec Wappalyzer et Gobuster

Le port 80 héberge un site vitrine (template "Smash"). L'extension **Wappalyzer** révèle la stack technique : Apache, PHP, Ubuntu, jQuery, Bootstrap.

Bruteforce des répertoires/fichiers avec Gobuster :

```bash
gobuster dir --url http://<IP_cible>/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,html
```

Résultat notable :

```
/login.php            (Status: 200) [Size: 1577]
/dashboard             (Status: 301) [--> http://<IP_cible>/dashboard/]
```

Une page de connexion (`/login.php`) est découverte.

## 5. Foothold — connexion au panneau d'administration

En testant les combinaisons username/password extraites du FTP (liste assez courte pour un test manuel), la connexion réussit sur `/login.php` et redirige vers un panneau **"Server Manager"**.

## 6. Récupération du flag

Le flag est affiché directement en haut du tableau de bord admin, au format :

```
Here is your flag: <valeur>
```

## Conclusion

Cette machine illustre un scénario de chaînage de vulnérabilités typique en pentest réel : un service auxiliaire mal sécurisé (FTP anonyme) fuit des identifiants qui, en apparence inutiles sur ce service, se révèlent valides ailleurs — ici sur un panneau d'administration web découvert par bruteforce de répertoires. Les leçons : désactiver l'accès FTP anonyme, ne jamais stocker d'identifiants en clair sur un service exposé, et imposer l'unicité des mots de passe entre services pour limiter l'impact d'une fuite.

## Résumé des commandes

| Étape | Commande |
|---|---|
| Scan de ports/versions | `sudo nmap -sC -sV <IP>` |
| Connexion FTP anonyme | `ftp <IP>` puis login `anonymous` |
| Lister/télécharger des fichiers | `dir`, `get <fichier>` |
| Bruteforce de répertoires web | `gobuster dir --url http://<IP>/ --wordlist <wordlist> -x php,html` |
| Connexion au panneau admin | via `/login.php` avec les identifiants trouvés |
