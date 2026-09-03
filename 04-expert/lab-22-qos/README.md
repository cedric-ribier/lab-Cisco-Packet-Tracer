# L-22 — QoS bases

> **Niveau :** 🟣 Expert · **Durée estimée :** 45 min · **Prérequis :** [L-16](../../02-intermediaire/lab-16-acl-etendues/)

Classifier et prioriser le trafic voix et vidéo face au trafic data  
avec le modèle MQC (Modular QoS CLI) et le marquage DSCP.

---

## 🎯 Objectifs

- Comprendre pourquoi la QoS est nécessaire sur un lien congestionné
- Classifier le trafic avec `class-map` (voix, vidéo, data)
- Marquer le trafic avec DSCP via `policy-map`
- Appliquer une politique de priorité stricte (`priority`) pour la voix
- Vérifier le marquage et les files d'attente

---

## 🗺️ Topologie

```
[Téléphone IP]  [PC-Visio]  [PC-Data]
       \             |            /
        \            |           /
         [ Switch-Access ] ── WAN restreint (lien lent) ── [ Router-Siège ]
```

> Le lien WAN entre le switch d'accès et le siège est volontairement limité  
> en bande passante (`bandwidth` réduite dans PT) pour observer l'effet  
> de la congestion sans QoS, puis avec QoS.

---

## 📋 Politique de classification

| Classe   | Trafic                     | Marquage DSCP | Traitement           |
|----------|------------------------------|----------------|------------------------|
| VOICE    | RTP voix (UDP 16384–32767)  | EF (46)        | Priority queue stricte |
| VIDEO    | Visioconférence              | AF41 (34)      | Bande garantie 30%     |
| DATA     | Reste du trafic               | Default (0)    | Best-effort            |

---

## 📝 Travail demandé

### Étape 1 — Identifier le trafic avec des ACL

```
configure terminal
ip access-list extended VOICE-TRAFFIC
 permit udp any any range 16384 32767
ip access-list extended VIDEO-TRAFFIC
 permit tcp any any eq 1720
 permit udp any any eq 5060
end
```

### Étape 2 — Créer les class-map

```
configure terminal
class-map match-all VOICE
 match access-group name VOICE-TRAFFIC
class-map match-all VIDEO
 match access-group name VIDEO-TRAFFIC
end
```

### Étape 3 — Créer la policy-map (MQC)

```
configure terminal
policy-map QOS-WAN
 class VOICE
  priority percent 20
  set dscp ef
 class VIDEO
  bandwidth percent 30
  set dscp af41
 class class-default
  fair-queue
  set dscp default
end
```

> `priority` crée une file d'attente prioritaire stricte (LLQ — Low Latency  
> Queuing) : la voix est toujours servie en premier, dans la limite du  
> pourcentage réservé, pour éviter qu'elle n'affame les autres classes.  
> `bandwidth` garantit un minimum sans empêcher les autres classes  
> d'utiliser la bande passante libre.

### Étape 4 — Appliquer la politique sur l'interface WAN

```
configure terminal
interface gi0/1
 service-policy output QOS-WAN
end
```

> La politique s'applique **en sortie** (`output`) sur l'interface la plus  
> proche du goulot d'étranglement — ici le lien WAN.

### Étape 5 — Générer de la congestion et observer

Dans PT, lance simultanément :
- Un appel voix simulé (IP Phone to IP Phone)
- Un transfert de fichier volumineux (FTP/TFTP) en arrière-plan

Sans QoS, la qualité voix se dégrade (gigue, coupures) pendant le transfert.  
Avec la policy-map appliquée, la voix doit rester fluide.

### Étape 6 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Vue d'ensemble de la policy-map appliquée
show policy-map interface gi0/1

! Statistiques par classe (paquets marqués/droppés)
show policy-map interface gi0/1 | section VOICE

! Vérifier le marquage DSCP en mode simulation PT
! (ouvrir un paquet capturé et inspecter le champ ToS/DSCP dans l'en-tête IP)
```

### Lire show policy-map interface

```
Class-map: VOICE (match-all)
    X packets, Y bytes
    Match: access-group name VOICE-TRAFFIC
    Priority: 20% (bandwidth), burst bytes ...
    	  0 packets dropped
```

| Champ | Signification |
|-------|-----------------|
| `packets dropped` | Paquets rejetés faute de place dans la file — doit rester proche de 0 pour VOICE |
| `Priority` | Confirme que la classe utilise la file prioritaire stricte |

---

## ✅ Critères de réussite

- [ ] `show policy-map interface gi0/1` — 3 classes visibles (VOICE, VIDEO, class-default)
- [ ] Classe VOICE — `Priority` confirmé, marquage `dscp ef`
- [ ] Classe VIDEO — `bandwidth percent 30`, marquage `dscp af41`
- [ ] En mode simulation PT — paquets voix marqués DSCP 46 (EF)
- [ ] Test de congestion — la voix reste fluide pendant le transfert de fichier volumineux

---

## 💡 Points pédagogiques

**Pourquoi la QoS ?**  
Sur un lien saturé, tous les paquets sont traités par défaut de façon  
égale (FIFO). La voix, sensible à la latence et à la gigue, se dégrade  
en premier. La QoS permet de garantir un traitement prioritaire aux flux  
qui en ont besoin, sans bloquer totalement les autres.

**DSCP vs ToS**  
DSCP (Differentiated Services Code Point) remplace l'ancien champ ToS  
IPv4 — 6 bits permettant 64 valeurs de marquage. `EF` (Expedited  
Forwarding, valeur 46) est la valeur standard pour la voix ; `AF41`  
(valeur 34) est couramment utilisée pour la vidéo.

**MQC — les trois briques**  
`class-map` (quoi classifier) → `policy-map` (quoi faire de chaque classe)  
→ `service-policy` (où appliquer la politique, et dans quel sens).

**LLQ (`priority`) vs CBWFQ (`bandwidth`)**  
`priority` garantit un service immédiat mais dans une limite stricte  
(pour éviter qu'un flux prioritaire n'affame tout le reste). `bandwidth`  
garantit un minimum de débit sans latence prioritaire stricte — adapté  
à un trafic tolérant un peu de délai comme la vidéo en différé.

---

## 📁 Fichiers

| Fichier | Description |
|---------|--------------|
| `lab-22-qos.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-21 — BGP bases (eBGP)](../../03-avance/lab-21-bgp/)*  
*Lab suivant : [L-23 — Réseau d'entreprise — partie 1](../lab-23-entreprise-1/)*
