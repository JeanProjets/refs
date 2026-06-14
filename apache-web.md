# 🌍 Cheat Sheet — Apache Web Server (httpd / apache2)

> **RHEL/Rocky Linux** → paquet `httpd`, service `httpd`
> **Debian/Ubuntu** → paquet `apache2`, service `apache2`
> Racine web : `/var/www/html/` sur les deux distributions.

---

## 1. Installation

### Sur Rocky Linux / RHEL
```bash
sudo dnf install -y httpd

# Démarrer ET activer au boot en une seule commande
sudo systemctl enable --now httpd

# Vérifier
sudo systemctl status httpd
# Chercher : Active: active (running)
```

### Sur Debian / Ubuntu
```bash
sudo apt update && sudo apt install -y apache2

sudo systemctl enable --now apache2

# Vérifier
sudo systemctl status apache2
```

---

## 2. Gestion du service Apache

```bash
# RHEL (httpd)
sudo systemctl start httpd        # Démarrer
sudo systemctl stop httpd         # Arrêter
sudo systemctl restart httpd      # Redémarrer (coupe puis relance)
sudo systemctl reload httpd       # Recharger la config (sans coupure)
sudo systemctl enable httpd       # Activer au démarrage
sudo systemctl disable httpd      # Désactiver au démarrage
sudo systemctl enable --now httpd # Activer + démarrer (le plus utile)

# Debian (apache2) — mêmes commandes, juste changer httpd → apache2
sudo systemctl enable --now apache2
```

---

## 3. Vérifier qu'Apache fonctionne

```bash
# Test local (depuis la même machine)
curl http://localhost
curl http://127.0.0.1

# Test depuis l'IP de la machine
curl http://10.10.1.10

# Test depuis une autre machine
curl http://10.10.1.10        # depuis VM-CLIENT
wget -qO- http://10.10.1.10  # alternative si curl absent
lynx http://10.10.1.10       # navigateur terminal

# Voir les ports en écoute (Apache doit être sur le port 80)
ss -tulpn | grep :80
# Doit afficher quelque chose comme :
# tcp  LISTEN 0 511 0.0.0.0:80  0.0.0.0:*  users:(("httpd",pid=...))
```

---

## 4. Créer / modifier une page web

```bash
# La racine web (identique RHEL et Debian)
ls /var/www/html/

# Écraser la page par défaut avec une page personnalisée
sudo tee /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Mon Serveur Web</title>
</head>
<body>
    <h1>Serveur Apache opérationnel</h1>
    <p>IP : 10.10.1.10 | Réseau : 10.10.1.0/24</p>
</body>
</html>
EOF

# Vérifier que le fichier est bien là
cat /var/www/html/index.html

# Tester immédiatement
curl http://localhost
```

---

## 5. Ouvrir le port 80 dans le firewall

> ⚠️ **Étape critique !** Sans cette étape, curl depuis VM-CLIENT échouera même si le ping fonctionne.

### Sur Rocky Linux (firewalld)
```bash
# Méthode 1 : par le nom de service (recommandé)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

# Méthode 2 : par numéro de port
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload

# Vérifier
sudo firewall-cmd --list-services
# Doit contenir : http
sudo firewall-cmd --list-all
```

### Sur Debian/Ubuntu (ufw)
```bash
sudo ufw allow http      # équivalent à allow 80/tcp
sudo ufw status
# Doit afficher : 80/tcp ALLOW
```

---

## 6. Logs Apache — Voir ce qui se passe

```bash
# RHEL
sudo journalctl -u httpd -n 30         # 30 dernières lignes de logs
sudo tail -f /var/log/httpd/access_log  # Voir les requêtes en temps réel
sudo tail -f /var/log/httpd/error_log   # Voir les erreurs

# Debian
sudo journalctl -u apache2 -n 30
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/error.log
```

---

## 7. Configuration Apache (fichiers importants)

```bash
# RHEL — Config principale
/etc/httpd/conf/httpd.conf
/etc/httpd/conf.d/          # Dossier de config additionnelles (vhosts, etc.)

# Debian
/etc/apache2/apache2.conf
/etc/apache2/sites-available/
/etc/apache2/sites-enabled/

# Tester la syntaxe de la config avant de recharger
sudo httpd -t              # RHEL (chercher : Syntax OK)
sudo apache2ctl configtest  # Debian
```

---

## 8. Troubleshooting — Séquence si ça ne marche pas

```
Symptôme : curl http://10.10.1.10 depuis VM-CLIENT → pas de réponse / erreur
```

```bash
# Étape 1 : Le ping fonctionne-t-il ?
ping -c 3 10.10.1.10
# Si NON → problème réseau/routage, voir proxmox-bridges.md
# Si OUI → continuer

# Étape 2 : Apache est-il démarré sur VM-WEB ?
# (se connecter sur VM-WEB via console Proxmox)
sudo systemctl is-active httpd    # doit afficher "active"
# Si inactif :
sudo systemctl start httpd

# Étape 3 : Apache écoute-t-il sur le port 80 ?
ss -tulpn | grep :80
# Si rien → Apache a un problème de démarrage
sudo journalctl -u httpd -n 20   # Lire les erreurs

# Étape 4 : Le firewall bloque-t-il le port 80 ?
sudo firewall-cmd --list-all | grep http
# Si http absent → ouvrir le port :
sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload

# Test de validation ultime :
# Depuis VM-CLIENT :
curl http://10.10.1.10     # Doit afficher le HTML
```

---

## 9. Tableau récapitulatif des erreurs curl courantes

| Erreur curl | Signification | Solution |
|-------------|---------------|----------|
| `curl: (7) Failed to connect` | Connexion refusée / service inactif | `systemctl start httpd` |
| `curl: (7) No route to host` | Pas de routage réseau | Activer ip_forward, vérifier les routes |
| `curl: (28) Connection timed out` | Firewall bloque | Ouvrir le port 80 dans firewall |
| `curl: (6) Could not resolve host` | Problème DNS | Utiliser l'IP directe plutôt que le nom |
| HTML reçu mais page vide | Fichier index.html absent | Créer `/var/www/html/index.html` |

---

## 10. Checklist complète avant validation

```bash
# Sur VM-WEB (10.10.1.10)
ip a show eth0                          # ✅ IP = 10.10.1.10
ip r | grep default                     # ✅ gateway = 10.10.1.254
sudo systemctl is-active httpd          # ✅ active
ss -tulpn | grep :80                    # ✅ Apache écoute
sudo firewall-cmd --list-services       # ✅ http présent
curl http://localhost | head -1         # ✅ HTML retourné

# Sur Proxmox
cat /proc/sys/net/ipv4/ip_forward       # ✅ = 1

# Sur VM-CLIENT (10.10.2.20)
ping -c 3 10.10.1.10                    # ✅ PING OK
curl http://10.10.1.10                  # ✅ Site affiché 🎉
```
