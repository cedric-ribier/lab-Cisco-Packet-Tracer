# L-24 — Réseau d'entreprise — partie 2

> **Niveau :** 🟣 Expert · **Durée estimée :** 90 min · **Prérequis :** [L-23](../lab-23-entreprise-1/) · [L-16](../../02-intermediaire/lab-16-acl-etendues/) · [L-20](../../03-avance/lab-20-vpn-ipsec/) · [L-22](../lab-22-qos/)

Lab de synthèse final : sécuriser le périmètre du réseau d'entreprise  
construit en L-23 avec NAT, ACL, un tunnel VPN vers un site distant  
et une politique QoS pour la voix — un réseau d'entreprise complet de bout en bout.

---

## 🎯 Objectifs

- Ajouter un routeur de périmètre (Edge) entre le cœur (L-23) et Internet
- Configurer NAT statique (DMZ) et PAT overload (utilisateurs internes)
- Filtrer le trafic entrant avec une ACL étendue nommée
- Établir un tunnel VPN IPsec vers un site distant (agence)
- Appliquer une politique QoS pour prioriser la voix sur le lien Internet

---

## 🗺️ Topologie

```
        [ Réseau interne — L-23 ]
                   |
              [ Edge-Router ] ── DMZ (WebServer)
                   |
              Internet (simulé)
                   |
         [ Router-Agence ] ── LAN Agence 172.30.0.0/24
```

> Edge-Router se branche sur le cœur construit en L-23 (Core-R1/Core-R2)  
> côté inside, et sur Internet simulé côté outside. Une DMZ héberge un  
> serveur web accessible depuis l'extérieur. Un site distant (agence)  
> se connecte via un tunnel IPsec.

---

## 📋 Plan d'adressage — périmètre

| Équipement    | Interface | Adresse IP         | Zone     |
|----------------|-----------|---------------------|-----------|
| Edge-Router    | Gi0/0     | 10.99.9.1/30         | inside (vers cœur L-23) |
| Edge-Router    | Gi0/1     | 203.0.113.1/24       | outside (Internet)      |
| Edge-Router    | Gi0/2     | 172.20.250.254/24    | DMZ                       |
| WebServer      | Fa0       | 172.20.250.10/24     | DMZ                       |
| Router-Agence  | Gi0/0     | 172.30.0.254/24      | LAN agence               |
| Router-Agence  | Gi0/1     | 203.0.113.2/24       | outside (Internet)       |
| PC-Agence      | Fa0       | 172.30.0.1/24        | LAN agence               |

---

## 📝 Travail demandé

### Étape 1 — Intégrer Edge-Router au cœur

Ajoute une route (statique ou via OSPF area 0, au choix cohérent avec L-23)  
entre le cœur et Edge-Router pour que tous les VLANs internes (172.20.0.0/16)  
sortent par Edge-Router.

```
! Sur Core-R1/Core-R2 (méthode au choix)
ip route 0.0.0.0 0.0.0.0 10.99.9.1
```

```
! Sur Edge-Router — retour vers les VLANs internes
ip route 172.20.0.0 255.255.0.0 10.99.9.2
```

### Étape 2 — NAT sur Edge-Router (méthode L-15/L-16)

```
configure terminal
interface gi0/0
 ip nat inside
interface gi0/2
 ip nat inside
interface gi0/1
 ip nat outside
ip nat inside source static 172.20.250.10 203.0.113.10
access-list 1 permit 172.20.0.0 0.0.255.255
ip nat inside source list 1 interface gi0/1 overload
end
```

### Étape 3 — ACL de filtrage périmètre (méthode L-16)

```
configure terminal
ip access-list extended FROM_INTERNET
 permit tcp any host 203.0.113.10 eq 80
 permit tcp any host 203.0.113.10 eq 443
 permit udp any any eq 500
 permit esp any any
 deny ip any 172.20.0.0 0.0.255.255
 permit ip any any
interface gi0/1
 ip access-group FROM_INTERNET in
end
```

> `udp eq 500` (ISAKMP) et `esp` sont nécessaires pour laisser passer le  
> trafic du tunnel VPN configuré à l'étape suivante.

### Étape 4 — Tunnel VPN vers l'agence (méthode L-20)

Sur **Edge-Router** et **Router-Agence** — configuration miroir :

```
configure terminal
ip access-list extended VPN-TRAFFIC
 permit ip 172.20.0.0 0.0.255.255 172.30.0.0 0.0.0.255
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key AgenceVpnKey! address 203.0.113.2
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TSET
 match address VPN-TRAFFIC
interface gi0/1
 crypto map CMAP
end
```

