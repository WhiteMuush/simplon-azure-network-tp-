# TP Pratique — Module 4 : Réseau Azure avec az CLI

**Durée estimée : 30-40 min**
**Prérequis : az CLI installé, connecté à Azure (`az login`), groupe de ressources existant**
**Niveau : Débutant — avoir fait le TP portail Module 4**

---

## 🎯 Scénario

L'API d'**AzureTech** tourne en production dans une App Service. Votre équipe décide de la sécuriser en l'isolant dans un réseau privé Azure avec des règles de filtrage du trafic.

Architecture cible :
```
Internet
    │
    ▼
[NSG] ← filtre le trafic (règles HTTP/HTTPS uniquement)
    │
[VNet 10.0.0.0/16]
    ├── subnet-frontend  10.0.1.0/24  → App Service / Container
    └── subnet-backend   10.0.2.0/24  → Base de données (usage futur)
```

Vous allez créer cette architecture avec `az CLI` — reproductible, versionnable, intégrable dans un pipeline.

> 💡 **Pourquoi CLI plutôt que portail ?**
> Un VNet créé dans le portail n'est pas documenté, pas reproductible, pas auditable. Avec le CLI, les commandes peuvent être commitées dans Git et ré-exécutées identiquement en staging et en production.

---

## Variables — à définir avant de commencer

> 📝 **À faire :** définissez et exportez vos variables d'environnement (`OWNER`, `RG`, `LOCATION`, `TAGS`, `VNET_NAME`, `NSG_NAME`) avant de commencer.

---

## Partie 1 — Créer le réseau virtuel (VNet) (8 min)

### 1.1 Créer le VNet

> 🧠 **Pourquoi ?** Un **VNet (Virtual Network)** est votre réseau privé dans Azure — équivalent à un réseau local dans un datacenter physique. Les ressources à l'intérieur peuvent communiquer entre elles. Les ressources à l'extérieur (Internet) ne peuvent pas y accéder directement sans règles explicites. On le crée en premier car les sous-réseaux et les NSG en dépendent.

**Décryptage :**
- `--address-prefix 10.0.0.0/16` : espace d'adressage total du VNet
  - `/16` = 65 536 adresses IP disponibles pour tous les sous-réseaux
  - C'est la "boîte" — les sous-réseaux sont des compartiments à l'intérieur

---

### 1.2 Créer le sous-réseau frontend

> 🧠 **Pourquoi ?** Les sous-réseaux permettent de **segmenter** le réseau par couche applicative. Le subnet-frontend héberge les ressources exposées vers Internet (App Service, ACI), le subnet-backend héberge les ressources internes (bases de données). Cette séparation permet d'appliquer des règles de sécurité différentes à chaque couche.

**Décryptage :**
- `/24` = 256 adresses, dont 5 réservées par Azure = **251 adresses utilisables**
- Les 5 réservées : adresse réseau, gateway Azure, DNS Azure x2, broadcast

> ❓ **Question :** Vous avez 251 adresses utilisables dans subnet-frontend. Si vous utilisez App Service VNet Integration, Azure réserve un bloc `/28` minimum par plan App Service. Combien de plans App Service maximum pouvez-vous intégrer dans ce sous-réseau ?

<details>
<summary>💡 Correction</summary>

Un bloc `/28` = 16 adresses (dont 5 réservées par Azure) = **11 adresses utilisables** par plan.

Avec 251 adresses disponibles dans un `/24` :
`251 ÷ 16 ≈ 15 plans App Service maximum`

En pratique, on réserve de la marge pour la croissance, donc on ne remplirait pas à 100%. Si l'application est amenée à scaler massivement, on préférerait un subnet `/23` (512 adresses) ou `/22` (1024 adresses).

</details>

---

### 1.3 Créer le sous-réseau backend

**Vérifier les deux sous-réseaux :**

Résultat attendu :
```
Nom               Plage          Statut
----------------  -------------  ---------
subnet-frontend   10.0.1.0/24    Succeeded
subnet-backend    10.0.2.0/24    Succeeded
```

---

### 1.4 Explorer le VNet créé

---

## Partie 2 — Créer le groupe de sécurité réseau (NSG) (10 min)

Un **NSG** est un pare-feu logiciel : il filtre le trafic entrant et sortant via des règles de priorité. **Plus le numéro de priorité est bas, plus la règle est appliquée en premier.**

### 2.1 Créer le NSG

> 🧠 **Pourquoi ?** Le NSG est dissocié du VNet par design — on peut créer plusieurs NSGs avec des règles différentes et les attacher à des subnets ou des NICs différents. On le crée d'abord vide, puis on ajoute les règles une par une pour pouvoir les comprendre individuellement.

---

