## Meow — Starting Point — Très facile

**Plateforme :** Hack The Box
**Catégorie :** Linux, Reconnaissance, Protocoles, Misconfiguration
**Date de résolution :** 08/10/2026

### Contexte

Meow est la toute première machine du module "Starting Point" de Hack The Box, pensée comme introduction à la méthodologie de base d'un test d'intrusion : connexion au VPN/Pwnbox, scan de ports, identification d'un service mal configuré, puis récupération du flag. Aucune information n'est donnée au départ hormis l'adresse IP de la cible.

### Reconnaissance

Après connexion au Pwnbox et vérification que la cible est joignable :

```bash
ping {target_IP}
```

J'ai ensuite lancé un scan de ports avec détection de version pour identifier les services exposés :

```bash
sudo nmap -sV {target_IP}
```

Résultat :

```
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
```

Un seul port ouvert : le **23/tcp**, correspondant au service **Telnet**. Telnet est un protocole ancien de gestion distante, aujourd'hui largement déprécié en raison de l'absence de chiffrement des échanges (identifiants et données transmis en clair sur le réseau).

### Découverte de la vulnérabilité

Sans autre port ouvert à explorer, la seule voie d'accès possible était ce service Telnet. En s'y connectant, le serveur affiche une invite de connexion classique (`login:` / `Password:`), ce qui suggère la présence de comptes utilisateurs configurés dessus.

L'hypothèse de départ a été de tester des comptes à privilèges par défaut, souvent laissés sans mot de passe par erreur de configuration lors du déploiement d'une machine — un classique des audits de sécurité sur des équipements réseau ou des serveurs mal durcis.

### Exploitation

1. Connexion au service Telnet :

```bash
telnet {target_IP}
```

2. Tentative de connexion avec des noms de comptes à privilèges standards (`admin`, `administrator`), toutes deux échouées (`Login incorrect`).

3. Tentative avec le compte `root` :

```
Meow login: root
```

Connexion acceptée **sans qu'aucun mot de passe ne soit demandé** — confirmant que le compte `root` était configuré avec un mot de passe vide, accessible publiquement via Telnet sans aucune authentification réelle.

4. Une fois connecté, exploration du répertoire courant :

```bash
ls
# flag.txt  snap

cat flag.txt
```

### Résultat

La lecture de `flag.txt` a affiché le hash de validation de la machine. Le flag a été soumis sur la page Hack The Box pour valider la machine "Meow".

### Ce que j'ai appris

Ce premier challenge illustre un principe fondamental qui revient très souvent en pentest réel : avant de chercher une vulnérabilité complexe, il faut systématiquement vérifier les erreurs de configuration les plus basiques — ici, un compte `root` accessible sans mot de passe sur un service non chiffré. Telnet lui-même, en tant que protocole, est une alerte à part entière dès qu'il est détecté lors d'un scan : sa simple présence sur une machine moderne est souvent le signe d'une configuration héritée ou négligée. La méthodologie suivie (ping → nmap avec détection de version → recherche d'information sur le service identifié → test de comptes par défaut) est la base de toute phase de reconnaissance, et sera réutilisée identiquement sur chaque nouvelle machine.

### Outils utilisés

- Pwnbox (environnement HTB préconfiguré)
- `ping` (vérification de connectivité)
- `nmap` (scan de ports et détection de version de service)
- `telnet` (connexion au service identifié)