> Exclus le trafic vers l'agence (172.30.0.0/24) de la liste NAT/PAT  
> (étape 2) — comme au L-20 — pour qu'il transite en clair dans le tunnel  
> plutôt que d'être traduit.

### Étape 5 — QoS sur le lien Internet (méthode L-22)

```
configure terminal
ip access-list extended VOICE-TRAFFIC
 permit udp any any range 16384 32767
class-map match-all VOICE
 match access-group name VOICE-TRAFFIC
policy-map QOS-EDGE
 class VOICE
  priority percent 20
  set dscp ef
 class class-default
  fair-queue
interface gi0/1
 service-policy output QOS-EDGE
end
```

### Étape 6 — Tester l'ensemble

```
! Depuis un PC interne (ex. VLAN Direction)
ping 8.8.8.8-like-Internet-PC       → succès via PAT overload
ping 172.30.0.1                     → succès via le tunnel VPN, en clair (non NAT)

! Depuis un PC Internet externe
ping 203.0.113.10                   → succès (WebServer DMZ)
ping 172.20.10.1                    → échec (LAN interne protégé)
```

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! NAT
show ip nat translations
show ip nat statistics

! ACL périmètre
show access-lists
show ip interface gi0/1

! VPN
show crypto isakmp sa
show crypto ipsec sa

! QoS
show policy-map interface gi0/1

! Vue globale
show ip route
```

---

## ✅ Critères de réussite

- [ ] `show ip nat translations` — entrée statique WebServer + traductions PAT actives
- [ ] `show access-lists` — FROM_INTERNET appliquée en entrée sur Gi0/1
- [ ] Internet-PC → WebServer DMZ : **succès** ; Internet-PC → LAN interne : **échec**
- [ ] `show crypto isakmp sa` — tunnel vers l'agence à l'état **QM_IDLE**
- [ ] Ping poste interne → PC-Agence : **succès**, trafic chiffré (non NAT)
- [ ] `show policy-map interface gi0/1` — classe VOICE avec `priority` active
- [ ] Poste interne → Internet (PAT) : **succès**

---

## 💡 Points pédagogiques — synthèse du parcours

Ce lab combine, sur une seule topologie, l'ensemble des briques vues  
depuis le niveau débutant :

| Brique | Origine | Rôle dans L-24 |
|--------|---------|------------------|
| VLANs, SVI, HSRP | L-04, L-11, L-18 | Réseau interne (repris de L-23) |
| OSPF | L-13, L-17 | Routage interne et vers le cœur |
| NAT/PAT | L-15 | Accès Internet des postes internes |
| ACL étendue, DMZ | L-16 | Filtrage et exposition contrôlée du WebServer |
| VPN IPsec | L-20 | Interconnexion sécurisée avec l'agence |
| QoS | L-22 | Priorisation de la voix sur le lien Internet |

**Ordre de traitement des paquets sortants sur Edge-Router**  
1. Routage (choix de l'interface de sortie)
2. Crypto map (si le paquet correspond au trafic VPN intéressant → chiffrement, pas de NAT)
3. NAT (sinon, traduction PAT/statique)
4. QoS (classification et mise en file d'attente en sortie)

Comprendre cet ordre est essentiel : un mauvais séquencement logique  
(par exemple appliquer le NAT avant l'exclusion VPN) casse silencieusement  
le tunnel, car le pair distant ne reconnaît plus l'adresse source attendue.

**Vers la production réelle**  
Cette maquette simplifie volontairement certains aspects (un seul  
Edge-Router au lieu d'une paire redondante, pas de firewall dédié,  
Internet simulé par un réseau directement connecté). En entreprise réelle,  
on ajouterait typiquement un pare-feu dédié en frontal, une redondance  
d'accès Internet (deux FAI) et une supervision centralisée (SNMP/Syslog).

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-24-entreprise-2.pka` | Fichier activité Packet Tracer |

---

## 🎓 Félicitations — parcours complet terminé !

Tu as parcouru l'ensemble des 24 labs, du premier câblage jusqu'à un  
réseau d'entreprise complet : VLANs, routage statique et dynamique,  
haute disponibilité, sécurité périmètre, VPN et QoS.

---

*Lab précédent : [L-23 — Réseau d'entreprise — partie 1](../lab-23-entreprise-1/)*
