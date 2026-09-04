# L-19 — Sécurité switch

> **Niveau :** 🟠 Avancé · **Durée estimée :** 45 min · **Prérequis :** [L-10](../../02-intermediaire/lab-10-etherchannel/) · [L-12](../../02-intermediaire/lab-12-dhcp-helper/)

Durcir la sécurité de la couche 2 sur le réseau d'entreprise construit depuis L-07 :  
limiter les adresses MAC par port, bloquer un serveur DHCP non autorisé et prévenir  
l'usurpation ARP — sans casser le DHCP centralisé mis en place en L-12.

> Point de départ : reprend l'état final de L-12 (VTP, VLANs 10/20/30, trunks,  
> EtherChannel Po1, lien redondant Switch2↔Switch3, SVIs routées, `ip helper-address`  
> vers Server0) — voir [`initial-config/`](initial-config/) pour les scripts à pousser  
> avant la séance. Un nouvel équipement, **Server-Rogue** (`Server-PT`), est ajouté  
> sur un port jusqu'ici inutilisé pour simuler la menace ; Server0 n'est pas déplacé,  
> son service DHCP existant (L-12) reste actif.

---

## 🎯 Objectifs

- Limiter le nombre d'adresses MAC apprises par port avec `port-security`
- Choisir et comprendre les modes de violation (`shutdown`, `restrict`, `protect`)
- Activer `DHCP snooping` pour protéger le DHCP centralisé de L-12 contre un serveur non autorisé
- Configurer `Dynamic ARP Inspection` (DAI) pour prévenir l'ARP spoofing
- Désactiver les ports inutilisés
- Revoir la portée de VTP : une VLAN "poubelle" se crée sur le serveur VTP, jamais sur un client

---

## 🗺️ Topologie

```
                         [ Multilayer Switch0 ]  ← VTP server, SVIs, ip helper-address
                        /  Gi0/1        Gi0/1  \  Gi0/1
                       /       \            \    \
              [ Switch2 ]══F0/24↔F0/24══[ Switch3 ]═══Po1═══[ Switch4 ]
                   |                    /   |   \             /  |  \
               Server0              PC3   PC0  PC2         PC1  PC5 PC4
              VLAN20 (F0/7)        V30    V10  V10         V30  V30 V10
                                          (F0/6 libre)
                                              |
                                        Server-Rogue   ← ajouté pour ce lab
```

