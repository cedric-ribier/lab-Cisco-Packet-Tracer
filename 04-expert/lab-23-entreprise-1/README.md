# L-23 — Réseau d'entreprise — partie 1

> **Niveau :** 🟣 Expert · **Durée estimée :** 90 min · **Prérequis :** [L-18](../../03-avance/lab-18-hsrp/) · [L-13](../../02-intermediaire/lab-13-ospf-mono/) · [L-12](../../02-intermediaire/lab-12-dhcp-helper/)

Lab de synthèse : construire l'infrastructure interne complète d'une PME  
sur une architecture 3-tier (accès / distribution / cœur), avec redondance  
de passerelle, routage dynamique et DHCP centralisé. La partie 2 (L-24)  
ajoutera la sécurisation du périmètre.

---

## 🎯 Objectifs

- Concevoir et câbler une architecture 3-tier (access / distribution / core)
- Répartir les VLANs par service et les router via des SVIs redondantes (HSRP)
- Interconnecter la distribution et le cœur en OSPF area 0
- Centraliser le DHCP pour tous les VLANs via `ip helper-address`
- Documenter un plan d'adressage cohérent pour l'ensemble du site

---

## 🗺️ Topologie

```
                         [ Core-R1 ]═══[ Core-R2 ]
                          |    OSPF area 0    |
                     [ Dist-A ]           [ Dist-B ]
                     HSRP actif VLAN10/30   HSRP actif VLAN20/40
                        /      \              /      \
                [Acc-1]      [Acc-2]     [Acc-3]    [Acc-4]
                  |             |           |          |
              VLAN10        VLAN30      VLAN20      VLAN40
             (Direction)   (Compta)    (RH)        (IT/Server)
```

> Dist-A et Dist-B forment une paire HSRP (comme en L-18) — chacun actif  
> sur la moitié des VLANs pour répartir la charge. Le cœur (Core-R1/R2)  
> route uniquement entre la distribution et la sortie vers le L-24  
> (NAT/ACL/VPN/QoS, ajoutés dans la partie 2).

---

## 📋 Plan d'adressage — VLANs de service

| VLAN | Nom       | Réseau            | Passerelle HSRP    | Actif  |
|------|-----------|-------------------|----------------------|--------|
| 10   | Direction | 172.20.10.0/24    | 172.20.10.254         | Dist-A |
| 20   | RH        | 172.20.20.0/24    | 172.20.20.254         | Dist-B |
| 30   | Compta    | 172.20.30.0/24    | 172.20.30.254         | Dist-A |
| 40   | IT/Server | 172.20.40.0/24    | 172.20.40.254         | Dist-B |
| 99   | Management| 172.20.99.0/24    | 172.20.99.254         | Dist-A |

---

## 📋 Liens de cœur (point à point, OSPF area 0)

| Lien              | Réseau           |
|--------------------|--------------------|
| Core-R1 ↔ Core-R2  | 10.99.0.0/30       |
| Core-R1 ↔ Dist-A   | 10.99.1.0/30       |
| Core-R2 ↔ Dist-B   | 10.99.2.0/30       |
| Dist-A ↔ Dist-B    | 10.99.3.0/30 (peer-link) |

---

## 📝 Travail demandé

### Étape 1 — Câblage et plan de VLANs

1. Câble les 4 switches d'accès vers leur switch de distribution respectif  
   en trunk (dot1q, `allowed vlan` limité aux VLANs nécessaires par site)
2. Câble Dist-A et Dist-B en peer-link direct (trunk + lien routé pour OSPF)
3. Câble Core-R1/Core-R2 vers leur switch de distribution respectif
4. Câble Core-R1 ↔ Core-R2 pour la redondance du cœur

### Étape 2 — VLANs et SVIs sur Dist-A / Dist-B

Reprends la méthode du **L-11** (SVI) pour créer les VLANs 10/20/30/40/99  
sur les deux switches de distribution, avec les adresses IP physiques  
distinctes par switch (ex. `.252`/`.253` comme en L-18) en plus de  
l'IP virtuelle HSRP.

