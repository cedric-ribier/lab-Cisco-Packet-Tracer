# 🟣 Niveau 4 — Expert

Bienvenue dans le niveau expert, dernière étape du parcours. Ces 3 labs couvrent
la qualité de service (QoS) puis la construction d'un réseau d'entreprise complet
en deux parties, combinant l'ensemble des notions vues dans les niveaux précédents.
Ils s'appuient sur tous les niveaux précédents et doivent être réalisés **dans l'ordre**.

---

## 📋 Prérequis

- Niveaux débutant, intermédiaire et avancé complétés (L-01 → L-21)
- Maîtrise de HSRP, OSPF, ACL étendues et VPN IPsec

---

## 📚 Labs de ce niveau

| Lab | Titre | Thèmes principaux | Durée | Prérequis |
|-----|-------|--------------------|-------|-----------|
| [L-22](lab-22-qos/) | QoS bases | DSCP, MQC, priority queue | 45 min | L-16 |
| [L-23](lab-23-entreprise-1/) | Réseau d'entreprise — partie 1 | 3-tier, OSPF, HSRP, DHCP | 90 min | L-18 · L-13 · L-12 |
| [L-24](lab-24-entreprise-2/) | Réseau d'entreprise — partie 2 | NAT, ACL, VPN, QoS | 90 min | L-23 · L-16 · L-20 · L-22 |

---

## 🧠 Ce que tu vas apprendre

### L-22 — QoS bases
- Classifier le trafic voix/vidéo/data avec `class-map`
- Marquer le trafic avec DSCP (`set dscp ef`, `af41`) via `policy-map`
- Prioriser la voix avec une file d'attente stricte (`priority`)
- Vérifier le marquage et les compteurs de classe

### L-23 — Réseau d'entreprise — partie 1
- Concevoir une architecture 3-tier (access / distribution / core)
- Répartir les VLANs de service avec redondance HSRP (load-balancing)
- Interconnecter distribution et cœur en OSPF area 0
- Centraliser le DHCP pour tous les VLANs

### L-24 — Réseau d'entreprise — partie 2
- Sécuriser le périmètre avec NAT statique (DMZ) et PAT overload
- Filtrer le trafic entrant avec une ACL étendue nommée
- Interconnecter un site distant via un tunnel VPN IPsec
- Appliquer une politique QoS sur le lien Internet

---

## 🔑 Commandes clés du niveau

```
! QoS — MQC
class-map match-all VOICE
 match access-group name VOICE-TRAFFIC
policy-map QOS-WAN
 class VOICE
  priority percent 20
  set dscp ef
interface gi0/1
 service-policy output QOS-WAN
show policy-map interface gi0/1

! HSRP (répartition de charge)
standby 1 ip 172.20.10.254
standby 1 priority 110
standby 1 preempt
show standby brief

! Périmètre entreprise
ip nat inside source static <ip-interne> <ip-publique>
ip access-list extended FROM_INTERNET
crypto map CMAP 10 ipsec-isakmp
show crypto isakmp sa
```

---

## ✅ Critères de réussite du niveau

À l'issue des 3 labs, tu dois être capable de :

- [ ] Classifier et prioriser le trafic voix/vidéo avec la QoS (MQC/DSCP)
- [ ] Concevoir une architecture 3-tier avec redondance à chaque étage
- [ ] Centraliser DHCP et router dynamiquement sur un réseau multi-sites internes
- [ ] Sécuriser le périmètre d'un réseau d'entreprise (NAT, ACL, VPN)
- [ ] Combiner NAT, VPN et QoS sur une même interface de périmètre sans conflit

---

*Niveau précédent : [03-avance/](../03-avance/)*
