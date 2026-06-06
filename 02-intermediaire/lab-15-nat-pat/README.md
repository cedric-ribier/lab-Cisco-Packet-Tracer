# L-15 — NAT & PAT

> **Niveau :** 🔵 Intermédiaire · **Durée estimée :** 30 min · **Prérequis :** [L-13](../lab-13-ospf-mono/)

Configurer le NAT statique pour un serveur accessible depuis l'extérieur  
et le PAT overload pour les postes clients, puis simuler l'accès Internet.

---

## 🎯 Objectifs

- Comprendre la différence entre NAT statique et PAT (NAT dynamique overload)
- Configurer le NAT statique pour un serveur interne
- Configurer le PAT overload pour les postes clients
- Vérifier avec `show ip nat translations`
- Simuler l'accès Internet depuis les postes internes

---

## 🗺️ Topologie

```
[PC0] ── Switch0 ── Router0 ── Router-ISP ── [Internet-PC]
      192.168.1.0/24   10.0.0.0/30   203.0.113.0/24
[Server]
192.168.1.10
NAT statique → 203.0.113.10
```

> Router0 fait la frontière NAT : inside (réseau interne) / outside (Internet simulé).  
> Router-ISP simule le réseau de l'opérateur avec une IP publique.

---

## 📋 Plan d'adressage

| Équipement    | Interface | Adresse IP        | Masque          | Rôle          |
|---------------|-----------|-------------------|-----------------|---------------|
| PC0           | Fa0       | 192.168.1.1/24    | 255.255.255.0   | Poste client  |
| Server        | Fa0       | 192.168.1.10/24   | 255.255.255.0   | Serveur web   |
| Router0       | Gi0/0     | 192.168.1.254/24  | 255.255.255.0   | inside NAT    |
| Router0       | Gi0/1     | 10.0.0.1/30       | 255.255.255.252 | outside NAT   |
| Router-ISP    | Gi0/0     | 10.0.0.2/30       | 255.255.255.252 | —             |
| Router-ISP    | Gi0/1     | 203.0.113.254/24  | 255.255.255.0   | —             |
| Internet-PC   | Fa0       | 203.0.113.1/24    | 255.255.255.0   | Client ext.   |

---

## 📋 NAT — règles de traduction

| Type | IP interne | IP publique | Usage |
|------|------------|-------------|-------|
| NAT statique | 192.168.1.10 | 203.0.113.10 | Serveur accessible depuis Internet |
| PAT overload | 192.168.1.0/24 | Gi0/1 (10.0.0.1) | Postes clients vers Internet |

---

## 📝 Travail demandé

### Étape 1 — Définir les interfaces inside/outside

Sur **Router0** :

```
configure terminal
interface gi0/0
 ip nat inside
interface gi0/1
 ip nat outside
end
```

### Étape 2 — NAT statique pour le serveur

```
configure terminal
ip nat inside source static 192.168.1.10 203.0.113.10
end
```

> Toute connexion vers 203.0.113.10 depuis Internet sera redirigée vers 192.168.1.10.

### Étape 3 — PAT overload pour les postes clients

```
configure terminal
access-list 1 permit 192.168.1.0 0.0.0.255
ip nat inside source list 1 interface gi0/1 overload
end
```

> L'ACL 1 définit les hôtes qui bénéficient du PAT.  
> `overload` permet à plusieurs hôtes de partager la même IP publique  
> en utilisant des ports TCP/UDP différents.

### Étape 4 — Route par défaut vers l'ISP

```
configure terminal
ip route 0.0.0.0 0.0.0.0 10.0.0.2
end
```

### Étape 5 — Route retour sur Router-ISP

```
configure terminal
ip route 203.0.113.10 255.255.255.255 10.0.0.1
end
```

### Étape 6 — Tester

Depuis **PC0** :
```
ping 203.0.113.1    → succès (PAT overload)
```

Depuis **Internet-PC** :
```
ping 203.0.113.10   → succès (NAT statique → Server interne)
```

### Étape 7 — Sauvegarder

```
copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
! Table de traduction NAT
show ip nat translations

! Statistiques NAT
show ip nat statistics

! Vider les traductions dynamiques
clear ip nat translation *
```

### Lire show ip nat translations

```
Pro  Inside global     Inside local      Outside local     Outside global
---  203.0.113.10      192.168.1.10      ---               ---
tcp  203.0.113.10:1024 192.168.1.1:1024  203.0.113.1:80    203.0.113.1:80
```

| Champ | Signification |
|-------|---------------|
| Inside local | IP réelle du poste interne |
| Inside global | IP publique vue depuis Internet |
| Outside global | IP du serveur distant |

---

## ✅ Critères de réussite

- [ ] `show ip nat translations` — entrée statique visible pour 192.168.1.10
- [ ] Ping PC0 → 203.0.113.1 : **succès** (PAT)
- [ ] Ping Internet-PC → 203.0.113.10 : **succès** (NAT statique)
- [ ] `show ip nat statistics` — compteurs de traductions actifs

---

## 💡 Points pédagogiques

**NAT statique vs PAT**  
Le NAT statique crée une correspondance permanente 1-pour-1 entre une IP  
interne et une IP publique — indispensable pour un serveur accessible depuis  
Internet. Le PAT permet à toute une plage d'hôtes de partager une seule  
IP publique via les ports — utilisé pour les postes clients.

**`overload`**  
Sans `overload`, chaque hôte interne nécessiterait une IP publique dédiée.  
Avec `overload`, des milliers de postes partagent une seule IP publique.

**inside / outside**  
Ces marqueurs sur les interfaces sont obligatoires — sans eux, NAT ne  
sait pas dans quel sens traduire les adresses.

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-15-nat-pat.pka` | Fichier activité Packet Tracer |

---

*Lab précédent : [L-14 — ACL standard](../lab-14-acl-standard/)*  
*Lab suivant : [L-16 — ACL étendues & DMZ](../lab-16-acl-etendues/)*
