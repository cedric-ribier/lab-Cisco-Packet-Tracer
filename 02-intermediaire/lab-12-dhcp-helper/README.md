# L-12 — DHCP centralisé & ip helper

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 30 min · **Prérequis :** [L-11](../lab-11-svi-l3/)

Migrer d'un DHCP distribué sur le routeur vers un serveur DHCP centralisé,  
configurer le relay `ip helper-address` sur les SVIs du 3560 et vérifier  
le renouvellement des baux depuis les postes clients.

---

## 🎯 Objectifs

- Désactiver les pools DHCP existants sur Router0
- Configurer les étendues DHCP sur Server0 (un pool par VLAN)
- Configurer `ip helper-address` sur chaque SVI du 3560
- Vérifier l'attribution automatique avec `ipconfig /release` et `ipconfig /renew`
- Comprendre le flux du relay DHCP (broadcast → unicast → relay)

---

## 🗺️ Topologie

```
                        [ Router0 ]
                             |
                   [ Multilayer Switch0 ]  ← ip helper-address sur SVIs
                  /          |            \
           [ Switch2 ]  [ Switch3 ]══[ Switch4 ]
                |         /  |  \      /  |  \
            [Server0]   PC3 PC0 PC2  PC4 PC1 PC5
            VLAN 20
            172.16.20.10
```

> Server0 est le serveur DHCP centralisé pour tous les VLANs.  
> Le 3560 relaie les requêtes DHCP des VLANs 10 et 30 vers Server0 via ip helper-address.  
> Le VLAN 20 n'a pas besoin de relay — Server0 est dans ce VLAN.

---

## 📋 Pools DHCP sur Server0

| Pool   | Réseau          | Plage distribuée    | Passerelle      | DNS     |
|--------|-----------------|---------------------|-----------------|---------|
| VLAN10 | 172.16.10.0/24  | .11 → .200          | 172.16.10.254   | 8.8.8.8 |
| VLAN30 | 172.16.30.0/24  | .11 → .200          | 172.16.30.254   | 8.8.8.8 |

> Server0 a une IP fixe (172.16.20.10) — ne pas le passer en DHCP.  
> Les adresses .1 → .10 sont réservées aux équipements fixes (PCs statiques, passerelles).

---

## 📋 ip helper-address sur les SVIs

| SVI        | ip helper-address |
|------------|-------------------|
| Vlan10     | 172.16.20.10      |
| Vlan30     | 172.16.20.10      |
| Vlan20     | — (pas de relay)  |

---

## 📝 Travail demandé

### Étape 1 — Désactiver le DHCP sur Router0

```
configure terminal
no ip dhcp pool LAN1
no ip dhcp pool LAN2
no ip dhcp excluded-address 192.168.1.1 192.168.1.10
no ip dhcp excluded-address 192.168.1.254
no ip dhcp excluded-address 192.168.2.1 192.168.2.10
no ip dhcp excluded-address 192.168.2.254
end
```

Vérifie que les pools sont supprimés :
```
show ip dhcp pool
```

### Étape 2 — Configurer DHCP sur Server0

Sur **Server0 → Services → DHCP** :

**Pool VLAN10 :**
- Pool Name : `VLAN10`
- Default Gateway : `172.16.10.254`
- DNS Server : `8.8.8.8`
- Start IP Address : `172.16.10.11`
- Subnet Mask : `255.255.255.0`
- Maximum Number of Users : `190`

**Pool VLAN30 :**
- Pool Name : `VLAN30`
- Default Gateway : `172.16.30.254`
- DNS Server : `8.8.8.8`
- Start IP Address : `172.16.30.11`
- Subnet Mask : `255.255.255.0`
- Maximum Number of Users : `190`

Active le service DHCP : **Service → ON**

### Étape 3 — Configurer ip helper-address sur les SVIs

Sur **Multilayer Switch0** :

```
configure terminal
interface vlan 10
 ip helper-address 172.16.20.10
interface vlan 30
 ip helper-address 172.16.20.10
end
```

### Étape 4 — Renouveler les baux sur les PCs

Sur chaque PC (VLAN 10 et 30) : **Desktop → Command Prompt**

```
ipconfig /release
ipconfig /renew
ipconfig
```

Vérifie que chaque PC reçoit :
- Une IP dans la bonne plage
- La bonne passerelle
- Le DNS 8.8.8.8

### Étape 5 — Vérifier les baux sur Server0

Sur **Server0 → Services → DHCP** — onglet **Binding** :  
Les baux actifs doivent apparaître avec l'IP attribuée et l'adresse MAC du client.

### Étape 6 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Sur le 3560 — vérifier le relay
show running-config | include helper

! Sur les PCs
ipconfig /release
ipconfig /renew
ipconfig

! Connectivité après renouvellement
ping 172.16.30.x    (PC0 → PC1 via DHCP)
ping 172.16.20.10   (PC0 → Server0)
```

---

## ✅ Critères de réussite

- [ ] `show ip dhcp pool` sur Router0 — aucun pool (supprimés)
- [ ] Server0 DHCP service — **ON**, 2 pools configurés
- [ ] PC0 reçoit une IP en `172.16.10.x` (x ≥ 11) après `ipconfig /renew`
- [ ] PC1 reçoit une IP en `172.16.30.x` (x ≥ 11) après `ipconfig /renew`
- [ ] Passerelle et DNS corrects sur chaque PC
- [ ] Server0 DHCP Binding — baux actifs visibles
- [ ] Ping PC0 → PC1 : **succès**

---

## 💡 Points pédagogiques

**Pourquoi un relay DHCP ?**  
DHCP utilise des broadcasts (255.255.255.255) qui ne traversent pas les routeurs  
ni les SVIs par défaut. `ip helper-address` convertit ce broadcast en unicast  
vers l'IP du serveur DHCP — le message traverse ainsi les frontières L3.

**Flux du relay :**
```
PC0 (VLAN10) → broadcast DHCP Discover
→ SVI Vlan10 reçoit le broadcast
→ ip helper-address 172.16.20.10
→ unicast vers Server0 (172.16.20.10)
→ Server0 répond en unicast via le 3560
→ PC0 reçoit son IP
```

**VLAN 20 sans relay**  
Server0 est dans le VLAN 20 — il reçoit directement les broadcasts de ce VLAN.  
Un relay n'est nécessaire que pour les VLANs distants du serveur DHCP.

**Centralisation vs distribution**  
Un serveur DHCP centralisé simplifie la gestion (un seul endroit pour les baux,  
les exclusions, les options). Le DHCP sur routeur (L-06) reste utile pour  
les petits sites sans serveur dédié.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-12-dhcp-helper.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-11 — SVIs & routage L3 switch](../lab-11-svi-l3/)*  
*Lab suivant : [L-13 — OSPF mono-area](../lab-13-ospf-mono/)*
