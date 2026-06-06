# L-14 — ACL standard

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 30 min · **Prérequis :** [L-05](../../01-debutant/lab-05-routage-statique/)

Créer des ACL standard pour filtrer le trafic par adresse source,  
les placer sur la bonne interface dans le bon sens et vérifier leur effet.

---

## 🎯 Objectifs

- Créer des ACL standard numérotées et nommées
- Comprendre la règle de placement : **au plus près de la destination**
- Appliquer les ACL en entrée (`in`) ou sortie (`out`) sur une interface
- Vérifier avec `show access-lists` et les compteurs de hits
- Tester les effets du filtrage depuis les PCs

---

## 🗺️ Topologie

```
PC0 ── Switch0 ── Router0 ──── Router1 ──── Router2 ── Switch1 ── PC1
    192.168.1.0/24  10.0.0.0/30  10.0.1.0/30  192.168.2.0/24
                                                         |
                                                       Server
                                                   192.168.2.10
```

> Point de départ : solution lab-13 (OSPF configuré).  
> Objectif : bloquer PC0 (192.168.1.1) d'accéder au réseau 192.168.2.0/24  
> tout en laissant passer le reste du trafic.

---

## 📋 Plan de filtrage

| ACL | Type | Règle | Placement |
|-----|------|-------|-----------|
| BLOCK_PC0 | Standard nommée | deny 192.168.1.1, permit any | Router2 Gi0/1 out |

> **Règle d'or des ACL standard** : les placer **au plus près de la destination**.  
> Une ACL standard filtre uniquement sur l'adresse source — si on la place  
> trop tôt, on bloque aussi l'accès aux réseaux intermédiaires.

---

## 📝 Travail demandé

### Étape 1 — Observer le trafic avant filtrage

```
ping 192.168.2.1    (PC0 → PC1 : succès)
ping 192.168.2.10   (PC0 → Server : succès)
```

Note les résultats — tu compareras après application de l'ACL.

### Étape 2 — Créer l'ACL nommée sur Router2

```
configure terminal
ip access-list standard BLOCK_PC0
 deny host 192.168.1.1
 permit any
end
```

> `deny host 192.168.1.1` = deny 192.168.1.1 0.0.0.0 (hôte exact).  
> Sans `permit any`, tout le reste serait aussi bloqué (implicit deny).

### Étape 3 — Appliquer l'ACL sur l'interface

```
configure terminal
interface gi0/1
 ip access-group BLOCK_PC0 out
end
```

### Étape 4 — Tester le filtrage

Depuis **PC0** :
```
ping 192.168.2.1     → échec attendu (bloqué par ACL)
ping 192.168.2.10    → échec attendu (bloqué par ACL)
```

Depuis **PC1** (192.168.2.1) :
```
ping 192.168.1.1     → succès (ACL filtre uniquement le trafic sortant vers 192.168.2.x)
```

### Étape 5 — Vérifier les compteurs

```
show access-lists
```

Les compteurs de hits s'incrémentent à chaque paquet correspondant à une règle :

```
Standard IP access list BLOCK_PC0
    10 deny host 192.168.1.1 (X matches)
    20 permit any (Y matches)
```

### Étape 6 — Tester avec une ACL numérotée (comparaison)

```
configure terminal
access-list 10 deny host 192.168.1.1
access-list 10 permit any
end
```

> Une ACL numérotée standard utilise les numéros **1-99** ou **1300-1999**.  
> L'ACL nommée est préférée en production — plus lisible, modifiable ligne par ligne.

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Voir les ACL et leurs compteurs
show access-lists
show ip access-lists

! Voir les ACL appliquées sur une interface
show ip interface gi0/1

! Réinitialiser les compteurs
clear ip access-list counters BLOCK_PC0
```

---

## ✅ Critères de réussite

- [ ] `show access-lists` — ACL BLOCK_PC0 présente avec règles deny et permit
- [ ] `show ip interface gi0/1` — BLOCK_PC0 appliquée **outbound**
- [ ] Ping PC0 → PC1 : **échec** (bloqué)
- [ ] Ping PC0 → PC1 depuis PC1 vers PC0 : **succès** (ACL unidirectionnelle)
- [ ] Compteurs de hits incrémentés après les pings

---

## 💡 Points pédagogiques

**ACL standard — filtrage par source uniquement**  
Une ACL standard ne regarde que l'adresse IP source.  
Pour filtrer par destination, protocole ou port → ACL étendue (L-16).

**Placement au plus près de la destination**  
Si on applique BLOCK_PC0 sur Router0 Gi0/1 (out), PC0 ne peut plus  
joindre Router1 non plus — trop restrictif.  
Sur Router2 Gi0/1 (out), on bloque uniquement l'accès au réseau 192.168.2.0/24.

**Implicit deny**  
Toute ACL se termine par un `deny any` implicite invisible.  
Sans `permit any` explicite, tout le trafic non matché est bloqué.

**ACL nommée vs numérotée**  
Nommée : modifiable ligne par ligne, plus lisible.  
Numérotée : doit être entièrement réécrite pour modification.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-14-acl-standard.pka` | Fichier activité Packet Tracer |
---

*Lab précédent : [L-13 — OSPF mono-area](../lab-13-ospf-mono/)*  
*Lab suivant : [L-15 — NAT & PAT](../lab-15-nat-pat/)*
