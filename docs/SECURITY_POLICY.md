# Security Policy — Défense en profondeur et policies

> Documentation des principes de sécurité appliqués — firewall, nftables, authentification, chiffrement, audit.

---

## Philosophie de sécurité / Security Philosophy

### Principes appliqués

```
┌─ 1. DÉFENSE EN PROFONDEUR (Defense in Depth)
│  ├─ Niveau 1 : Périmètre (pfSense firewall)
│  ├─ Niveau 2 : Segmentation (VLANs)
│  ├─ Niveau 3 : Hôte (nftables policy DROP)
│  ├─ Niveau 4 : Application (service-level auth)
│  ├─ Niveau 5 : Données (chiffrement AES-256)
│  └─ Niveau 6 : Monitoring (détection intrusions)
│
├─ 2. MOINDRE PRIVILÈGE (Principle of Least Privilege)
│  ├─ Services tourne avec user non-root
│  ├─ Ports minimaux exposés
│  ├─ VLAN access : whitelist uniquement
│  ├─ SSH : clés Ed25519 (pas de mot de passe)
│  └─ Sudo : ACL strictes (audit trail)
│
├─ 3. DÉFAUT DENY (Fail Secure / Deny by Default)
│  ├─ Firewall pfSense : DROP all inbound
│  ├─ nftables sur serveurs : DROP all
│  ├─ VLAN access : explicit ALLOW uniquement
│  └─ Services : bind on localhost par défaut
│
├─ 4. TRAÇABILITÉ COMPLÈTE (Audit Trail)
│  ├─ Logs centralisés : Graylog (syslog)
│  ├─ Audit SSH : logs bastion + syslog
│  ├─ Firewall logs : pfSense + nftables
│  ├─ Application logs : service-specific
│  └─ Rétention : 90 jours minimum
│
├─ 5. AUTHENTIFICATION CENTRALISÉE
│  ├─ Backend : OpenLDAP (annuaire unique)
│  ├─ Authentification : PEAP/EAP-PEAP
│  ├─ Authorization : groupes LDAP + VLAN assignment
│  └─ Gestion secrets : Passbolt CE (vault)
│
└─ 6. CHIFFREMENT SYSTÉMATIQUE
   ├─ Transport : TLS/SSL (certificats internes + publics)
   ├─ Données : AES-256 (PBS backups)
   ├─ Keys : gestion centralisée Passbolt
   └─ Pas de credentials en clair (config files)
```

---

## Défense en profondeur / Defense in Depth Layers

```
┌──────────────────────────────────────────────┐
│ NIVEAU 6 : MONITORING & DÉTECTION            │
│ Zabbix (alertes), Graylog (logs), GLPI       │
├──────────────────────────────────────────────┤
│ NIVEAU 5 : APPLICATION SECURITY              │
│ Auth centralisée (LDAP), bastion SSH, tokens │
├──────────────────────────────────────────────┤
│ NIVEAU 4 : CHIFFREMENT                       │
│ TLS/SSL (transport), AES-256 (données)       │
├──────────────────────────────────────────────┤
│ NIVEAU 3 : FIREWALL HÔTE (nftables)          │
│ Policy DROP + rules explicites par service   │
├──────────────────────────────────────────────┤
│ NIVEAU 2 : SEGMENTATION RÉSEAU (VLANs)       │
│ Isolation inter-VLAN, firewall pfSense       │
├──────────────────────────────────────────────┤
│ NIVEAU 1 : PÉRIMÈTRE (pfSense)               │
│ Firewall stateful, NAT, IDS/IPS              │
├──────────────────────────────────────────────┤
│ NIVEAU 0 : PHYSIQUE                          │
│ Accès contrôlé                               │
└──────────────────────────────────────────────┘

À chaque NIVEAU :
- Une intrusion est ralentie/bloquée
- L'attaquant doit contourner plusieurs systèmes
- Logs à chaque étape (traçabilité)
```

---

## nftables Policy DROP / nftables Configuration

**Principe : chaque VM/CT dispose de ses propres règles nftables — uniquement les ports nécessaires à ses services sont ouverts (moindre privilège).**

### Structure commune à toutes les VMs

