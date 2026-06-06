# 🟠 Niveau 3 — Avancé

Bienvenue dans le niveau avancé. Ces 5 labs couvrent la haute disponibilité,
le routage dynamique multi-area, la sécurité de la commutation, les VPNs
et les bases du routage inter-AS. Ils s'appuient sur l'ensemble des niveaux
précédents et doivent être réalisés **dans l'ordre**.

---

## 📋 Prérequis

- Niveaux débutant et intermédiaire complétés (L-01 → L-16)
- Maîtrise de OSPF mono-area, ACL, NAT et SVIs

---

## 📚 Labs de ce niveau

| Lab | Titre | Thèmes principaux | Durée | Prérequis |
|-----|-------|-------------------|-------|-----------|
| [L-17](lab-17-ospf-multi/) | OSPF multi-area | ABR, summarization, inter-area | 45 min | L-13 |
| [L-18](lab-18-hsrp/) | HSRP & redondance passerelle | HSRP, failover, preempt | 45 min | L-11 · L-13 |
| [L-19](lab-19-securite-switch/) | Sécurité switch | Port-security, DHCP snooping, DAI | 45 min | L-10 · L-12 |
| [L-20](lab-20-vpn-ipsec/) | VPN site-à-site IPsec | IKEv1, crypto map, tunnel | 60 min | L-16 |
| [L-21](lab-21-bgp/) | BGP bases (eBGP) | eBGP, AS-PATH, préfixes | 60 min | L-17 |

---

## 🧠 Ce que tu vas apprendre

### L-17 — OSPF multi-area
- Structurer un réseau OSPF en plusieurs areas autour du backbone area 0
- Configurer les ABR (Area Border Routers)
- Comprendre les routes inter-area (O IA) dans la table de routage
- Réduire la taille de la LSDB par area
- Configurer le résumé de routes sur un ABR

### L-18 — HSRP & redondance passerelle
- Configurer deux routeurs en HSRP avec une IP virtuelle par VLAN
- Définir les priorités et activer `preempt`
- Simuler la panne du routeur actif et observer le basculement automatique
- Comprendre la différence HSRP / VRRP / GLBP

### L-19 — Sécurité switch
- Limiter le nombre d'adresses MAC par port avec `port-security`
- Activer DHCP snooping pour bloquer les serveurs DHCP non autorisés
- Configurer Dynamic ARP Inspection (DAI) pour prévenir l'ARP spoofing
- Désactiver les ports inutilisés

### L-20 — VPN site-à-site IPsec
- Configurer un tunnel IPsec IKEv1 entre deux sites distants
- Définir la crypto policy, le transform-set et la crypto map
- Vérifier l'établissement du tunnel avec `show crypto isakmp sa`
- Tester la connectivité chiffrée entre les deux sites

### L-21 — BGP bases (eBGP)
- Configurer une session eBGP entre deux AS distincts
- Annoncer des préfixes via `network`
- Analyser les attributs AS-PATH et LOCAL_PREF
- Vérifier avec `show bgp summary` et `show ip bgp`

---

## 🔑 Commandes clés du niveau

```
! OSPF multi-area
router ospf 1
 area 1 range 10.1.0.0 255.255.0.0    ← résumé sur ABR
show ip ospf border-routers
show ip route ospf

! HSRP
interface gi0/0
 standby 1 ip 172.16.10.254
 standby 1 priority 110
 standby 1 preempt
show standby brief

! Port-security
switchport port-security maximum 2
switchport port-security violation restrict
switchport port-security
show port-security interface fa0/1

! DHCP snooping
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
interface gi0/1
 ip dhcp snooping trust
show ip dhcp snooping binding

! DAI
ip arp inspection vlan 10,20,30
interface gi0/1
 ip arp inspection trust
show ip arp inspection

! IPsec IKEv1
crypto isakmp policy 10
 encryption aes / hash sha / authentication pre-share
crypto isakmp key CiscoKey address [peer-ip]
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
crypto map CMAP 10 ipsec-isakmp
 set peer [peer-ip]
 set transform-set TSET
 match address [acl]
show crypto isakmp sa
show crypto ipsec sa

! BGP
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 network 192.168.1.0 mask 255.255.255.0
show bgp summary
show ip bgp
```

---

## ✅ Critères de réussite du niveau

À l'issue des 5 labs, tu dois être capable de :

- [ ] Concevoir et configurer OSPF sur un réseau multi-area
- [ ] Mettre en place la redondance de passerelle avec HSRP
- [ ] Sécuriser les ports d'un switch contre les attaques L2
- [ ] Établir un tunnel IPsec entre deux sites
- [ ] Configurer une session eBGP et annoncer des préfixes

---

*Niveau précédent : [02-intermediaire/](../02-intermediaire/)*  
*Niveau suivant : [04-expert/](../04-expert/) — QoS & réseau d'entreprise complet*
