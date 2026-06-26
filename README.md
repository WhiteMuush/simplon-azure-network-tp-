# TP Pratique â€” Module 4 : RÃ©seau Azure avec az CLI

**DurÃ©e estimÃ©e : 30-40 min**
**PrÃ©requis : az CLI installÃ©, connectÃ© Ã  Azure (`az login`), groupe de ressources existant**
**Niveau : DÃ©butant â€” avoir fait le TP portail Module 4**

---

## ðŸŽ¯ ScÃ©nario

L'API d'**AzureTech** tourne en production dans une App Service. Votre Ã©quipe dÃ©cide de la sÃ©curiser en l'isolant dans un rÃ©seau privÃ© Azure avec des rÃ¨gles de filtrage du trafic.

Architecture cible :
```
Internet
    â”‚
    â–¼
[NSG] â† filtre le trafic (rÃ¨gles HTTP/HTTPS uniquement)
    â”‚
[VNet 10.0.0.0/16]
    â”œâ”€â”€ subnet-frontend  10.0.1.0/24  â†’ App Service / Container
    â””â”€â”€ subnet-backend   10.0.2.0/24  â†’ Base de donnÃ©es (usage futur)
```

Vous allez crÃ©er cette architecture avec `az CLI` â€” reproductible, versionnable, intÃ©grable dans un pipeline.

> ðŸ’¡ **Pourquoi CLI plutÃ´t que portail ?**
> Un VNet crÃ©Ã© dans le portail n'est pas documentÃ©, pas reproductible, pas auditable. Avec le CLI, les commandes peuvent Ãªtre commitÃ©es dans Git et rÃ©-exÃ©cutÃ©es identiquement en staging et en production.

---

## Variables â€” Ã  dÃ©finir avant de commencer

```bash
export OWNER="prenom-nom"            # votre prÃ©nom-nom (minuscules, sans espaces)
export RG="rg-${OWNER}"              # resource group fourni par le formateur
export LOCATION="francecentral"

# Tags appliquÃ©s Ã  toutes les ressources (cleanup du vendredi)
export TAGS="managed_by=cli environment=tp owner=${OWNER}"

# Noms des ressources rÃ©seau
export VNET_NAME="vnet-${OWNER}-cli"
export NSG_NAME="nsg-frontend-${OWNER}-cli"

echo "OWNER     = $OWNER"
echo "VNET_NAME = $VNET_NAME"
echo "NSG_NAME  = $NSG_NAME"
```

---

## Partie 1 â€” CrÃ©er le rÃ©seau virtuel (VNet) (8 min)

### 1.1 CrÃ©er le VNet

> ðŸ§  **Pourquoi ?** Un **VNet (Virtual Network)** est votre rÃ©seau privÃ© dans Azure â€” Ã©quivalent Ã  un rÃ©seau local dans un datacenter physique. Les ressources Ã  l'intÃ©rieur peuvent communiquer entre elles. Les ressources Ã  l'extÃ©rieur (Internet) ne peuvent pas y accÃ©der directement sans rÃ¨gles explicites. On le crÃ©e en premier car les sous-rÃ©seaux et les NSG en dÃ©pendent.

```bash
az network vnet create \
  --name           "$VNET_NAME" \
  --resource-group "$RG" \
  --location       "$LOCATION" \
  --address-prefix "10.0.0.0/16" \
  --tags           $TAGS
```

**DÃ©cryptage :**
- `--address-prefix 10.0.0.0/16` : espace d'adressage total du VNet
  - `/16` = 65 536 adresses IP disponibles pour tous les sous-rÃ©seaux
  - C'est la "boÃ®te" â€” les sous-rÃ©seaux sont des compartiments Ã  l'intÃ©rieur

---

### 1.2 CrÃ©er le sous-rÃ©seau frontend