```
┌────────────────────────────────────────────────┐
│ nftables Firewall (Linux kernel packet filter) │
└────────────┬───────────────────────────────────┘
             │
        ┌────┴────┐
        ↓         ↓
    CHAINS      RULES
        │
    input   ←─ policy: DROP (default deny)
    output  ←─ policy: ACCEPT (syslog, NTP, DNS, métriques Zabbix)
    forward ←─ policy: DROP (pas de routage)

Règles communes à toutes les VMs :
├─ loopback (lo) : ACCEPT
├─ established/related : ACCEPT
├─ invalid : DROP
├─ ICMP : ACCEPT (diagnostics)
└─ Zabbix agent (TCP 10050) : ACCEPT
   (supervision centralisée depuis VM102)
```

### Règles par VM/CT

```
┌─ VM100 : OpenLDAP + LAM
│  TCP 389   — LDAP (depuis VLAN99 + VLAN50)
│  TCP 636   — LDAPS (depuis VLAN99 + VLAN50)
│  TCP 443   — LAM interface web (depuis VLAN99)
│  TCP 10050 — Zabbix agent
│
├─ VM101 : FreeRADIUS
│  UDP 1812  — RADIUS auth (depuis pfSense/WAX200)
│  UDP 1813  — RADIUS accounting
│  TCP 10050 — Zabbix agent
│
├─ VM102 : Zabbix Server
│  TCP 10051 — Zabbix server (depuis agents)
│  TCP 443   — Interface web (depuis VLAN99)
│  TCP 10050 — Zabbix agent (auto-supervision)
│
├─ VM103 : Grafana
│  TCP 3000  — Interface web (depuis VLAN99)
│  TCP 10050 — Zabbix agent
│
├─ VM104 : Graylog
│  TCP 9000  — Interface web (depuis VLAN99)
│  UDP 514   — Syslog (depuis tous les serveurs)
│  TCP 514   — Syslog TCP
│  TCP 5555  — GELF (depuis applications)
│  TCP 10050 — Zabbix agent
│
├─ VM105 : GLPI
│  TCP 443   — Interface web (depuis VLAN99)
│  TCP 10050 — Zabbix agent
│
├─ VM106 : Bastion SSH
│  TCP 22    — SSH (depuis VLAN99 uniquement)
│  TCP 10050 — Zabbix agent
│  ⚠️  Aucun autre port ouvert
│
├─ VM107 : Nextcloud (Docker)
│  TCP 443   — HTTPS (depuis VLAN50 via HAProxy)
│  TCP 10050 — Zabbix agent
│  Note : MariaDB (3306) bind localhost uniquement
│
└─ CT108 : Passbolt CE (LXC + Docker)
   TCP 443   — HTTPS (depuis VLAN99)
   TCP 10050 — Zabbix agent
   Note : MySQL bind localhost uniquement
```

### Exemple — VM100 (OpenLDAP) `/etc/nftables.conf`

```bash
#!/usr/bin/nftables -f

# ============================================
# nftables — VM100 OpenLDAP
# Ports ouverts : 389, 636, 443, 10050
# ============================================

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Loopback
        iif "lo" accept

        # Connexions établies
        ct state established,related accept
        ct state invalid drop

        # ICMP
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # LDAP — depuis VLAN99 et VLAN50 uniquement
        ip saddr { 10.10.99.0/24, 10.10.50.0/24 } tcp dport 389 ct state new accept comment "LDAP"
        ip saddr { 10.10.99.0/24, 10.10.50.0/24 } tcp dport 636 ct state new accept comment "LDAPS"

        # LAM (interface web admin) — depuis VLAN99 uniquement
        ip saddr 10.10.99.0/24 tcp dport 443 ct state new accept comment "LAM web"

        # Zabbix agent — depuis VM Zabbix uniquement
        ip saddr 10.10.99.X tcp dport 10050 ct state new accept comment "Zabbix agent"

        # Drop tout le reste
        counter drop comment "DROP all unmatched"
    }

    chain output {
        type filter hook output priority 0; policy accept;
        # Sorties autorisées implicitement, dont :
        # UDP 514 → Graylog (10.10.99.X) — syslog
        # TCP 10051 → Zabbix server — métriques
        # UDP 123 → pfSense — NTP
        # UDP 53 → pfSense — DNS
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }
}
```

### Gestion des règles nftables

```bash
# Charger la configuration au démarrage
sudo systemctl enable nftables
sudo systemctl start nftables

# Appliquer les changements
sudo nft -f /etc/nftables.conf

# Vérifier les règles
sudo nft list table inet filter

# Voir les statistiques (règles déclenchées)
sudo nft list table inet filter verbose

# Dump les règles en JSON
sudo nft -j list table inet filter
```

