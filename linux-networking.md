# 🌐 Cheat Sheet — Réseau Linux (nmcli, ip, VLAN, Bridge)

> Testé sur : **Rocky Linux 9** (RHEL) et **Debian 12**
> Outil principal RHEL : `nmcli` (persistant). `ip` = temporaire seulement.

---

## 1. Diagnostic rapide — Toujours commencer par là

```bash
# État de toutes les interfaces
ip a

# Voir l'interface d'une VM (souvent eth0 ou ens18 dans les VMs Proxmox)
ip link show

# Table de routage — indispensable pour débugger
ip r
# La ligne "default via X.X.X.X" = ma gateway (routeur)
# Les autres lignes = réseaux directement connectés

# Identifier le nom exact de ma connexion NetworkManager
nmcli connection show
# Retient bien le nom entre guillemets ! Ex: "System eth0" ou "ens18"
```

---

## 2. Configurer une IP statique — RHEL/Rocky (nmcli)

>  Remplace `"eth0"` par le **vrai nom** de ta connexion (`nmcli connection show`)

```bash
# Étape 1 : Passer en mode manuel (arrêter le DHCP)
sudo nmcli connection modify "eth0" ipv4.method manual

# Étape 2 : Définir l'IP et le masque
sudo nmcli connection modify "eth0" ipv4.addresses "10.10.1.10/24"

# Étape 3 : Définir la gateway (CRITIQUE pour le routage inter-réseau)
sudo nmcli connection modify "eth0" ipv4.gateway "10.10.1.254"

# Étape 4 : DNS (optionnel si pas de résolution DNS nécessaire)
sudo nmcli connection modify "eth0" ipv4.dns "8.8.8.8 8.8.4.4"

# Étape 5 : APPLIQUER (obligatoire !)
sudo nmcli connection up "eth0"

# Vérifier
ip a show eth0
ip r
# Doit afficher : inet 10.10.1.10/24 et default via 10.10.1.254
```

### Tout en une seule commande (plus rapide) :

```bash
sudo nmcli connection modify "eth0" \
  ipv4.method manual \
  ipv4.addresses "10.10.1.10/24" \
  ipv4.gateway "10.10.1.254" \
  ipv4.dns "8.8.8.8"
sudo nmcli connection up "eth0"
```

---

## 3. Configurer une IP statique — Debian/Ubuntu/Proxmox

>  **Important :** Vérifie toujours le nom réel de ta carte réseau (souvent `ens18` sur Proxmox) avec la commande `ip link show` avant de configurer.
> Fichier de config : `/etc/network/interfaces`

```bash
# 1. Identifier le nom de l'interface
ip link show

# 2. Éditer le fichier
sudo nano /etc/network/interfaces
```

```text
# Exemple pour VM-WEB sur Proxmox (remplace ens18 par ton interface si différent)
auto ens18
iface ens18 inet static
    address 10.10.1.10/24
    gateway 10.10.1.254
```

```bash
# Appliquer
sudo systemctl restart networking
# Ou juste l'interface :
sudo ifdown ens18 && sudo ifup ens18

# Vérifier
ip a
ip r
```

---

## 4. Changer le hostname

```bash
# RHEL et Debian
sudo hostnamectl set-hostname srv-supervision

# Vérifier immédiatement
hostname
hostnamectl status
```

---

## 5. Créer un VLAN (interface taguée)

> Prérequis : module `8021q` chargé

```bash
# Vérifier / charger le module
lsmod | grep 8021q
sudo modprobe 8021q

# Créer VLAN 10 sur l'interface eth0 (RHEL/nmcli)
sudo nmcli connection add \
  type vlan \
  con-name "vlan10" \
  ifname "eth0.10" \
  dev eth0 \
  id 10 \
  ipv4.method manual \
  ipv4.addresses "172.16.10.1/24"

sudo nmcli connection up "vlan10"

# Vérifier
ip a show eth0.10
ip link show eth0.10
```

```text
# [Debian] Dans /etc/network/interfaces :
auto eth0.10
iface eth0.10 inet static
    address 172.16.10.1/24
    vlan-raw-device eth0
```

---

## 6. Créer un bridge (switch virtuel local) — RHEL

> Utile pour simuler les bridges Proxmox sur une Rocky Linux

```bash
sudo nmcli connection add \
  type bridge \
  con-name "br-isole" \
  ifname "br-isole" \
  ipv4.method manual \
  ipv4.addresses "10.200.0.254/24"

sudo nmcli connection up "br-isole"

# Vérifier
ip a show br-isole
ip r  # Doit voir 10.200.0.0/24 dev br-isole
```

---

## 7. Routage — Table de routage expliquée

```bash
ip r
# Exemple de sortie :
# default via 10.10.2.254 dev eth0          ← la gateway (tout ce qui est "ailleurs")
# 10.10.2.0/24 dev eth0 proto kernel        ← mon propre réseau (direct)
```

### Ajouter une route temporaire (débogage rapide) :

```bash
# Ajouter une route vers un autre réseau via une gateway
ip route add 10.10.1.0/24 via 10.10.2.254

# Supprimer
ip route del 10.10.1.0/24
```

### Ajouter une gateway par défaut temporairement :

```bash
ip route add default via 10.10.1.254
```

---

## 8. Retour en DHCP (si besoin de reset)

```bash
# RHEL
sudo nmcli connection modify "eth0" ipv4.method auto
sudo nmcli connection modify "eth0" ipv4.addresses ""
sudo nmcli connection modify "eth0" ipv4.gateway ""
sudo nmcli connection modify "eth0" ipv4.dns ""
sudo nmcli connection up "eth0"
```

---

## 9. Troubleshooting réseau — Séquence de diagnostic

```bash
# 1. Mon interface est-elle UP ?
ip link show eth0
# chercher : state UP

# 2. Est-ce que j'ai une IP ?
ip a show eth0
# chercher : inet X.X.X.X/XX

# 3. Est-ce que j'ai une gateway ?
ip r
# chercher : default via X.X.X.X

# 4. Puis-je joindre la gateway ?
ping -c 3 <IP_GATEWAY>
# Si ça ne marche pas → problème L2 (câble virtuel, bridge, VLAN)

# 5. Puis-je joindre l'autre réseau ?
ping -c 3 <IP_AUTRE_VM>
# Si ça ne marche pas mais gateway OK → routage pas activé (ip_forward !)

# 6. Le service répond-il ?
curl http://<IP>
# Si ping OK mais curl KO → firewall bloque le port !
ss -tulpn | grep :80  # Apache écoute-t-il vraiment ?
```

---

## 10. Équivalences RHEL ↔ Debian à connaître

| Action | RHEL/Rocky | Debian/Ubuntu |
|--------|-----------|---------------|
| Paquets | `dnf install` | `apt install` |
| Config réseau | `nmcli` | `/etc/network/interfaces` |
| Appliquer réseau | `nmcli connection up` | `systemctl restart networking` |
| Firewall | `firewall-cmd` | `ufw` |
| Apache | `httpd` | `apache2` |
| Racine web | `/var/www/html/` | `/var/www/html/` |