> ðŸ§  **Pourquoi ?** Les sous-rÃ©seaux permettent de **segmenter** le rÃ©seau par couche applicative. Le subnet-frontend hÃ©berge les ressources exposÃ©es vers Internet (App Service, ACI), le subnet-backend hÃ©berge les ressources internes (bases de donnÃ©es). Cette sÃ©paration permet d'appliquer des rÃ¨gles de sÃ©curitÃ© diffÃ©rentes Ã  chaque couche.

```bash
az network vnet subnet create \
  --name           "subnet-frontend" \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --address-prefix "10.0.1.0/24"
```

**DÃ©cryptage :**
- `/24` = 256 adresses, dont 5 rÃ©servÃ©es par Azure = **251 adresses utilisables**
- Les 5 rÃ©servÃ©es : adresse rÃ©seau, gateway Azure, DNS Azure x2, broadcast

> â“ **Question :** Vous avez 251 adresses utilisables dans subnet-frontend. Si vous utilisez App Service VNet Integration, Azure rÃ©serve un bloc `/28` minimum par plan App Service. Combien de plans App Service maximum pouvez-vous intÃ©grer dans ce sous-rÃ©seau ?

<details>
<summary>ðŸ’¡ Correction</summary>

Un bloc `/28` = 16 adresses (dont 5 rÃ©servÃ©es par Azure) = **11 adresses utilisables** par plan.

Avec 251 adresses disponibles dans un `/24` :
`251 Ã· 16 â‰ˆ 15 plans App Service maximum`

En pratique, on rÃ©serve de la marge pour la croissance, donc on ne remplirait pas Ã  100%. Si l'application est amenÃ©e Ã  scaler massivement, on prÃ©fÃ©rerait un subnet `/23` (512 adresses) ou `/22` (1024 adresses).

</details>

---

### 1.3 CrÃ©er le sous-rÃ©seau backend

```bash
az network vnet subnet create \
  --name           "subnet-backend" \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --address-prefix "10.0.2.0/24"
```

**VÃ©rifier les deux sous-rÃ©seaux :**
```bash
az network vnet subnet list \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --query          "[].{Nom:name, Plage:addressPrefix, Statut:provisioningState}" \
  --output         table
```

RÃ©sultat attendu :
```
Nom               Plage          Statut
----------------  -------------  ---------
subnet-frontend   10.0.1.0/24    Succeeded
subnet-backend    10.0.2.0/24    Succeeded
```

---

### 1.4 Explorer le VNet crÃ©Ã©

```bash
az network vnet show \
  --name           "$VNET_NAME" \
  --resource-group "$RG" \
  --query          "{nom:name, adresses:addressSpace.addressPrefixes, subnets:subnets[].name}" \
  --output         json
```

---

## Partie 2 â€” CrÃ©er le groupe de sÃ©curitÃ© rÃ©seau (NSG) (10 min)

Un **NSG** est un pare-feu logiciel : il filtre le trafic entrant et sortant via des rÃ¨gles de prioritÃ©. **Plus le numÃ©ro de prioritÃ© est bas, plus la rÃ¨gle est appliquÃ©e en premier.**

### 2.1 CrÃ©er le NSG

> ðŸ§  **Pourquoi ?** Le NSG est dissociÃ© du VNet par design â€” on peut crÃ©er plusieurs NSGs avec des rÃ¨gles diffÃ©rentes et les attacher Ã  des subnets ou des NICs diffÃ©rents. On le crÃ©e d'abord vide, puis on ajoute les rÃ¨gles une par une pour pouvoir les comprendre individuellement.

```bash
az network nsg create \
  --name           "$NSG_NAME" \
  --resource-group "$RG" \
  --location       "$LOCATION" \
  --tags           $TAGS
```

---

### 2.2 Observer les rÃ¨gles par dÃ©faut

> ðŸ§  **Pourquoi ?** Azure crÃ©e automatiquement 6 rÃ¨gles qu'on ne peut pas modifier ni supprimer. Il est essentiel de les comprendre avant d'ajouter vos propres rÃ¨gles â€” certaines interactions ne sont pas intuitives.

