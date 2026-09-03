# L-17 — OSPF multi-area

> **Niveau :** 🟠 Avancé · **Durée estimée :** 45 min · **Prérequis :** [L-13](../../02-intermediaire/lab-13-ospf-mono/)

Structurer un réseau OSPF en plusieurs areas autour du backbone, configurer les ABR,  
observer les routes inter-area et vérifier la tolérance aux pannes grâce à un  
second chemin structurellement plus long sur le backbone.

> Point de départ : toutes les interfaces et adresses IP sont déjà configurées  
> sur chaque équipement (voir plan d'adressage). Le travail demandé porte  
> uniquement sur l'implémentation d'OSPF.

---

## 🎯 Objectifs

- Comprendre la hiérarchie OSPF : backbone area 0 + areas secondaires
- Configurer les ABR (Area Border Routers)
- Distinguer les routes intra-area (O) et inter-area (O IA)
- Réduire la taille de la LSDB par area
- Configurer un résumé de routes sur un ABR
- Comprendre comment OSPF choisit le chemin le moins coûteux (SPF) entre deux routes possibles
- Observer la reconvergence automatique d'OSPF après la panne du chemin principal

---

## 🗺️ Topologie

```
   Area 1                          Area 0 (backbone)                        Area 2
192.168.1.0/24                                                           192.168.2.0/24
10.1.0.0/24 (Lo1)      10.0.0.0/30        10.0.1.0/30

   PC0 ── R0 ───────────────────── R1 ───────────────────── R2 ── PC1
           │      chemin principal (2 sauts, coût 2)               │
           │                                                       │
           └── R3 ─────────────── R4 ────────────────────────────┘
        10.0.2.0/30    10.0.3.0/30              10.0.4.0/30
              chemin de secours (3 sauts, coût 3)
        utilisé uniquement si R0–R1 ou R1–R2 tombe en panne
```

> R1 et R3/R4 sont de simples routeurs de transit, sans aucun client — comme R1  
> dans la version mono-area (L-13). Le chemin via R3–R4 comporte volontairement  
> **un saut de plus** que le chemin via R1 : à coût de lien identique (GigabitEthernet  
> partout), OSPF additionne un coût par saut traversé, donc le chemin le plus court  
> en nombre de sauts gagne toujours. C'est cette différence structurelle — et non  
> un réglage de coût manuel — qui garantit que R3–R4 reste un vrai chemin de secours,  
> jamais utilisé en fonctionnement normal.
>
> `Lo1` sur R0 est une interface **loopback** (virtuelle, toujours up) qui simule  
> un second segment LAN de l'Area 1, uniquement pour illustrer le résumé de routes  
> à l'étape 6.

---

## 📋 Plan d'adressage (déjà configuré sur chaque équipement)

| Équipement | Interface   | Adresse IP       | Area   | Rôle                          |
|------------|-------------|------------------|--------|-------------------------------|
| PC0        | Fa0         | 192.168.1.1/24   | Area 1 | Poste client                  |
| R0         | Gi0/0       | 192.168.1.254/24 | Area 1 | Passerelle PC0                |
| R0         | Loopback1   | 10.1.0.1/24      | Area 1 | 2ᵉ réseau Area 1 (virtuel)     |
| R0         | Gi0/1       | 10.0.0.1/30      | Area 0 | Lien vers R1 (chemin principal) |
| R0         | Gi0/2       | 10.0.2.1/30      | Area 0 | Lien vers R3 (chemin de secours) |
| R1         | Gi0/0       | 10.0.0.2/30      | Area 0 | Lien vers R0                  |
| R1         | Gi0/1       | 10.0.1.1/30      | Area 0 | Lien vers R2                  |
| R2         | Gi0/0       | 10.0.1.2/30      | Area 0 | Lien vers R1 (chemin principal) |
| R2         | Gi0/2       | 10.0.4.2/30      | Area 0 | Lien vers R4 (chemin de secours) |
| R2         | Gi0/1       | 192.168.2.254/24 | Area 2 | Passerelle PC1                |
| R3         | Gi0/0       | 10.0.2.2/30      | Area 0 | Lien vers R0                  |
| R3         | Gi0/1       | 10.0.3.1/30      | Area 0 | Lien vers R4                  |
| R4         | Gi0/0       | 10.0.3.2/30      | Area 0 | Lien vers R3                  |
| R4         | Gi0/1       | 10.0.4.1/30      | Area 0 | Lien vers R2                  |
| PC1        | Fa0         | 192.168.2.1/24   | Area 2 | Poste client                  |

---

## 📝 Travail demandé

### Étape 1 — OSPF sur R0 (ABR Area1/Area0, 2 sorties backbone)

```
configure terminal
router ospf 1
 network 192.168.1.0 0.0.0.255 area 1
 network 10.1.0.0 0.0.0.255 area 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
end
```

### Étape 2 — OSPF sur R1 (transit, chemin principal)

```
configure terminal
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
end
```

### Étape 3 — OSPF sur R2 (ABR Area0/Area2, 2 entrées backbone)

```
configure terminal
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.255 area 2
end
```

### Étape 4 — OSPF sur R3 (transit, chemin de secours)

```
configure terminal
router ospf 1
 network 10.0.2.0 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
end
```

### Étape 5 — OSPF sur R4 (transit, chemin de secours)

```
configure terminal
router ospf 1
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
end
```

### Étape 6 — Vérifier les voisinages et le chemin retenu

```
show ip ospf neighbor
show ip route ospf
```

R0 doit avoir 2 voisins directs (R1 et R3), R2 également (R1 et R4).  
Sur R0, `show ip route` vers 192.168.2.0/24 doit indiquer un **next-hop vers R1**  
(chemin principal, coût le plus bas) — jamais vers R3.

### Étape 7 — Résumé de routes sur R0 (ABR)

Au lieu d'envoyer 192.168.1.0/24 et 10.1.0.0/24 séparément vers Area 0,  
R0 peut les résumer en un seul préfixe :

```
configure terminal
router ospf 1
 area 1 range 10.1.0.0 255.255.255.0
 area 1 range 192.168.1.0 255.255.255.0
end
```

Observe la différence dans `show ip route` sur R1.

### Étape 8 — Tester la tolérance aux pannes

Dans Packet Tracer, coupe le lien **R0 – R1** (débranche le câble ou fais  
`shutdown` sur l'interface Gi0/1 de R0).

Lance un ping continu depuis PC0 :
```
ping 192.168.2.1 -t
```

Après une brève interruption (temps de convergence OSPF), le ping doit reprendre  
automatiquement — le trafic bascule sur le chemin de secours **R0 – R3 – R4 – R2**.  
Vérifie sur R0 :
```
show ip ospf neighbor
show ip route
```
R0 n'a plus que R3 comme voisin direct, mais conserve une route complète vers  
Area 2 via ce chemin plus long — sans aucune intervention manuelle.

Reconnecte le lien R0 – R1 et vérifie qu'OSPF reprend le chemin principal (coût  
le plus bas) via R1.

### Étape 9 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Voisinages OSPF
show ip ospf neighbor

! Routes avec distinction O / O IA
show ip route ospf

! ABR et ASBR
show ip ospf border-routers

! LSDB par area
show ip ospf database

! Coût OSPF d'une interface
show ip ospf interface gi0/1
```

### Lire show ip route — routes OSPF

```
O    10.0.0.0/30    [110/2] via ...    ← intra-area
O IA 192.168.2.0/24 [110/3] via ...   ← inter-area (autre area)
```

| Code | Signification |
|------|---------------|
| `O` | Route OSPF intra-area |
| `O IA` | Route OSPF inter-area (via ABR) |
| `O E1/E2` | Route externe redistribuée dans OSPF |

---

## ✅ Critères de réussite

- [ ] `show ip ospf neighbor` — tous les voisins en état **FULL** (R0 et R2 en ont 2 chacun)
- [ ] `show ip route` sur R1 — routes **O IA** vers Area 1 et Area 2
- [ ] `show ip ospf border-routers` — R0 et R2 identifiés comme ABR
- [ ] `show ip route` sur R0 — route vers 192.168.2.0/24 via **R1** (chemin principal, coût le plus bas)
- [ ] Ping PC0 → PC1 : **succès**
- [ ] Après résumé sur R0 : `show ip route` sur R1 montre les préfixes résumés
- [ ] Après coupure du lien R0 – R1 : ping PC0 → PC1 **toujours réussi**, via le chemin de secours R0–R3–R4–R2
- [ ] Après reconnexion : `show ip route` sur R0 repasse par **R1**

---

## 💡 Points pédagogiques

**Pourquoi plusieurs areas ?**  
En mono-area, tous les routeurs échangent toute la LSDB — coûteux en  
mémoire et CPU sur les grands réseaux. Le multi-area réduit la LSDB  
à la portée de chaque area, avec des routes résumées entre elles.

**Area 0 obligatoire**  
Toutes les areas doivent être connectées à l'area 0 (backbone).  
Un routeur connecté à deux areas est un ABR.

**O IA vs O**  
`O` = la route est dans la même area. `O IA` = la route vient d'une autre  
area via un ABR. Distance administrative identique (110) mais le chemin  
passe par un ABR supplémentaire.

**Résumé de routes**  
Configuré sur l'ABR, il réduit le nombre de préfixes injectés dans  
le backbone — moins d'instabilité propagée entre les areas.

**Pourquoi pas un simple lien direct R0–R2 comme chemin de secours ?**  
Par défaut, Cisco calcule le coût OSPF d'une interface avec  
`bande passante de référence (100 Mbps) / bande passante de l'interface` —  
et avec cette formule, FastEthernet **et** GigabitEthernet obtiennent tous les  
deux un coût de 1. Un lien direct R0–R2 (1 saut, coût 1) serait donc **toujours**  
préféré au chemin via R1 (2 sauts, coût 2) : ce ne serait pas un chemin de secours,  
mais le chemin principal en permanence, et R1 ne servirait jamais à rien.  
En ajoutant deux routeurs de transit (R3, R4), le chemin alternatif fait 3 sauts —  
structurellement plus coûteux que les 2 sauts du chemin principal, quel que soit  
le débit des liens. C'est le nombre de sauts traversés, et non la vitesse des liens,  
qui garantit ici la hiérarchie principal/secours.

**Tolérance aux pannes d'un IGP dynamique**  
Une topologie en chaîne (R0 – R1 – R2) n'offre qu'un seul chemin : la panne  
d'un lien coupe toute communication entre les areas, quel que soit le protocole  
de routage utilisé. C'est précisément ce qu'OSPF sait exploiter quand une  
alternative existe physiquement : il détecte la panne, recalcule le plus court  
chemin restant (algorithme SPF) et bascule le trafic automatiquement — sans  
reconfiguration manuelle, contrairement au routage statique.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-17-ospf-multi.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-16 — ACL étendues & DMZ](../../02-intermediaire/lab-16-acl-etendues/)*  
*Lab suivant : [L-18 — HSRP & redondance passerelle](../lab-18-hsrp/)*
