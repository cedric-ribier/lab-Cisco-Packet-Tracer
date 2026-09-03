# L-17 — OSPF multi-area

> **Niveau :** 🟠 Avancé · **Durée estimée :** 45 min · **Prérequis :** [L-13](../../02-intermediaire/lab-13-ospf-mono/)

Structurer un réseau OSPF en plusieurs areas autour du backbone, configurer les ABR,  
observer les routes inter-area et vérifier la tolérance aux pannes grâce à un lien  
redondant sur le backbone.

---

## 🎯 Objectifs

- Comprendre la hiérarchie OSPF : backbone area 0 + areas secondaires
- Configurer les ABR (Area Border Routers)
- Distinguer les routes intra-area (O) et inter-area (O IA)
- Réduire la taille de la LSDB par area
- Configurer un résumé de routes sur un ABR
- Observer la reconvergence automatique d'OSPF après la panne d'un lien du backbone

---

## 🗺️ Topologie

```
   Area 1                        Area 0 (backbone)                     Area 2
192.168.1.0/24                                                      192.168.2.0/24
10.1.0.0/24 (Lo1 sur R0)    10.0.0.0/30        10.0.1.0/30

   PC0 ── R0 ──────────────────────── R1 ──────────────────────── R2 ── PC1
           │        ABR                   transit                  ABR │
           └───────────────────────── 10.0.2.0/30 ──────────────────────┘
                          lien redondant (secours) — Area 0
```

> Le backbone forme un triangle R0 – R1 – R2 : le chemin normal passe par R1,  
> mais le lien direct R0 – R2 fournit un second chemin si R1 tombe en panne  
> ou si l'un des deux liens vers R1 est coupé. C'est ce qui permet de démontrer  
> l'intérêt réel d'un protocole de routage dynamique : reconvergence automatique,  
> sans intervention manuelle, contrairement au routage statique.
>
> `Lo1` sur R0 est une interface **loopback** (virtuelle, toujours up) — elle simule  
> un second segment LAN de l'Area 1 sans nécessiter de câblage ni de PC supplémentaire,  
> uniquement pour illustrer le résumé de routes à l'étape 5.

---

## 📋 Plan d'adressage

| Équipement | Interface   | Adresse IP       | Masque          | Area   | Rôle                          |
|------------|-------------|------------------|-----------------|--------|-------------------------------|
| PC0        | Fa0         | 192.168.1.1/24   | 255.255.255.0   | Area 1 | Poste client                  |
| R0         | Gi0/0       | 192.168.1.254/24 | 255.255.255.0   | Area 1 | Passerelle PC0                |
| R0         | Loopback1   | 10.1.0.1/24      | 255.255.255.0   | Area 1 | 2ᵉ réseau Area 1 (virtuel)     |
| R0         | Gi0/1       | 10.0.0.1/30      | 255.255.255.252 | Area 0 | Lien vers R1                  |
| R0         | Gi0/2       | 10.0.2.1/30      | 255.255.255.252 | Area 0 | Lien redondant vers R2        |
| R1         | Gi0/0       | 10.0.0.2/30      | 255.255.255.252 | Area 0 | Lien vers R0                  |
| R1         | Gi0/1       | 10.0.1.1/30      | 255.255.255.252 | Area 0 | Lien vers R2                  |
| R2         | Gi0/0       | 10.0.1.2/30      | 255.255.255.252 | Area 0 | Lien vers R1                  |
| R2         | Gi0/2       | 10.0.2.2/30      | 255.255.255.252 | Area 0 | Lien redondant vers R0        |
| R2         | Gi0/1       | 192.168.2.254/24 | 255.255.255.0   | Area 2 | Passerelle PC1                |
| PC1        | Fa0         | 192.168.2.1/24   | 255.255.255.0   | Area 2 | Poste client                  |

> Chaque interface a un rôle concret : accès client, lien backbone principal ou lien  
> backbone de secours. Il n'y a plus d'interface déclarée sans appareil en face.

---

## 📝 Travail demandé

### Étape 1 — OSPF sur R0 (ABR Area1/Area0)

```
configure terminal
interface loopback 1
 ip address 10.1.0.1 255.255.255.0
router ospf 1
 network 192.168.1.0 0.0.0.255 area 1
 network 10.1.0.0 0.0.0.255 area 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
end
```

### Étape 2 — OSPF sur R1 (backbone uniquement)

```
configure terminal
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
end
```

### Étape 3 — OSPF sur R2 (ABR Area0/Area2)

```
configure terminal
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.255 area 2
end
```

### Étape 4 — Vérifier les voisinages et les routes

```
show ip ospf neighbor
show ip route ospf
```

Identifie les routes **O** (intra-area) et **O IA** (inter-area). R0 doit avoir  
deux voisins OSPF (R1 et R2 directement), de même pour R2.

### Étape 5 — Résumé de routes sur R0 (ABR)

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

### Étape 6 — Tester la tolérance aux pannes du backbone

Dans Packet Tracer, coupe le lien **R0 – R1** (débranche le câble ou fais  
`shutdown` sur l'interface Gi0/1 de R0).

Lance un ping continu depuis PC0 :
```
ping 192.168.2.1 -t
```

Après une brève interruption (temps de convergence OSPF), le ping doit reprendre  
automatiquement — le trafic passe désormais par le lien redondant **R0 – R2**.  
Vérifie sur R0 :
```
show ip ospf neighbor
show ip route
```
R0 n'a plus qu'un seul voisin direct (R2), mais conserve une route complète  
vers Area 0 et Area 2 via ce nouveau chemin — sans aucune intervention manuelle.

Reconnecte le lien R0 – R1 et vérifie qu'OSPF reprend le chemin initial.

### Étape 7 — Sauvegarder

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

! Détail d'une interface OSPF
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
- [ ] Ping PC0 → PC1 : **succès**
- [ ] Après résumé sur R0 : `show ip route` sur R1 montre les préfixes résumés
- [ ] Après coupure du lien R0 – R1 : ping PC0 → PC1 **toujours réussi**, via le lien redondant R0 – R2
- [ ] Après reconnexion : `show ip ospf neighbor` sur R0 montre à nouveau 2 voisins

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

**Pourquoi un lien redondant sur le backbone ?**  
Une topologie en chaîne (R0 – R1 – R2) n'offre qu'un seul chemin : la panne  
d'un lien coupe toute communication entre les areas, quel que soit le protocole  
de routage utilisé. C'est précisément ce qu'un IGP dynamique comme OSPF sait  
exploiter quand une alternative existe physiquement : il détecte la panne,  
recalcule le plus court chemin restant (algorithme SPF) et bascule le trafic  
automatiquement — sans reconfiguration manuelle, contrairement au routage statique.  
Sans second lien, il n'y aurait simplement rien à recalculer.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-17-ospf-multi.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-16 — ACL étendues & DMZ](../../02-intermediaire/lab-16-acl-etendues/)*  
*Lab suivant : [L-18 — HSRP & redondance passerelle](../lab-18-hsrp/)*
