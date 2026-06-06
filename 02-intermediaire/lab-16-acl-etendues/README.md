# L-16 — ACL étendues & DMZ

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 45 min · **Prérequis :** [L-14](../lab-14-acl-standard/) · [L-15](../lab-15-nat-pat/)

Mettre en place une DMZ avec un serveur web, filtrer le trafic par protocole  
et port avec des ACL étendues nommées, et sécuriser le périmètre réseau.

---

## 🎯 Objectifs

- Comprendre l'architecture DMZ (zone démilitarisée)
- Créer des ACL étendues nommées avec filtrage L4 (protocole + port)
- Placer les ACL au plus près de la source
- Autoriser HTTP/HTTPS vers le serveur DMZ, bloquer le reste
- Combiner NAT et ACL pour sécuriser le périmètre

---

## 🗺️ Topologie

```
[LAN]              [DMZ]             [Internet]
PC0 ─┐             WebServer         Internet-PC
     ├── Router0 ──┤ 172.16.2.10 ─── Router-ISP
PC1 ─┘  inside     DMZ: 172.16.2.0/24  outside
    172.16.1.0/24
```

---

## 📋 Plan d'adressage

| Équipement  | Interface | Adresse IP        | Masque          | Zone    |
|-------------|-----------|-------------------|-----------------|---------|
| PC0         | Fa0       | 172.16.1.1/24     | 255.255.255.0   | LAN     |
| PC1         | Fa0       | 172.16.1.2/24     | 255.255.255.0   | LAN     |
| Router0     | Gi0/0     | 172.16.1.254/24   | 255.255.255.0   | inside  |
| Router0     | Gi0/1     | 10.0.0.1/30       | 255.255.255.252 | outside |
| Router0     | Gi0/2     | 172.16.2.254/24   | 255.255.255.0   | DMZ     |
| WebServer   | Fa0       | 172.16.2.10/24    | 255.255.255.0   | DMZ     |
| Router-ISP  | Gi0/0     | 10.0.0.2/30       | 255.255.255.252 | —       |
| Router-ISP  | Gi0/1     | 203.0.113.254/24  | 255.255.255.0   | —       |
| Internet-PC | Fa0       | 203.0.113.1/24    | 255.255.255.0   | Internet|

---

## 📋 Politique de filtrage

| Source | Destination | Protocole/Port | Action |
|--------|-------------|----------------|--------|
| Internet | WebServer DMZ | TCP 80 (HTTP) | permit |
| Internet | WebServer DMZ | TCP 443 (HTTPS) | permit |
| Internet | LAN | tout | deny |
| Internet | DMZ (autres) | tout | deny |
| LAN | tout | tout | permit |

---

## 📝 Travail demandé

### Étape 1 — NAT statique pour le WebServer

```
configure terminal
interface gi0/0
 ip nat inside
interface gi0/2
 ip nat inside
interface gi0/1
 ip nat outside
ip nat inside source static 172.16.2.10 203.0.113.10
ip route 0.0.0.0 0.0.0.0 10.0.0.2
end
```

### Étape 2 — ACL étendue — filtrage depuis Internet

```
configure terminal
ip access-list extended FROM_INTERNET
 permit tcp any host 203.0.113.10 eq 80
 permit tcp any host 203.0.113.10 eq 443
 deny ip any 172.16.1.0 0.0.0.255
 deny ip any 172.16.2.0 0.0.0.255
 permit ip any any
end
```

### Étape 3 — Appliquer l'ACL sur l'interface outside

```
configure terminal
interface gi0/1
 ip access-group FROM_INTERNET in
end
```

> Placement au plus près de la source (Internet) — interface outside en entrée.

### Étape 4 — Tester le filtrage

Depuis **Internet-PC** :
```
! Via navigateur web PT ou ping
ping 203.0.113.10       → succès (ICMP autorisé car permit ip any any)
```

Depuis **Internet-PC**, tentative d'accès au LAN :
```
ping 172.16.1.1         → échec (deny ip any 172.16.1.0)
```

Depuis **PC0** (LAN) :
```
ping 203.0.113.1        → succès (LAN → Internet autorisé)
ping 172.16.2.10        → succès (LAN → DMZ autorisé)
```

### Étape 5 — Vérifier les compteurs

```
show access-lists
show ip nat translations
```

### Étape 6 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! ACL et compteurs
show access-lists
show ip access-lists

! ACL appliquées sur les interfaces
show ip interface gi0/1

! NAT
show ip nat translations
```

### Syntaxe ACL étendue

```
permit|deny  protocole  source  wildcard  destination  wildcard  [eq port]

Exemples :
permit tcp any host 172.16.2.10 eq 80
deny   ip  any 172.16.1.0 0.0.0.255
permit icmp any any
```

---

## ✅ Critères de réussite

- [ ] `show access-lists` — ACL FROM_INTERNET présente avec toutes les règles
- [ ] `show ip interface gi0/1` — FROM_INTERNET appliquée **inbound**
- [ ] Internet-PC → ping 203.0.113.10 : **succès**
- [ ] Internet-PC → ping 172.16.1.1 : **échec** (LAN protégé)
- [ ] PC0 → ping 172.16.2.10 : **succès** (LAN → DMZ autorisé)
- [ ] PC0 → ping 203.0.113.1 : **succès** (LAN → Internet autorisé)

---

## 💡 Points pédagogiques

**DMZ — zone démilitarisée**  
La DMZ isole les serveurs accessibles depuis Internet du LAN interne.  
Si le serveur DMZ est compromis, l'attaquant n'a pas accès direct au LAN.

**ACL étendue vs standard**  
L'ACL standard ne filtre que par source — impossible de n'autoriser que HTTP  
vers un serveur précis. L'ACL étendue filtre par source, destination,  
protocole ET port.

**Placement au plus près de la source**  
Contrairement à l'ACL standard, l'ACL étendue se place **au plus près  
de la source** — ici sur l'interface outside en entrée. On bloque le  
trafic indésirable le plus tôt possible.

**`eq 80` et `eq 443`**  
`eq` = equal, suivi du numéro de port.  
Ports courants : 80 (HTTP), 443 (HTTPS), 22 (SSH), 21 (FTP), 25 (SMTP).

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-16-acl-etendues.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-15 — NAT & PAT](../lab-15-nat-pat/)*  
*Niveau suivant : [03-avance/](../../03-avance/) — OSPF multi-area, HSRP, sécurité switch, VPN, BGP*
