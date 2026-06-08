# L-17 — OSPF multi-area

> **Niveau :** 🟠 Avancé · **Durée estimée :** 45 min · **Prérequis :** [L-13](../../02-intermediaire/lab-13-ospf-mono/)

Structurer un réseau OSPF en plusieurs areas autour du backbone,  
configurer les ABR et observer les routes inter-area dans la table de routage.

---

## 🎯 Objectifs

- Comprendre la hiérarchie OSPF : backbone area 0 + areas secondaires
- Configurer les ABR (Area Border Routers)
- Distinguer les routes intra-area (O) et inter-area (O IA)
- Réduire la taille de la LSDB par area
- Configurer un résumé de routes sur un ABR

---

## 🗺️ Topologie

```
Area 1                  Area 0                  Area 2
192.168.1.0/24          10.0.0.0/30             192.168.2.0/24
10.1.0.0/24             10.0.1.0/30             10.2.0.0/24

PC0 ── R0 ──────────── R1 ─────────────── R2 ── PC1
       ABR          Backbone             ABR
    (Area1/Area0)                    (Area0/Area2)
```

---

## 📋 Plan d'adressage

| Équipement | Interface | Adresse IP       | Masque          | Area   |
|------------|-----------|------------------|-----------------|--------|
| PC0        | Fa0       | 192.168.1.1/24   | 255.255.255.0   | Area 1 |
| R0         | Gi0/0     | 192.168.1.254/24 | 255.255.255.0   | Area 1 |
| R0         | Gi0/1     | 10.0.0.1/30      | 255.255.255.252 | Area 0 |
| R0         | Gi0/2     | 10.1.0.1/24      | 255.255.255.0   | Area 1 |
| R1         | Gi0/0     | 10.0.0.2/30      | 255.255.255.252 | Area 0 |
| R1         | Gi0/1     | 10.0.1.1/30      | 255.255.255.252 | Area 0 |
| R2         | Gi0/0     | 10.0.1.2/30      | 255.255.255.252 | Area 0 |
| R2         | Gi0/1     | 192.168.2.254/24 | 255.255.255.0   | Area 2 |
| R2         | Gi0/2     | 10.2.0.1/24      | 255.255.255.0   | Area 2 |
| PC1        | Fa0       | 192.168.2.1/24   | 255.255.255.0   | Area 2 |

---

## 📝 Travail demandé

### Étape 1 — OSPF sur R0 (ABR Area1/Area0)

```
configure terminal
router ospf 1
 network 192.168.1.0 0.0.0.255 area 1
 network 10.1.0.0 0.0.0.255 area 1
 network 10.0.0.0 0.0.0.3 area 0
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
 network 192.168.2.0 0.0.0.255 area 2
 network 10.2.0.0 0.0.0.255 area 2
end
```

### Étape 4 — Vérifier les voisinages et les routes

```
show ip ospf neighbor
show ip route ospf
```

Identifie les routes **O** (intra-area) et **O IA** (inter-area).

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

### Étape 6 — Sauvegarder

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

- [ ] `show ip ospf neighbor` — tous les voisins en état **FULL**
- [ ] `show ip route` sur R1 — routes **O IA** vers Area 1 et Area 2
- [ ] `show ip ospf border-routers` — R0 et R2 identifiés comme ABR
- [ ] Ping PC0 → PC1 : **succès**
- [ ] Après résumé sur R0 : `show ip route` sur R1 montre les préfixes résumés

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

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-17-ospf-multi.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-16 — ACL étendues & DMZ](../../02-intermediaire/lab-16-acl-etendues/)*  
*Lab suivant : [L-18 — HSRP & redondance passerelle](../lab-18-hsrp/)*
