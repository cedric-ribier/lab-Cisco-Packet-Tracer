# 🔬 lab-Cisco-Packet-Tracer

Série de labs progressifs pour apprendre la configuration réseau Cisco avec **Packet Tracer**.  
Chaque lab est un fichier `.pka` autonome avec énoncé intégré, scoring automatique et corrigé.

---

## 🎯 Objectif

Fournir une progression pédagogique complète, du premier câblage jusqu'à l'architecture d'entreprise,  
utilisable en autonomie ou en accompagnement de formation TSSR / CCNA.

---

## 📋 Prérequis

- [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) 8.x ou supérieur
- Compte Cisco NetAcad (gratuit) pour télécharger PT

---

## 🗂️ Structure du dépôt

```
lab-Cisco-Packet-Tracer/
├── 01-debutant/          # Bases réseau, VLANs, routage statique, DHCP
├── 02-intermediaire/     # STP, EtherChannel, SVIs, OSPF, ACL, NAT
├── 03-avance/            # OSPF multi-area, HSRP, sécurité, VPN, BGP
└── 04-expert/            # QoS, réseau d'entreprise complet
```

---

## 📚 Liste des labs

### 🟢 Niveau 1 — Débutant

| Lab | Titre | Thèmes | Durée |
|-----|-------|--------|-------|
| [L-01](01-debutant/lab-01-premier-pas/) | Premier pas | Interface CPT, câblage, ping | 15 min |
| [L-02](01-debutant/lab-02-osi-adressage/) | Modèle OSI & plan d'adressage | OSI, IPv4, subnetting | 20 min |
| [L-03](01-debutant/lab-03-ios-base/) | Commandes IOS de base | IOS, SSH, show, sauvegarde | 20 min |
| [L-04](01-debutant/lab-04-vlans-access/) | VLANs & ports access | VLAN, access, isolation L2 | 20 min |
| [L-05](01-debutant/lab-05-routage-statique/) | Routage statique | ip route, default route | 30 min |
| [L-06](01-debutant/lab-06-dhcp/) | DHCP sur routeur | DHCP pool, options, baux | 20 min |
| [L-07](01-debutant/lab-07-vtp-trunks/) | VTP & trunks dot1q | VTP server/client, trunk, dot1q | 40 min |
| [L-08](01-debutant/lab-08-roas/) | Router-on-a-stick | Sous-interfaces, inter-VLAN | 30 min |

### 🔵 Niveau 2 — Intermédiaire

| Lab | Titre | Thèmes | Durée |
|-----|-------|--------|-------|
| [L-09](02-intermediaire/lab-09-stp/) | STP & bridge priority | STP, root bridge, PortFast | 30 min |
| [L-10](02-intermediaire/lab-10-etherchannel/) | EtherChannel LACP | LACP, Port-channel, redondance | 30 min |
| [L-11](02-intermediaire/lab-11-svi-l3/) | SVIs & routage L3 switch | ip routing, SVI, L3 switch | 30 min |
| [L-12](02-intermediaire/lab-12-dhcp-helper/) | DHCP centralisé & ip helper | DHCP server, helper-address | 30 min |
| [L-13](02-intermediaire/lab-13-ospf-mono/) | OSPF mono-area | OSPF area 0, DR/BDR | 45 min |
| [L-14](02-intermediaire/lab-14-acl-standard/) | ACL standard | permit, deny, filtrage source | 30 min |
| [L-15](02-intermediaire/lab-15-nat-pat/) | NAT & PAT | NAT statique, PAT overload | 30 min |
| [L-16](02-intermediaire/lab-16-acl-etendues/) | ACL étendues & DMZ | Named ACL, DMZ, filtrage L4 | 45 min |

### 🟠 Niveau 3 — Avancé

| Lab | Titre | Thèmes | Durée |
|-----|-------|--------|-------|
| [L-17](03-avance/lab-17-ospf-multi/) | OSPF multi-area | ABR, summarization, redondance backbone | 45 min |
| [L-18](03-avance/lab-18-hsrp/) | HSRP & redondance | HSRP, failover, preempt | 45 min |
| [L-19](03-avance/lab-19-securite-switch/) | Sécurité switch | Port-security, DHCP snooping, DAI | 45 min |
| [L-20](03-avance/lab-20-vpn-ipsec/) | VPN site-à-site IPsec | IKEv1, crypto map, tunnel | 60 min |
| [L-21](03-avance/lab-21-bgp/) | BGP bases (eBGP) | eBGP, AS-PATH, préfixes | 60 min |

### 🟣 Niveau 4 — Expert

| Lab | Titre | Thèmes | Durée |
|-----|-------|--------|-------|
| [L-22](04-expert/lab-22-qos/) | QoS bases | DSCP, MQC, priority queue | 45 min |
| [L-23](04-expert/lab-23-entreprise-1/) | Réseau d'entreprise — partie 1 | 3-tier, OSPF, HSRP, DHCP | 90 min |
| [L-24](04-expert/lab-24-entreprise-2/) | Réseau d'entreprise — partie 2 | NAT, ACL, VPN, QoS | 90 min |

---

## 📁 Contenu de chaque lab

```
lab-XX-nom/
├── README.md              # Corrigé complet : objectifs, prérequis, topologie, commandes de vérification
├── instructions.html      # Énoncé à coller dans le Plain Editor du Activity Wizard (local, non versionné — voir .gitignore)
└── lab-XX-nom.pka         # Fichier Packet Tracer Activity (apprenant)
```

---

## 🚀 Comment utiliser un lab

1. Télécharge le fichier `.pka` du lab souhaité
2. Ouvre-le dans Packet Tracer 8.x
3. Lis l'énoncé dans le panneau **Instructions** (coche **Dock** pour l'ancrer)
4. Configure les équipements selon les consignes
5. Clique **Check Results** pour voir ta progression

---

## 🗺️ Graphe de dépendances

```
L-01 → L-02 → L-05 → L-06 → L-08
         ↓              ↓
        L-03 → L-04 → L-07
                            ↓
                  L-09 → L-10 → L-11 → L-12
                                   ↓
                  L-13 → L-14 → L-15 → L-16 → L-20
                    ↓
                  L-17 → L-21
                  L-18 → L-23 → L-24
                  L-19 ↗
                  L-22 ↗
```

---

## 🤝 Contribuer

Les fichiers `.pka` ne sont pas versionnables ligne à ligne (format binaire).  
Les contributions sont les bienvenues sur les fichiers texte : `README.md`, `instructions.html`.

1. Fork le dépôt
2. Crée une branche `feat/lab-XX-nom`
3. Propose une Pull Request

---

## 📄 Licence

Ce dépôt est partagé à des fins pédagogiques.  
Cisco, Packet Tracer et IOS sont des marques déposées de Cisco Systems, Inc.

---

*Maintenu par [@cedric-ribier](https://github.com/cedric-ribier) — Formation TSSR · Normandie*
