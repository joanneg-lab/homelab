# Homelab — Infrastructure personnelle de niveau production

> Documentation publique de mon homelab, conçu et opéré selon les principes d'une infrastructure d'entreprise : sécurité, séparation des rôles, moindre privilège, supervision complète et continuité de service.

---

## Overview / Vue d'ensemble

Ce homelab est une infrastructure multi-services déployée sur **Proxmox VE**, opérée en continu et maintenue selon les standards professionnels. Il me sert à la fois d'environnement d'apprentissage et de démonstration de compétences en administration d'infrastructures sécurisées.

**Objectifs :**
- Appliquer en conditions réelles les concepts vus en formation AIS
- Respecter les principes de sécurité (moindre privilège, défense en profondeur, RGPD)
- Maintenir une documentation à jour et une traçabilité complète
- Assurer la continuité de service (PCA/PRA)

---

## Hardware / Matériel

| Rôle | Matériel | Specs |
|---|---|---|
| Hyperviseur | Mini PC | AMD Ryzen 5 5500U · 32 Go RAM · SSD 500 Go |
| Backup | Serveur physique dédié (PBS) | SSD 500 Go + RAID5 3×2 To |
| Réseau | Switch principal | VLAN-aware |
| Réseau | Switch DMZ | Isolation DMZ |
| Wi-Fi | Point d'accès | WPA2 · Portails captifs |

---

## Architecture réseau / Network Architecture

### Segmentation VLAN

L'ensemble des VMs et conteneurs (LXC) sont déployés sur Proxmox VE, hébergé sur un Mini PC dédié.

| VLAN | Usage | Accès |
|---|---|---|
| VLAN Admin | Serveurs d'administration | Admins uniquement |
| VLAN DMZ | Services exposés (Nextcloud) | Accès contrôlé depuis internet |
| VLAN Employés | Postes filaires professionnels | Accès filaire authentifié |
| VLAN Famille | Wi-Fi famille | Portail captif LDAP |
| VLAN Invités | Wi-Fi visiteurs | Portail captif vouchers |
| VLAN Outils | Machines et équipements | Isolé |

### Principes appliqués
- **Isolation stricte** entre VLANs — pas de communication inter-VLAN sans règle explicite
- **Politique de pare-feu** : deny all par défaut, autorisation explicite uniquement
- **DMZ** : Nextcloud isolé du réseau d'administration
- **Split DNS** : résolution interne/externe séparée pour les services exposés

### Firewall / pfSense
- Pare-feu principal : pfSense
- Fonctions : firewall, DHCP, DNS, HAProxy (reverse proxy), WireGuard VPN, CA SSL interne
- Règles : politique drop par défaut, ouvertures minimales et documentées

---

## Services déployés / Deployed Services

### Virtualisation

| VM/CT | Service | Rôle |
|---|---|---|
| VM | OpenLDAP + LAM | Annuaire centralisé, gestion des identités |
| VM | FreeRADIUS | Authentification réseau, portails captifs, attribution VLAN |
| VM | Zabbix | Supervision et alertes |
| VM | Grafana | Dashboards et métriques |
| VM | Graylog | Centralisation et analyse des logs |
| VM | GLPI | Ticketing (intégration Zabbix automatique) |
| VM | Bastion SSH | Point d'entrée SSH sécurisé |
| VM | Nextcloud | Stockage cloud privé (Docker + Nginx + MariaDB) |
| CT | Passbolt CE | Gestionnaire de mots de passe d'équipe |
| Physique | PBS | Sauvegardes chiffrées Proxmox |

### Authentification centralisée (OpenLDAP)
- Annuaire unique pour tous les services
- Groupes mappés sur les VLANs (attribution automatique FreeRADIUS)
- Self-service changement de mot de passe via Nextcloud
- Convention de nommage stricte (uid, uidNumber)

### Accès distant (WireGuard VPN)
- Profil admin : accès complet au réseau d'administration
- Profil famille : accès Nextcloud uniquement
- Clés Ed25519, rotation planifiée

---

## Sécurité / Security

### Pare-feu local (nftables)
- **Policy drop** sur tous les serveurs — aucun service accessible par défaut
- Règles minimales et explicites par service
- Gestion spécifique des interfaces Docker (`br-*`)

### Bastion SSH
- Point d'entrée unique pour toute administration SSH
- Authentification par clé Ed25519 uniquement (pas de mot de passe)
- Fail2Ban actif
- Port non standard

### Certificats SSL
- CA interne gérée par pfSense
- Certificats déployés sur tous les services internes
- Certificats publics via Cloudflare pour les services exposés

### Gestion des secrets
- Passbolt CE : gestionnaire de mots de passe centralisé
- Aucun credential en clair dans les configurations
- Rotation des clés SSH planifiée

### Conformité RGPD
- Données personnelles limitées au strict nécessaire
- Logs centralisés avec durée de rétention définie
- Accès aux données tracé et auditable

---

## Supervision / Monitoring

### Stack de supervision
```
Zabbix (collecte + alertes)
    └── Grafana (dashboards et visualisation)

Graylog (centralisation des logs)
    ├── MongoDB (stockage)
    └── OpenSearch (indexation et recherche)
```

### Ce qui est supervisé
- Disponibilité de tous les services (uptime)
- Ressources système (CPU, RAM, disque, réseau)
- Logs système et applicatifs centralisés
- Alertes automatiques en cas d'anomalie
- Tickets GLPI générés automatiquement depuis les alertes Zabbix

---

## Sauvegardes / Backup Strategy

### Proxmox Backup Server (PBS)
- Sauvegardes chiffrées de toutes les VMs et CTs
- Stockage dédié : SSD système + RAID5 3×2 To pour les données
- **Stratégie GFS** (Grandfather-Father-Son) :
  - 7 sauvegardes quotidiennes
  - 4 sauvegardes hebdomadaires
  - 12 sauvegardes mensuelles
  - 1 sauvegarde annuelle
- Script de sauvegarde des configurations : exécuté chaque dimanche à 2h
- Vérification régulière de l'intégrité des sauvegardes

---

## Principes et méthodes / Principles & Methods

| Principe | Application |
|---|---|
| Moindre privilège | Chaque service tourne avec les droits minimaux nécessaires |
| Séparation des rôles | VLANs distincts, comptes dédiés par service |
| Défense en profondeur | Pare-feu périmétrique + nftables local + bastion SSH |
| Documentation continue | Chaque modification documentée avant déploiement |
| Traçabilité | Logs centralisés, commits Git signés |
| PCA/PRA | Sauvegardes chiffrées GFS, procédures de restauration testées |
| RGPD | Minimisation des données, durées de rétention définies |

---

## Tâches en cours / Work in Progress

- [ ] Configuration Cloudflare DNS + proxy pour domaine public
- [ ] Déploiement site vitrine public
- [ ] Audit de sécurité : Lynis, AIDE, rkhunter
- [ ] Benchmarks : sysbench, fio, iperf3
- [ ] Tests d'intrusion : Nikto

---

## Stack technique complète / Full Tech Stack

`Proxmox VE` `PBS` `Debian 12` `pfSense` `OpenLDAP` `FreeRADIUS` `Zabbix` `Grafana` `Graylog` `GLPI` `Nextcloud` `Passbolt CE` `Docker` `Nginx` `MariaDB` `HAProxy` `WireGuard` `nftables` `Fail2Ban` `Cloudflare` `Bash`

---

*Infrastructure opérée depuis avril 2026 — documentation mise à jour en continu.*
