# L-20 — VPN site-à-site IPsec

> **Niveau :** 🟠 Avancé · **Durée estimée :** 60 min · **Prérequis :** [L-16](../../02-intermediaire/lab-16-acl-etendues/)

Établir un tunnel VPN IPsec IKEv1 site-à-site entre deux routeurs distants  
pour chiffrer le trafic entre deux réseaux privés à travers Internet.

---

## 🎯 Objectifs

- Comprendre le rôle d'un VPN site-à-site face à des liaisons non chiffrées
- Configurer la phase 1 (ISAKMP/IKE) : policy et clé pré-partagée
- Configurer la phase 2 (IPsec) : transform-set et crypto map
- Définir le trafic intéressant avec une ACL étendue
- Vérifier l'établissement du tunnel et tester le chiffrement

---

## 🗺️ Topologie

```
[LAN Site A]                                          [LAN Site B]
192.168.1.0/24                                        192.168.2.0/24
     |                                                      |
  Router-A ──── 203.0.113.1 ═══ Internet ═══ 203.0.113.2 ──── Router-B
   (inside)         (outside, public)      (outside, public)   (inside)
```

> Point de départ : solution lab-16 (NAT + ACL déjà en place sur les deux  
> sites). Le tunnel IPsec doit chiffrer uniquement le trafic entre les  
> deux LANs — le reste du trafic (accès Internet normal) continue de  
> passer en clair via NAT/PAT.

---

## 📋 Plan d'adressage

| Équipement | Interface | Adresse IP        | Rôle           |
|------------|-----------|-------------------|-----------------|
| PC-A       | Fa0       | 192.168.1.1/24    | LAN Site A      |
| Router-A   | Gi0/0     | 192.168.1.254/24  | inside Site A   |
| Router-A   | Gi0/1     | 203.0.113.1/24    | outside Site A  |
| Router-B   | Gi0/0     | 192.168.2.254/24  | inside Site B   |
| Router-B   | Gi0/1     | 203.0.113.2/24    | outside Site B  |
| PC-B       | Fa0       | 192.168.2.1/24    | LAN Site B      |

> 203.0.113.0/24 simule Internet — un réseau intermédiaire directement  
> connecté aux deux routeurs pour simplifier la maquette PT.

---

## 📝 Travail demandé

### Étape 1 — Définir le trafic intéressant (ACL cryptée)

Sur **Router-A** :
```
configure terminal
ip access-list extended VPN-TRAFFIC
 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
end
```

Sur **Router-B** (miroir exact — sens inversé) :
```
configure terminal
ip access-list extended VPN-TRAFFIC
 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
end
```

> Seul le trafic entre les deux LANs correspond à cette ACL — c'est ce  
> trafic-là, et uniquement lui, qui sera chiffré et encapsulé dans le tunnel.

### Étape 2 — Phase 1 — ISAKMP (IKEv1)

Sur **Router-A** et **Router-B** (configuration identique) :
```
configure terminal
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
crypto isakmp key VpnSiteKey123! address 203.0.113.2   ! (sur Router-A, IP du pair)
end
```

> Sur Router-B, la clé pointe vers l'IP publique de Router-A : `address 203.0.113.1`.  
> La clé pré-partagée doit être **strictement identique** des deux côtés.

### Étape 3 — Phase 2 — IPsec

```
configure terminal
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
end
```

### Étape 4 — Crypto map

Sur **Router-A** :
```
configure terminal
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TSET
 match address VPN-TRAFFIC
interface gi0/1
 crypto map CMAP
end
```

Sur **Router-B** (peer inversé) :
```
configure terminal
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TSET
 match address VPN-TRAFFIC
interface gi0/1
 crypto map CMAP
end
```

### Étape 5 — Exclure le trafic VPN du NAT

Le trafic destiné au LAN distant ne doit **pas** être traduit en NAT/PAT,  
sinon le tunnel ne verra jamais les adresses privées d'origine.

```
configure terminal
ip access-list extended NAT-PAT-TRAFFIC
 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
 permit ip 192.168.1.0 0.0.0.255 any
end
```

> Remplace l'ACL utilisée par `ip nat inside source list ... overload` par  
> cette version qui exclut explicitement le trafic vers le LAN distant.

### Étape 6 — Tester le tunnel

Depuis **PC-A** :
```
ping 192.168.2.1
```

Le premier ping peut échouer ou être lent (négociation IKE), les suivants  
doivent réussir.

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! État de la négociation phase 1
show crypto isakmp sa

! État des associations de sécurité phase 2
show crypto ipsec sa

! Crypto map appliquée
show crypto map

! Compteurs de paquets chiffrés/déchiffrés
show crypto ipsec sa | include encaps|decaps
```

### Lire show crypto isakmp sa

```
dst             src             state          conn-id status
203.0.113.2     203.0.113.1     QM_IDLE           1    ACTIVE
```

| État | Signification |
|------|-----------------|
| `QM_IDLE` | Phase 1 établie, tunnel prêt pour le trafic |
| `MM_NO_STATE` | Négociation phase 1 non aboutie — vérifier la clé pré-partagée |

---

## ✅ Critères de réussite

- [ ] `show crypto isakmp sa` — état **QM_IDLE** entre les deux routeurs
- [ ] `show crypto ipsec sa` — compteurs `pkts encaps` / `pkts decaps` non nuls
- [ ] Ping PC-A → PC-B : **succès**
- [ ] En mode simulation PT, le paquet capturé entre Router-A et Router-B  
      est illisible (ESP), contrairement à un ping direct hors tunnel
- [ ] Le trafic PC-A → Internet (hors LAN distant) continue de passer en NAT/PAT normal

---

## 💡 Points pédagogiques

**Pourquoi un VPN site-à-site ?**  
Sans chiffrement, tout le trafic entre deux sites distants transite en clair  
sur Internet — visible par n'importe quel équipement intermédiaire.  
IPsec chiffre et authentifie ce trafic de bout en bout entre les deux routeurs.

**Phase 1 vs Phase 2**  
La phase 1 (ISAKMP) établit un canal sécurisé pour négocier les paramètres  
— c'est l'équivalent d'une poignée de main authentifiée. La phase 2 (IPsec)  
utilise ce canal pour négocier les clés qui chiffreront réellement le trafic utilisateur.

**Le trafic intéressant (`match address`)**  
Seuls les paquets qui correspondent à l'ACL crypto sont chiffrés. Tout  
le reste (navigation web normale, etc.) suit son chemin habituel — souvent  
via NAT/PAT — sans passer par le tunnel.

**Symétrie obligatoire**  
La clé pré-partagée, le transform-set et les paramètres ISAKMP doivent être  
identiques des deux côtés. Seuls le `peer` et le sens de l'ACL crypto s'inversent.

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-20-vpn-ipsec.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-19 — Sécurité switch](../lab-19-securite-switch/)*  
*Lab suivant : [L-21 — BGP bases (eBGP)](../lab-21-bgp/)*