```bash
az network nsg show \
  --name           "$NSG_NAME" \
  --resource-group "$RG" \
  --query          "defaultSecurityRules[].{Nom:name, Priorite:priority, Direction:direction, Action:access, Port:destinationPortRange}" \
  --output         table
```

RÃ©sultat attendu :
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

> â“ **Questions :**
> 1. Quelle rÃ¨gle bloque tout le trafic entrant depuis Internet par dÃ©faut ? Quel est son numÃ©ro de prioritÃ© ?
> 2. Pourquoi `AllowVnetInBound` a-t-elle la prioritÃ© 65000 et `DenyAllInBound` la prioritÃ© 65500 ?
> 3. Si vous crÃ©ez une rÃ¨gle avec la prioritÃ© 100, sera-t-elle appliquÃ©e avant ou aprÃ¨s `AllowVnetInBound` ?

<details>
<summary>ðŸ’¡ Correction</summary>

**1.** La rÃ¨gle `DenyAllInBound` (prioritÃ© 65500) bloque tout le trafic entrant qui n'a pas Ã©tÃ© explicitement autorisÃ© par une rÃ¨gle de prioritÃ© infÃ©rieure. Internet n'est pas dans `AllowVnetInBound` (qui couvre seulement l'espace d'adressage du VNet).

**2.** Les prioritÃ©s fonctionnent du plus petit au plus grand. Azure Ã©value d'abord la prioritÃ© 65000 (`AllowVnetInBound`), puis 65001, puis 65500 (`DenyAllInBound`). Un paquet venant du VNet matche `AllowVnetInBound` et est autorisÃ© â€” `DenyAllInBound` n'est jamais atteint pour ce paquet. Un paquet venant d'Internet ne matche aucune rÃ¨gle avant 65500, donc il est bloquÃ©.

**3.** Votre rÃ¨gle prioritÃ© 100 est Ã©valuÃ©e **avant** `AllowVnetInBound` (65000). Si votre rÃ¨gle autorise le port 80 depuis partout, un paquet HTTP venant d'Internet matchera votre rÃ¨gle 100 et sera autorisÃ© â€” sans jamais atteindre `DenyAllInBound`.

</details>

---

### 2.3 Ajouter une rÃ¨gle pour autoriser HTTP (port 80)

> ðŸ§  **Pourquoi ?** Sans cette rÃ¨gle, tout le trafic HTTP venant d'Internet serait bloquÃ© par `DenyAllInBound`. On l'ajoute avec la prioritÃ© 100 pour qu'elle soit Ã©valuÃ©e avant les rÃ¨gles par dÃ©faut d'Azure (toutes Ã  65000+).

```bash
az network nsg rule create \
  --name                   "Allow-HTTP" \
  --nsg-name               "$NSG_NAME" \
  --resource-group         "$RG" \
  --priority               100 \
  --direction              Inbound \
  --access                 Allow \
  --protocol               Tcp \
  --source-address-prefix  "*" \
  --source-port-range      "*" \
  --destination-address-prefix "*" \
  --destination-port-range "80" \
  --description            "Autoriser le trafic HTTP entrant"
```

**DÃ©cryptage des paramÃ¨tres :**
- `--priority 100` : s'applique avant les rÃ¨gles par dÃ©faut (65000+)
- `--direction Inbound` : filtre le trafic **entrant** vers les ressources du subnet
- `--source-address-prefix "*"` : depuis n'importe quelle adresse IP
- `--destination-port-range "80"` : uniquement le port HTTP

---

### 2.4 Ajouter une rÃ¨gle pour autoriser HTTPS (port 443)

