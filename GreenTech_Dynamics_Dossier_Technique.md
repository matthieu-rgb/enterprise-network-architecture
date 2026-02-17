# GreenTech Dynamics Inc. — Dossier Technique Architecture Réseau

> **Projet** : Siège Social Lyon 
> **Version** : 1.0
> **Classification** :Exercice pour ma formation jedha 
> **Date** : Février 2026

---

## Table des matières

1. [Synthèse exécutive](#1-synthèse-exécutive)
2. [Architecture réseau globale](#2-architecture-réseau-globale)
3. [Design campus — Modèle 3 tiers](#3-design-campus--modèle-3-tiers)
4. [Design datacenter — Leaf-Spine](#4-design-datacenter--leaf-spine)
5. [Plan de segmentation VLAN](#5-plan-de-segmentation-vlan)
6. [Plan d'adressage IP](#6-plan-dadressage-ip)
7. [Architecture WAN et connectivité](#7-architecture-wan-et-connectivité)
8. [Architecture de sécurité](#8-architecture-de-sécurité)
9. [NAC et contrôle d'accès](#9-nac-et-contrôle-daccès)
10. [Automatisation et gestion](#10-automatisation-et-gestion)
11. [Monitoring et observabilité](#11-monitoring-et-observabilité)
12. [Matrice de flux](#12-matrice-de-flux)
13. [Liste des équipements](#13-liste-des-équipements)
14. [Plan de haute disponibilité et DR](#14-plan-de-haute-disponibilité-et-dr)
15. [Budget prévisionnel](#15-budget-prévisionnel)
16. [Planning de déploiement](#16-planning-de-déploiement)

---

## 1. Synthèse exécutive

### 1.1 Contexte

GreenTech Dynamics Inc. déploie un nouveau siège social à Lyon accueillant 400 employés, un centre R&D européen, et des équipes commerciales EMEA. L'architecture réseau doit répondre à des exigences strictes de performance (99.9% SLA), de sécurité (ISO 27001, GDPR), d'innovation et de scalabilité.

### 1.2 Décisions architecturales clés

| Domaine | Choix | Justification |
|---------|-------|---------------|
| Campus | Core-Distribution-Access (3 tiers) | Trafic nord-sud dominant, 6 étages hiérarchiques |
| Datacenter | Leaf-Spine avec ECMP + VXLAN | Trafic est-ouest Docker/K8s, scalabilité horizontale |
| WAN | SD-WAN hybride (MPLS + fibre + 4G) | Utilisation intelligente des 3 liaisons, local breakout SaaS |
| Sécurité | Défense en profondeur (7 couches) | ISO 27001, données R&D sensibles, 200+ IoT |
| NAC | Cisco ISE (802.1X + MAB + portail captif) | Partenariat Cisco, PKI existante, profiling IoT |
| Automatisation | GitOps (NetBox → Ansible → Oxidized → Git + CI/CD) | 400+ équipements, ISO 27001 auditabilité |
| DR | Réplication sync/async + basculement automatique | SLA 99.9% exige < 9h indisponibilité/an |
| Évolution | ZTNA + microsegmentation | Zero Trust pour télétravailleurs et datacenter |

### 1.3 Schéma d'architecture globale

```
                              ┌─────────────────────────┐
                              │       INTERNET           │
                              └──────────┬──────────────┘
                                ╱        │         ╲
                    ┌──────────┐  ┌──────┴──────┐  ┌──────────┐
                    │ Orange   │  │ Bouygues    │  │ SFR      │
                    │ MPLS 10G │  │ Fibre 1G    │  │ 4G/5G    │
                    └────┬─────┘  └──────┬──────┘  └────┬─────┘
                         │               │              │
                    ┌────┴───────────────┴──────────────┴────┐
                    │          SD-WAN / Firewalls             │
                    │     FortiGate HA (Actif/Passif)         │
                    │        + CPE SD-WAN intégré             │
                    └──────────────────┬─────────────────────┘
                                       │
                    ┌──────────────────┴─────────────────────┐
                    │             CORE LAYER                  │
                    │      2× Switches Core L3 (stack)        │
                    │          10/40 Gbps                     │
                    └───┬─────┬─────┬─────┬─────┬──────┬────┘
                        │     │     │     │     │      │
                     ┌──┴──┐┌─┴──┐┌─┴──┐┌─┴──┐┌─┴──┐┌─┴───┐
                     │Dist ││Dist││Dist││Dist││Dist││Dist  │
                     │RDC  ││1er ││2ème││3ème││4ème││5ème  │
                     └──┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└──┬──┘
                        │     │     │     │     │      │
                     Access Access Access Access Access Access
                     Switches par étage (PoE+ pour WiFi/VoIP/IoT)
                        │
                    ┌───┴────────────────────────────────────┐
                    │         DATACENTER (Sous-sol)           │
                    │                                        │
                    │     Spine 1 ════════════ Spine 2       │
                    │      ╱  ╲      ECMP      ╱  ╲         │
                    │   Leaf1 Leaf2          Leaf3 Leaf4     │
                    │     │     │              │     │       │
                    │   Serveurs Docker/K8s  Apps  Stockage  │
                    └────────────────────────────────────────┘
```

---

## 2. Architecture réseau globale

### 2.1 Principes de design

L'architecture repose sur six principes fondamentaux qui guident chaque décision technique.

**Défense en profondeur** : Chaque couche de l'architecture intègre ses propres contrôles de sécurité. Si une couche est compromise, les suivantes protègent encore l'infrastructure.

**Séparation des plans** : Le campus (trafic utilisateur nord-sud) et le datacenter (trafic applicatif est-ouest) utilisent des topologies différentes, chacune optimisée pour son type de trafic.

**Redondance N+1** : Chaque composant critique existe en double. Aucun point unique de défaillance (SPOF) ne peut provoquer une indisponibilité.

**Automatisation systématique** : Toute configuration est définie comme du code, versionnée dans Git, et déployée automatiquement. Aucune modification manuelle en production.

**Segmentation Zero Trust** : Chaque zone réseau est isolée. L'accès entre zones est explicitement autorisé, jamais implicitement permis.

**Scalabilité horizontale** : L'architecture peut absorber la croissance (nouveaux étages, nouveaux sites, nouveaux services) sans refonte.

### 2.2 Deux topologies complémentaires

```
CAMPUS (Étages RDC à 5ème) :         DATACENTER (Sous-sol) :
Core-Distribution-Access              Leaf-Spine

       Core                                Spine
      ╱    ╲                              ╱    ╲
   Dist    Dist                        Leaf    Leaf
   ╱ ╲    ╱ ╲                          │  │    │  │
  Acc Acc Acc Acc                      Srv Srv Srv Srv

Trafic : Nord-Sud                     Trafic : Est-Ouest
(User → Server/Cloud)                 (Server ↔ Server)
Redondance : STP/RSTP                 Redondance : ECMP
Max sauts : 5                         Max sauts : 2
Overlay : VLANs classiques            Overlay : VXLAN
```

---

## 3. Design campus — Modèle 3 tiers

### 3.1 Couche Core

Le cœur de réseau assure le routage inter-VLAN haute performance et la connectivité vers le datacenter, le WAN et les services cloud.

**Équipement** : 2× switches Core L3 en stack/VSS (haute disponibilité active/active)

**Caractéristiques** :
- Ports 40 Gbps vers chaque switch de distribution
- Ports 10 Gbps vers les Spine switches du datacenter
- Ports 10 Gbps vers les firewalls FortiGate
- Routage OSPF en interne, BGP vers le WAN
- HSRP/VRRP pour la redondance de passerelle (ou routing L3 jusqu'à la distribution)

```
            ┌────────────┐     ┌────────────┐
            │  Core SW1  │═════│  Core SW2  │    Stack/VSS
            │  (Actif)   │     │  (Actif)   │    Active/Active
            └──┬──┬──┬───┘     └──┬──┬──┬───┘
               │  │  │            │  │  │
          40G  │  │  │ 10G   40G │  │  │ 10G
               │  │  │            │  │  │
    Vers       │  │  └─── Vers ───┘  │  └─── Vers
    Distribution  │      Datacenter   │       Firewalls
                  │      (Spines)     │       (FortiGate HA)
                  │                   │
            Vers Distribution    Vers Distribution
```

### 3.2 Couche Distribution

Un switch de distribution par étage (ou par groupe de 2 étages selon la densité). Assure l'agrégation du trafic des switches d'accès et applique les politiques de routage inter-VLAN.

**Équipement** : 6× switches Distribution L3 (un par étage)

**Caractéristiques** :
- Uplinks 40 Gbps (2×) vers les Core switches (redondance)
- Downlinks 10 Gbps vers les switches d'accès
- Routage inter-VLAN pour les VLANs locaux à l'étage
- ACL inter-zones appliquées à ce niveau
- Spanning Tree Root Bridge pour le domaine L2 de l'étage

### 3.3 Couche Access

Les switches d'accès connectent directement les utilisateurs, les téléphones IP, les bornes WiFi, les caméras et les équipements IoT.

**Équipement** : 24-48× switches Access L2+ (PoE+) selon la densité par étage

**Caractéristiques** :
- Ports 1 Gbps PoE+ (802.3af/at) pour postes, téléphones, WiFi AP, caméras
- Uplinks 10 Gbps (2×) vers le switch Distribution de l'étage
- 802.1X activé sur tous les ports (NAC enforcement)
- Port-security et DHCP snooping pour la sécurité L2
- Storm control et BPDU guard
- QoS marquage au plus près de la source (DSCP)

```
Exemple étage 3 — R&D Hardware (80 employés) :

    ┌──────────────────────────────────────────┐
    │           Distribution Étage 3            │
    │         (Switch L3 — 10G uplinks)         │
    └──┬──────────┬──────────┬────────────┬────┘
       │ 10G      │ 10G      │ 10G        │ 10G
    ┌──┴──┐    ┌──┴──┐    ┌──┴──┐     ┌───┴──┐
    │Acc 1│    │Acc 2│    │Acc 3│     │Acc 4 │
    │48p  │    │48p  │    │48p  │     │48p   │
    │PoE+ │    │PoE+ │    │PoE+ │     │PoE+  │
    └──┬──┘    └──┬──┘    └──┬──┘     └──┬───┘
       │          │          │           │
    Postes     Postes     WiFi AP     Équip. IoT
    R&D        R&D        + Caméras   Labo
    VLAN 30    VLAN 30    VLAN 70/60  VLAN 50
```

### 3.4 WiFi

**Équipement** : Contrôleur WiFi centralisé + bornes WiFi 6E (WiFi 802.11ax)

**SSIDs** :
- `GTD-Corp` : WPA3-Enterprise, 802.1X, VLAN dynamique par rôle
- `GTD-Guest` : WPA3, portail captif, VLAN Invités (Internet seul)
- `GTD-IoT` : WPA3-Personal, VLAN IoT dédié (caché)

---

## 4. Design datacenter — Leaf-Spine

### 4.1 Topologie

Le datacenter au sous-sol utilise une topologie Leaf-Spine pour optimiser le trafic est-ouest entre les serveurs Docker/Kubernetes, MongoDB, et les applications internes.

```
                 ┌──────────┐         ┌──────────┐
                 │ Spine 1  │═════════│ Spine 2  │
                 │  (BGP)   │  ECMP   │  (BGP)   │
                 └─┬──┬──┬──┘         └─┬──┬──┬──┘
                   │  │  │              │  │  │
              ┌────┘  │  └────┐    ┌───┘  │  └────┐
              │       │       │    │      │       │
           ┌──┴──┐ ┌──┴──┐ ┌─┴────┴┐  ┌──┴──┐ (extensible)
           │Leaf1│ │Leaf2│ │Leaf 3 │  │Leaf4│
           │     │ │     │ │       │  │     │
           └──┬──┘ └──┬──┘ └───┬───┘  └──┬──┘
              │       │        │         │
         Serveurs  Docker/  MongoDB    Stockage
         Physiques Kubernetes Atlas     SAN/NAS
         (Compute) (Compute)  (Data)   (Storage)

  Protocole : eBGP entre Leaf et Spine (RFC 7938)
  Overlay   : VXLAN/EVPN pour la mobilité des VMs/conteneurs
  ECMP      : Tous les chemins Leaf→Spine actifs simultanément
  Latence   : Max 2 sauts entre n'importe quels serveurs
```

### 4.2 VXLAN/EVPN

VXLAN (Virtual Extensible LAN) encapsule les trames L2 dans UDP, permettant de créer des segments logiques au-dessus du fabric IP Leaf-Spine. EVPN (Ethernet VPN) gère le plan de contrôle pour la distribution des adresses MAC et IP.

**Avantages pour GreenTech** : Les conteneurs Kubernetes peuvent migrer entre n'importe quels serveurs (sur différents Leaf switches) sans reconfiguration réseau. La limite de 4096 VLANs classiques est dépassée (VXLAN supporte 16 millions de VNIs). La segmentation datacenter est indépendante de la segmentation campus.

### 4.3 Connexion Campus ↔ Datacenter

Les Spine switches du datacenter se connectent aux Core switches du campus via des liens 10 Gbps redondants. Le routage entre les deux domaines est assuré par OSPF (même aire) ou BGP si une isolation plus forte est souhaitée.

```
  Campus Core SW1 ──── 10G ──── Spine 1 (DC)
          │     ╲              ╱    │
          │      ╲ 10G   10G ╱     │
          │       ╲        ╱       │
  Campus Core SW2 ──── 10G ──── Spine 2 (DC)

  4 liens physiques, ECMP actif = résilience totale
  Perte d'un lien ou d'un switch = aucun impact
```

---

## 5. Plan de segmentation VLAN

### 5.1 VLANs Campus

| VLAN ID | Nom | Département / Fonction | Niveau sécurité | Étage(s) |
|---------|-----|----------------------|-----------------|----------|
| 10 | VLAN_ACCUEIL | Accueil / Showroom | Standard | RDC |
| 20 | VLAN_COMMERCIAL | Commercial EMEA | Élevé | 1er |
| 30 | VLAN_FINANCE | Finance & Comptabilité | Très Élevé | 2ème |
| 40 | VLAN_RD_HW | R&D Hardware | Très Élevé | 3ème |
| 41 | VLAN_RD_SW | R&D Software | Très Élevé | 4ème |
| 50 | VLAN_IOT | Équipements IoT (capteurs, smart building) | Élevé (isolé) | Tous |
| 60 | VLAN_CAMERAS | Caméras IP | Élevé (isolé) | Tous |
| 70 | VLAN_VOIP | Téléphones IP | Élevé | Tous |
| 80 | VLAN_DIRECTION | Direction & RH | Très Élevé | 5ème |
| 90 | VLAN_IT_MGMT | Management réseau / IT | Très Élevé | Tous |
| 100 | VLAN_GUEST | Invités / WiFi Guest | Standard (isolé) | Tous |
| 110 | VLAN_QUARANTINE | Quarantaine NAC | Restreint | Tous |
| 999 | VLAN_NATIVE | Native VLAN (non utilisé) | — | Tous |

### 5.2 VLANs Datacenter (VXLAN VNIs)

| VNI | Nom | Fonction |
|-----|-----|----------|
| 5010 | VNI_COMPUTE | Serveurs de calcul / Docker / K8s |
| 5020 | VNI_DATABASE | MongoDB Atlas, bases de données |
| 5030 | VNI_STORAGE | Stockage SAN/NAS |
| 5040 | VNI_DMZ | Services exposés (web, API publiques) |
| 5050 | VNI_MGMT_DC | Management datacenter (iLO, iDRAC, IPMI) |

### 5.3 Principes de segmentation

Le VLAN IoT (50) et le VLAN Caméras (60) n'ont **aucun accès à Internet direct** et ne peuvent communiquer qu'avec leurs serveurs de gestion dédiés dans le datacenter. Le VLAN Invités (100) a accès à Internet uniquement, via un pare-feu dédié, sans aucune visibilité sur le réseau interne. Le VLAN Quarantaine (110) n'a accès qu'aux serveurs de remédiation (patchs, antivirus). Chaque VLAN métier est séparé par un firewall inter-zones (FortiGate) avec des règles de flux explicites.

---

## 6. Plan d'adressage IP

### 6.1 Supernet global

GreenTech utilise le bloc privé **10.0.0.0/8** segmenté par site et par fonction.

```
Allocation globale :

  10.1.0.0/16   → Lyon (Siège)
  10.2.0.0/16   → Paris (Bureau satellite)
  10.3.0.0/16   → San Francisco (Siège US)
  10.10.0.0/16  → Datacenter Lyon (Production)
  10.11.0.0/16  → Datacenter Lyon (DR)
  10.100.0.0/16 → Infrastructure réseau (management, loopbacks)
  
  172.16.0.0/16 → VPN télétravailleurs
  192.168.0.0/16 → Réservé labs R&D (environnements isolés)
```

### 6.2 Adressage détaillé Lyon (10.1.0.0/16)

| Sous-réseau | VLAN | Passerelle | Plage DHCP | Capacité |
|-------------|------|-----------|------------|----------|
| 10.1.10.0/24 | 10 — Accueil | 10.1.10.1 | .50 à .200 | 150 hôtes |
| 10.1.20.0/24 | 20 — Commercial | 10.1.20.1 | .50 à .200 | 150 hôtes |
| 10.1.30.0/24 | 30 — Finance | 10.1.30.1 | .50 à .100 | 50 hôtes |
| 10.1.40.0/24 | 40 — R&D HW | 10.1.40.1 | .50 à .200 | 150 hôtes |
| 10.1.41.0/24 | 41 — R&D SW | 10.1.41.1 | .50 à .200 | 150 hôtes |
| 10.1.50.0/23 | 50 — IoT | 10.1.50.1 | .10 à .254 + .1.254 | 500 hôtes |
| 10.1.60.0/24 | 60 — Caméras | 10.1.60.1 | .10 à .100 | 90 hôtes |
| 10.1.70.0/23 | 70 — VoIP | 10.1.70.1 | .10 à .254 + .1.254 | 500 hôtes |
| 10.1.80.0/24 | 80 — Direction | 10.1.80.1 | .50 à .100 | 50 hôtes |
| 10.1.90.0/24 | 90 — IT Mgmt | 10.1.90.1 | .50 à .100 | 50 hôtes |
| 10.1.100.0/24 | 100 — Guest | 10.1.100.1 | .50 à .250 | 200 hôtes |
| 10.1.110.0/24 | 110 — Quarantine | 10.1.110.1 | .50 à .200 | 150 hôtes |

### 6.3 Adressage Datacenter (10.10.0.0/16)

| Sous-réseau | VNI | Fonction |
|-------------|-----|----------|
| 10.10.10.0/24 | 5010 | Compute (Serveurs physiques) |
| 10.10.11.0/24 | 5010 | Compute (Kubernetes Pods — CIDR K8s) |
| 10.10.20.0/24 | 5020 | Databases |
| 10.10.30.0/24 | 5030 | Stockage |
| 10.10.40.0/24 | 5040 | DMZ |
| 10.10.90.0/24 | 5050 | Management DC (iLO/iDRAC/IPMI) |

### 6.4 Adressage infrastructure réseau (10.100.0.0/16)

| Sous-réseau | Fonction |
|-------------|----------|
| 10.100.0.0/24 | Loopbacks switches Core |
| 10.100.1.0/24 | Loopbacks switches Distribution |
| 10.100.2.0/24 | Loopbacks switches Access |
| 10.100.10.0/24 | Loopbacks Spine/Leaf DC |
| 10.100.100.0/24 | Point-à-point liens inter-switches (/31) |
| 10.100.200.0/24 | Management band (Ansible, Oxidized, Zabbix) |

---

## 7. Architecture WAN et connectivité

### 7.1 SD-WAN hybride

L'architecture WAN utilise un overlay SD-WAN sur les trois liaisons, avec le MPLS conservé pour les flux ultra-critiques bénéficiant des SLA opérateur.

```
                           Internet
                          ╱    │    ╲
                   ┌──────┐ ┌──┴───┐ ┌───────┐
                   │Orange│ │Bouygues│ │ SFR   │
                   │MPLS  │ │Fibre  │ │ 4G/5G │
                   │10 Gbps│ │1 Gbps│ │300Mbps│
                   └──┬───┘ └──┬───┘ └──┬────┘
                      │        │        │
                 ┌────┴────────┴────────┴─────┐
                 │    FortiGate HA Cluster     │
                 │  + SD-WAN intégré (FortiOS) │
                 │                             │
                 │  Politiques applicatives :   │
                 │  • VoIP/Vidéo → MPLS        │
                 │  • O365/Salesforce → Local   │
                 │    breakout fibre            │
                 │  • Backup/non-critique → 4G  │
                 │  • Basculement automatique   │
                 └─────────────┬───────────────┘
                               │
                          Core Campus
```

### 7.2 Politiques de routage SD-WAN

| Application | Liaison préférée | SLA requis | Fallback |
|-------------|-----------------|-----------|----------|
| VoIP / Visioconférence | MPLS Orange | Latence < 100ms, Perte < 1%, Gigue < 30ms | Fibre → 4G |
| ERP / Applications internes | MPLS Orange | Latence < 50ms, Perte < 0.1% | Fibre → 4G |
| Office 365 / Salesforce | Local breakout Fibre | Latence < 200ms, Perte < 3% | MPLS → 4G |
| Trafic web général | Fibre Bouygues | Best effort | MPLS → 4G |
| Backup / Réplication DR | Fibre (heures creuses) / 4G | Pas de SLA strict | N'importe quel chemin |
| IoT Cloud (AWS) | AWS Direct Connect via Fibre | Latence < 50ms | VPN IPsec via MPLS |

### 7.3 Connectivité sites distants

```
Lyon (Siège) ─────────────────────────────────────────┐
  │                                                    │
  ├── MPLS Orange ──────────── Paris (Bureau 80 pers.) │
  │   1 Gbps, BGP + OSPF      + CPE SD-WAN            │
  │                            + Local breakout SaaS   │
  │                                                    │
  ├── VPN IPsec via Internet ── San Francisco          │
  │   500 Mbps, BGP            + CPE SD-WAN            │
  │                            (ZTP déployable)        │
  │                                                    │
  ├── AWS Direct Connect ────── AWS eu-west-3 (Paris)  │
  │   1 Gbps, BGP              Données IoT + IaaS     │
  │                                                    │
  └── VPN SSL (OpenVPN/ZTNA)── Télétravailleurs        │
      Variable                  ZTNA via FortiClient    │
      Accès par application     (remplace VPN classique)│
```

### 7.4 Détection de pannes et basculement

**BFD** (Bidirectional Forwarding Detection) est activé sur toutes les sessions BGP et OSPF pour une détection de panne en moins de 50 ms au niveau du lien direct.

**IP SLA** surveille la qualité de bout en bout avec des sondes adaptées à chaque type de trafic : ICMP echo vers les passerelles FAI, HTTP GET vers les portails SaaS critiques (O365, Salesforce), UDP Jitter vers les serveurs VoIP pour mesurer la qualité voix.

**SD-WAN** assure la sélection dynamique de chemin en temps réel, basculant les flux applicatifs en sub-seconde en cas de dégradation (pas seulement en cas de panne).

---

## 8. Architecture de sécurité

### 8.1 Défense en profondeur — Les 7 couches

```
┌─────────────────────────────────────────────────────────┐
│ Couche 7 : Chiffrement                                  │
│ AES-256 pour R&D/Finance, IPsec pour tous les tunnels   │
├─────────────────────────────────────────────────────────┤
│ Couche 6 : SIEM (Splunk)                                │
│ Corrélation d'événements, détection d'anomalies, 24/7   │
├─────────────────────────────────────────────────────────┤
│ Couche 5 : Endpoint Protection (FortiEDR / CrowdStrike) │
│ Protection de chaque poste, détection comportementale    │
├─────────────────────────────────────────────────────────┤
│ Couche 4 : IDS/IPS (FortiGate intégré)                  │
│ Détection et blocage des menaces en temps réel           │
├─────────────────────────────────────────────────────────┤
│ Couche 3 : Firewalls inter-zones (FortiGate HA)         │
│ Filtrage des flux entre chaque zone de sécurité          │
├─────────────────────────────────────────────────────────┤
│ Couche 2 : Segmentation VLAN / VXLAN / Microsegmentation│
│ Isolation logique de chaque département et workload      │
├─────────────────────────────────────────────────────────┤
│ Couche 1 : NAC (Cisco ISE) au point d'entrée            │
│ 802.1X + MAB + Portail captif + Posture assessment       │
└─────────────────────────────────────────────────────────┘
```

### 8.2 Zones de sécurité et firewalling

Le FortiGate en cluster HA (actif/passif) segmente le réseau en zones de sécurité avec des règles de flux inter-zones explicites :

```
                    ┌────────────────────┐
                    │   FortiGate HA     │
                    │   (Policy Engine)  │
                    └──┬──┬──┬──┬──┬──┬─┘
                       │  │  │  │  │  │
               ┌───────┘  │  │  │  │  └────────┐
               ▼          ▼  ▼  ▼  ▼           ▼
           ┌───────┐  ┌────┐┌──┐┌───┐┌──────┐┌──────┐
           │  WAN  │  │R&D ││FI││COM││  IoT ││ DMZ  │
           │(Untrust)│ │    ││NA││MER││      ││      │
           └───────┘  │    ││NCE││CIAL│      ││      │
                      │TRÈS││TRÈS││ÉLE││ISOLÉ ││EXPOSÉ│
                      │ÉLEVÉ││ÉLEVÉ││VÉ ││     ││      │
                      └────┘└──┘└───┘└──────┘└──────┘

  Chaque flèche entre zones = règle de flux explicite
  Default deny : tout ce qui n'est pas explicitement autorisé est bloqué
```

### 8.3 Zone DMZ

La DMZ héberge les services accessibles depuis Internet : serveurs web publics, API publiques, serveur mail entrant, reverse proxy. La DMZ est doublement isolée (firewall côté WAN + firewall côté LAN).

### 8.4 Authentification et identité

L'infrastructure d'identité repose sur **Active Directory** avec **Azure AD Connect** pour la synchronisation cloud. Tous les accès utilisent l'authentification multi-facteur (MFA). Le Single Sign-On (SSO) est assuré via SAML/OIDC pour les applications SaaS (O365, Salesforce, Atlassian). Les certificats PKI internes sont délivrés par une autorité de certification (CA) interne pour 802.1X EAP-TLS.

### 8.5 Conformité

**ISO 27001** : Piste d'audit complète via Git (Ansible) + Oxidized. Logs centralisés 12 mois dans Splunk. Politique de gestion des accès documentée et automatisée via NAC.

**GDPR** : Chiffrement des données en transit (IPsec, TLS) et au repos (AES-256). Segmentation réseau empêchant l'accès non autorisé aux données personnelles. Logs d'accès pour le droit d'audit.

---

## 9. NAC et contrôle d'accès

### 9.1 Architecture Cisco ISE

```
  ┌──────────────────────────────────────────────────┐
  │                   Cisco ISE                       │
  │              (Cluster HA — 2 nœuds)               │
  │                                                   │
  │  ┌─────────────┐  ┌────────────┐  ┌───────────┐  │
  │  │ Policy Node │  │ Monitoring │  │  pxGrid   │  │
  │  │ (RADIUS)    │  │ Node       │  │ (partage  │  │
  │  │             │  │            │  │ contexte  │  │
  │  │ 802.1X      │  │ Logs       │  │ avec FW,  │  │
  │  │ MAB         │  │ Dashboards │  │ SIEM)     │  │
  │  │ Guest       │  │ Alertes    │  │           │  │
  │  │ Posture     │  │            │  │           │  │
  │  └─────────────┘  └────────────┘  └───────────┘  │
  └──────────────────────────────────────────────────┘
           │                              │
           ▼                              ▼
  Switches d'accès (802.1X)       FortiGate (pxGrid)
  Bornes WiFi (802.1X)            Splunk (logs ISE)
  Tous les ports campus           FortiEDR (posture)
```

### 9.2 Politiques d'accès

| Profil | Authentification | Posture | VLAN assigné | ACL |
|--------|-----------------|---------|-------------|-----|
| Employé (laptop géré) | 802.1X EAP-TLS (certificat) | AV + patchs + firewall + chiffrement | VLAN département | Accès complet département |
| Employé (BYOD) | 802.1X PEAP (login/mdp + MFA) | Basique (AV + OS à jour) | VLAN limité (Internet + O365) | Pas d'accès réseau interne |
| Prestataire | Portail captif (sponsorisé) | Aucune | VLAN Prestataire (100) | Accès projet spécifique |
| Visiteur | Portail captif (CGU) | Aucune | VLAN Guest (100) | Internet seul |
| Téléphone IP | MAB + profiling CDP/LLDP | N/A | VLAN VoIP (70) | SIP/RTP + management |
| Imprimante | MAB + profiling | N/A | VLAN département | Ports impression seuls |
| Caméra IP | MAB + profiling | N/A | VLAN Caméras (60) | Flux vidéo vers NVR seul |
| Capteur IoT | MAB + profiling | N/A | VLAN IoT (50) | API cloud + management |
| Non conforme | 802.1X (auth OK, posture KO) | Échec | VLAN Quarantaine (110) | Serveurs patchs seuls |
| Inconnu | Échec auth | N/A | Bloqué ou VLAN Guest | Selon politique |

### 9.3 Déploiement progressif NAC

**Phase 1 (Semaines 1-2)** : Mode monitor. 802.1X activé mais en open mode. ISE authentifie et logue sans bloquer. Objectif : inventorier tous les appareils.

**Phase 2 (Semaines 3-4)** : Low-impact. VLAN Guest pour les appareils non reconnus. Portail captif invités. MAB pour les IoT identifiés.

**Phase 3 (Semaines 5-8)** : Full enforcement. 802.1X obligatoire sur tous les ports. Posture assessment activé. VLAN Quarantaine opérationnel.

---

## 10. Automatisation et gestion

### 10.1 Écosystème GitOps

```
┌────────────┐     ┌──────────┐     ┌───────────┐     ┌──────────────┐
│   NetBox   │────▶│   Git    │────▶│ GitLab CI │────▶│   Ansible    │
│   (SSoT)   │     │(GitLab)  │     │ Pipeline  │     │              │
│            │     │          │     │           │     │ Déploiement  │
│ Inventaire │     │ Playbooks│     │ Lint      │     │ sur 400+     │
│ IPAM       │     │ Templates│     │ Check     │     │ équipements  │
│ Topologie  │     │ Variables│     │ Approve   │     │              │
└────────────┘     └──────────┘     └───────────┘     └──────┬───────┘
      ▲                                                       │
      │                                                       ▼
      │            ┌──────────┐                     ┌──────────────────┐
      └────────────│ Oxidized │◀────────────────────│  Équipements     │
   Feedback        │          │    SSH (collecte)   │  réseau          │
   (écart détecté) │ Backup   │                     │  (switches, FW,  │
                   │ Git      │                     │   bornes, CPE)   │
                   │ Diff     │                     └──────────────────┘
                   │ Alertes  │                              │
                   └──────────┘                              ▼
                                                    ┌──────────────────┐
                                                    │     Zabbix       │
                                                    │   Monitoring     │
                                                    │   SNMP + API     │
                                                    └──────────────────┘
```

### 10.2 Rôles Ansible

| Rôle | Fonction | Cibles |
|------|----------|--------|
| `base_config` | Hostname, NTP, DNS, logging, SNMP, bannière | Tous les équipements |
| `security_hardening` | SSH, ACL management, port-security, DHCP snooping | Switches d'accès |
| `vlan_management` | Création et assignation des VLANs | Switches d'accès et distribution |
| `nac_802.1x` | Configuration 802.1X, MAB, RADIUS | Switches d'accès |
| `qos_policy` | Marquage DSCP, queuing VoIP | Tous les switches |
| `wan_config` | BGP, SD-WAN policies, IP SLA, BFD | Routeurs WAN, FortiGate |
| `monitoring_agent` | SNMP community, Zabbix agent, syslog | Tous les équipements |
| `wifi_config` | SSIDs, WPA3, RADIUS, policies | Contrôleur WiFi, bornes |
| `dc_fabric` | BGP Leaf-Spine, VXLAN/EVPN | Spine et Leaf switches |
| `backup_restore` | Sauvegarde running-config, rollback | Tous les équipements |

### 10.3 Stratégie de sauvegarde des configurations

**Oxidized** collecte les configurations toutes les heures et les stocke dans un dépôt Git dédié. Chaque changement déclenche un commit automatique avec diff. Les alertes de changement non planifié sont envoyées vers Slack et le SIEM.

**Ansible** peut restaurer n'importe quelle version de configuration via `git checkout` + playbook de déploiement. Le rollback est testable en mode `--check --diff` avant exécution.

**Rétention** : 12 mois d'historique dans Git (conforme ISO 27001).

---

## 11. Monitoring et observabilité

### 11.1 Stack de monitoring

| Outil | Fonction | Cibles |
|-------|----------|--------|
| **Zabbix** | Monitoring infrastructure (SNMP, ICMP, agents) | Tous les équipements réseau, serveurs |
| **Splunk** | SIEM — corrélation d'événements sécurité, logs centralisés | Firewalls, NAC, switches, serveurs, endpoints |
| **FortiAnalyzer** | Analyse des logs Fortinet, reporting sécurité | FortiGate, FortiSwitch, FortiAP |
| **NetBox** | Source de vérité inventaire | Référence pour tous les outils |
| **Grafana** | Dashboards de visualisation | Données Zabbix, métriques custom |

### 11.2 Alertes critiques

| Événement | Sévérité | Notification | Action |
|-----------|---------|-------------|--------|
| Lien WAN DOWN | Critique | Slack + SMS + SIEM | SD-WAN bascule automatique |
| Switch DOWN | Critique | Slack + SMS + ticket auto | Investigation immédiate |
| NAC : appareil inconnu sur port critique | Haute | SIEM + email sécurité | Isolement + investigation |
| Dérive de configuration détectée | Haute | Slack + SIEM | Ansible correction auto ou manuelle |
| Utilisation bande passante > 80% | Moyenne | Slack + Zabbix | Planification capacité |
| Certificat PKI expire dans 30 jours | Moyenne | Email IT + ticket | Renouvellement |
| Posture : échec massif (> 10 appareils) | Haute | SIEM + email sécurité | Vérification serveur patchs |

---

## 12. Matrice de flux

### 12.1 Flux inter-zones autorisés

| Source | Destination | Ports/Protocoles | Action | Justification |
|--------|------------|-----------------|--------|---------------|
| VLAN_COMMERCIAL | Internet | 443, 80 | ALLOW | Accès SaaS (O365, Salesforce) |
| VLAN_COMMERCIAL | VNI_DMZ | 443 | ALLOW | API/Web internes |
| VLAN_COMMERCIAL | VLAN_VOIP | SIP, RTP | ALLOW | Téléphonie |
| VLAN_FINANCE | VNI_DATABASE | 27017 (MongoDB), 1433 | ALLOW | Accès bases financières |
| VLAN_FINANCE | Internet | 443 | ALLOW (restreint) | Banque en ligne, reporting |
| VLAN_RD_HW | VNI_COMPUTE | Tous | ALLOW | Accès environnements dev |
| VLAN_RD_SW | VNI_COMPUTE | Tous | ALLOW | Accès Docker/K8s, CI/CD |
| VLAN_RD_* | Internet | 443, 22 | ALLOW | GitHub, registres Docker, cloud |
| VLAN_IOT | VNI_COMPUTE | 8883 (MQTTS), 443 | ALLOW | Publication données capteurs |
| VLAN_IOT | Internet | DENY | DENY | IoT isolé du web |
| VLAN_CAMERAS | VNI_STORAGE | RTSP (554) | ALLOW | Flux vidéo vers NVR |
| VLAN_CAMERAS | Internet | DENY | DENY | Caméras isolées du web |
| VLAN_GUEST | Internet | 443, 80, 53 | ALLOW | Navigation web seule |
| VLAN_GUEST | Tous VLANs internes | Tous | DENY | Isolation totale invités |
| VLAN_QUARANTINE | Serveur patchs | 80, 443 | ALLOW | Remédiation |
| VLAN_QUARANTINE | Tout le reste | Tous | DENY | Isolation quarantaine |
| VLAN_IT_MGMT | Tous | SSH, HTTPS, SNMP | ALLOW | Administration réseau |
| VNI_DMZ | Internet | 443, 80, 25 | ALLOW | Services publics |
| VNI_DMZ | VLANs internes | DENY | DENY | DMZ ne peut pas initier vers l'interne |

### 12.2 Flux WAN

| Source | Destination | Liaison | Protocole |
|--------|------------|---------|-----------|
| Lyon | Paris | MPLS Orange | BGP + OSPF, tunnel SD-WAN |
| Lyon | San Francisco | Fibre Bouygues (Internet) | VPN IPsec + BGP, tunnel SD-WAN |
| Lyon | AWS eu-west-3 | AWS Direct Connect via Fibre | BGP |
| Lyon | O365 / Salesforce | Local breakout Fibre | HTTPS direct |
| Paris | O365 / Salesforce | Local breakout Internet | HTTPS direct |
| Télétravailleurs | Lyon | Internet (variable) | ZTNA (FortiClient) |

---

## 13. Liste des équipements

### 13.1 Équipements réseau Campus

| Rôle | Modèle suggéré | Quantité | Justification |
|------|---------------|----------|---------------|
| Core Switch L3 | Cisco Catalyst 9500 (ou FortiSwitch 4xx) | 2 | HA active/active, 40G, OSPF/BGP |
| Distribution Switch L3 | Cisco Catalyst 9300 (ou FortiSwitch 2xx) | 6 | Un par étage, 10G uplinks |
| Access Switch L2+ PoE+ | Cisco Catalyst 9200 (ou FortiSwitch 1xx) | 30-40 | 48 ports PoE+, 802.1X |
| Firewall / SD-WAN | FortiGate 600F (HA pair) | 2 | FW inter-zones, SD-WAN, IPS, VPN |
| WiFi Controller | FortiWLC ou Cisco 9800-CL | 1 (+ backup) | Gestion centralisée WiFi |
| WiFi Access Points | FortiAP 431G ou Cisco 9136 (WiFi 6E) | 40-50 | Couverture tous étages |
| NAC | Cisco ISE (VM cluster) | 2 | HA, 802.1X, posture, guest |

### 13.2 Équipements Datacenter

| Rôle | Modèle suggéré | Quantité | Justification |
|------|---------------|----------|---------------|
| Spine Switch | Cisco Nexus 9336 ou Arista 7280 | 2 | BGP, ECMP, VXLAN/EVPN |
| Leaf Switch | Cisco Nexus 93180 ou Arista 7050 | 4 | 10/25G serveurs, VXLAN |
| Serveurs Rack | Dell PowerEdge R760 ou HPE DL380 | 25 | Compute, Docker/K8s |
| Stockage SAN | NetApp AFF A250 ou Dell PowerStore | 2 | Réplication DR |
| UPS | APC Smart-UPS 10kVA | 2 | Alimentation redondante |

### 13.3 Équipements WAN / Sécurité

| Rôle | Modèle suggéré | Quantité | Justification |
|------|---------------|----------|---------------|
| CPE SD-WAN Paris | FortiGate 80F | 1 | SD-WAN + FW intégré |
| CPE SD-WAN San Francisco | FortiGate 80F | 1 | SD-WAN + FW intégré |
| FortiAnalyzer | FortiAnalyzer VM | 1 | Logs Fortinet centralisés |

### 13.4 Outils logiciels

| Outil | Type | Fonction |
|-------|------|----------|
| Cisco ISE | VM (2 nœuds) | NAC |
| NetBox | VM ou conteneur Docker | SSoT, inventaire, IPAM |
| Ansible + AWX | VM | Automatisation, GUI |
| Oxidized | VM ou conteneur Docker | Backup configs |
| GitLab | VM ou SaaS | Git + CI/CD |
| Zabbix | VM | Monitoring infrastructure |
| Splunk | VM ou SaaS | SIEM |
| Grafana | VM ou conteneur | Dashboards |

---

## 14. Plan de haute disponibilité et DR

### 14.1 Redondance par couche

| Composant | Mécanisme de redondance | RTO | RPO |
|-----------|------------------------|-----|-----|
| Core switches | Stack/VSS active/active | 0 (transparent) | N/A |
| Distribution | Dual uplinks vers 2 Core | < 1 sec (RSTP/ECMP) | N/A |
| Access | Dual uplinks vers Distribution | < 1 sec | N/A |
| Firewalls | FortiGate HA actif/passif | < 5 sec | 0 |
| WAN | SD-WAN 3 liaisons + BFD | < 1 sec | N/A |
| Spine/Leaf DC | ECMP multi-chemin | 0 (transparent) | N/A |
| NAC (ISE) | Cluster 2 nœuds + fail-open | < 30 sec | N/A |
| Serveurs | Kubernetes HA (multi-nœud) | < 30 sec | 0 |
| Stockage | Réplication synchrone vers DR | < 5 min | 0 (sync) |
| Alimentation | UPS + groupe électrogène | 0 (transparent) | N/A |

### 14.2 Datacenter DR

Le datacenter de secours (DR) est connecté au datacenter principal via un lien dédié 10 Gbps (DCI — Data Center Interconnect).

**Réplication synchrone** pour les données critiques (bases financières, données R&D) : RPO = 0 (aucune perte de données).

**Réplication asynchrone** pour les données volumineuses (logs, backups, archives) : RPO = 15 minutes.

**Basculement** : Automatique pour les services Kubernetes (orchestrateur détecte la panne et redéploie les pods sur le DR). Semi-automatique pour les services legacy (validation humaine avant basculement).

### 14.3 Calcul de disponibilité

```
Objectif SLA : 99.9% = max 8h46min d'indisponibilité par an

Architecture redondante :
  • 2 Core switches : 99.999% disponibilité combinée
  • 2 Firewalls HA : 99.99%
  • 3 liaisons WAN : 99.99% (probabilité que les 3 tombent simultanément)
  • 2 Spine + 4 Leaf : 99.999%
  • DR avec réplication : RPO 0, RTO < 5 min

Disponibilité globale estimée : > 99.95% ✅ (dépasse l'objectif 99.9%)
```

---

## 15. Budget prévisionnel

### 15.1 CAPEX (Investissement initial)

| Catégorie | Estimation | Notes |
|-----------|-----------|-------|
| Switches Campus (Core + Dist + Access) | 180 000 € — 250 000 € | ~40 switches, PoE+ |
| Switches Datacenter (Spine + Leaf) | 80 000 € — 120 000 € | 6 switches haute performance |
| Firewalls FortiGate HA + FortiAnalyzer | 60 000 € — 90 000 € | Cluster HA + licences UTM |
| WiFi (Contrôleur + 40-50 AP) | 40 000 € — 60 000 € | WiFi 6E, PoE |
| Cisco ISE (licences + VM) | 50 000 € — 80 000 € | Licences Base + Plus + Device Admin |
| CPE SD-WAN (Paris + San Francisco) | 10 000 € — 15 000 € | 2× FortiGate 80F |
| Câblage / Infrastructure passive | 30 000 € — 50 000 € | Cat6A existant, quelques extensions |
| Serveurs + Stockage DC | 200 000 € — 300 000 € | 25 serveurs + SAN |
| UPS + Alimentation | 30 000 € — 50 000 € | Redondance N+1 |
| **Total CAPEX** | **680 000 € — 1 015 000 €** | |

### 15.2 OPEX (Coûts récurrents annuels)

| Catégorie | Estimation/an | Notes |
|-----------|-------------|-------|
| Liaisons WAN (Orange MPLS + Bouygues + SFR) | 60 000 € — 80 000 € | MPLS représente ~70% du coût |
| AWS Direct Connect | 12 000 € — 18 000 € | 1 Gbps dédié |
| Licences logicielles (Splunk, FortiGuard, ISE) | 80 000 € — 120 000 € | Renouvellement annuel |
| Support et maintenance matériel | 40 000 € — 60 000 € | SmartNet / FortiCare |
| Électricité datacenter | 15 000 € — 25 000 € | Climatisation incluse |
| **Total OPEX annuel** | **207 000 € — 303 000 €** | |

### 15.3 TCO sur 5 ans

```
TCO 5 ans = CAPEX + (OPEX × 5)
         = ~850 000 € + (255 000 € × 5)
         = ~850 000 € + 1 275 000 €
         = ~2 125 000 €

Soit environ 425 000 €/an tout compris
Ou environ 1 060 €/employé/an

ROI attendu :
  • Réduction coûts WAN de 40% à terme (SD-WAN remplace MPLS progressivement)
  • Réduction temps d'administration de 60% (automatisation Ansible)
  • Réduction incidents réseau de 70% (NAC + monitoring proactif)
  • Temps de déploiement nouveaux sites : jours au lieu de mois
```

---

## 16. Planning de déploiement

### Phase 1 : Analyse et conception (Semaines 1-2)

- Validation de l'architecture avec les parties prenantes
- Finalisation du plan d'adressage IP et de la matrice de flux
- Commande des équipements
- Préparation de l'environnement NetBox (inventaire + IPAM)
- Création des templates Ansible et des rôles de base

### Phase 2 : Déploiement infrastructure (Semaines 3-6)

- Installation du câblage et des équipements physiques
- Configuration Core + Distribution + Firewalls
- Déploiement Leaf-Spine datacenter
- Mise en service des liaisons WAN (MPLS, fibre, 4G)
- Configuration SD-WAN et politiques de routage
- Déploiement WiFi (contrôleur + bornes)

### Phase 3 : Sécurité et NAC (Semaines 7-10)

- Déploiement Cisco ISE (cluster HA)
- Phase monitor NAC (2 semaines)
- Phase low-impact NAC (2 semaines)
- Configuration des politiques de posture
- Intégration ISE ↔ AD ↔ FortiGate ↔ Splunk

### Phase 4 : Automatisation et monitoring (Semaines 11-14)

- Déploiement NetBox, Oxidized, GitLab, Ansible/AWX
- Création des playbooks de production
- Configuration Zabbix + Splunk + Grafana
- Tests de basculement WAN et DR
- Full enforcement NAC

### Phase 5 : Recette et mise en production (Semaines 15-16)

- Tests de charge et de performance
- Validation SLA (99.9%)
- Tests de sécurité (pentest interne)
- Formation des équipes IT
- Rédaction du runbook opérationnel
- Go-live progressif par étage

---

> **Document rédigé par** : Matthieu
> **Validé par** : CTO GreenTech Dynamics Inc.
> **Classification** : Exercice pour ma formation jedha
