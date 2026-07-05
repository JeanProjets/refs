# Cheat Sheet — Proxmox : Bridges, Réseaux Isolés & Routage IP

> Référence pour la gestion réseau dans Proxmox VE.
> Architecture : **Proxmox (Debian)** avec bridges comme switches virtuels.

---

## 1. Architecture — Comment ça marche

```
Serveur Proxmox
│
├── nic0 (carte réseau physique) → connecté au monde extérieur
│    └── asservie à vmbr0 (sans IP sur nic0 !)
│
├── vmbr0 (Bridge principal) → porte l'IP du serveur Proxmox
│    IP: 192.168.122.100/24, GW: 192.168.122.1
│    bridge-ports: nic0 → connecté au réseau physique
│
├── vmbr1 (Bridge isolé #1) → réseau enfoui, pas de bridge-ports
│    IP: 10.10.1.254/24
│    Toutes les VMs branchées dessus = réseau 10.10.1.0/24
│
└── vmbr2 (Bridge isolé #2) → autre réseau enfoui
     IP: 10.10.2.254/24
     Toutes les VMs branchées dessus = réseau 10.10.2.0/24
```

**Règle d'or :** La carte physique (`nic0`) n'a **jamais** d'IP dans Proxmox.
C'est le bridge (`vmbr0`) qui porte l'IP de management.

---

## 2. `/etc/network/interfaces` — Syntaxe complète

```bash
# Voir / éditer
cat /etc/network/interfaces
sudo nano /etc/network/interfaces

# Appliquer les changements (Proxmox/Debian)
sudo ifreload -a
# Ou depuis l'interface web : System → Network → Apply Configuration
```

```text
# Loopback (toujours présent)
auto lo
iface lo inet loopback

# Carte physique — SANS IP (mode manuel)
iface nic0 inet manual

# Bridge principal (IP de management Proxmox)
auto vmbr0
iface vmbr0 inet static
    address 192.168.122.100/24
    gateway 192.168.122.1
    bridge-ports nic0        # ← branche la carte physique dedans
    bridge-stp off
    bridge-fd 0

# Bridge isolé #1 (réseau enfoui — VM-WEB)
auto vmbr1
iface vmbr1 inet static
    address 10.10.1.254/24
    bridge-ports none        # ← AUCUN port physique = réseau isolé !
    bridge-stp off
    bridge-fd 0

# Bridge isolé #2 (réseau enfoui — VM-CLIENT)
auto vmbr2
iface vmbr2 inet static
    address 10.10.2.254/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

---

## 3. Créer un bridge via l'interface web Proxmox

```
Proxmox → Nœud → System → Network → Create → Linux Bridge

Champs à remplir :
  Name:          vmbr1
  IPv4/CIDR:     10.10.1.254/24
  Gateway:       (laisser vide)
  Bridge ports:  (laisser vide) ← réseau enfoui
  Comment:       Réseau Web isolé

Puis : bouton "Apply Configuration"
```

---

## 4. Vérifier l'état des bridges

```bash
# Voir les IPs des bridges
ip a show vmbr1
ip a show vmbr2

# Voir la table de routage de Proxmox (doit avoir une entrée par bridge)
ip route
# Doit afficher :
# 10.10.1.0/24 dev vmbr1 proto kernel scope link src 10.10.1.254
# 10.10.2.0/24 dev vmbr2 proto kernel scope link src 10.10.2.254

# Lister tous les bridges
ip link show type bridge

# Voir quelles VMs sont connectées (interfaces TAP)
brctl show vmbr1
```

---

## 5. IP Forwarding — L'étape clé du routage inter-VM

> **Contexte :** Le Proxmox a une patte dans chaque réseau (vmbr1 et vmbr2).
> Il faut lui dire de **transférer les paquets** entre les deux réseaux.
> Sans cette étape, le ping inter-VM ne fonctionnera PAS.

```bash
# 1. Vérifier l'état actuel (0 = désactivé, 1 = activé)
cat /proc/sys/net/ipv4/ip_forward

# 2. Activer TEMPORAIREMENT (perdu au reboot)
echo 1 > /proc/sys/net/ipv4/ip_forward

# 3. Activer de façon PERMANENTE
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p        # Recharger sysctl sans rebooter

# 4. Vérifier
cat /proc/sys/net/ipv4/ip_forward
# Doit afficher : 1
```

---

## 6. Scénario complet : 2 VMs sur 2 réseaux différents

### Architecture du scénario

```
VM-WEB  (10.10.1.10)  ←→  [vmbr1] ←→ Proxmox ←→ [vmbr2]  ←→  VM-CLIENT (10.10.2.20)
        GW: 10.10.1.254                                          GW: 10.10.2.254
```

### Étapes dans l'ordre

#### Sur Proxmox (avant de créer les VMs)
```bash
# Créer vmbr1 et vmbr2 dans /etc/network/interfaces (voir section 2)
sudo ifreload -a
# Vérifier avec ip route → doit voir les deux réseaux
```

#### Dans VM-WEB (via console Proxmox)
```bash
# Identifier l'interface (souvent eth0 ou ens18)
ip link show

