# L-19 — Sécurité switch

> **Niveau :** 🟠 Avancé · **Durée estimée :** 45 min · **Prérequis :** [L-10](../../02-intermediaire/lab-10-etherchannel/) · [L-12](../../02-intermediaire/lab-12-dhcp-helper/)

Durcir la sécurité de la couche 2 : limiter les adresses MAC par port,  
bloquer les serveurs DHCP non autorisés et prévenir l'usurpation ARP.

---

## 🎯 Objectifs

- Limiter le nombre d'adresses MAC apprises par port avec `port-security`
- Choisir et comprendre les modes de violation (`shutdown`, `restrict`, `protect`)
- Activer `DHCP snooping` pour bloquer les serveurs DHCP non autorisés
- Configurer `Dynamic ARP Inspection` (DAI) pour prévenir l'ARP spoofing
- Désactiver les ports inutilisés

---

## 🗺️ Topologie

```
PC0 (légitime) ──┐
PC-Rogue-DHCP ────┤── Switch-Access ── Uplink (trust) ── Switch-Distribution
Server0 (DHCP) ───┘        VLAN 10
```

> Point de départ : VLAN 10 déjà en place, DHCP centralisé (L-12) sur Server0.  
> Un PC-Rogue-DHCP est ajouté sur la topologie pour simuler un serveur DHCP  
> non autorisé branché par erreur ou malveillance.

---

## 📋 Plan de sécurisation

| Port switch     | Équipement       | Mesure                                    |
|------------------|------------------|--------------------------------------------|
| Fa0/1            | PC0              | port-security max 2, violation restrict    |
| Fa0/2            | PC-Rogue-DHCP    | port-security max 2, violation restrict    |
| Fa0/7            | Server0 (DHCP)   | `ip dhcp snooping trust`                   |
| Gi0/1 (uplink)   | Switch-Distribution | `ip dhcp snooping trust` + `ip arp inspection trust` |
| Fa0/10 → Fa0/24  | Non utilisés     | `shutdown` + VLAN poubelle                 |

---

## 📝 Travail demandé

### Étape 1 — Port-security sur les ports access

Sur **Switch-Access** :

```
configure terminal
interface fa0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
interface fa0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
end
```

> `switchport port-security` doit être activé **avant** de pouvoir configurer  
> `maximum` ou `violation`. `sticky` convertit les MAC apprises dynamiquement  
> en entrées statiques sauvegardées dans la config.

### Étape 2 — Tester le port-security

Branche un second PC sur le même câble que PC0 via un mini-switch simulé,  
ou change l'adresse MAC de PC0 dans PT, pour dépasser la limite de 2.

```
show port-security interface fa0/1
```

En mode `restrict`, le port reste up mais les trames de la MAC en trop  
sont bloquées et un compteur de violation s'incrémente (pas de coupure  
du port comme en mode `shutdown`).

### Étape 3 — Activer DHCP snooping

```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10
interface fa0/7
 ip dhcp snooping trust
interface gi0/1
 ip dhcp snooping trust
end
```

> Par défaut, **tous les ports sont untrusted**. Seuls les ports menant à un  
> serveur DHCP légitime (Fa0/7) ou vers l'upstream (Gi0/1) doivent être trust.  
> Fa0/1 et Fa0/2 restent untrusted — les réponses DHCP (`OFFER`/`ACK`) y sont bloquées.

### Étape 4 — Tester le blocage du rogue DHCP

Sur **PC-Rogue-DHCP**, active un service DHCP simulé (Server-PT ou fonction  
DHCP du PC dans PT si disponible) sur le port Fa0/2 (untrusted).

Sur un PC client, lance un renouvellement DHCP :
```
ipconfig /release
ipconfig /renew
```

Le PC doit recevoir son bail depuis **Server0** (le port légitime), jamais  
depuis PC-Rogue-DHCP — les trames `DHCP OFFER` émises depuis un port  
untrusted sont supprimées par le switch.

```
show ip dhcp snooping
show ip dhcp snooping binding
```

### Étape 5 — Activer Dynamic ARP Inspection (DAI)

DAI s'appuie sur la table de bindings DHCP snooping pour valider les  
paires IP/MAC dans les paquets ARP.

```
configure terminal
ip arp inspection vlan 10
interface fa0/7
 ip arp inspection trust
interface gi0/1
 ip arp inspection trust
end
```

> Les ports untrusted (Fa0/1, Fa0/2) filtrent les ARP : seule une IP/MAC  
> présente dans la table `ip dhcp snooping binding` est acceptée.  
> Un poste avec IP statique doit être ajouté via `ip source binding` ou  
> une ARP ACL, sinon son trafic ARP sera rejeté sur un port DAI-protégé.

### Étape 6 — Désactiver les ports inutilisés

```
configure terminal
interface range fa0/10 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
end
```

> VLAN 999 = VLAN "poubelle" sans routage — un câble branché par erreur  
> sur un port inutilisé ne peut accéder à aucun VLAN de production, même  
> si l'administrateur oublie de le réactiver correctement.

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Port-security
show port-security
show port-security interface fa0/1
show port-security address

! DHCP snooping
show ip dhcp snooping
show ip dhcp snooping binding

! DAI
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics vlan 10
```

### Modes de violation port-security

| Mode | Comportement |
|------|---------------|
| `shutdown` | Le port passe en `err-disabled` — coupure totale, nécessite `shutdown`/`no shutdown` manuel |
| `restrict` | Trafic de la MAC en trop bloqué, port reste up, compteur incrémenté, log/SNMP trap |
| `protect` | Trafic de la MAC en trop bloqué, port reste up, **aucun** log |

---

## ✅ Critères de réussite

- [ ] `show port-security interface fa0/1` — maximum 2, violation restrict, sticky actif
- [ ] `show ip dhcp snooping` — activé, VLAN 10, Fa0/7 et Gi0/1 en trust
- [ ] Renouvellement DHCP client — bail attribué uniquement par Server0
- [ ] `show ip dhcp snooping binding` — bindings visibles pour les clients légitimes
- [ ] `show ip arp inspection` — activé sur VLAN 10, Fa0/7 et Gi0/1 en trust
- [ ] Ports Fa0/10 à Fa0/24 — état **administratively down**, VLAN 999

---

## 💡 Points pédagogiques

**Pourquoi sécuriser la couche 2 ?**  
La plupart des attaques L2 (rogue DHCP, ARP spoofing, MAC flooding) sont  
invisibles pour les ACL ou le firewall — elles se produisent avant même  
que le paquet atteigne la couche 3. La sécurité doit commencer au port switch.

**DHCP snooping — brique de base**  
DAI et l'IP Source Guard s'appuient tous les deux sur la table de bindings  
construite par DHCP snooping. Sans DHCP snooping actif, DAI ne peut pas  
fonctionner correctement — la table de référence serait vide.

**MAC flooding vs port-security**  
Une attaque de MAC flooding sature la table CAM du switch pour le forcer  
à broadcaster tout le trafic (comme un hub) — permettant l'interception.  
`port-security maximum` limite le nombre de MAC apprises par port et  
empêche ce remplissage massif depuis un seul port compromis.

**Ports inutilisés = surface d'attaque**  
Un port access non sécurisé et non désactivé est une porte ouverte :  
n'importe qui avec un accès physique peut brancher un câble et rejoindre  
le réseau de production.

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-19-securite-switch.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-18 — HSRP & redondance passerelle](../lab-18-hsrp/)*  
*Lab suivant : [L-20 — VPN site-à-site IPsec](../lab-20-vpn-ipsec/)*
