# L-03 — Commandes IOS de base

> **Niveau :** 🟢 Débutant · **Durée estimée :** 20 min · **Prérequis :** [L-01](../lab-01-premier-pas/)

Configurer un switch Cisco depuis zéro : sécurisation de l'accès, bannière,  
console, SSH et sauvegarde de la configuration.

---

## 🎯 Objectifs

- Naviguer entre les modes IOS
- Configurer hostname, enable secret, bannière
- Sécuriser l'accès console
- Configurer SSH v2 et tester la connexion depuis un PC
- Sauvegarder la configuration

---

## 🗺️ Topologie

```
PC-Admin ──── Switch2960 (Sw-Lab)
              VLAN 1 : 192.168.1.0/24
```

| Équipement | Interface  | Adresse IP    | Masque        |
|------------|------------|---------------|---------------|
| Sw-Lab     | VLAN 1     | 192.168.1.1   | 255.255.255.0 |
| PC-Admin   | Fa0        | 192.168.1.100 | 255.255.255.0 |

---

## 📝 Travail demandé

### Étape 1 — Modes IOS & hostname

| Commande | Mode | Invite |
|----------|------|--------|
| *(Entrée)* | Utilisateur | `Switch>` |
| `enable` | Privilégié | `Switch#` |
| `configure terminal` | Config globale | `Switch(config)#` |
| `interface fa0/1` | Config interface | `Switch(config-if)#` |
| `line console 0` | Config ligne | `Switch(config-line)#` |

```
Switch> enable
Switch# configure terminal
Switch(config)# hostname Sw-Lab
Sw-Lab(config)#
```

### Étape 2 — Sécurisation de base

```
Sw-Lab(config)# enable secret Cisco123!
Sw-Lab(config)# no ip domain-lookup
Sw-Lab(config)# banner motd #Acces reserve aux personnes autorisees#
Sw-Lab(config)# service password-encryption
```

### Étape 3 — Console

```
Sw-Lab(config)# line console 0
Sw-Lab(config-line)# password Cisco123!
Sw-Lab(config-line)# login
Sw-Lab(config-line)# logging synchronous
Sw-Lab(config-line)# exec-timeout 5
Sw-Lab(config-line)# exit
```

### Étape 4 — IP sur le VLAN 1

```
Sw-Lab(config)# interface vlan 1
Sw-Lab(config-if)# ip address 192.168.1.1 255.255.255.0
Sw-Lab(config-if)# no shutdown
Sw-Lab(config-if)# exit
```

### Étape 5 — SSH v2

```
Sw-Lab(config)# ip domain-name lab-tssr.local
Sw-Lab(config)# username admin password Cisco123!
Sw-Lab(config)# ip ssh version 2
Sw-Lab(config)# ip ssh authentication-retries 3
Sw-Lab(config)# ip ssh time-out 120
Sw-Lab(config)# crypto key generate rsa
! Entrer la valeur : 1024
Sw-Lab(config)# line vty 0 3
Sw-Lab(config-line)# transport input ssh
Sw-Lab(config-line)# login local
Sw-Lab(config-line)# exit
```

### Étape 6 — Tester SSH depuis PC-Admin

Sur **PC-Admin → Desktop → SSH** :
- Host : `192.168.1.1`
- Username : `admin`
- Password : `Cisco123!`

La connexion SSH doit s'établir.

### Étape 7 — Sauvegarder

```
Sw-Lab# copy running-config startup-config
```

---

## 🔑 Commandes de vérification

```
show running-config
show ip ssh
show users
show version
```

---

## ✅ Critères de réussite

- [ ] `hostname` = `Sw-Lab`
- [ ] `enable secret` configuré
- [ ] `banner motd` configuré
- [ ] SSH v2 actif : `show ip ssh` → version 2
- [ ] Connexion SSH depuis PC-Admin : **succès**
- [ ] `copy running-config startup-config` effectué

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-03-ios-base.pka` | Fichier activité Packet Tracer |

---

> 📖 Référence complète : [00-documentation/Switch-configuration-base/](../../../00-documentation/Switch-configuration-base/CONFIGURATION-BASE.md)

---

*Lab précédent : [L-02 — Modèle OSI & plan d'adressage](../lab-02-osi-adressage/)*  
*Lab suivant : [L-04 — VLANs & ports access](../lab-04-vlans-access/)*