```bash
az network nsg rule create \
  --name                   "Allow-HTTPS" \
  --nsg-name               "$NSG_NAME" \
  --resource-group         "$RG" \
  --priority               110 \
  --direction              Inbound \
  --access                 Allow \
  --protocol               Tcp \
  --source-address-prefix  "*" \
  --source-port-range      "*" \
  --destination-address-prefix "*" \
  --destination-port-range "443" \
  --description            "Autoriser le trafic HTTPS entrant"
```

> ðŸ§  **Pourquoi 110 et pas 101 ?** On laisse des espaces entre les prioritÃ©s (100, 110, 120â€¦) pour pouvoir insÃ©rer des rÃ¨gles entre deux existantes sans tout renumÃ©roter. Une rÃ¨gle prioritÃ© 105 pourrait Ãªtre insÃ©rÃ©e entre HTTP et HTTPS si besoin.

---

### 2.5 Ajouter une rÃ¨gle pour bloquer tout autre trafic entrant

```bash
az network nsg rule create \
  --name                   "Deny-All-Inbound" \
  --nsg-name               "$NSG_NAME" \
  --resource-group         "$RG" \
  --priority               4000 \
  --direction              Inbound \
  --access                 Deny \
  --protocol               "*" \
  --source-address-prefix  "*" \
  --source-port-range      "*" \
  --destination-address-prefix "*" \
  --destination-port-range "*" \
  --description            "Bloquer tout autre trafic entrant"
```

> ðŸ’¡ Cette rÃ¨gle est redondante avec `DenyAllInBound` (prioritÃ© 65500) mais elle la rend **explicite** dans vos rÃ¨gles personnalisÃ©es â€” bonne pratique en sÃ©curitÃ© pour que l'intention soit visible sans avoir Ã  regarder les rÃ¨gles par dÃ©faut.

---

### 2.6 VÃ©rifier toutes vos rÃ¨gles personnalisÃ©es

```bash
az network nsg rule list \
  --nsg-name       "$NSG_NAME" \
  --resource-group "$RG" \
  --query          "[].{Nom:name, Priorite:priority, Direction:direction, Action:access, Port:destinationPortRange}" \
  --output         table
```

RÃ©sultat attendu :
```
Nom               Priorite  Direction  Action  Port
----------------  --------  ---------  ------  -----
Allow-HTTP        100       Inbound    Allow   80
Allow-HTTPS       110       Inbound    Allow   443
Deny-All-Inbound  4000      Inbound    Deny    *
```

> â“ **Question :** Un paquet arrive sur le port 22 (SSH) depuis Internet. Dans quel ordre les rÃ¨gles sont-elles Ã©valuÃ©es et quelle est la dÃ©cision finale ?

<details>
<summary>ðŸ’¡ Correction</summary>

Ordre d'Ã©valuation pour un paquet SSH (port 22) entrant depuis Internet :

1. **Allow-HTTP (prioritÃ© 100)** â†’ port 80 seulement â†’ âŒ ne matche pas
2. **Allow-HTTPS (prioritÃ© 110)** â†’ port 443 seulement â†’ âŒ ne matche pas
3. **Deny-All-Inbound (prioritÃ© 4000)** â†’ port `*` â†’ âœ… matche â†’ **DENY**

Le paquet est bloquÃ© Ã  la rÃ¨gle 4000. Les rÃ¨gles par dÃ©faut Azure (65000+) ne sont jamais atteintes car la rÃ¨gle 4000 a dÃ©jÃ  pris une dÃ©cision.

C'est exactement le but : seuls les ports 80 et 443 passent, tout le reste est refusÃ© explicitement.

</details>

---

## Partie 3 â€” Associer le NSG au sous-rÃ©seau frontend (5 min)

> ðŸ§  **Pourquoi ?** Un NSG crÃ©Ã© mais non associÃ© **n'a aucun effet** â€” c'est un rÃ¨glement qui n'est appliquÃ© nulle part. L'association NSG â†’ subnet signifie que toutes les ressources qui rejoignent ce subnet seront soumises aux rÃ¨gles du NSG, automatiquement.

