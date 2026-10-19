# HTB — Redeemer (Starting Point)

**Difficulté :** Very Easy
**OS :** Linux
**Catégorie :** Starting Point

## Résumé

Redeemer est une machine d'introduction axée sur l'énumération d'un serveur **Redis** exposé sans authentification, permettant de lister et lire directement les clés de la base de données, dont le flag.

## 1. Connexion au VPN

Connexion au lab HTB via OpenVPN (pool *Starting Point*), puis vérification de la connectivité :

```bash
ping -c 5 10.129.29.161
```

## 2. Reconnaissance / Scan de ports

```bash
sudo nmap -p- -sV 10.129.29.161
```

Résultat :

```
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```

Seul le port **6379/tcp** est ouvert, exécutant un serveur **Redis**.

## 3. Qu'est-ce que Redis ?

Redis (*REmote DIctionary Server*) est une base de données NoSQL clé-valeur en mémoire, souvent utilisée comme cache ou message broker. Par défaut, Redis n'impose **aucune authentification**, ce qui en fait une cible fréquente en cas d'exposition réseau non maîtrisée.

## 4. Énumération du service Redis

Le client officiel `redis-cli` nécessite une installation (`sudo apt install redis-tools` sous Linux). En l'absence de ce client, il est possible d'interagir directement avec le serveur en envoyant le protocole Redis brut via un script Python (librairie standard, aucune dépendance externe) :

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.129.29.161", 6379))
s.sendall(b"KEYS *\r\n")
print(s.recv(4096))
```

Cela retourne la liste des clés présentes dans la base de données sélectionnée par défaut (index 0).

## 5. Récupération du flag

Une fois la clé pertinente identifiée (ex: `flag`), sa valeur est récupérée avec la commande `GET` :

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.129.29.161", 6379))
s.sendall(b"GET flag\r\n")
print(s.recv(4096))
```

(Avec le client officiel `redis-cli -h <IP>`, l'équivalent est simplement `keys *` puis `get flag`.)

## Conclusion

Cette machine illustre un risque fréquent avec les bases de données « in-memory » comme Redis : par défaut, aucune authentification n'est requise, et si le service est exposé sans pare-feu ni `requirepass` configuré, n'importe quel client réseau peut lire (voire écrire) l'intégralité des données stockées. La remédiation consiste à activer l'authentification (`requirepass`), restreindre l'accès réseau au service, et ne jamais exposer Redis directement sur Internet.

## Résumé des commandes

| Étape | Commande |
|---|---|
| Scan de ports/versions | `sudo nmap -p- -sV <IP>` |
| Connexion Redis (client officiel) | `redis-cli -h <IP>` |
| Sélectionner la base | `select 0` |
| Lister les clés | `keys *` |
| Lire une valeur | `get <clé>` |
| Alternative sans client (Python) | `socket.connect((<IP>, 6379))` + `sendall(b"KEYS *\r\n")` |