### Logging nftables

Pour auditer les paquets rejetés (envoyer vers Graylog) :

```bash
# Ajouter logging dans les règles DROP
counter drop comment "DROP all unmatched inbound" \
    | log prefix "nftables-drop: " level warning

# Configuration rsyslog pour nftables
# /etc/rsyslog.d/99-nftables.conf
:msg,contains,"nftables-drop" @@10.10.99.X:514
```

---

## Firewall pfSense / pfSense Firewall Rules

### Architecture pfSense

```
┌─────────────────────────────────────────┐
│ pfSense (Routing + Firewall)             │
├─────────────────────────────────────────┤
│                                         │
│  WAN Interface ──────────────────────┐  │
│   (Internet, VPN)                    │  │
│       │                              │  │
│   [Firewall Rules]                   │  │
│       │                              │  │
│   [NAT / DNAT]                       │  │
│       │                              │  │
│   [Routing]                          │  │
│       │                              │  │
│  LAN Interfaces ◄────────────────────┘  │
│   VLAN99, VLAN50, VLAN20, etc.         │
│       │                              │  │
│   [Inter-VLAN Rules]                 │  │
│       │                              │  │
│   [DHCP / DNS]                       │  │
│       │                              │  │
│  [Syslog export] ──→ Graylog         │  │
│                                      │  │
└─────────────────────────────────────────┘
```

### Règles WAN (Internet → LAN)

```
┌─ INBOUND WAN RULES (stateful)
│
├─ Rule 1 : HTTPS (443) → HAProxy (Nextcloud)
│  Source : Any
│  Destination : WAN IP (pfSense)
│  Protocol : TCP
│  Port : 443
│  Action : ALLOW
│  Redirect : 10.10.50.X:443 (NAT)
│  Logging : YES
│  Description : "Allow Nextcloud HTTPS"
│
├─ Rule 2 : SSH Bastion (2222)
│  Source : Any
│  Destination : WAN IP (pfSense)
│  Protocol : TCP
│  Port : 2222
│  Action : ALLOW
│  Redirect : 10.10.99.X:22 (NAT)
│  Logging : YES (critical)
│  Description : "SSH Bastion (with Fail2Ban)"
│
├─ Rule 3 : WireGuard VPN (51820)
│  Source : Any
│  Destination : WAN IP (pfSense)
│  Protocol : UDP
│  Port : 51820
│  Action : ALLOW
│  Logging : YES
│  Description : "WireGuard VPN endpoint"
│
└─ [DEFAULT] : DROP all inbound
   Logging : YES (rate-limited to prevent log spam)
```

### Rules Inter-VLAN (LAN ↔ LAN)

Voir `NETWORK_DESIGN.md` pour la liste complète.

---

## Authentification & Authorization

### OpenLDAP (Annuaire centralisé)

```
Base DN : dc=prod,dc=exemple-info,dc=org

Organisational Structure :
├─ cn=admin (admin user)
├─ ou=users
│  ├─ uid=joanne (uidNumber: 2000, gidNumber: 1000)
│  ├─ uid=john (uidNumber: 2001, gidNumber: 1001)
│  └─ [autre users]
│
├─ ou=groups
│  ├─ cn=admins (gidNumber: 1000) → VLAN99
│  ├─ cn=employes (gidNumber: 1001) → VLAN20
│  ├─ cn=membres (gidNumber: 1002) → VLAN20
│  ├─ cn=famille (gidNumber: 1003) → VLAN70
│  ├─ cn=invites (gidNumber: 1004) → VLAN80
│  └─ cn=guest-vouchers (gidNumber: 1099) → Vouchers
│
└─ ou=services
   ├─ uid=nextcloud-svc
   ├─ uid=zabbix-svc
   └─ [service accounts]

Access Control List (ACL) :
- Admin : can write all
- Users : can read own entry + change password
- Service accounts : can read LDAP tree (restricted DN)
```

### FreeRADIUS (PEAP-MSCHAPv2)

```
┌─ Protocol Flow
│
├─ Client → FreeRADIUS : EAP-PEAP request
├─ FreeRADIUS → OpenLDAP : uid=john query
├─ OpenLDAP ← FreeRADIUS : cn=john password hash
├─ FreeRADIUS : validate password (PEAP tunnel)
├─ Success ? Check group (LDAP)
├─ Assign VLAN : based on cn=groupe → gidNumber
├─ FreeRADIUS → Client : Access-Accept + VLAN ID
└─ Client : joins VLAN + receives DHCP
```

