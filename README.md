# 🏢 Enterprise Network Architecture — GreenTech Dynamics

> Conception d'une architecture réseau enterprise complète pour un siège social de 400 employés, incluant campus, datacenter, SD-WAN, NAC, automatisation GitOps et stratégie de sécurité en profondeur.

📄 **[Article Medium détaillé](https://medium.com/@broquard.matthieu/)** — Le parcours complet avec les erreurs et les apprentissages

---

## 📋 Le projet

Exercice de synthèse réalisé dans le cadre de ma formation cybersécurité chez [Jedha Academy](https://www.jedha.co/). Le scénario : concevoir l'infrastructure réseau complète d'un nouveau siège social pour GreenTech Dynamics Inc., une entreprise spécialisée dans les solutions IoT pour bâtiments intelligents.

**Contraintes principales :**
- 400 employés sur 6 étages + datacenter au sous-sol
- Bureau satellite à Paris (80 employés) + site San Francisco
- 200+ capteurs IoT, 60 caméras, 450 téléphones IP
- SLA 99.9% (< 9h d'indisponibilité/an)
- Conformité ISO 27001, GDPR
- 3 liaisons WAN (MPLS 10G + Fibre 1G + 4G/5G)

---

## 🏗 Architecture globale

```
                              INTERNET / CLOUD
                             ╱       │        ╲
                      Orange MPLS  Bouygues   SFR 4G
                        10 Gbps    1 Gbps    300 Mbps
                             ╲       │        ╱
                        ┌────────────────────────┐
                        │  FortiGate HA + SD-WAN │
                        └───────────┬────────────┘
                                    │
                        ┌───────────┴────────────┐
                        │     CORE (2× Stack)     │
                        └──┬─────────┬────────┬──┘
                           │         │        │
                     ┌─────┴──┐  ┌───┴───┐  ┌─┴────────┐
                     │DISTRIB.│  │ SPINE │  │FIREWALLS │
                     │×6 étages│ │  ×2   │  │  inter-  │
                     └───┬────┘  └───┬───┘  │  zones   │
                         │           │      └──────────┘
                     ┌───┴────┐  ┌───┴───┐
                     │ ACCESS │  │ LEAF  │
                     │switches│  │  ×4   │
                     │ PoE+   │  └───┬───┘
                     └───┬────┘      │
                         │           │
                     Utilisateurs  Serveurs
                     WiFi, VoIP   Docker/K8s
                     IoT, Caméras MongoDB
```

---

## 🔑 Décisions techniques

### Topologie réseau

| Domaine | Choix | Justification |
|---------|-------|---------------|
| Campus | Core-Distribution-Access (3 tiers) | Trafic nord-sud, 6 étages hiérarchiques |
| Datacenter | Leaf-Spine + ECMP + VXLAN/EVPN | Trafic est-ouest Docker/K8s, max 2 sauts |
| Overlay DC | VXLAN | 16M segments, mobilité conteneurs K8s |

### WAN & Connectivité

| Domaine | Choix | Justification |
|---------|-------|---------------|
| WAN Lyon | SD-WAN hybride sur 3 liaisons | Utilisation simultanée MPLS + fibre + 4G |
| Paris ↔ Lyon | SD-WAN + tunnel chiffré | Local breakout SaaS, sélection dynamique |
| San Francisco | CPE SD-WAN + ZTP | Déployable en heures, scalable |
| AWS | Direct Connect 1 Gbps | Trafic IoT dédié, bande passante garantie |
| Télétravailleurs | ZTNA (FortiClient) | Accès par application, pas par réseau |
| Détection pannes | BFD + IP SLA | Basculement < 50ms |

### Sécurité (30% de la note)

| Domaine | Choix | Justification |
|---------|-------|---------------|
| Stratégie | Défense en profondeur (7 couches) | ISO 27001, données R&D sensibles |
| NAC | Cisco ISE (802.1X + MAB + portail captif) | PKI existante, profiling IoT |
| Segmentation | 13 VLANs campus + 5 VNIs datacenter | Isolation par rôle et par risque |
| Firewalling | FortiGate HA inter-zones | Default deny, règles explicites |
| Monitoring | Splunk (SIEM) + Zabbix + FortiAnalyzer | Corrélation 24/7, logs 12 mois |
| Évolution | Zero Trust (ZTNA + microsegmentation) | Télétravailleurs + datacenter |

### Automatisation

| Domaine | Choix | Justification |
|---------|-------|---------------|
| Source de vérité | NetBox (inventaire + IPAM) | SSoT, inventaire dynamique Ansible |
| Déploiement | Ansible + AWX | 400+ équipements, 10 rôles |
| Versioning | Git (GitLab) | Traçabilité ISO 27001 |
| Validation | GitLab CI (lint → check → approve) | Aucun déploiement sans revue |
| Backup configs | Oxidized → Git | Détection de dérive, diff, alertes |
| Monitoring | Zabbix + Grafana | SNMP, dashboards |

---

## 📁 Contenu du repo

```
enterprise-network-architecture/
│
├── README.md                          ← Ce fichier
│
├── docs/
│   ├── 01_dossier_technique.md        ← Dossier complet (16 sections)
│   ├── 02_plan_adressage_ip.md        ← Adressage détaillé campus + DC
│   ├── 03_matrice_flux.md             ← Règles inter-zones
│   └── 04_budget_tco.md              ← CAPEX + OPEX + TCO 5 ans
│
├── schemas/
│   ├── architecture_globale.png       ← Schéma d'ensemble
│   ├── campus_3tiers.png              ← Design campus
│   ├── datacenter_leafspine.png       ← Design datacenter
│   ├── wan_sdwan.png                  ← Architecture WAN
│   ├── securite_defense_profondeur.png← Les 7 couches
│   └── gitops_workflow.png            ← Workflow automatisation
│
├── ansible/
│   ├── inventory/
│   │   └── netbox_inventory.yml       ← Plugin inventaire dynamique
│   ├── roles/
│   │   ├── base_config/               ← Hostname, NTP, DNS, logging
│   │   ├── security_hardening/        ← SSH, ACL, port-security
│   │   ├── vlan_management/           ← Création et assignation VLANs
│   │   ├── nac_802.1x/                ← Config 802.1X, MAB, RADIUS
│   │   ├── qos_policy/                ← Marquage DSCP, queuing VoIP
│   │   └── monitoring_agent/          ← SNMP, syslog, Zabbix
│   └── site.yml                       ← Playbook principal
│
└── article/
    └── medium_article.md              ← Brouillon de l'article Medium
```

---

## 🔐 Perspective cybersécurité

Ce projet a été réalisé avec une double vision **défense + offensive**. Chaque choix de design a été évalué sous l'angle : "comment un attaquant contournerait ça ?"

Quelques exemples :

- **MAB est spoofable** → `macchanger` suffit pour usurper une imprimante. Contre-mesure : device profiling
- **Fail-open NAC** → DoS sur le serveur ISE = tout le monde entre. Contre-mesure : cluster HA
- **Prises physiques non protégées** → Salles de réunion, couloirs. Contre-mesure : 802.1X sur 100% des ports
- **Serveur Ansible** → Cible de haute valeur (credentials SSH de tous les switches). Contre-mesure : segmentation, MFA, audit logs
- **Local breakout SD-WAN** → Le trafic SaaS ne passe pas par le firewall central. Contre-mesure : CASB ou firewall local

---

## 🛠 Stack technique

**Réseau** : Cisco Catalyst (campus), Cisco Nexus ou Arista (datacenter), FortiGate (firewalls + SD-WAN)

**Sécurité** : Cisco ISE (NAC), Splunk (SIEM), FortiEDR (endpoint), FortiAnalyzer

**Automatisation** : Ansible + AWX, Oxidized, NetBox, GitLab CI

**Monitoring** : Zabbix, Grafana

**Protocoles** : OSPF, BGP, VXLAN/EVPN, 802.1X, RADIUS, BFD, IP SLA

---

## 📚 Contexte d'apprentissage

Ce projet fait partie de ma transition professionnelle vers la cybersécurité après 20 ans comme technicien diagnostic électronique dans l'automobile. Il synthétise les cours d'architecture réseau de ma formation Jedha :

- WAN Failover & Redondance
- SD-WAN
- Automatisation réseau avec Ansible
- NAC (Network Access Control)

**Autres projets** :
- 🔒 [VPN WireGuard sur AWS avec Terraform](https://github.com/matthieu-rgb)
- 🐳 [Déploiement Flask avec Docker, Nginx et HTTPS](https://github.com/matthieu-rgb)
- 🎯 Write-ups HTB & TryHackMe (à venir)

---

## 📬 Contact

- **Medium** : [@broquard.matthieu](https://medium.com/@broquard.matthieu/)
- **GitHub** : [matthieu-rgb](https://github.com/matthieu-rgb)
- **LinkedIn** : [Matthieu Broquard](https://linkedin.com/)

---

*Projet réalisé dans le cadre de la formation Cybersécurité — Jedha Academy, 2026*
