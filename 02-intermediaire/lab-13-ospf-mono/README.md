# L-13 — OSPF mono-area

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 45 min · **Prérequis :** [L-05](../../01-debutant/lab-05-routage-statique/)

Remplacer le routage statique par OSPF sur une topologie 3 routeurs,  
analyser les voisinages, la LSDB et observer la convergence dynamique.

---

## 🎯 Objectifs

- Supprimer les routes statiques et configurer OSPF area 0
- Analyser les voisinages OSPF avec `show ip ospf neighbor`
- Comprendre l'élection DR/BDR sur un lien multi-accès
- Observer la reconvergence après panne d'un lien
- Lire les routes OSPF dans `show ip route`

---

## 🗺️ Topologie

```
PC0 ── Switch0 ── Router0 ──── Router1 ──── Router2 ── Switch1 ── PC1
    192.168.1.0/24  10.0.0.0/30  10.0.1.0/30  192.168.2.0/24
```

---

## 📋 Plan d'adressage

| Équipement | Interface | Adresse IP       | Masque          |
|------------|-----------|------------------|-----------------|
| PC0        | Fa0       | 192.168.1.1/24   | 255.255.255.0   |
| Router0    | Gi0/0     | 192.168.1.254/24 | 255.255.255.0   |
| Router0    | Gi0/1     | 10.0.0.1/30      | 255.255.255.252 |
| Router1    | Gi0/0     | 10.0.0.2/30      | 255.255.255.252 |
| Router1    | Gi0/1     | 10.0.1.1/30      | 255.255.255.252 |
| Router2    | Gi0/0     | 10.0.1.2/30      | 255.255.255.252 |
| Router2    | Gi0/1     | 192.168.2.254/24 | 255.255.255.0   |
| PC1        | Fa0       | 192.168.2.1/24   | 255.255.255.0   |

---

## 📝 Travail demandé

### Étape 1 — Supprimer les routes statiques

Sur chaque routeur, supprime les routes statiques héritées de lab-05 :

```
configure terminal
no ip route 192.168.2.0 255.255.255.0 10.0.0.2
end
```

Vérifie : `show ip route` — plus de routes **S**.

### Étape 2 — Configurer OSPF sur Router0

```
configure terminal
router ospf 1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
end
```

### Étape 3 — Configurer OSPF sur Router1

```
configure terminal
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
end
```

### Étape 4 — Configurer OSPF sur Router2

```
configure terminal
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.255 area 0
end
```

### Étape 5 — Vérifier les voisinages

```
show ip ospf neighbor
```

Router1 doit afficher 2 voisins (Router0 et Router2) en état **FULL**.

### Étape 6 — Observer la convergence

Déconnecte le câble entre Router0 et Router1 dans PT.  
Observe que `show ip route` sur Router2 se met à jour automatiquement.  
Reconnecte le câble — les routes OSPF reviennent sans intervention.

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Voisinages OSPF
show ip ospf neighbor

! Routes OSPF dans la table
show ip route ospf
show ip route

! Base de données OSPF
show ip ospf database

! Détail d'une interface OSPF
show ip ospf interface gi0/1
```

### Lire show ip route avec OSPF

```
O    192.168.2.0/24 [110/3] via 10.0.0.2
```

| Champ | Signification |
|-------|---------------|
| `O` | Route apprise via OSPF |
| `110` | Distance administrative OSPF |
| `3` | Métrique (coût cumulé) |

### États des voisins OSPF

| État | Signification |
|------|---------------|
| `FULL` | Voisinage établi, LSDB synchronisée |
| `2WAY` | Voisin détecté, pas encore DR/BDR |
| `INIT` | Hello reçu, pas encore bidirectionnel |

---

## ✅ Critères de réussite

- [ ] `show ip route` — plus de routes **S**, présence de routes **O**
- [ ] `show ip ospf neighbor` sur Router1 — 2 voisins en état **FULL**
- [ ] Ping PC0 → PC1 : **succès**
- [ ] Après déconnexion d'un lien : ping toujours **succès** (reconvergence)
- [ ] `show ip ospf database` — LSAs visibles

---

## 💡 Points pédagogiques

**OSPF vs routage statique**  
Le routage statique nécessite une intervention manuelle à chaque changement  
de topologie. OSPF détecte automatiquement les pannes et recalcule les routes.

**Wildcard mask**  
`network 10.0.0.0 0.0.0.3 area 0` — le wildcard est l'inverse du masque :  
255.255.255.252 → 0.0.0.3. Il définit quelles interfaces participent à OSPF.

**Process ID**  
`router ospf 1` — le numéro 1 est local au routeur, il n'a pas besoin  
d'être identique sur tous les routeurs (contrairement à EIGRP AS number).

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-13-ospf-mono.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-12 — DHCP centralisé & ip helper](../lab-12-dhcp-helper/)*  
*Lab suivant : [L-14 — ACL standard](../lab-14-acl-standard/)*
