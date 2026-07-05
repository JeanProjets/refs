#  Cheat Sheet — Firewall Linux (firewalld & UFW)

> **RHEL/Rocky Linux** → `firewall-cmd`
> **Debian/Ubuntu/Proxmox** → `ufw`
> Les deux sont couverts ici.

---

## 1. firewalld — Diagnostic initial

```bash
# Le service tourne-t-il ?
sudo systemctl status firewalld
sudo firewall-cmd --state        # running ou not running

# Vue complète de la config active
sudo firewall-cmd --list-all
# Montre : zone, interfaces, services autorisés, ports, rich rules

# Voir uniquement les rich rules
sudo firewall-cmd --list-rich-rules

# Voir sur quelle zone est mon interface
sudo firewall-cmd --get-active-zones
```

---

## 2. firewalld — Ouvrir des services/ports

>  **TOUJOURS** utiliser `--permanent` puis `--reload` pour persister !

```bash
# Autoriser un SERVICE connu (http, https, ssh, ftp...)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh

# Lister tous les services disponibles
sudo firewall-cmd --get-services

# Autoriser un PORT précis
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp

# ⚡ APPLIQUER (obligatoire après --permanent)
sudo firewall-cmd --reload

# Vérifier
sudo firewall-cmd --list-all
```

---

## 3. firewalld — Supprimer des règles

```bash
# Retirer un service
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload

# Retirer un port
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
```

---

## 4. firewalld — Rich Rules (restriction par IP source)

> Les "rich rules" permettent de restreindre l'accès à un service/port
> à une **adresse IP ou un sous-réseau spécifique**.

```bash
# Autoriser SSH uniquement depuis le réseau 192.168.100.0/24
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.100.0/24"
  service name="ssh"
  accept'

# Autoriser le port 9090 depuis un seul réseau
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.100.0/24"
  port port="9090" protocol="tcp"
  accept'

# Bloquer une IP/réseau entier
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="10.0.0.0/24"
  reject'

#  Appliquer
sudo firewall-cmd --reload

# Vérifier
sudo firewall-cmd --list-rich-rules
```

---

## 5. firewalld — Scénario type test technique

> **Énoncé type :** HTTP et HTTPS ouverts à tous, SSH uniquement depuis le LAN, port 9090 uniquement depuis le LAN.

```bash
# 1. Retirer SSH du public (souvent déjà là par défaut)
sudo firewall-cmd --permanent --remove-service=ssh

# 2. Ouvrir HTTP et HTTPS à tout le monde
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# 3. SSH restreint au LAN
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.100.0/24"
  service name="ssh"
  accept'

# 4. Port 9090 restreint au LAN
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4"
  source address="192.168.100.0/24"
  port port="9090" protocol="tcp"
  accept'

# 5. Appliquer tout d'un coup
sudo firewall-cmd --reload

# 6. Valider
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
```

---

## 6. UFW (Debian/Ubuntu) — Équivalent complet

```bash
# Statut
sudo ufw status            # inactif / actif
sudo ufw status numbered   # Voir toutes les règles numérotées

# Politique par défaut (IMPORTANT : bloquer tout l'entrant par défaut)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Ouvrir des services
sudo ufw allow http          # port 80
sudo ufw allow https         # port 443
sudo ufw allow ssh           # port 22

# Restreindre à un réseau source
sudo ufw allow from 192.168.100.0/24 to any port 22
sudo ufw allow from 192.168.100.0/24 to any port 9090

# Activer (une seule fois, garde les règles)
sudo ufw enable

# Supprimer une règle
sudo ufw delete allow http
# Ou par numéro :
sudo ufw delete 3

# Désactiver temporairement (débogage)
sudo ufw disable
```

---

## 7. Dépannage firewall — Séquence logique

```bash
# Symptôme : ping marche mais curl/HTTP ne répond pas

# 1. Vérifier que Apache écoute vraiment
ss -tulpn | grep :80
# Si rien → Apache n'est pas démarré

# 2. Vérifier que le port est ouvert dans le firewall
sudo firewall-cmd --list-all | grep http
# ou
sudo ufw status | grep 80

# 3. Test sans firewall (diagnos. temporaire)
sudo systemctl stop firewalld     # RHEL (attention !)
sudo ufw disable                  # Debian (attention !)
curl http://<IP>                  # Si ça marche → c'est le firewall
# REMETTRE le firewall après le test !
sudo systemctl start firewalld
sudo ufw enable
```

---

## 8. Mémo des ports importants

| Port | Protocole | Service |
|------|-----------|---------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 9090 | TCP | Cockpit (admin web RHEL) |
| 8006 | TCP | Proxmox Web UI |
| 3306 | TCP | MySQL/MariaDB |
| 5432 | TCP | PostgreSQL |
