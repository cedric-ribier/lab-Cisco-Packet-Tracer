# L-01 — Premier pas

> **Niveau :** 🟢 Débutant · **Durée estimée :** 15 min · **Prérequis :** Aucun

Premier contact avec Cisco Packet Tracer : construire une topologie simple, la mettre  
en forme avec les outils CPT, et tester la communication entre deux PCs.

---

## 🎯 Objectifs

- Placer et câbler des équipements dans Packet Tracer
- Annoter la topologie avec les adresses IP
- Ajouter une forme colorée sur chaque PC
- Configurer des adresses IP statiques sur les PCs
- Tester la connectivité avec `ping`

---

## 🗺️ Topologie

```
[Forme bleue]                    [Forme orange]
   PC0    ──────  Switch2960  ──────   PC1
192.168.1.1                        192.168.1.2
   [IP]                               [IP]
```

| Équipement | Interface | Adresse IP  | Masque        |
|------------|-----------|-------------|---------------|
| PC0        | Fa0       | 192.168.1.1  | 255.255.255.0 |
| PC1        | Fa0       | 192.168.1.2  | 255.255.255.0 |
| Switch0    | —         | —           | —             |

---

## 📝 Travail demandé

### Étape 1 — Placer les équipements
1. Placer un switch **2960-24TT** au centre du plan de travail
2. Placer un **PC-PT** à gauche et un autre à droite du switch

### Étape 2 — Câbler
3. Relier chaque PC au switch avec un câble **Copper Straight-Through**  
   PC0 → `Fa0` vers Switch → `Fa0/1`  
   PC1 → `Fa0` vers Switch → `Fa0/2`
4. Attendre que les voyants passent au **vert**

### Étape 3 — Mise en forme CPT
5. Ajouter une **annotation** sous chaque PC avec son adresse IP  
   *(barre d'outils → icône texte **A** → cliquer sur le plan)*
6. Ajouter une **forme** sur chaque PC :  
   - PC0 → forme **bleue**  
   - PC1 → forme **orange**  
   *(barre d'outils → icône ellipse ou rectangle → choisir la couleur)*

### Étape 4 — Configurer les IPs
7. Cliquer sur **PC0** → onglet **Desktop** → **IP Configuration**
   - IP Address : `192.168.1.1`
   - Subnet Mask : `255.255.255.0`
8. Répéter pour **PC1** avec l'adresse `192.168.1.2`

### Étape 5 — Tester
9. PC0 → **Desktop** → **Command Prompt** → `ping 192.168.1.2`
10. PC1 → **Desktop** → **Command Prompt** → `ping 192.168.1.1`

---

## 🎨 Outils CPT utilisés

| Outil | Icône | Usage |
|-------|-------|-------|
| Texte | **A** | Annoter les adresses IP |
| Rectangle / Ellipse | ⬜ ⭕ | Entourer les équipements |
| Couleur de remplissage | 🎨 | Différencier visuellement les PCs |

> Ces outils se trouvent dans la **barre d'outils de dessin** en haut du plan de travail.

---

## 🔑 Commandes de vérification

```
! Depuis le Command Prompt d'un PC
ping 192.168.1.2
```

---

## ✅ Critères de réussite

- [ ] Les deux PCs sont câblés au switch (voyants verts)
- [ ] Chaque PC a une annotation avec son adresse IP
- [ ] PC0 entouré d'une forme **bleue**, PC1 d'une forme **orange**
- [ ] Ping PC0 → PC1 : **succès** (4/4 réponses)
- [ ] Ping PC1 → PC0 : **succès** (4/4 réponses)

---

## 📁 Fichiers

| Fichier | Description |
|---------|-------------|
| `lab-01-premier-pas.pka` | Fichier activité Packet Tracer |

---

*Lab suivant : [L-02 — Modèle OSI & plan d'adressage](../lab-02-osi-adressage/)*
