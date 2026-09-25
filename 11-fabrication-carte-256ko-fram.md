# Proposition de fabrication — carte FRAM 256 Ko

*Rédigé le 2026-08-26 — mis à jour le 2026-08-26*

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier reprend, pour une carte **256 Ko**, le même exercice que `10-fabrication-carte-1024ko-fram.md` : nomenclature chiffrée, schéma fonctionnel, proposition de PCB — pour une conception équivalente à la **carte FRAM commerciale japonaise** décrite en `08-cartes-memoire.md` §4 (vendeur 高松製作所/Takamatsu Seisakusho, `tmfg.jp`).
>
> **Avertissement identique à `10-fabrication-carte-1024ko-fram.md`** : le brochage exact du connecteur reste non confirmé publiquement — hypothèse de conception à vérifier par traçage de continuité avant routage définitif (`08-cartes-memoire.md` §6).

## 0. Référence de marché (nouveau, trouvé lors de cette recherche)

La fiche produit réelle du vendeur japonais a pu être consultée directement (`tmfg.jp/products/detail/202`, juillet 2026) et apporte des données précises qui n'étaient qu'évoquées indirectement en §4 :

| Élément | Valeur confirmée par la fiche produit |
|---|---|
| Prix de vente (carte finie, port non compris) | **¥ 14 850 TTC** (≈ 80 € / 87 $ au taux du 30/06/2026, 1 € ≈ 185,6 ¥) |
| Dimensions officielles | **42 mm × 54 mm × 3 mm** (saillies comprises) — très proche des 54×42×2,8 mm des cartes Sharp officielles (`08-cartes-memoire.md` §2), légère différence probablement due à l'arrondi/à la mesure avec les saillies |
| Poids | ≈ 8 g |
| Protection | Interrupteur physique de protection en écriture (comme la carte M. Kemper 256 Ko de `08-cartes-memoire.md` §3.2) |
| Compatibilité | PC-E500, E550, 1480U, 1490U, 1490UII, E650, U6000 |
| Usage | « strictement identique à une carte RAM classique » — **aucune commande `POKE` spéciale mentionnée**, confirmation indirecte qu'aucun registre de banque n'est nécessaire à 256 Ko (cohérent avec `08-cartes-memoire.md` §1 : 256 Ko < 512 Ko, limite native du bus) |
| Produit apparenté (même vendeur) | Carte FRAM **128 Ko** à ¥ 14 300, et carte **512 Ko (256 Ko×2)** à ¥ 15 400 (probablement SRAM avec pile, nom « RAMカード » et non « FRAMカード ») |
| **Découverte annexe pertinente pour `09-cartes-meres-ram-interne.md`** | Le même vendeur propose un **service payant de transformation de la RAM interne 32/64 Ko → 256 Ko** (`tmfg.jp/products/detail/104`, ¥ 6 380 ≈ 34 €, machine à leur envoyer) — confirmation commerciale directe que la transformation 32K→256K évoquée en `09-cartes-meres-ram-interne.md` §3 est réalisable et proposée commercialement au Japon en 2026, en complément du témoignage forum de 2015. |

Ce prix de **~80 €** pour le produit fini sert de référence de comparaison pour l'estimation DIY ci-dessous.

## 1. Nomenclature (BOM) et coût estimé

Contrairement à la carte 1024 Ko, **aucun registre de banque n'est nécessaire** : 256 Ko tient entièrement dans la fenêtre native de 256 Ko fournie par `CE1` (`08-cartes-memoire.md` §1), donc pas de `74HC175`, pas de `74HC02`, pas de décodage d'adresse `10000H`. C'est le schéma déjà identifié pour la carte M. Kemper 256 Ko (§3.2) et repris en §5 : 2 puces + décodeur simple — **désormais confirmé par deux exemplaires physiques indépendants** du même concepteur, à 5 ans d'écart (carte « M. Kemper 4.93 » de 1993 et carte « Böttcher Datentechnik »/M.K. v4.1 de 1998, `08-cartes-memoire.md` §3.4), ce qui renforce la confiance dans ce schéma comme base de conception plutôt qu'une simple hypothèse isolée.

