# L-11 — SVIs & routage L3 switch

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 30 min · **Prérequis :** [L-08](../../01-debutant/lab-08-roas/) · [L-10](../lab-10-etherchannel/)

Remplacer le Router-on-a-stick par des SVIs sur le 3560, comparer les deux  
approches et comprendre quand choisir l'une ou l'autre en entreprise.

---

## 🎯 Objectifs

- Activer `ip routing` sur le Multilayer Switch0
- Créer les SVIs (interfaces VLAN) comme passerelles inter-VLAN
- Supprimer le ROAS sur Router0
- Comparer ROAS vs SVI : chemin, latence, cas d'usage
- Vérifier le routage via `show ip route` sur le 3560

---

## 🗺️ Topologie

```
                        [ Router0 ]
                             | Gi0/0 (trunk — désactivé)
                   [ Multilayer Switch0 ]  ← ip routing activé
                  /          |            \
           [ Switch2 ]  [ Switch3 ]══[ Switch4 ]
                |         /  |  \      /  |  \
             Server0   PC3 PC0 PC2  PC4 PC1 PC5
```

> Point de départ : solution lab-10 (EtherChannel Po1 en place).  
> Router0 reste câblé mais ses sous-interfaces seront supprimées —  
> le routage inter-VLAN sera assuré directement par le 3560.

---

## 📋 SVIs à créer sur Multilayer Switch0

| Interface SVI  | Adresse IP       | VLAN | Rôle              |
|----------------|------------------|------|-------------------|
| interface vlan 10 | 172.16.10.254/24 | 10   | Passerelle Compta |
| interface vlan 20 | 172.16.20.254/24 | 20   | Passerelle Server |
| interface vlan 30 | 172.16.30.254/24 | 30   | Passerelle DG     |

---

## 📝 Travail demandé

### Étape 1 — Supprimer le ROAS sur Router0

```
configure terminal
no interface gi0/0.10
no interface gi0/0.20
no interface gi0/0.30
end
```

Vérifie que le routage inter-VLAN ne fonctionne plus :
```
ping 172.16.30.1    (PC0 → PC1 : échec attendu)
```

### Étape 2 — Activer ip routing sur le 3560

```
configure terminal
ip routing
end
```

### Étape 3 — Créer les SVIs

```
configure terminal
interface vlan 10
 ip address 172.16.10.254 255.255.255.0
 no shutdown
interface vlan 20
 ip address 172.16.20.254 255.255.255.0
 no shutdown
interface vlan 30
 ip address 172.16.30.254 255.255.255.0
 no shutdown
end
```

### Étape 4 — Vérifier la table de routage du 3560

```
show ip route
```

Tu dois voir 3 routes **C** (connected) — une par SVI :

```
C    172.16.10.0/24  is directly connected, Vlan10
C    172.16.20.0/24  is directly connected, Vlan20
C    172.16.30.0/24  is directly connected, Vlan30
```

### Étape 5 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Routing activé sur le 3560
show ip routing

! Table de routage
show ip route

! État des SVIs
show interfaces vlan 10
show ip interface brief

! Connectivité inter-VLAN
ping 172.16.30.1    (PC0 VLAN10 → PC1 VLAN30 : succès)
ping 172.16.20.10   (PC0 VLAN10 → Server0 VLAN20 : succès)
tracert 172.16.30.1 (1 seul saut — pas de passage par Router0)
```

---

## ✅ Critères de réussite

- [ ] `show ip routing` sur 3560 — routing activé
- [ ] `show ip route` — 3 routes **C** visibles (Vlan10, Vlan20, Vlan30)
- [ ] `show ip interface brief` — Vlan10, Vlan20, Vlan30 en **up/up**
- [ ] Ping PC0 → PC1 : **succès** (VLAN 10 → VLAN 30)
- [ ] Ping PC0 → Server0 : **succès** (VLAN 10 → VLAN 20)
- [ ] Traceroute PC0 → PC1 : **1 seul saut** (routage local sur le 3560)

---

## 💡 ROAS vs SVI — comparaison

| Critère | ROAS (L-08) | SVI (L-11) |
|---------|-------------|------------|
| Équipement | Routeur dédié | Switch L3 |
| Chemin du trafic | PC → Switch → Routeur → Switch → PC | PC → Switch → PC |
| Latence | Plus élevée (2 traversées switch) | Plus faible (routage local) |
| Lien physique | 1 lien trunk dédié vers le routeur | Aucun lien supplémentaire |
| Goulot d'étranglement | Oui (tout passe par 1 lien) | Non |
| Coût | Routeur supplémentaire | Switch L3 suffisant |
| Quand l'utiliser | Réseau simple, routeur déjà en place | Infrastructure de commutation L3 |

> **Règle pratique** : en entreprise avec un switch L3 disponible, on préfère les SVIs.  
> Le ROAS reste utile quand on n'a pas de switch L3 ou qu'on veut appliquer  
> des politiques de routage avancées sur le routeur (QoS, ACL, NAT).

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-11-svi-l3.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-10 — EtherChannel LACP](../lab-10-etherchannel/)*  
*Lab suivant : [L-12 — DHCP centralisé & ip helper](../lab-12-dhcp-helper/)*
