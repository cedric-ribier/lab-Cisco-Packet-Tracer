# L-21 — BGP bases (eBGP)

> **Niveau :** 🟠 Avancé · **Durée estimée :** 60 min · **Prérequis :** [L-17](../lab-17-ospf-multi/)

Établir une session eBGP entre deux systèmes autonomes distincts,  
annoncer des préfixes et analyser les attributs de chemin AS-PATH.

---

## 🎯 Objectifs

- Comprendre la différence entre routage IGP (OSPF) et EGP (BGP)
- Configurer une session eBGP entre deux AS
- Annoncer des réseaux avec `network`
- Analyser l'attribut `AS-PATH` et son rôle dans la sélection de route
- Vérifier l'état des voisinages avec `show bgp summary`

---

## 🗺️ Topologie

```
AS 65001                          AS 65002
192.168.1.0/24                    192.168.2.0/24
    |                                  |
  R1 (interne, OSPF area 0) ── eBGP ── R2 (interne, OSPF area 0)
    |                                  |
10.0.0.0/30 (lien eBGP)
```

> R1 représente la frontière de l'AS 65001 (ex. un client), R2 la frontière  
> de l'AS 65002 (ex. un fournisseur ou un site partenaire). Chaque AS a son  
> propre réseau interne, indépendant de l'autre — BGP n'annonce que les  
> préfixes explicitement déclarés, contrairement à OSPF qui annonce tout  
> ce qui est directement connecté dans le process.

---

## 📋 Plan d'adressage

| Équipement | Interface | Adresse IP        | AS     |
|------------|-----------|-------------------|--------|
| PC0        | Fa0       | 192.168.1.1/24    | 65001  |
| R1         | Gi0/0     | 192.168.1.254/24  | 65001  |
| R1         | Gi0/1     | 10.0.0.1/30       | 65001  |
| R2         | Gi0/0     | 10.0.0.2/30       | 65002  |
| R2         | Gi0/1     | 192.168.2.254/24  | 65002  |
| PC1        | Fa0       | 192.168.2.1/24    | 65002  |

---

## 📝 Travail demandé

### Étape 1 — Configurer les interfaces et la connectivité de base

```
! Sur R1
configure terminal
interface gi0/0
 ip address 192.168.1.254 255.255.255.0
 no shutdown
interface gi0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
end
```

```
! Sur R2
configure terminal
interface gi0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
interface gi0/1
 ip address 192.168.2.254 255.255.255.0
 no shutdown
end
```

Vérifie la connectivité directe :
```
ping 10.0.0.2    (depuis R1)
```

### Étape 2 — Configurer eBGP sur R1

```
configure terminal
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 network 192.168.1.0 mask 255.255.255.0
end
```

### Étape 3 — Configurer eBGP sur R2

```
configure terminal
router bgp 65002
 neighbor 10.0.0.1 remote-as 65001
 network 192.168.2.0 mask 255.255.255.0
end
```

> **`network`** ne "crée" pas de route — le préfixe doit déjà exister dans  
> la table de routage (via une interface directement connectée, une route  
> statique, ou un IGP) pour que BGP l'annonce au voisin.

### Étape 4 — Vérifier le voisinage BGP

```
show bgp summary
```

Ou selon la version IOS :
```
show ip bgp summary
```

L'état doit passer de `Active`/`Idle` à un nombre (nombre de préfixes reçus),  
indiquant une session **Established**.

### Étape 5 — Analyser la table BGP

```
show ip bgp
```

```
   Network            Next Hop        Metric  LocPrf  Weight  Path
*> 192.168.1.0/24     0.0.0.0              0           0      32768  i
*> 192.168.2.0/24     10.0.0.2             0           0          0  65002 i
```

Identifie l'attribut **AS-PATH** (`65002`) sur la route apprise depuis R2.

### Étape 6 — Vérifier la table de routage

```
show ip route bgp
```

```
B    192.168.2.0/24 [20/0] via 10.0.0.2
```

### Étape 7 — Tester la connectivité bout en bout

```
ping 192.168.2.1    (depuis PC0)
```

### Étape 8 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Résumé des voisinages BGP
show bgp summary
show ip bgp summary

! Table BGP complète
show ip bgp

! Routes BGP dans la table de routage
show ip route bgp

! Détail d'un préfixe
show ip bgp 192.168.2.0
```

### Lire show ip route avec BGP

```
B    192.168.2.0/24 [20/0] via 10.0.0.2
```

| Champ | Signification |
|-------|-----------------|
| `B` | Route apprise via BGP |
| `20` | Distance administrative eBGP (par défaut) |
| `[20/0]` | Distance administrative / métrique (souvent 0 en BGP) |

### Codes de statut show ip bgp

| Symbole | Signification |
|---------|-----------------|
| `*` | Route valide |
| `>` | Route meilleure (best path), installée dans la table de routage |
| `i` | Origine IGP (annoncée via `network`) |

---

## ✅ Critères de réussite

- [ ] `show bgp summary` — voisin 10.0.0.2 à l'état **Established** (compteur de préfixes ≠ Idle/Active)
- [ ] `show ip bgp` — préfixe 192.168.2.0/24 visible avec AS-PATH `65002`
- [ ] `show ip route bgp` — route **B** vers 192.168.2.0/24
- [ ] Ping PC0 → PC1 : **succès**
- [ ] `show ip bgp` sur R2 — préfixe 192.168.1.0/24 visible avec AS-PATH `65001`

---

## 💡 Points pédagogiques

**IGP vs EGP**  
OSPF (IGP) route **à l'intérieur** d'un AS et fait confiance à tous ses  
voisins. BGP (EGP) route **entre** AS distincts, appartenant potentiellement  
à des organisations différentes — la confiance et le contrôle du trafic  
échangé sont donc beaucoup plus stricts et explicites.

**AS-PATH**  
Chaque AS traversé ajoute son numéro à l'attribut AS-PATH. Cet attribut  
sert à la fois à choisir la meilleure route (chemin le plus court en  
nombre d'AS, par défaut) et à détecter les boucles : un routeur BGP  
rejette une annonce contenant déjà son propre numéro d'AS.

**Distance administrative eBGP (20) vs iBGP (200)**  
Une route eBGP (entre AS différents) est considérée plus fiable qu'une  
route iBGP (à l'intérieur du même AS) par défaut — reflet du fait  
qu'eBGP est généralement directement connecté à la frontière du réseau.

**`network` en BGP ≠ `network` en OSPF**  
En OSPF, `network` active le protocole sur une interface. En BGP,  
`network` déclare un préfixe à annoncer — le préfixe doit déjà être  
présent dans la table de routage locale, sinon l'annonce échoue silencieusement.

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-21-bgp.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-20 — VPN site-à-site IPsec](../lab-20-vpn-ipsec/)*  
*Niveau suivant : [04-expert/](../../04-expert/) — QoS & réseau d'entreprise complet*