> Topologie identique à celle de L-07 → L-12 (mêmes switches, mêmes PCs, mêmes  
> VLANs), **plus** le lien redondant Switch2↔Switch3 (F0/24↔F0/24, trunk géré  
> par STP) déjà présent depuis L-09 mais absent de la documentation de ce lab  
> jusqu'ici. Seul **Server-Rogue** est nouveau, branché sur **Switch3 Fa0/6**  
> (libre jusqu'ici).

---

## 📋 Rappel des ports (topologie réelle L-07 → L-12)

| Switch  | Port    | VLAN | Équipement / lien |
|---------|---------|------|--------------------|
| Multilayer Switch0 | Fa0/1 → Switch2 Gi0/1 | trunk | uplink |
| Multilayer Switch0 | Fa0/2 → Switch3 Gi0/1 | trunk | uplink |
| Multilayer Switch0 | Fa0/3 → Switch4 Gi0/1 | trunk | uplink |
| Switch2 | Fa0/7   | 20   | Server0 (DHCP légitime, 172.16.20.10) |
| Switch2 | Fa0/24 → Switch3 Fa0/24 | trunk | lien redondant (STP) |
| Switch3 | Fa0/1   | 30   | PC3                |
| Switch3 | Fa0/4   | 10   | PC0                |
| Switch3 | Fa0/5   | 10   | PC2                |
| Switch3 | Fa0/6   | 10   | **Server-Rogue (nouveau)** |
| Switch3 | Fa0/22 → Switch4 Fa0/23 | trunk | Po1 (LACP), membre 1 |
| Switch3 | Fa0/23 → Switch4 Fa0/24 | trunk | Po1 (LACP), membre 2 |
| Switch4 | Fa0/1   | 30   | PC1                |
| Switch4 | Fa0/2   | 30   | PC5                |
| Switch4 | Fa0/6   | 10   | PC4                |

**Ports libres avant ajout de Server-Rogue :**
- Switch3 : Fa0/2-3, Fa0/6-21, Gi0/2
- Switch4 : Fa0/3-5, Fa0/7-22, Gi0/2

---

## 📋 Plan de sécurisation

| Port(s)                                | Switch    | Mesure                                    |
|------------------------------------------|-----------|--------------------------------------------|
| Fa0/1, Fa0/4, Fa0/5, Fa0/6 (Server-Rogue) | Switch3   | port-security max 2, violation restrict, sticky |
| Fa0/1, Fa0/2, Fa0/6                       | Switch4   | port-security max 2, violation restrict, sticky |
| Fa0/7 (Server0)                           | Switch2   | `ip dhcp snooping trust`                   |
| Fa0/24 (lien vers Switch3)                | Switch2   | `ip dhcp snooping trust`                   |
| Fa0/24 (lien vers Switch2)                | Switch3   | `ip dhcp snooping trust` + `ip arp inspection trust` |
| Gi0/1 (uplink vers Multilayer Switch0)    | Switch3, Switch4 | `ip dhcp snooping trust` + `ip arp inspection trust` |
| Ports libres restants                     | Switch3, Switch4 | `shutdown` + VLAN poubelle (999)          |

> Tout lien **inter-switch** (uplink vers Multilayer Switch0, ou lien redondant  
> Switch2↔Switch3) passe en trust, quel que soit son état STP actuel (forwarding  
> ou blocking) : le blocage STP peut changer après un événement de topologie,  
> la confiance DHCP/ARP doit être basée sur la nature du lien, pas sur son état  
> momentané.

---

## 📝 Travail demandé

### Étape 1 — Port-security sur les ports access existants

Sur **Switch3** (Fa0/1, Fa0/4, Fa0/5) et **Switch4** (Fa0/1, Fa0/2, Fa0/6) :

```
configure terminal
interface fa0/x
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
end
```

> `switchport port-security` doit être activé **avant** de pouvoir configurer  
> `maximum` ou `violation`. `sticky` convertit les MAC apprises dynamiquement  
> en entrées statiques sauvegardées dans la config.

### Étape 2 — Brancher et configurer Server-Rogue

Ajoute un équipement **Server-PT** nommé `Server-Rogue`, branché sur **Switch3 Fa0/6**.

```
configure terminal
interface fa0/6
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
end
```

> Un `Server-PT` est nécessaire ici : un `PC-PT` n'a pas de service DHCP serveur  
> dans Packet Tracer, seul `Server-PT` (ou un routeur avec `ip dhcp pool`) en a un.  
> Server0 n'est pas touché — son service DHCP (L-12) continue de tourner normalement.

### Étape 3 — Activer DHCP snooping

Sur **Switch2** :
```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 20
interface fa0/7
 ip dhcp snooping trust
interface fa0/24
 ip dhcp snooping trust
end
```

Sur **Switch3** :
```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,30
interface gi0/1
 ip dhcp snooping trust
interface fa0/24
 ip dhcp snooping trust
end
```

Sur **Switch4** :
```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,30
interface gi0/1
 ip dhcp snooping trust
end
```

> Par défaut, **tous les ports sont untrusted**. Seuls les liens inter-switches  
> (Fa0/7 vers Server0, Fa0/24 entre Switch2 et Switch3, Gi0/1 vers Multilayer  
> Switch0) passent en trust. Tous les ports PC — y compris Fa0/6 sur Switch3,  
> où se trouve Server-Rogue — restent untrusted.

### Étape 4 — Tester le blocage du DHCP non autorisé

Sur **Server-Rogue**, active un service DHCP (Services → DHCP → ON) avec une  
étendue arbitraire (ex. passerelle et DNS différents de ceux de L-12).

Sur un PC du VLAN 10 (ex. PC0) :
```
ipconfig /release
ipconfig /renew
```

Le PC doit recevoir son bail depuis **Server0** (relayé via `ip helper-address`),  
jamais depuis Server-Rogue — les trames `DHCP OFFER` émises depuis un port  
untrusted sont supprimées par le switch.

```
show ip dhcp snooping
show ip dhcp snooping binding
```

### Étape 5 — Activer Dynamic ARP Inspection (DAI)

DAI s'appuie sur la table de bindings DHCP snooping pour valider les  
paires IP/MAC dans les paquets ARP.

Sur **Switch3** :
```
configure terminal
ip arp inspection vlan 10,30
interface gi0/1
 ip arp inspection trust
interface fa0/24
 ip arp inspection trust
end
```

Sur **Switch4** :
```
configure terminal
ip arp inspection vlan 10,30
interface gi0/1
 ip arp inspection trust
end
```

> Les ports untrusted filtrent les ARP : seule une IP/MAC présente dans la  
> table `ip dhcp snooping binding` est acceptée.

### Étape 6 — Désactiver les ports inutilisés

Sur **Multilayer Switch0** (serveur VTP — c'est le seul endroit où une VLAN  
peut être créée, les autres switches sont en mode `client`) :

```
configure terminal
vlan 999
 name PARKING
end
```

Sur **Switch3** (ports libres restants : Fa0/2-3, Fa0/7-21, Gi0/2) :

```
configure terminal
interface range fa0/2 - 3 , fa0/7 - 21
 switchport mode access
 switchport access vlan 999
 shutdown
interface gi0/2
 shutdown
end
```

Sur **Switch4** (ports libres restants : Fa0/3-5, Fa0/7-22, Gi0/2) :

```
configure terminal
interface range fa0/3 - 5 , fa0/7 - 22
 switchport mode access
 switchport access vlan 999
 shutdown
interface gi0/2
 shutdown
end
```

> Une VLAN créée directement sur Switch3 ou Switch4 (mode VTP client) serait  
> refusée — c'est le même principe qu'en L-07. VLAN 999 = VLAN "poubelle"  
> sans routage — un câble branché par erreur sur un port inutilisé ne peut  
> accéder à aucun VLAN de production, même si l'administrateur oublie de  
> le réactiver correctement.

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

- [ ] `show port-security interface fa0/1` (Switch3) — maximum 2, violation restrict, sticky actif
- [ ] `show ip dhcp snooping` sur Switch3/Switch4 — activé, VLAN 10 et 30, Gi0/1 (et Fa0/24 sur Switch3) en trust
- [ ] Renouvellement DHCP sur PC0 (VLAN10) — bail attribué uniquement par Server0, jamais par Server-Rogue
- [ ] `show ip dhcp snooping binding` — bindings visibles pour les PC légitimes
- [ ] `show ip arp inspection` — activé sur VLAN 10 et 30, mêmes ports en trust
- [ ] `vlan 999` visible via `show vlan brief` sur Switch3 **et** Switch4 (propagée par VTP depuis Multilayer Switch0)
- [ ] Ports libres restants sur Switch3/Switch4 — état **administratively down**, VLAN 999
- [ ] Ping PC0 → Server0, PC0 → PC2 : **toujours réussi** (le durcissement ne casse rien de L-07 → L-12)

---

## 💡 Points pédagogiques

**Pourquoi sécuriser la couche 2 ?**  
La plupart des attaques L2 (rogue DHCP, ARP spoofing, MAC flooding) sont  
invisibles pour les ACL ou le firewall — elles se produisent avant même  
que le paquet atteigne la couche 3. La sécurité doit commencer au port switch.

**DHCP snooping — brique de base**  
DAI s'appuie sur la table de bindings construite par DHCP snooping. Sans  
DHCP snooping actif, DAI ne peut pas fonctionner correctement — la table  
de référence serait vide.

**Pourquoi trust Gi0/1 ET Fa0/24, et pas les ports PC ?**  
Le DHCP centralisé de L-12 répond depuis Server0 (VLAN20, sur Switch2) via  
le relais `ip helper-address` de Multilayer Switch0. Pour les PCs des VLANs  
10 et 30 (sur Switch3/Switch4), cette réponse légitime entre par l'uplink  
Gi0/1 — mais elle pourrait aussi transiter par le lien redondant Switch2↔  
Switch3 (Fa0/24) si la topologie STP change. Les deux doivent donc être trust.

**MAC flooding vs port-security**  
Une attaque de MAC flooding sature la table CAM du switch pour le forcer  
à broadcaster tout le trafic (comme un hub) — permettant l'interception.  
`port-security maximum` limite le nombre de MAC apprises par port et  
empêche ce remplissage massif depuis un seul port compromis.

**VTP et VLAN poubelle**  
Comme en L-07, seul le serveur VTP (Multilayer Switch0) peut créer une VLAN —  
les switches clients (Switch3, Switch4) ne font que la recevoir. Oublier ce  
détail est une erreur fréquente : créer `vlan 999` directement sur un switch  
client est silencieusement rejeté.

**Ports inutilisés = surface d'attaque**  
Un port access non sécurisé et non désactivé est une porte ouverte :  
n'importe qui avec un accès physique peut brancher un câble et rejoindre  
le réseau de production — c'est exactement le scénario simulé avec Server-Rogue.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-19-securite-switch.pka` | Fichier activité Packet Tracer |
| `initial-config/Switch2.txt` | État hérité de L-07/L-09/L-12 (local, non versionné) |
| `initial-config/Switch3.txt` | État hérité de L-07/L-09/L-10 (local, non versionné) |
| `initial-config/Switch4.txt` | État hérité de L-07/L-09/L-10 (local, non versionné) |
| `initial-config/Multilayer-Switch0.txt` | État hérité de L-07/L-09/L-11/L-12 (local, non versionné) |

---

*Lab précédent : [L-18 — HSRP & redondance passerelle](../lab-18-hsrp/)*  
*Lab suivant : [L-20 — VPN site-à-site IPsec](../lab-20-vpn-ipsec/)*
