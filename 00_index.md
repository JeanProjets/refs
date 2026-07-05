# Références Personnelles — Linux & Réseau

https://blog.stephane-robert.info/docs/admin-serveurs/linux/reseau/

---

##  Index

| Fichier | Contenu |
|---------|---------|
| [linux-networking.md](./linux-networking.md) | IP statique, nmcli, table de routage, VLAN, bridge |
| [firewall-linux.md](./firewall-linux.md) | firewalld (RHEL), UFW (Debian), rich rules |
| [proxmox-bridges.md](./proxmox-bridges.md) | Bridges Proxmox, réseaux isolés, ip_forward, `/etc/network/interfaces` |
| [apache-web.md](./apache-web.md) | Apache/httpd, pages web, curl/wget, dépannage |

---

## Commandes de survie (à mémoriser absolument)

```bash
ip a                         # Voir mes IPs
ip r                         # Voir la table de routage (gateway !)
nmcli connection show        # Lister les connexions (RHEL)
sudo firewall-cmd --list-all # Voir les règles firewall
systemctl status <service>   # État d'un service
journalctl -u <service> -n 30 # Logs d'un service
ss -tulpn                    # Ports en écoute
```

---

## Mémo des plages IP privées (RFC 1918)

| Plage | Masque | Usage typique |
|-------|--------|---------------|
| `10.0.0.0/8` | `255.0.0.0` | Grands réseaux entreprise |
| `172.16.0.0/12` | `255.240.0.0` | Taille moyenne |
| `192.168.0.0/16` | `255.255.0.0` | Petits réseaux, labs |

**Convention gateway :** `.1` ou `.254` sur le dernier octet.
Ex : réseau `10.10.1.0/24` → gateway souvent `10.10.1.1` ou `10.10.1.254`

---

## Checklist avant de rendre un travail

- [ ] `ip a` → IP correcte sur la bonne interface
- [ ] `ip r` → gateway correcte (`default via ...`)
- [ ] `sudo firewall-cmd --list-all` → règles firewall correctes
- [ ] `sudo firewall-cmd --list-rich-rules` → restrictions IP ok
- [ ] `nmcli connection show` → connexion active
- [ ] `systemctl is-active <service>` → service actif
- [ ] Redémarrer et re-vérifier → tout doit **persister** !