| Réf. | Composant | Rôle | Prix indicatif (unité) | Source vérifiée |
|---|---|---|---|---|
| U1, U2 | **Fujitsu `MB85R1001ANC-GE1`** — FRAM 1 Mbit (128K×8), parallèle, TSOP-48 | 2 puces de 128 Ko = 256 Ko | **≈ 5-9 $** pièce (qté 1-10) | Octopart (juillet 2026) : de 5,29 $ (Worldway Electronics) à 9,81 $ (AAA Chips) selon revendeur ; comme pour la carte 1024 Ko, pas de stock direct constaté chez un distributeur de premier rang (Digikey/Mouser) — composant de niche à sourcer via revendeur ou circuit japonais. |
| U3 | **74HC138** (décodeur 3→8) | Sélection de puce sur le seul bit `A17` (128 Ko bas / 128 Ko haut) | **≈ 0,35-1,30 $** | DigiKey (juillet 2026, même donnée que `10-fabrication-carte-1024ko-fram.md`) |
| SW1 | Interrupteur glissière/DIP 2 positions | Protection en écriture (sur le modèle de l'interrupteur `Prot.`/`R/W.O` de la carte M. Kemper 256 Ko, §3.2) | **≈ 0,30-0,50 $** | Estimation (composant standard) |
| RN1 | Réseau résistif pull-up (`A15`-`A17`) | Compatibilité machines aux lignes flottantes (`09-cartes-meres-ram-interne.md` §4-5) | **≈ 0,20-0,40 $** | Estimation |
| C1-C2 | Condensateurs céramique 100 nF ×2 | Découplage par puce | **≈ 0,10 $** total | Estimation |
| PCB | Carte 2 couches, **42×54×3 mm** (dimensions confirmées par la fiche produit ci-dessus), contacts dorés | Support + connecteur | **≈ 8-15 $/carte** en série de 5 | JLCPCB (base publique) + supplément gold fingers non confirmé par devis réel |
| — | Coque plastique (optionnel) | Protection mécanique | **0-3 $** (récupérée ou imprimée en 3D) | Estimation |
| — | Assemblage CMS (optionnel) | TSOP-48 pas 0,5 mm | **≈ 10-20 $/carte** en petite série | Estimation |

**Total estimé par carte (auto-assemblée, série de 5, hors port/douane) : ≈ 20-30 $** (≈ 18-27 €) — sensiblement **moins cher que le produit commercial japonais (~80 €)**, l'écart s'expliquant par la marge commerciale, le conditionnement (boîtier injecté, notice), et le fait que le vendeur assemble/teste chaque carte individuellement. Avec assemblage CMS sous-traité : **≈ 30-45 $/carte**, ce qui reste inférieur au prix de vente japonais.

## 2. Schéma fonctionnel proposé

*(voir le diagramme affiché dans la conversation)*

Bien plus simple que la carte 1024 Ko, cohérent avec l'absence de commande `POKE` documentée pour ce produit :

1. **Bus d'adresse/données** : `A0`-`A17`, `D0`-`D7`, `CE1`, `RD`, `WR` arrivent du connecteur directement à `U3` et aux deux puces FRAM.
2. **Décodeur (`U3`, 74HC138)** : un seul bit d'adresse, `A17` (le plus significatif de la fenêtre `CE1` de 256 Ko), suffit à choisir entre les deux puces de 128 Ko — pas de registre à charger, la sélection est purement combinatoire et instantanée (contrairement à la carte 1024 Ko où le choix de banc doit être mémorisé par `POKE`).
3. **Protection en écriture (`SW1`)** : interrupteur physique en série sur la ligne `WR` (ou sur une entrée d'inhibition dédiée des puces FRAM), à l'identique de la carte M. Kemper 256 Ko (§3.2) plutôt qu'un bit de registre logiciel — plus simple et cohérent avec la description « protection par interrupteur » de la fiche produit japonaise.
4. **Pull-up `A15`-`A17`** (`RN1`) : même recommandation que pour la carte 1024 Ko (`09-cartes-meres-ram-interne.md` §4-5).

**Différence clé avec la carte 1024 Ko** : aucune logique de mémorisation (`74HC175`), aucune logique de combinaison (`74HC02`), aucun décodage de l'adresse spéciale `10000H` — donc moins de composants, moins de risque lié à l'hypothèse de connecteur non vérifiée (seul le décodage simple par `A17` + les signaux de bus standard sont nécessaires, pas de circuit supplémentaire dépendant d'un accès en dehors de la fenêtre `CE1`).

## 3. Proposition de PCB

- **Format** : **42×54×3 mm**, désormais confirmé par une fiche produit commerciale réelle (§0) plutôt que par déduction seule — légèrement différent des 54×42×2,8 mm cités pour les cartes Sharp officielles, à considérer comme une variance normale de fabricant tiers plutôt qu'une incompatibilité.
- **Empilage** : 2 couches, largement suffisant (encore moins de composants que la carte 1024 Ko).
- **Finition de bord** : contacts dorés (« gold fingers »), identique à `10-fabrication-carte-1024ko-fram.md` §3.
- **Placement proposé** : bord doré → `U3` (74HC138) et `SW1` (interrupteur, accessible sur la tranche de la carte pour rester actionnable une fois la carte insérée, comme sur la carte M. Kemper 256 Ko) → les deux FRAM `U1`/`U2` côte à côte → `RN1` proche des broches `A15`-`A17` du connecteur → découplage au plus près de chaque puce.
- **Prérequis bloquant identique** : brochage du connecteur à vérifier par traçage de continuité (`08-cartes-memoire.md` §6 point 1) avant de dessiner l'empreinte connecteur dans un outil de CAO.

## 4. Voir aussi

- `08-cartes-memoire.md` §3.2, §4, §5, §6 — carte M. Kemper 256 Ko (schéma de référence), carte FRAM commerciale, candidats FRAM, vérifications recommandées.
- `10-fabrication-carte-1024ko-fram.md` — même exercice pour la carte 1024 Ko (registre de banque, coût plus élevé).
- `09-cartes-meres-ram-interne.md` — transformation 32K→256K de la RAM interne (à recouper avec le service commercial découvert en §0).
- `07-sources-et-bibliographie.md` — fiche produit `tmfg.jp`.