# Configurer l'IP statique
sudo nmcli connection modify "eth0" \
  ipv4.method manual \
  ipv4.addresses "10.10.1.10/24" \
  ipv4.gateway "10.10.1.254"
sudo nmcli connection up "eth0"

# Test : pinger la gateway
ping -c 3 10.10.1.254   # ✅ doit marcher
```

#### Dans VM-CLIENT (via console Proxmox)
```bash
sudo nmcli connection modify "eth0" \
  ipv4.method manual \
  ipv4.addresses "10.10.2.20/24" \
  ipv4.gateway "10.10.2.254"
sudo nmcli connection up "eth0"

# Test : pinger la gateway
ping -c 3 10.10.2.254   #  doit marcher
ping -c 3 10.10.1.10    #  échoue encore (ip_forward pas activé)
```

#### Sur Proxmox — Activer le routage
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

#### Vérification finale depuis VM-CLIENT
```bash
ping -c 3 10.10.1.10    #  doit marcher maintenant !
curl http://10.10.1.10  #  doit afficher le site Apache
```

---

## 7. Comprendre le trajet d'un paquet (à expliquer à voix haute)

```
VM-CLIENT (10.10.2.20) → veut joindre VM-WEB (10.10.1.10)

1. VM-CLIENT regarde sa table de routage :
   → 10.10.1.10 n'est PAS dans mon réseau (10.10.2.0/24)
   → J'envoie au gateway : 10.10.2.254 (= le Proxmox via vmbr2)

2. Le Proxmox reçoit le paquet sur vmbr2 :
   → ip_forward = 1 → OK je peux transférer
   → Je cherche une route vers 10.10.1.10 dans ma table
   → 10.10.1.0/24 dev vmbr1 → je l'envoie sur vmbr1

3. VM-WEB reçoit le paquet, répond en sens inverse

4. La réponse refait le même chemin en sens inverse
```

---

## 8. Donner accès Internet à une VM sur réseau enfoui

> Les VMs sur vmbr1/vmbr2 (sans bridge-ports) n'ont pas Internet.
> Pour installer des paquets dessus :

### Option A — Brancher temporairement sur vmbr0
```
Proxmox → VM → Hardware → Network Device → changer vmbr1 en vmbr0
Installer les paquets avec apt/dnf
Remettre sur vmbr1
```

### Option B — Ajouter une 2ème carte réseau temporaire
```
Proxmox → VM → Hardware → Add → Network Device → vmbr0
(La VM aura 2 cartes : eth0 sur vmbr1, eth1 sur vmbr0 en DHCP)
Installer les paquets
Retirer eth1
```

### Option C — NAT/Masquerade sur Proxmox (avancé)
```bash
# Si vmbr0 a Internet, on NAT les réseaux enfouis vers l'extérieur
iptables -t nat -A POSTROUTING -s 10.10.1.0/24 -o vmbr0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.10.2.0/24 -o vmbr0 -j MASQUERADE
```

---

## 9. Troubleshooting Proxmox — Problèmes courants

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Bridge n'apparaît pas | Config pas appliquée | `ifreload -a` ou "Apply Config" dans l'UI |
| Ping gateway KO | IP/GW mal configurée dans la VM | Vérifier `ip a` et `ip r` dans la VM |
| Ping inter-VM KO | `ip_forward` = 0 | `echo 1 > /proc/sys/net/ipv4/ip_forward` |
| Ping OK mais curl KO | Firewall bloque port 80 | `firewall-cmd --add-service=http --permanent && --reload` |
| VM n'a pas Internet | Réseau enfoui sans accès ext. | Brancher temporairement sur vmbr0 |
| Apache : `curl: (7) Failed to connect` | Service httpd pas démarré | `systemctl start httpd` |

---

## 10. Variante : VM-ROUTEUR dédiée (3 VMs)

> Si le test demande une VM dédiée au routage (au lieu du Proxmox lui-même) :

```bash
# VM-ROUTEUR a 2 cartes réseau :
# eth0 → vmbr1 (réseau 10.10.1.0/24)
# eth1 → vmbr2 (réseau 10.10.2.0/24)

# Configurer eth0
sudo nmcli connection modify "eth0" \
  ipv4.method manual ipv4.addresses "10.10.1.1/24"
sudo nmcli connection up "eth0"

# Configurer eth1
sudo nmcli connection modify "eth1" \
  ipv4.method manual ipv4.addresses "10.10.2.1/24"
sudo nmcli connection up "eth1"

# Activer le forwarding sur la VM-ROUTEUR
echo 1 > /proc/sys/net/ipv4/ip_forward
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# Modifier les gateways des autres VMs :
# VM-WEB  : gateway = 10.10.1.1 (la VM routeur, pas le Proxmox)
# VM-CLIENT : gateway = 10.10.2.1
```