### Bastion SSH (Point d'entrée unique)

```
┌─ SSH Config (/etc/ssh/sshd_config)
│
├─ Port : 2222 (non-standard)
├─ Protocol : 2 only
├─ PermitRootLogin : NO
├─ PasswordAuthentication : NO (keys only)
├─ PubkeyAuthentication : YES
├─ AuthorizedKeysFile : %h/.ssh/authorized_keys
├─ LogLevel : VERBOSE (audit trail)
├─ SyslogFacility : AUTH
├─ Match User *
│  └─ PasswordAuthentication : NO
│     (all users must use keys)
│
└─ Key Management
   ├─ Key Type : Ed25519 only (no RSA)
   ├─ Keys stored : Passbolt CE
   ├─ Access : via Bastion → Proxmox machines
   └─ Audit : all SSH sessions logged

┌─ Fail2Ban Configuration
│
├─ Jail : sshd
├─ Max Retries : 3
├─ Ban Time : 24h (first offense)
├─ Find Time : 10 min (detection window)
├─ Action : iptables + email alert
└─ Logs : /var/log/auth.log → Graylog
```

---

## Gestion des secrets / Secret Management

### Passbolt CE (Vault centralisé)

```
┌─ Passbolt Server (10.10.99.X)
│
├─ Credentials stored
│  ├─ LDAP admin password
│  ├─ FreeRADIUS shared secrets
│  ├─ MariaDB root password
│  ├─ Nextcloud admin password
│  ├─ SSH private keys (Ed25519)
│  ├─ PBS encryption keys
│  └─ API tokens (Zabbix, etc.)
│
├─ Access Control
│  ├─ Users need Passbolt login
│  ├─ Password encrypted with user's GPG key
│  ├─ Share folder model (read/write/admin)
│  └─ Audit trail (who accessed what/when)
│
└─ Deployment
   ├─ Docker container (LXC CT108)
   ├─ HTTPS only (self-signed cert)
   ├─ MySQL backend (encrypted)
   └─ Backups : PBS GFS retention
```

### Secret Rotation Policy

```
┌─ Schedule
├─ SSH keys : every 90 days
├─ LDAP password : every 180 days
├─ API tokens : every 30 days
├─ Root passwords : every year
└─ Encryption keys : only on major events

Procédure :
1. Générer un nouveau secret (dans Passbolt)
2. Tester le nouveau secret sur une VM de préproduction
3. Déployer en production
4. Mettre à jour les services dépendants
5. Marquer l’ancien secret comme obsolète
6. Le supprimer après une période de rétention de 30 jours
```

---

## Chiffrement / Encryption

### Transport Layer (TLS/SSL)

```
┌─ Internal Services (VLAN99)
│  ├─ OpenLDAP : LDAPS (port 636)
│  ├─ Zabbix : HTTPS (self-signed cert)
│  ├─ Grafana : HTTPS (self-signed cert)
│  └─ Graylog : HTTPS (self-signed cert)
│  Certs : CA interne (pfSense)
│
├─ Public Services (Internet)
│  ├─ Nextcloud : HTTPS (Let's Encrypt)
│  └─ Domain : cloud.exemple-info.org
│  Certs : via Cloudflare
│
└─ Certificate Management
   ├─ Authority : pfSense CA (internal)
   ├─ Validity : 365 days
   ├─ Renewal : 30 days before expiry
   └─ Monitoring : Zabbix cert expiry alerts
```

### Data at Rest (Backup Encryption)

```
┌─ PBS (Proxmox Backup Server)
│
├─ Encryption Method : AES-256-CBC
├─ Key Management : Passbolt CE (master key)
├─ Key Rotation : Annual (with alert)
├─ Storage : /mnt/raid (RAID5 3×2TB)
│
├─ Backup Workflow
│  ├─ VM100-VM107 : encrypted snapshot
│  ├─ Sent to PBS : over HTTPS
│  ├─ Stored encrypted : on disk
│  └─ Can only decrypt with master key
│
└─ Recovery
   ├─ Retrieve backup : PBS API
   ├─ Decrypt : with master key (from Passbolt)
   ├─ Restore to Proxmox : full VM restore
   └─ No key = no recovery possible
```