### Étape 3 — HSRP — répartition de charge

Reprends la méthode du **L-18** :
- VLAN 10 et 30 → actif sur Dist-A, standby sur Dist-B
- VLAN 20 et 40 → actif sur Dist-B, standby sur Dist-A
- `preempt` activé sur les deux switches pour chaque groupe

### Étape 4 — OSPF sur le cœur et la distribution

Sur **Core-R1**, **Core-R2**, **Dist-A**, **Dist-B** :

```
configure terminal
router ospf 1
 router-id <ip-unique-par-routeur>
 network 10.99.0.0 0.0.0.3 area 0
 network 10.99.1.0 0.0.0.3 area 0
 ! ... déclarer chaque lien point à point concerné
end
```

> Les VLANs de service (172.20.x.0/24) ne sont **pas** annoncées dans OSPF  
> — elles sont uniquement routées localement sur Dist-A/Dist-B via les SVIs.  
> Seuls les liens de cœur participent à OSPF pour cette partie 1.

### Étape 5 — DHCP centralisé

Un serveur DHCP (VLAN 40 — IT/Server) distribue les baux pour tous les VLANs.

Sur **Dist-A** et **Dist-B**, ajoute `ip helper-address` sur chaque SVI  
active localement (méthode du **L-12**) :

```
configure terminal
interface vlan 10
 ip helper-address 172.20.40.10
interface vlan 30
 ip helper-address 172.20.40.10
end
```

> VLAN 20 (RH) et VLAN 99 (Management) suivent la même logique sur Dist-B  
> et Dist-A respectivement.

### Étape 6 — Tester la convergence

1. Vérifie que chaque poste de chaque VLAN reçoit un bail DHCP
2. Débranche le lien entre Dist-A et Core-R1 — vérifie qu'OSPF reconverge  
   via Dist-B/Core-R2
3. Simule la panne de Dist-A — vérifie que Dist-B reprend les VLANs 10/30

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! HSRP
show standby brief

! OSPF
show ip ospf neighbor
show ip route ospf

! DHCP
show ip dhcp snooping binding    (si sécurité déjà en place)
ipconfig /renew                  (sur les PCs)

! Vue d'ensemble
show ip interface brief
show vlan brief
```

---

## ✅ Critères de réussite

- [ ] Les 5 VLANs (10/20/30/40/99) sont actifs sur Dist-A et Dist-B avec HSRP
- [ ] `show standby brief` — répartition de charge conforme (VLAN10/30 → Dist-A, VLAN20/40 → Dist-B)
- [ ] `show ip ospf neighbor` — tous les voisins de cœur en **FULL**
- [ ] Chaque poste de chaque VLAN reçoit un bail DHCP cohérent avec son réseau
- [ ] Ping inter-VLAN entre tous les services : **succès**
- [ ] Après panne d'un switch de distribution : bascule HSRP fonctionnelle,  
      services toujours accessibles

---

## 💡 Points pédagogiques

**Pourquoi une architecture 3-tier ?**  
Séparer access / distribution / core limite la portée d'une panne ou  
d'une boucle, facilite le dimensionnement de chaque couche selon son  
rôle (densité de ports en access, routage et redondance en distribution,  
haut débit et simplicité en core) et clarifie la maintenance sur un  
réseau qui grandit.

**Redondance à chaque étage**  
HSRP protège la passerelle des VLANs (distribution), OSPF protège les  
chemins entre distribution et cœur, et le double lien Core-R1/Core-R2  
protège le cœur lui-même. Aucun point unique de défaillance ne doit  
subsister sur les liens de production.

**Ce que la partie 2 (L-24) ajoute**  
Ce lab s'arrête à la frontière du réseau interne. Le périmètre — NAT,  
ACL, VPN vers un site distant et QoS pour la voix — sera traité en L-24  
pour compléter un réseau d'entreprise réaliste de bout en bout.

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-23-entreprise-1.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-22 — QoS bases](../lab-22-qos/)*  
*Lab suivant : [L-24 — Réseau d'entreprise — partie 2](../lab-24-entreprise-2/)*
