# L-18 — HSRP & redondance passerelle

> **Niveau :** 🟠 Avancé · **Durée estimée :** 45 min · **Prérequis :** [L-11](../../02-intermediaire/lab-11-svi-l3/) · [L-13](../../02-intermediaire/lab-13-ospf-mono/)

Mettre en place deux switches L3 en redondance de passerelle avec HSRP,  
répartir la charge entre deux VLANs et observer le basculement automatique  
en cas de panne du routeur actif.

---

## 🎯 Objectifs

- Comprendre le principe de la redondance de passerelle (First Hop Redundancy Protocol)
- Configurer HSRP sur deux switches L3 avec une IP virtuelle par VLAN
- Répartir la charge : SW-A actif sur VLAN 10, SW-B actif sur VLAN 20
- Définir les priorités et activer `preempt`
- Simuler la panne du routeur actif et observer le basculement
- Comprendre la différence HSRP / VRRP / GLBP

---

## 🗺️ Topologie

```
                [ PC0 VLAN10 ]      [ PC1 VLAN20 ]
                       \                 /
                    [ Switch-Access (trunk) ]
                       /                 \
              [ SW-A ]═══ Po1 (peer-link) ═══[ SW-B ]
                  |                              |
                  └──────── OSPF area 0 ─────────┘
                              |
                         [ Router-ISP ]
```

> SW-A et SW-B sont deux switches L3 (3560) reliés à l'access switch en trunk  
> et reliés entre eux par un lien direct pour le trafic inter-VLAN et OSPF.  
> Point de départ : SVIs (L-11) et OSPF (L-13) déjà fonctionnels sur les deux switches.

---

## 📋 Plan d'adressage

| Équipement | Interface   | Adresse IP        | Rôle                          |
|------------|-------------|-------------------|-------------------------------|
| SW-A       | Vlan10      | 172.16.10.252/24  | Physique — actif VLAN10       |
| SW-B       | Vlan10      | 172.16.10.253/24  | Physique — standby VLAN10     |
| —          | Vlan10 HSRP | 172.16.10.254/24  | IP virtuelle — passerelle PCs |
| SW-A       | Vlan20      | 172.16.20.252/24  | Physique — standby VLAN20     |
| SW-B       | Vlan20      | 172.16.20.253/24  | Physique — actif VLAN20       |
| —          | Vlan20 HSRP | 172.16.20.254/24  | IP virtuelle — passerelle PCs |
| PC0        | Fa0         | 172.16.10.1/24    | Passerelle 172.16.10.254      |
| PC1        | Fa0         | 172.16.20.1/24    | Passerelle 172.16.20.254      |

---

## 📋 Groupes HSRP cibles

| VLAN | Groupe | Actif | Standby | Priority actif | Priority standby |
|------|--------|-------|---------|-----------------|-------------------|
| 10   | 1      | SW-A  | SW-B    | 110             | 100 (défaut)      |
| 20   | 2      | SW-B  | SW-A    | 110             | 100 (défaut)      |

---

## 📝 Travail demandé

### Étape 1 — Adresser les SVIs physiques

Sur **SW-A** :
```
configure terminal
interface vlan 10
 ip address 172.16.10.252 255.255.255.0
 no shutdown
interface vlan 20
 ip address 172.16.20.252 255.255.255.0
 no shutdown
end
```

Sur **SW-B** :
```
configure terminal
interface vlan 10
 ip address 172.16.10.253 255.255.255.0
 no shutdown
interface vlan 20
 ip address 172.16.20.253 255.255.255.0
 no shutdown
end
```

> Chaque switch garde une IP physique distincte sur chaque SVI — l'IP HSRP  
> vient s'ajouter par-dessus, elle n'est pas l'IP de l'interface.

### Étape 2 — HSRP groupe 1 (VLAN 10) — SW-A actif

Sur **SW-A** :
```
configure terminal
interface vlan 10
 standby 1 ip 172.16.10.254
 standby 1 priority 110
 standby 1 preempt
end
```

Sur **SW-B** :
```
configure terminal
interface vlan 10
 standby 1 ip 172.16.10.254
end
```