---

## Monitoring & Alerting Security

### Zabbix Security Monitoring

```
┌─ Monitored Items
│
├─ Firewall
│  ├─ Dropped packets rate (nftables)
│  ├─ SSH failed login attempts
│  ├─ Port scans detected (Fail2Ban)
│  └─ VLAN access violations
│
├─ Services
│  ├─ Service availability (up/down)
│  ├─ Port listening (unexpected ports)
│  ├─ Process integrity (expected daemons)
│  └─ Configuration changes (file integrity)
│
├─ System
│  ├─ File integrity (AIDE)
│  ├─ Log file access
│  ├─ Sudo usage
│  └─ Cron job execution
│
└─ Thresholds & Alerting
   ├─ Alert on : suspicious patterns
   ├─ Create GLPI ticket : auto for critical
   ├─ Escalation : email + SMS for critical
   └─ Logs : centralized to Graylog
```

### Graylog Security Logging

```
┌─ Log Sources
│
├─ pfSense firewall
├─ nftables (all DROP rules)
├─ SSH (Bastion access logs)
├─ SUDO usage
├─ File access (AIDE)
├─ LDAP authentication
├─ FreeRADIUS auth attempts
└─ Application logs (all services)

Parsing Rules :
├─ Extract source IP
├─ Extract username
├─ Extract action (allow/deny)
├─ Extract service/port
└─ Correlation : IP → user → time → action

Retention Policy :
├─ All logs : 90 days minimum
├─ Security events : 1 year
├─ Backups : unlimited (GFS)
└─ Compliance : RGPD data handling
```

---

## Audit & Compliance

### RGPD Compliance

```
┌─ Data Inventory
│
├─ Personal Data Stored
│  ├─ User LDAP entries : name, email, uid
│  ├─ SSH keys : associated with users
│  ├─ Nextcloud storage : user files
│  ├─ Logs : IP addresses, timestamps
│  └─ Backups : includes all above
│
├─ Data Processing
│  ├─ Purpose : infrastructure operation
│  ├─ Retention : see below
│  ├─ Processing : within VLAN99 (admin only)
│  └─ Transfer : none to third parties
│
├─ Retention Schedule
│  ├─ Active accounts : indefinite (while in use)
│  ├─ Deleted accounts : 30 days (for cleanup)
│  ├─ Audit logs : 90 days
│  ├─ Backups : 12 months
│  └─ Incident logs : 2 years (legal hold)
│
└─ User Rights
   ├─ Right to access : via Passbolt/GLPI
   ├─ Right to delete : request to admin
   ├─ Right to portability : via Nextcloud export
   └─ Right to rectify : self-service LDAP password change
```

### Change Management & Audit Trail

```
┌─ Change Log (documented before deployment)
│
├─ Entry fields
│  ├─ Date + Time
│  ├─ Changed by (user)
│  ├─ What changed (service/config)
│  ├─ Before → After (specific diff)
│  ├─ Reason (ticket reference)
│  ├─ Testing (test plan + result)
│  └─ Approval (admin sign-off)
│
└─ Storage
   ├─ Primary : Git repo (signed commits)
   ├─ Backup : GLPI ticket
   └─ Logs : Graylog (audit trail)
```

---

## Incident Response / Incident Response Plan

```
┌─ Detection Phase
│  ├─ Zabbix alert triggered
│  ├─ GLPI ticket auto-created
│  ├─ Logs analyzed in Graylog
│  └─ Timeline established
│
├─ Containment Phase
│  ├─ Isolate affected VM (VLAN)
│  ├─ Kill suspicious processes
│  ├─ Block attacker IP (firewall)
│  └─ Preserve forensic evidence (logs)
│
├─ Investigation Phase
│  ├─ Review nftables/firewall logs
│  ├─ Check SSH bastion logs
│  ├─ Check file integrity (AIDE)
│  ├─ Analyze suspicious traffic (tcpdump)
│  └─ Document findings in GLPI
│
├─ Recovery Phase
│  ├─ Restore VM from backup (PBS)
│  ├─ Verify service integrity
│  ├─ Reapply security patches (CVE scan)
│  └─ Change credentials (Passbolt)
│
└─ Lessons Learned Phase
   ├─ Post-incident review (meeting)
   ├─ Root cause analysis
   ├─ Policy updates (if needed)
   └─ Security awareness training
```

---

**Dernière mise à jour** : Mai 2026
