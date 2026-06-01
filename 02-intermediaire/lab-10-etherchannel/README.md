# L-10 — EtherChannel LACP

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 30 min · **Prérequis :** [L-09](../lab-09-stp/)

Agréger les liens redondants entre Switch3 et Switch4 avec le protocole LACP,  
vérifier l'état du bundle et observer le comportement en cas de panne d'un lien.

---

## 🎯 Objectifs

- Comprendre l'intérêt de l'EtherChannel vs liens redondants gérés par STP
- Configurer un bundle LACP (active/active) entre Switch3 et Switch4
- Vérifier l'état du port-channel avec `show etherchannel summary`
- Observer que STP voit le bundle comme un seul lien logique
- Simuler la panne d'un lien membre et vérifier la continuité

---

## 🗺️ Topologie

```
                        [ Router0 ]
                             | Gi0/0 (trunk — ROAS)
                   [ Multilayer Switch0 ]
                  /          |            \
           [ Switch2 ]  [ Switch3 ]══[ Switch4 ]
                |         /  |  \      /  |  \
             Server0   PC3 PC0 PC2  PC4 PC1 PC5

         Switch3 Fa0/22 ════ Fa0/23 Switch4  } Po1 LACP
         Switch3 Fa0/23 ════ Fa0/24 Switch4  }
```

> **Ports asymétriques** — les numéros de ports ne se correspondent pas de chaque côté.  
> C'est parfaitement valide avec LACP : ce qui compte c'est que les deux ports  
> d'un même switch soient dans le même `channel-group` avec la même config trunk.  
> Fa0/24 de Switch3 est déjà utilisé pour le lien vers Switch2.

---

## 📋 EtherChannel LACP

| Switch  | Ports membres   | Port-channel | Mode        |
|---------|-----------------|--------------|-------------|
| Switch3 | Fa0/22 + Fa0/23 | Po1          | LACP active |
| Switch4 | Fa0/23 + Fa0/24 | Po1          | LACP active |

---

## 📝 Travail demandé

### Étape 1 — Ajouter le second câble dans PT

Dans Packet Tracer, ajoute un câble **Copper Cross-Over** entre :
- Switch3 **Fa0/22** → Switch4 **Fa0/23**

Le câble Switch3 Fa0/23 ↔ Switch4 Fa0/24 est déjà en place (lien existant).

### Étape 2 — Configurer EtherChannel sur Switch3

```
configure terminal
interface range fa0/22 - 23
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
end
```

### Étape 3 — Configurer EtherChannel sur Switch4

```
configure terminal
interface range fa0/23 - 24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
end
```

### Étape 4 — Vérifier l'état du bundle

```
show etherchannel summary
```

Résultat attendu sur Switch3 :

```
Group  Port-channel  Protocol  Ports
------+-------------+---------+--------------------
1      Po1(SU)       LACP      Fa0/22(P)  Fa0/23(P)
```

Résultat attendu sur Switch4 :

```
Group  Port-channel  Protocol  Ports
------+-------------+---------+--------------------
1      Po1(SU)       LACP      Fa0/23(P)  Fa0/24(P)
```

| Code | Signification |
|------|---------------|
| `SU` | Layer2, in use — bundle opérationnel |
| `P`  | Port membre actif |
| `D`  | Port down |

### Étape 5 — Vérifier STP avec le bundle

```
show spanning-tree vlan 10
```

STP doit voir **Po1** comme une seule interface — plus les ports physiques séparément.

### Étape 6 — Simuler une panne

Dans PT, déconnecte l'un des deux câbles du bundle.  
Observe que :
- Po1 reste **SU** avec un seul port **P**
- La connectivité est maintenue
- STP ne recompute pas

Reconnecte le câble — Po1 revient à 2 ports P.

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! État du bundle
show etherchannel summary
show etherchannel 1 detail

! Négociation LACP
show lacp neighbor

! STP avec EtherChannel
show spanning-tree vlan 10

! Connectivité
ping 172.16.10.3    (PC0 → PC4 : succès via Po1)
ping 172.16.30.1    (PC0 → PC1 : succès inter-VLAN via ROAS)
```

---

## ✅ Critères de réussite

- [ ] `show etherchannel summary` sur Switch3 — Po1 **SU**, Fa0/22 et Fa0/23 en **P**
- [ ] `show etherchannel summary` sur Switch4 — Po1 **SU**, Fa0/23 et Fa0/24 en **P**
- [ ] `show lacp neighbor` — négociation LACP confirmée des deux côtés
- [ ] `show spanning-tree vlan 10` — **Po1** visible comme interface trunk
- [ ] Ping PC0 → PC4 : **succès**
- [ ] Après suppression d'un câble : ping toujours **succès**

---

## 💡 Points pédagogiques

**Ports asymétriques — pourquoi ça marche ?**  
LACP identifie les ports par leurs adresses MAC système et leurs clés  
d'agrégation, pas par leurs numéros. Ce qui doit être identique des deux côtés :  
configuration trunk, vitesse/duplex, mode LACP.

**EtherChannel vs STP**  
En L-09, STP bloquait l'un des liens redondants — bande passante gaspillée.  
Avec Po1, STP voit un seul lien logique et les deux câbles transportent  
du trafic simultanément.

**LACP active/active**  
Les deux switches initient la négociation. `active/passive` fonctionne aussi.  
`passive/passive` ne fonctionne jamais.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-10-etherchannel.pka` | Fichier activité Packet Tracer |
---

*Lab précédent : [L-09 — STP & bridge priority](../lab-09-stp/)*  
*Lab suivant : [L-11 — SVIs & routage L3 switch](../lab-11-svi-l3/)*