### 3.1 Associer le NSG Ã  subnet-frontend

```bash
az network vnet subnet update \
  --name           "subnet-frontend" \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --network-security-group "$NSG_NAME"
```

---

### 3.2 VÃ©rifier l'association

```bash
az network vnet subnet show \
  --name           "subnet-frontend" \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --query          "{Subnet:name, NSG:networkSecurityGroup.id}" \
  --output         json
```

Vous devriez voir l'ID du NSG dans le champ `NSG`.

---

### 3.3 Comparer les deux sous-rÃ©seaux

```bash
az network vnet subnet list \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --query          "[].{Nom:name, Plage:addressPrefix, NSG:networkSecurityGroup.id}" \
  --output         table
```

| Subnet | Plage | NSG |
|--------|-------|-----|
| subnet-frontend | 10.0.1.0/24 | nsg-frontend-... (protÃ©gÃ©) |
| subnet-backend | 10.0.2.0/24 | null (non protÃ©gÃ© pour l'instant) |

> â“ **Question :** `subnet-backend` n'a pas de NSG. Dans une architecture rÃ©elle, quelles rÃ¨gles mettriez-vous sur un NSG backend pour n'autoriser que le trafic venant du frontend (et bloquer Internet directement) ?

<details>
<summary>ðŸ’¡ Correction</summary>

Un NSG backend restrictif suivrait ce modÃ¨le :

| RÃ¨gle | PrioritÃ© | Source | Port | Action |
|-------|----------|--------|------|--------|
| Allow-From-Frontend | 100 | 10.0.1.0/24 | 5432 (PostgreSQL) | Allow |
| Allow-From-Frontend-App | 110 | 10.0.1.0/24 | 3306 (MySQL) | Allow |
| Deny-Internet-Direct | 200 | Internet | * | Deny |
| Deny-All | 4000 | * | * | Deny |

L'idÃ©e clÃ© : utiliser l'**adresse du subnet-frontend** (`10.0.1.0/24`) comme source autorisÃ©e, plutÃ´t que `*`. Ainsi, mÃªme si quelqu'un pÃ©nÃ¨tre dans le rÃ©seau Azure par un autre chemin, il ne peut pas atteindre la base de donnÃ©es directement.

Cette architecture s'appelle **dÃ©fense en profondeur** : chaque couche a ses propres contrÃ´les de sÃ©curitÃ©, indÃ©pendamment des autres.

</details>

---

## Partie 4 â€” VÃ©rifier les rÃ¨gles effectives (5 min)

> ðŸ§  **Pourquoi ?** Azure combine vos rÃ¨gles personnalisÃ©es avec les rÃ¨gles par dÃ©faut. `list-effective-nsg` montre exactement ce qui sera appliquÃ© sur une interface rÃ©seau â€” utile pour dÃ©boguer un problÃ¨me de connectivitÃ© ("pourquoi mon trafic est-il bloquÃ© ?").

### 4.1 CrÃ©er une interface rÃ©seau de test

Pour voir les rÃ¨gles effectives, nous avons besoin d'une NIC (Network Interface Card) attachÃ©e au subnet :

```bash
az network nic create \
  --name           "nic-test-${OWNER}-cli" \
  --resource-group "$RG" \
  --location       "$LOCATION" \
  --vnet-name      "$VNET_NAME" \
  --subnet         "subnet-frontend" \
  --tags           $TAGS
```

### 4.2 Afficher les rÃ¨gles de sÃ©curitÃ© effectives

```bash
az network nic list-effective-nsg \
  --name           "nic-test-${OWNER}-cli" \
  --resource-group "$RG" \
  --query          "effectiveNetworkSecurityGroups[0].effectiveSecurityRules[].{Nom:name, Priorite:priority, Direction:direction, Action:access, Port:destinationPortRanges}" \
  --output         table
```

Vous voyez ici la vue consolidÃ©e : vos rÃ¨gles + les rÃ¨gles par dÃ©faut Azure, dans l'ordre d'application rÃ©el.

> â“ **Question :** Vous voyez `AllowVnetInBound` (prioritÃ© 65000) dans les rÃ¨gles effectives. Est-ce que cette rÃ¨gle permet Ã  une VM dans `subnet-backend` de contacter une VM dans `subnet-frontend` ? Pourquoi ?

<details>
<summary>ðŸ’¡ Correction</summary>

**Oui**, `AllowVnetInBound` autorise le trafic entre toutes les ressources **du mÃªme VNet** (`10.0.0.0/16`), peu importe le subnet.

`AllowVnetInBound` a pour source `VirtualNetwork` â€” une balise Azure qui reprÃ©sente l'espace d'adressage complet du VNet (y compris tous ses subnets). Donc une VM dans `subnet-backend` (10.0.2.x) peut contacter une VM dans `subnet-frontend` (10.0.1.x) sans rÃ¨gle supplÃ©mentaire.

C'est pourquoi, si vous voulez **isoler** subnet-backend de subnet-frontend, il faut crÃ©er une rÃ¨gle `Deny` explicite avec prioritÃ© infÃ©rieure Ã  65000 et source `10.0.2.0/24`.

</details>

---

## Partie 5 â€” IntÃ©gration dans provision.sh

Voici le bloc Ã  ajouter dans votre `provision.sh` aprÃ¨s la crÃ©ation des ressources de calcul :

```bash
# â”€â”€ RÃ©seau (VNet + NSG) â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
echo ""
echo "â–¶ [8/8] RÃ©seau (VNet + subnets + NSG)..."

az network vnet create \
  --name           "$VNET_NAME" \
  --resource-group "$RG" \
  --location       "$LOCATION" \
  --address-prefix "10.0.0.0/16" \
  --tags           $TAGS

az network vnet subnet create \
  --name "subnet-frontend" --vnet-name "$VNET_NAME" \
  --resource-group "$RG" --address-prefix "10.0.1.0/24"

az network vnet subnet create \
  --name "subnet-backend" --vnet-name "$VNET_NAME" \
  --resource-group "$RG" --address-prefix "10.0.2.0/24"

echo "âœ… VNet crÃ©Ã© : $VNET_NAME (frontend: 10.0.1.0/24 / backend: 10.0.2.0/24)"

az network nsg create \
  --name "$NSG_NAME" --resource-group "$RG" \
  --location "$LOCATION" --tags $TAGS

az network nsg rule create \
  --name "Allow-HTTP" --nsg-name "$NSG_NAME" --resource-group "$RG" \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefix "*" --source-port-range "*" \
  --destination-address-prefix "*" --destination-port-range "80"

az network nsg rule create \
  --name "Allow-HTTPS" --nsg-name "$NSG_NAME" --resource-group "$RG" \
  --priority 110 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefix "*" --source-port-range "*" \
  --destination-address-prefix "*" --destination-port-range "443"

az network nsg rule create \
  --name "Deny-All-Inbound" --nsg-name "$NSG_NAME" --resource-group "$RG" \
  --priority 4000 --direction Inbound --access Deny --protocol "*" \
  --source-address-prefix "*" --source-port-range "*" \
  --destination-address-prefix "*" --destination-port-range "*"

az network vnet subnet update \
  --name "subnet-frontend" --vnet-name "$VNET_NAME" \
  --resource-group "$RG" \
  --network-security-group "$NSG_NAME"

echo "âœ… NSG crÃ©Ã© et associÃ© Ã  subnet-frontend (HTTP:100 / HTTPS:110 / Deny:4000)"
```

Et dans `destroy.sh`, ajouter **avant** la suppression des resources de calcul (le NSG doit Ãªtre dÃ©sassociÃ© avant suppression) :

```bash
# â”€â”€ RÃ©seau â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
# DÃ©sassocier le NSG du subnet avant suppression (obligatoire)
if az network vnet subnet show \
    --name "subnet-frontend" --vnet-name "$VNET_NAME" \
    --resource-group "$RG" &>/dev/null; then
  az network vnet subnet update \
    --name "subnet-frontend" --vnet-name "$VNET_NAME" \
    --resource-group "$RG" --network-security-group "" 2>/dev/null || true
fi

if az network nsg show --name "$NSG_NAME" --resource-group "$RG" &>/dev/null; then
  az network nsg delete --name "$NSG_NAME" --resource-group "$RG"
  echo "âœ… NSG supprimÃ©"
fi

# Supprimer la NIC de test puis le VNet
az network nic delete \
  --name "nic-test-${OWNER}-cli" --resource-group "$RG" 2>/dev/null || true

if az network vnet show --name "$VNET_NAME" --resource-group "$RG" &>/dev/null; then
  az network vnet delete --name "$VNET_NAME" --resource-group "$RG"
  echo "âœ… VNet supprimÃ©"
fi
```

---

## ðŸ§¹ Nettoyage

```bash
# DÃ©sassocier le NSG avant suppression
az network vnet subnet update \
  --name           "subnet-frontend" \
  --vnet-name      "$VNET_NAME" \
  --resource-group "$RG" \
  --network-security-group "" 2>/dev/null || true

# Supprimer la NIC de test
az network nic delete \
  --name           "nic-test-${OWNER}-cli" \
  --resource-group "$RG" 2>/dev/null || true

# Supprimer le NSG
az network nsg delete \
  --name           "$NSG_NAME" \
  --resource-group "$RG"

# Supprimer le VNet (supprime automatiquement les subnets)
az network vnet delete \
  --name           "$VNET_NAME" \
  --resource-group "$RG"

echo "âœ… Ressources rÃ©seau supprimÃ©es"

# VÃ©rification
az network vnet list \
  --resource-group "$RG" \
  --query          "[].name" \
  --output         table
```

---

## âœ… Ce que vous avez appris

- CrÃ©er un **VNet** avec un espace d'adressage `/16` et deux sous-rÃ©seaux `/24` avec az CLI
- Comprendre les **calculs d'adressage** : /24 = 256 adresses âˆ’ 5 rÃ©servÃ©es Azure = 251 utilisables
- CrÃ©er un **NSG** et ajouter des rÃ¨gles de filtrage avec des prioritÃ©s
- Comprendre l'**ordre d'Ã©valuation** des rÃ¨gles NSG (prioritÃ© la plus basse = appliquÃ©e en premier)
- **Associer** un NSG Ã  un sous-rÃ©seau pour protÃ©ger toutes les ressources qui s'y trouvent
- Voir les **rÃ¨gles effectives** consolidÃ©es sur une interface rÃ©seau
- IntÃ©grer ces commandes dans un `provision.sh` rÃ©utilisable

---

## ðŸ“š Pour aller plus loin

```bash
# Lister tous les VNets de votre resource group
az network vnet list --resource-group "$RG" --output table

# Voir les adresses IP utilisÃ©es dans un subnet
az network vnet subnet show \
  --name "subnet-frontend" --vnet-name "$VNET_NAME" \
  --resource-group "$RG" \
  --query "{ipConfigurations:ipConfigurations[].id}" \
  --output json

# Ajouter une rÃ¨gle SSH (port 22) â€” Ã  utiliser uniquement en dev depuis votre IP !
MY_IP=$(curl -s ifconfig.me)
az network nsg rule create \
  --name "Allow-SSH-DevOnly" --nsg-name "$NSG_NAME" --resource-group "$RG" \
  --priority 200 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefix "${MY_IP}/32" \
  --source-port-range "*" \
  --destination-address-prefix "*" \
  --destination-port-range "22" \
  --description "SSH depuis mon IP uniquement â€” Ã  supprimer en production"
```

---

*Formation DevSecOps Azure â€” Simplon*