### 2.2 Observer les règles par défaut

> 🧠 **Pourquoi ?** Azure crée automatiquement 6 règles qu'on ne peut pas modifier ni supprimer. Il est essentiel de les comprendre avant d'ajouter vos propres règles — certaines interactions ne sont pas intuitives.

Résultat attendu :
```
Nom                              Priorite  Direction  Action  Port
-------------------------------  --------  ---------  ------  -----
AllowVnetInBound                 65000     Inbound    Allow   *
AllowAzureLoadBalancerInBound    65001     Inbound    Allow   *
DenyAllInBound                   65500     Inbound    Deny    *
AllowVnetOutBound                65000     Outbound   Allow   *
AllowInternetOutBound            65001     Outbound   Allow   *
DenyAllOutBound                  65500     Outbound   Deny    *
```

> ❓ **Questions :**
> 1. Quelle règle bloque tout le trafic entrant depuis Internet par défaut ? Quel est son numéro de priorité ?
> 2. Pourquoi `AllowVnetInBound` a-t-elle la priorité 65000 et `DenyAllInBound` la priorité 65500 ?
> 3. Si vous créez une règle avec la priorité 100, sera-t-elle appliquée avant ou après `AllowVnetInBound` ?

<details>
<summary>💡 Correction</summary>

**1.** La règle `DenyAllInBound` (priorité 65500) bloque tout le trafic entrant qui n'a pas été explicitement autorisé par une règle de priorité inférieure. Internet n'est pas dans `AllowVnetInBound` (qui couvre seulement l'espace d'adressage du VNet).

**2.** Les priorités fonctionnent du plus petit au plus grand. Azure évalue d'abord la priorité 65000 (`AllowVnetInBound`), puis 65001, puis 65500 (`DenyAllInBound`). Un paquet venant du VNet matche `AllowVnetInBound` et est autorisé — `DenyAllInBound` n'est jamais atteint pour ce paquet. Un paquet venant d'Internet ne matche aucune règle avant 65500, donc il est bloqué.

**3.** Votre règle priorité 100 est évaluée **avant** `AllowVnetInBound` (65000). Si votre règle autorise le port 80 depuis partout, un paquet HTTP venant d'Internet matchera votre règle 100 et sera autorisé — sans jamais atteindre `DenyAllInBound`.

</details>

---

### 2.3 Ajouter une règle pour autoriser HTTP (port 80)

> 🧠 **Pourquoi ?** Sans cette règle, tout le trafic HTTP venant d'Internet serait bloqué par `DenyAllInBound`. On l'ajoute avec la priorité 100 pour qu'elle soit évaluée avant les règles par défaut d'Azure (toutes à 65000+).

---

### 2.4 Ajouter une règle pour autoriser HTTPS (port 443)

> 🧠 **Pourquoi 110 et pas 101 ?** On laisse des espaces entre les priorités (100, 110, 120…) pour pouvoir insérer des règles entre deux existantes sans tout renuméroter. Une règle priorité 105 pourrait être insérée entre HTTP et HTTPS si besoin.

---

### 2.5 Ajouter une règle pour bloquer tout autre trafic entrant

> 💡 Cette règle est redondante avec `DenyAllInBound` (priorité 65500) mais elle la rend **explicite** dans vos règles personnalisées — bonne pratique en sécurité pour que l'intention soit visible sans avoir à regarder les règles par défaut.

---

### 2.6 Vérifier toutes vos règles personnalisées

Résultat attendu :
```
Nom               Priorite  Direction  Action  Port
----------------  --------  ---------  ------  -----
Allow-HTTP        100       Inbound    Allow   80
Allow-HTTPS       110       Inbound    Allow   443
Deny-All-Inbound  4000      Inbound    Deny    *
```

> ❓ **Question :** Un paquet arrive sur le port 22 (SSH) depuis Internet. Dans quel ordre les règles sont-elles évaluées et quelle est la décision finale ?

<details>
<summary>💡 Correction</summary>

Ordre d'évaluation pour un paquet SSH (port 22) entrant depuis Internet :

1. **Allow-HTTP (priorité 100)** → port 80 seulement → ❌ ne matche pas
2. **Allow-HTTPS (priorité 110)** → port 443 seulement → ❌ ne matche pas
3. **Deny-All-Inbound (priorité 4000)** → port `*` → ✅ matche → **DENY**

Le paquet est bloqué à la règle 4000. Les règles par défaut Azure (65000+) ne sont jamais atteintes car la règle 4000 a déjà pris une décision.

C'est exactement le but : seuls les ports 80 et 443 passent, tout le reste est refusé explicitement.

</details>

---

## Partie 3 — Associer le NSG au sous-réseau frontend (5 min)

