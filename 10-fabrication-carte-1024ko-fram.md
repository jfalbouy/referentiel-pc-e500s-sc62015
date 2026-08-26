# Proposition de fabrication — carte FRAM 1024 Ko

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier propose une **conception nouvelle** (pas une reconstitution exacte de la carte M. Kemper/Dynatech de `08-cartes-memoire.md` §3.3) : un nomenclature (BOM) chiffrée et un schéma fonctionnel pour une carte 1024 Ko en FRAM (sans pile), en s'appuyant sur le fonctionnement confirmé du registre `POKE 65536` (§3.3) et les contraintes de bus du SC62015 (`01-architecture-cpu-sc62015.md` §3, `08-cartes-memoire.md` §1).
>
> **Avertissement** : comme indiqué dans `08-cartes-memoire.md` (préambule et §6), le brochage exact du connecteur de carte Sharp n'est publié nulle part et n'a pas été retrouvé lors des recherches (y compris recherche web dédiée, juillet 2026). Tout ce fichier repose donc sur une hypothèse de fonctionnement (le connecteur expose le bus d'adresse/données/contrôle du SC62015 de façon suffisamment directe pour permettre le décodage embarqué décrit ci-dessous) qui **doit être vérifiée par traçage de continuité** sur une carte réelle avant de router un PCB définitif.

## 1. Nomenclature (BOM) et coût estimé

Composants nécessaires pour **1 carte** (hypothèse : petite série de 5 PCB, coûts de fabrication amortis sur 5 ; prix composants en quantité unitaire/faible, hors port et douane) :

| Réf. | Composant | Rôle | Prix indicatif (unité) | Source vérifiée |
|---|---|---|---|---|
| U1, U2 | **Fujitsu `MB85R4001ANC-GE1`** — FRAM 4 Mbit (512K×8), parallèle, TSOP-48 | 2 puces de 512 Ko = 1024 Ko | **≈ 9-15 $** pièce (qté 1-10) | Octopart (agrégateur, juillet 2026) : offres de 9,26 $ (Worldway Electronics) à 20,63 $ (Win Source) selon revendeur ; **pas de stock direct constaté chez Digikey/Mouser en tant que distributeur officiel** — composant à sourcer via revendeur/broker (Octopart) ou circuit japonais (Fujitsu direct, Chip1Stop) puisque c'est un composant Fujitsu d'origine japonaise. |
| U3 | **74HC138** (décodeur 3→8) | Reconnaissance d'adresse `10000H` + sélection des puces | **≈ 0,35-1,30 $** selon fabricant (ex. `TC74HC138APF` Toshiba ≈ 0,34 $ ; `SN74HC138N` TI ≈ 1,27 $) | DigiKey (juillet 2026) |
| U4 | **74HC175** (registre 4×D, avec `CLR`) | Stockage des 4 bits `D0`-`D3` du registre `POKE 65536` | **≈ 0,30-0,55 $** (ex. `74HC175D,653` Nexperia ≈ 0,32-0,46 $ selon quantité) | DigiKey (juillet 2026) |
| U5 | **74HC02** (quad NOR) | Glue logic (combinaison décodage/CE/protection écriture) | **≈ 0,30-0,50 $** | Estimation (famille standard, non recherchée individuellement) |
| RN1 | Réseau résistif 4-8 broches (pull-up, ≈ 10-47 kΩ) | Tirage `A15`-`A17` (§`09-cartes-meres-ram-interne.md` §4-5) | **≈ 0,20-0,40 $** | Estimation (composant standard courant) |
| C1-C5 | Condensateurs céramique 100 nF (découplage) | Découplage alimentation par puce | **≈ 0,05 $** pièce (0,25 $ total) | Estimation |
| PCB | Carte 2 couches, ≈ 54×42×2,8 mm, **connecteur à contacts dorés (« gold fingers »)** | Support + connecteur (le connecteur EST le bord de carte plaqué or, pas une pièce séparée) | **≈ 8-15 $/carte** en série de 5 (base ≈ 2 $/5 cartes + supplément « gold fingers », généralement un forfait fixe ≈ 30-50 $ par commande, amorti sur la série) | JLCPCB (tarif de base public, juillet 2026 : prototypes à partir de 2 $ pour 5 PCB — supplément gold fingers non confirmé précisément, à cocher au devis) |
| — | Coque plastique (optionnel) | Protection mécanique, forme Sharp standard | **0 $ si récupérée** d'une carte officielle bon marché (ex. CE-2H32M déclassée) ; **≈ 1-3 $** si impression 3D | Estimation |
| — | Assemblage CMS (optionnel, si pas de soudure manuelle) | TSOP-48 pas 0,5 mm : soudure manuelle possible mais délicate (fer fin + flux, ou air chaud) | **≈ 10-20 $/carte** en petite série (service JLCPCB SMT ou équivalent) | Estimation |

**Total estimé par carte (auto-assemblée, série de 5, hors port/douane) : ≈ 30-45 $** — dominé à ≈ 60-70 % par les 2 puces FRAM. Avec assemblage CMS sous-traité : **≈ 45-65 $/carte**.

> **Fiabilité des prix** : les prix des FRAM `MB85R4001A` proviennent d'un agrégateur (Octopart) interrogeant des revendeurs tiers, pas d'un distributeur agréé de premier rang — composant ancien/de niche, prix et disponibilité réellement volatils. Les prix 74HC138/74HC175 proviennent de pages produit DigiKey consultées le jour même. Le supplément PCB « gold fingers » n'a pas pu être confirmé par un devis en ligne réel dans cette recherche (page JLCPCB trop lourde à charger) — **à vérifier par un devis direct avant commande**.

## 2. Schéma fonctionnel proposé

*(voir le diagramme affiché dans la conversation)*

Principe retenu, cohérent avec le fonctionnement confirmé du registre `POKE 65536` (`08-cartes-memoire.md` §3.3) :

1. **Bus d'adresse/données** : `A0`-`A18`, `D0`-`D7`, `CE1` (zone `S2:`), `RD`, `WR` arrivent du connecteur directement aux deux puces FRAM et à la logique de décodage.
2. **Registre de banque (`U4`, 74HC175)** : chargé par une impulsion d'écriture générée par `U3` (74HC138) lorsque l'adresse `10000H` est reconnue en écriture (`WR` actif) — reproduit le comportement `POKE 65536,valeur` du manuel fabricant. Stocke `D0` (bit de poids faible du banc), `D1` (bit de poids fort du banc / sélection de puce), `D2` (Card OFF), `D3` (Protect).
3. **Sélection de puce (`U5`, 74HC02)** : combine `D1` (sortie du registre) avec `CE1` (venant du connecteur) pour produire `CE_A` (FRAM A, bancs 0-1) et `CE_B` (FRAM B, bancs 2-3) — un seul étage OU/NON suffit pour un choix binaire entre 2 puces.
4. **Bit de poids faible du banc (`D0`)** : câblé directement sur la ligne d'adresse `A18` de chaque puce FRAM (puisque la fenêtre native de 256 Ko fournie par `CE1` ne couvre que 18 lignes d'adresse, il manque exactement 1 ligne pour adresser les 512 Ko d'une puce — `D0` la fournit).
5. **`D2` (Card OFF)** : combiné (via `U5`) à `CE_A`/`CE_B` pour forcer les deux puces en haute impédance quand la carte est désactivée.
6. **`D3` (Protect)** : combiné à `WR` pour bloquer toute écriture vers les puces FRAM quand la protection est active.
7. **Pull-up `A15`-`A17`** (réseau résistif `RN1`) : ajouté directement sur la carte, conformément à la recommandation de `08-cartes-memoire.md` §6 point 5 (compatibilité machines dont ces lignes sont flottantes).

**Ce qui est confirmé** (§3.3) : la structure logique 4 bancs de 256 Ko, la fonction de chaque bit du registre. **Ce qui reste une hypothèse de conception** (marqué sur le diagramme) : que l'adresse `10000H` soit décodable directement depuis les signaux disponibles au connecteur — c'est plausible mais non vérifié, comme déjà noté dans `08-cartes-memoire.md` §6.

## 3. Proposition de PCB

- **Format** : reprendre les dimensions officielles Sharp demi-format, **54×42×2,8 mm** (`08-cartes-memoire.md` §2), pour garantir la compatibilité mécanique avec le logement `S2:` et une éventuelle coque récupérée.
- **Empilage** : 2 couches suffisent largement (peu de composants, bus relativement lent) — pas besoin de 4 couches.
- **Finition de bord** : contacts « gold fingers » (or dur) sur l'arête d'insertion, seule zone du PCB à traiter ainsi — c'est cette finition qui **constitue le connecteur**, il n'y a pas de connecteur rapporté à acheter séparément.
- **Placement proposé** (de l'arête connecteur vers l'intérieur) : bord doré → `U3` (74HC138) et `U4` (74HC175) et `U5` (74HC02) groupés près du bord pour des pistes de décodage courtes → les deux FRAM `U1`/`U2` côte à côte plus loin, bus de données/adresse en bus parallèle direct → réseau résistif `RN1` proche des broches `A15`-`A17` du connecteur → condensateurs de découplage au plus près de chaque puce (un `100 nF` par `VCC`).
- **Ce qui manque avant de router réellement le PCB** : le **brochage exact du connecteur** (quel contact = quelle ligne `A0`-`A18`/`D0`-`D7`/`CE`/`RD`/`WR`/`VCC`/`GND`). `08-cartes-memoire.md` §6 point 1 recommande déjà la méthode (traçage de continuité depuis la carte « CC Sharp Card » nue, `IMG_2412.jpg`) — **c'est le prérequis bloquant avant de dessiner l'empreinte du connecteur dans un outil comme KiCad**. Une fois ce brochage confirmé, le reste du routage (bus parallèle simple, peu de contraintes de vitesse) est un exercice standard.

## 4. Voir aussi

- `08-cartes-memoire.md` §3.3, §5, §6 — carte 1024 Ko d'origine, candidats FRAM, vérifications recommandées.
- `09-cartes-meres-ram-interne.md` §4-5 — pull-up `A15`-`A17`, limites `MEM$="B"`.
- `01-architecture-cpu-sc62015.md` §3 — bus externe, 19 lignes d'adresse.