> Sans `priority`, SW-B reste à la valeur par défaut (100) — donc standby.

### Étape 3 — HSRP groupe 2 (VLAN 20) — SW-B actif

Sur **SW-B** :
```
configure terminal
interface vlan 20
 standby 2 ip 172.16.20.254
 standby 2 priority 110
 standby 2 preempt
end
```

Sur **SW-A** :
```
configure terminal
interface vlan 20
 standby 2 ip 172.16.20.254
end
```

### Étape 4 — Configurer les passerelles des PCs

- PC0 (VLAN10) → passerelle `172.16.10.254`
- PC1 (VLAN20) → passerelle `172.16.20.254`

### Étape 5 — Vérifier l'état HSRP

```
show standby brief
```

```
Interface   Grp  Pri  P State    Active          Standby         Virtual IP
Vl10        1    110  P Active   local           172.16.10.253   172.16.10.254
Vl20        2    100    Standby  172.16.20.253   local           172.16.20.254
```

### Étape 6 — Simuler la panne du routeur actif

Dans PT, débranche le câble reliant **SW-A** au reste du réseau (ou fais  
`shutdown` sur `interface vlan 10` de SW-A).

Observe sur **SW-B** :
```
show standby brief
```
SW-B doit passer en **Active** pour le groupe 1 après le timer HSRP (par défaut ~10s).

Depuis **PC0** :
```
ping 172.16.20.1    → succès sans interruption prolongée (bascule transparente)
```

Reconnecte SW-A. Grâce à `preempt`, SW-A redevient actif car sa priority (110) est supérieure.

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! État HSRP résumé
show standby brief

! Détail d'un groupe
show standby vlan 10

! Timers et état de préemption
show standby vlan 10 | include Preempt|Active|Standby
```

### Lire show standby brief

| Champ | Signification |
|-------|----------------|
| `P` | Preempt activé sur ce groupe |
| `Active` | Ce switch traite le trafic pour ce groupe |
| `Standby` | Ce switch est prêt à prendre le relais |
| `Virtual IP` | IP de passerelle vue par les PCs |

---

## ✅ Critères de réussite

- [ ] `show standby brief` sur SW-A — groupe 1 **Active**, groupe 2 **Standby**
- [ ] `show standby brief` sur SW-B — groupe 1 **Standby**, groupe 2 **Active**
- [ ] Ping PC0 → PC1 : **succès** en fonctionnement normal
- [ ] Après panne de SW-A : SW-B passe **Active** sur le groupe 1, ping toujours **succès**
- [ ] Après retour de SW-A : SW-A reprend le rôle **Active** (preempt)

---

## 💡 Points pédagogiques

**Pourquoi HSRP ?**  
Sans redondance, la panne du switch L3 qui sert de passerelle coupe tout  
le VLAN concerné. HSRP fournit une IP de passerelle virtuelle partagée  
par deux équipements physiques.

**`preempt`**  
Sans cette commande, un routeur qui redevient disponible avec une priority  
plus élevée ne reprend PAS automatiquement le rôle actif — il reste standby  
jusqu'à la prochaine panne du routeur actif en place. En production,  
`preempt` est généralement activé pour garantir un chemin prévisible.

**Répartition de charge (load-balancing) HSRP**  
HSRP de base élit un seul actif par groupe. En alternant le rôle actif entre  
SW-A (groupe 1) et SW-B (groupe 2), on répartit le trafic sortant des deux  
VLANs sur les deux liens montants au lieu de tout faire passer par un seul switch.

**HSRP vs VRRP vs GLBP**

| Protocole | Éditeur | Load-balancing natif |
|-----------|---------|-----------------------|
| HSRP | Cisco (propriétaire) | Non (nécessite plusieurs groupes) |
| VRRP | Standard IETF | Non |
| GLBP | Cisco (propriétaire) | Oui (plusieurs MAC virtuelles) |

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-18-hsrp.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-17 — OSPF multi-area](../lab-17-ospf-multi/)*  
*Lab suivant : [L-19 — Sécurité switch](../lab-19-securite-switch/)*