> 🧠 **Pourquoi ?** Un NSG créé mais non associé **n'a aucun effet** — c'est un règlement qui n'est appliqué nulle part. L'association NSG → subnet signifie que toutes les ressources qui rejoignent ce subnet seront soumises aux règles du NSG, automatiquement.

### 3.1 Associer le NSG à subnet-frontend

---

### 3.2 Vérifier l'association

Vous devriez voir l'ID du NSG dans le champ `NSG`.

---

### 3.3 Comparer les deux sous-réseaux

| Subnet | Plage | NSG |
|--------|-------|-----|
| subnet-frontend | 10.0.1.0/24 | nsg-frontend-... (protégé) |
| subnet-backend | 10.0.2.0/24 | null (non protégé pour l'instant) |

> ❓ **Question :** `subnet-backend` n'a pas de NSG. Dans une architecture réelle, quelles règles mettriez-vous sur un NSG backend pour n'autoriser que le trafic venant du frontend (et bloquer Internet directement) ?

<details>
<summary>💡 Correction</summary>

Un NSG backend restrictif suivrait ce modèle :

| Règle | Priorité | Source | Port | Action |
|-------|----------|--------|------|--------|
| Allow-From-Frontend | 100 | 10.0.1.0/24 | 5432 (PostgreSQL) | Allow |
| Allow-From-Frontend-App | 110 | 10.0.1.0/24 | 3306 (MySQL) | Allow |
| Deny-Internet-Direct | 200 | Internet | * | Deny |
| Deny-All | 4000 | * | * | Deny |

L'idée clé : utiliser l'**adresse du subnet-frontend** (`10.0.1.0/24`) comme source autorisée, plutôt que `*`. Ainsi, même si quelqu'un pénètre dans le réseau Azure par un autre chemin, il ne peut pas atteindre la base de données directement.

Cette architecture s'appelle **défense en profondeur** : chaque couche a ses propres contrôles de sécurité, indépendamment des autres.

</details>

---

## Partie 4 — Vérifier les règles effectives (5 min)

> 🧠 **Pourquoi ?** Azure combine vos règles personnalisées avec les règles par défaut. `list-effective-nsg` montre exactement ce qui sera appliqué sur une interface réseau — utile pour déboguer un problème de connectivité ("pourquoi mon trafic est-il bloqué ?").

### 4.1 Créer une interface réseau de test

Pour voir les règles effectives, nous avons besoin d'une NIC (Network Interface Card) attachée au subnet :

### 4.2 Afficher les règles de sécurité effectives

Vous voyez ici la vue consolidée : vos règles + les règles par défaut Azure, dans l'ordre d'application réel.

> ❓ **Question :** Vous voyez `AllowVnetInBound` (priorité 65000) dans les règles effectives. Est-ce que cette règle permet à une VM dans `subnet-backend` de contacter une VM dans `subnet-frontend` ? Pourquoi ?

<details>
<summary>💡 Correction</summary>

**Oui**, `AllowVnetInBound` autorise le trafic entre toutes les ressources **du même VNet** (`10.0.0.0/16`), peu importe le subnet.

`AllowVnetInBound` a pour source `VirtualNetwork` — une balise Azure qui représente l'espace d'adressage complet du VNet (y compris tous ses subnets). Donc une VM dans `subnet-backend` (10.0.2.x) peut contacter une VM dans `subnet-frontend` (10.0.1.x) sans règle supplémentaire.

C'est pourquoi, si vous voulez **isoler** subnet-backend de subnet-frontend, il faut créer une règle `Deny` explicite avec priorité inférieure à 65000 et source `10.0.2.0/24`.

</details>

---

## Partie 5 — Intégration dans provision.sh

> 📝 **À faire :** reportez vous-même les commandes des Parties 1 à 4 dans `provision.sh` (après les ressources de calcul) et dans `destroy.sh` (avant, le NSG doit être désassocié du subnet avant suppression).

---

## 🧹 Nettoyage

---

## ✅ Ce que vous avez appris

- Créer un **VNet** avec un espace d'adressage `/16` et deux sous-réseaux `/24` avec az CLI
- Comprendre les **calculs d'adressage** : /24 = 256 adresses − 5 réservées Azure = 251 utilisables
- Créer un **NSG** et ajouter des règles de filtrage avec des priorités
- Comprendre l'**ordre d'évaluation** des règles NSG (priorité la plus basse = appliquée en premier)
- **Associer** un NSG à un sous-réseau pour protéger toutes les ressources qui s'y trouvent
- Voir les **règles effectives** consolidées sur une interface réseau
- Intégrer ces commandes dans un `provision.sh` réutilisable

---

## 📚 Pour aller plus loin

---

*Formation DevSecOps Azure — Simplon*