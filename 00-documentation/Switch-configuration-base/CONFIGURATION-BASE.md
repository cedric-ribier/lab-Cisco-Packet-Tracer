# Documentation des configurations initial d'un Switch
---

## Mode et invite de commande 

| Commande | Mode | invite de commande | 
|---|---|---|
|```Entrer```| **Mode utilisateur** | ```Switch> ``` | 
| ```enable``` | **Mode privilégié** | ```Switch# ``` |  
| ```configuration terminal``` | **Mode de configuration globale** | ```Switch(config)# ``` | 
| ```interface fastethernet0/1```| **Mode de configuration interface** | ```Switch(config-if)# ``` |
| ```line console 0```| **Mode de configuration de ligne** | ```Switch(config-line)# ``` |

---

## Changer nom du switch

```
Switch> enable
Switch# configure terminal
Switch(config)# hostname Sw1
Sw1(config)#
```

## Protéger le mode privilégié

- Mot de passe non chiffré :
  
```Sw1(config)# enable password 123```

- Mot de passe chiffré :
  
```Sw1(config)# enable secret 12345```

## Définir un nom de domaine

```Sw1(config)# ip domain-name lab-tssr.local # Personnalisée le domaine```

## Désactiver la recherche DNS “Domain Name System”, ”Système de Noms de Domaine”

```Sw1(config)# no ip domain-lookup```

La commande no ip domain-lookup empêche le périphérique de tenter une
résolution DNS lorsqu’une commande est mal saisie, ce qui accélère l’affichage
du message d’erreur.


## Configurer la bannière du jour (Message of the Day)

```Sw1(config)# banner motd #Acces reserve aux personnes autorisees#```

## Configurer l'interface console

```
Sw1(config)# line console 0
Sw1(config-line)# password 123 # Utilisé mot de passe fort
Sw1(config-line)# logging synchronous
Sw1(config-line)# login
Sw1(config-line)# exec-timeout 5
Sw1(config-line)# exit
```

## Configuration de Telnet (Deprecated)

```
S1(config)# line vty 0 15
S1(config-line)# password 1234567 # Utilisé mot de passe fort
S1(config-line)# login
S1(config-line)# exit
```

## Configuration ssh sur un Switch
```
Switch(config)# hostname SW1
SW1(config)# ip domain-name lab-tssr.local # Personnalisée le domaine
SW1(config)# username admin password 12345678 # Utilisé mot de passe fort
SW1(config)# username superadmin privilege 15 password 12345678 # Utilisé mot de passe fort
SW1(config)# ip ssh version 2
SW1(config)# ip ssh authentication-retries 3
SW1(config)# crypto key generate rsa
**Entrer la valeur "1024"**
SW1(config)# line vty 0 3
SW1(config-line)# transport input ssh
SW1(config-line)# login local
SW1(config-line)# exit
SW1(config)# ip ssh time-out 120
```

L'utilisateur "admin" a le privilège par défaut 1 avec des droits limités, tandis
que "superadmin" a le privilège 15, lui accordant un accès complet à toutes
les commandes administratives.
Le niveau de privilège 15 accorde un accès complet à toutes les
commandes de configuration et d'exécution sur les appareils Cisco.

## Chiffrement des mots de passe

```
Sw1(config)# service password-encryption
```

## Réinitialisation des paramètres du Switch

```
Sw1# erase startup-config
Sw1#reload
```

**erase startup-config**
Supprime le fichier startup-config stocké dans la NVRAM.
Toutes les configurations sauvegardées disparaissent.

**reload**
Redémarre le routeur ou switch.
Après le redémarrage, le périphérique démarre avec les paramètres par
défaut.

## Enregistrer les modifications

```Sw1# write memory  # Fonctionnel mais plus recommandé```
ou

```Sw1# copy running-config startup-config  # A privilégié```

