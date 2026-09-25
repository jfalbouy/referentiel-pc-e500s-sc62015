# Cartes mémoire — officielles, tierces, et pistes FRAM

*Rédigé le 2026-08-26 — mis à jour le 2026-09-25*

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier documente les cartes mémoire connues pour PC-E500/PC-E500S (Sharp et tierces), avec un inventaire technique des **cinq cartes tierces** photographiées dans `Photos Carte memoire Sharp/` (128 Ko, 256 Ko ×3 fabricants indépendants, 1024 Ko), dans le but de préparer la fabrication de nouvelles cartes 256 Ko / 512 Ko / 1024 Ko, si possible en FRAM.
>
> **Avertissement méthodologique** : au-delà de ce qui est directement lisible sur les cartes (marquages de composants) ou documenté dans un manuel Sharp, plusieurs points ci-dessous (mécanisme exact de bascule de banque, brochage complet du connecteur) restent des **hypothèses techniques argumentées**, pas des certitudes — ce référentiel n'a pas trouvé de brochage connecteur ni de schéma Sharp publiés. Ces points sont marqués « **à vérifier avant fabrication** » et la vérification recommandée (traçage de continuité sur une carte réelle possédée par l'utilisateur) est indiquée.

## 1. Ce que la mémoire card doit respecter côté CPU

Rappel de `01-architecture-cpu-sc62015.md` et complété par le brochage du SC62015 documenté par Andrew Woods (1995, cf. `07-sources-et-bibliographie.md`) :

| Signal | Rôle |
|---|---|
| `A0`-`A18` | Bus d'adresses externe — **19 lignes**, donc **512 Ko (2¹⁹ octets) adressables nativement** dans une même zone de sélection de boîtier. Confirmé indépendamment par le forum silicium.org (§1bis) : une carte 256 Ko utilise notamment `A15`-`A17` sans résistances de tirage intégrées. |
| `CE0`-`CE7` | 8 sorties de sélection de boîtier (*chip select*) — répartissent l'espace externe de 1 Mo. Table précise par adresse désormais disponible en `03-memoire-et-systeme-pc-e500s.md` §1bis (source : forum silicium.org, voir §1bis ci-dessous). |
| `DIO0`-`DIO7` | Bus de données 8 bits. |
| `MRQ`, `RD`, `WR` | Contrôle mémoire (requête, lecture, écriture). |

**Conséquence directe pour la conception d'une carte** : une seule zone de sélection (`CE`) ne peut adresser que 512 Ko au maximum avec les 19 lignes d'adresse disponibles. C'est cohérent avec :

- la puce FRAM `MB85R4001A` (512 Ko, voir §5) qui utilise exactement 19 lignes d'adresse — un remplacement « naturel » à ce format ;
- le fait que toute carte visant **plus de 512 Ko** (la carte M. Kemper/Dynatech ci-dessous, 1024 Ko) **doive nécessairement ajouter de la logique de commutation de banque à bord de la carte elle-même** — le CPU seul ne peut pas adresser plus de 512 Ko par zone `CE`.

### 1bis. Correction : un seul slot physique, mais trois zones `CE` distinctes

Le PC-E500S ne dispose que d'**un seul emplacement physique** de carte à l'arrière (« RAM card slot in back (Half-size) », confirmé par `sharppocketcomputers.com`). La confusion à éviter (corrigée ici grâce au fil « PC-E500S 256K » du forum silicium.org, voir `09-cartes-meres-ram-interne.md` §4) : **`S1:`, `S2:` et `S3:` ne sont pas trois emplacements physiques**, mais trois zones `CE` aux rôles très différents sur cette machine précise :

| Lecteur | Zone `CE` | Adresses | Nature physique |
|---|---|---|---|
| `S1:` (et lecteur `E:`) | `CE0` | `80000H`-`BFFFFH` | **RAM interne soudée sur la carte mère** (32 Ko ou 256 Ko selon le modèle, voir `09-cartes-meres-ram-interne.md`) — **pas amovible**. |
| `S2:` (et lecteur `F:`) | `CE1` | `40000H`-`7FFFFH` | **L'unique emplacement de carte amovible** à l'arrière de l'appareil — celui que visent les cartes tierces de ce fichier. |
| `S3:` (et lecteur `G:`) | `CE2` | `C0000H`-`FFFFFH` | ROM système — non amovible sur PC-E500(S). |

C'est donc bien **une seule zone `CE` (`CE1`, `S2:`) qui correspond au connecteur physique de carte** ; `S1:` désigne la RAM interne et n'est un « emplacement de carte » que par analogie logicielle. Cette contrainte d'un seul connecteur amovible est ce qui pousse les fabricants tiers à empiler plusieurs bancs de mémoire *derrière* de la logique active de la carte elle-même plutôt que d'utiliser plusieurs cartes simples en parallèle.

## 2. Cartes officielles Sharp

Gamme demi-format (« half-size », compatible PC-E500/PC-E500S), d'après `sharppocketcomputers.com/ram_cards.htm` :

| Modèle | Capacité | Notes |
|---|---|---|
| CE-210M | 2 Ko | |
| CE-211M | 4 Ko | |
| CE-212M | 8 Ko | |
| CE-2H16M | 16 Ko | |
| CE-2H32M | 32 Ko | Confirmée par plusieurs annonces/collections (Arno Welzel, Worthpoint). |
| CE-2H64M | 64 Ko | |

Caractéristiques communes : alimentation 3 V (batteries de l'appareil quand la carte est installée), pile de sauvegarde **CR-1616** (⚠ à ne pas confondre avec la pile principale de l'appareil, **CR-2016**, qui alimente la RAM interne du PC-E500S lui-même), dimensions 54×42×2,8 mm, autonomie de sauvegarde hors appareil ≈ 12 mois. Gamme plein format antérieure (CE-201M/202M/203M, 8/16/32 Ko) électriquement équivalente mais deux fois plus large — non compatible mécaniquement avec le PC-E500(S).

Au-delà de 64 Ko, Sharp n'a pas documenté de carte demi-format officielle dans cette gamme numérotée ; la spécification d'Andrew Woods (1995) mentionne toutefois une variante 128 Ko en option pour le PC-E500, sans référence commerciale précise. Le PC-E500S annonce officiellement jusqu'à **256 Ko** pour sa carte (Arno Welzel, Wikipedia) — c'est-à-dire l'emplacement **`S2:`** : ⛔ une version antérieure écrivait `S1`, qui est la RAM interne (§1bis) : au-delà de 64 Ko documenté par la gamme CE-2Hxx, **le marché s'est donc reporté sur des cartes tierces** — objet du §3.

## 3. Cartes tierces documentées (photos utilisateur, dossier local `Photos Carte memoire Sharp/`)

Cinq cartes distinctes ont été photographiées et démontées. Elles illustrent la progression technique nécessaire pour dépasser la limite officielle de 64 Ko : simple puce unique (128 Ko), deux puces + décodeur avec protection matérielle (256 Ko, chez **trois fabricants indépendants**, tous basés à Aix-la-Chapelle/Aachen), puis deux puces + décodeur avec commutation de banque logicielle (1024 Ko).

> **Correction (regroupement des photos)** : une première lecture avait à tort séparé `IMG_2415` (étiquette Dynatech) de `IMG_2407`/`IMG_2416`/`IMG_2417` (carte à deux puces Samsung) en les traitant comme deux cartes différentes. Il s'agit en réalité d'une **seule et même carte**, photographiée sous plusieurs angles/faces : `IMG_2416` et `IMG_2417` montrent la face composants (carte nue), `IMG_2407` montre la carte installée dans son boîtier noir (loquet `LOCK`, vis), et `IMG_2415` montre la face arrière, où est apposée l'étiquette du revendeur **Dynatech**. Voir §3.3.

### 3.1 « CC Sharp Card » (1992) — 128 Ko

*Photos : `IMG_2410.jpg`, `IMG_2411.jpg`, `IMG_2412.jpg`, `IMG_2414.jpg` (mêmes composants, faces/angles différents — `IMG_2414` montre la carte retournée).*

Carte nue (sans coque plastique), sérigraphie au dos « **CC Sharp Card 23.11.92** » + marque « LS » et logo losange « N ». Composants côté face :

| Réf. | Marquage | Fonction |
|---|---|---|
| U1 (mémoire) | `MITSUBISHI 42010A M5M51008AFP-70L` | SRAM CMOS 128K×8 (1 Mbit), boîtier SOP, temps d'accès 70 ns. **Capacité de la carte : 128 Ko.** |
| U2 | `TIF327ER HC04` | Hex inverseur (façon 74HC04) — vraisemblablement utilisé pour l'oscillateur de contrôle de la pile de sauvegarde / détection de retrait. |
| — | 2× condensateur `104` (100 nF) + 1× `33`/`6V` (tantale 33 µF/6,3 V) | Découplage alimentation. |
| Pile | Bouton lithium (type CR, non identifié précisément) | Sauvegarde hors appareil. |
| — | Fils de réparation visibles (`IMG_2411`) | La carte a été réparée/modifiée (au moins un pont soudé côté pile/oscillateur). |

Une seule puce mémoire suffit ici (128K×8 = 128 Ko, largement sous la limite native de 512 Ko du bus).

### 3.2 Carte « (C) M. Kemper 4.93 » — 256 Ko, avec protection en écriture matérielle

*Photos : `531135.jpg` (face composants), `594245.jpg` (face sérigraphiée, identité du fabricant).*

Carte nue, sérigraphie au dos très explicite : **« (C) M. Kemper 4.93 »**, adresse **« Hanbrucher Str. 46, 52064 Aachen »**, téléphone **« Tel. 0241/75118 »** — un fabricant/concepteur individuel allemand basé à Aix-la-Chapelle, carte datée **avril 1993**. Composants côté face :

| Réf. | Marquage | Fonction |
|---|---|---|
| U1, U2 (mémoire) | 2× `SAMSUNG KM681000ALG-7 422C KOREA` | SRAM CMOS **128K×8 chacune** (1 Mbit), boîtier SOP. **2 puces → 256 Ko au total.** |
| U3 | `74HC138D` (marquage complet lisible : `917560Q`) | Décodeur 3 vers 8 — sélectionne laquelle des deux puces de 128 Ko répond à une adresse donnée (les deux puces se partagent une fenêtre unique de 256 Ko, sous la limite native de 512 Ko : **pas besoin de registre de banque, un simple décodage d'adresse suffit**). |
| — | Condensateurs `473` (47 nF) et `125` (1,2 µF, ou code similaire) + composants CMS non identifiés | Découplage. |
| **Interrupteur physique** | Repéré « **Prot.** » et « **R/W.O** » sur le PCB | **Commutateur mécanique à 2 positions pour la protection en écriture** — contrairement à la carte Dynatech (§3.3) qui protège par registre logiciel (`POKE`), cette carte 256 Ko utilise un **DIP switch physique**, actionné directement sur la tranche de la carte. |
| Pile | Bouton lithium (type CR, format plat rectangulaire visible) | Sauvegarde hors appareil. |

**Lecture technique** : avec seulement 256 Ko (< 512 Ko, la limite native du bus 19 bits, §1), aucune commutation de banque n'est nécessaire — le `74HC138` ne fait que du décodage d'adresse classique entre les deux puces pour former une fenêtre continue de 256 Ko. C'est la démonstration, sur une carte réelle, du seuil identifié en §1 : en dessous de 512 Ko, un simple décodeur suffit ; au-delà (§3.3), il faut un registre de banque piloté par logiciel.

### 3.3 « (C) M.K. » / DYNATECH « Aktive 512/1024-KB-Karte für SHARP PC-E500/S » — 1024 Ko

*Photos : `IMG_2407.jpg` (en coque), `IMG_2416.jpg`/`IMG_2417.jpg` (carte nue, face composants), `IMG_2415.jpg` (face arrière, étiquette Dynatech).*

Carte montée dans une coque noire de format Sharp standard (loquet `LOCK`, vis, forme identique aux cartes officielles), marquage PCB « **(C) M.K.** » — très probablement le même **M. Kemper** que la carte 256 Ko du §3.2 (mêmes initiales, même famille de composants, même style de sérigraphie), revendue ou distribuée sous la marque allemande **Dynatech** (`www.dynatech.de`), dont l'étiquette produit est collée au dos :

```
- DYNATECH -
www.dynatech.de   [CR1220]
1024 KB Karte SHARP PC-E500S
D0 + D1: Kart.Nr.; D2: Card OFF
D3: Protect; Poke 65536,ΣDx
```

Composants côté face :

| Réf. | Marquage | Fonction |
|---|---|---|
| U1, U2 (mémoire) | 2× `SAMSUNG K6X4008C1F-GF55` | SRAM CMOS **512K×8 chacune** (4 Mbit), boîtier TSOP2. **2 puces → 1024 Ko (1 Mo) au total.** |
| U3 | `[F] MM74HC02M` (marquage partiel `P15SB`) | Portes NOR — logique de sélection/activation, probablement associée au registre de banque. |
| U4 | `74HC138` (marquage `...306LC HC138`) | Décodeur 3 vers 8 — même rôle de sélection de puce que sur la carte 256 Ko (§3.2), mais ici combiné à un **registre de banque** (voir ci-dessous) puisque 1024 Ko dépasse la limite native de 512 Ko. |
| — | 2× réseau résistif `2203` (22 kΩ) + `4702` (47 kΩ), ×5 et ×2 | Pull-up/pull-down sur les lignes de commande. |
| Pile | **CR1220** (confirmé par l'étiquette Dynatech) | Sauvegarde hors appareil. |

Éléments de fonctionnement documentés directement par l'étiquette Dynatech :

- **Registre de contrôle accessible depuis BASIC par `POKE 65536,valeur`** (adresse décimale 65536 = `10000h`), valeur = somme de bits `Dx` :
  - `D0` + `D1` (2 bits → 4 combinaisons) = **numéro de carte/banc** ;
  - `D2` = **Card OFF** (désactivation de la carte) ;
  - `D3` = **Protect** (protection en écriture, ici *logicielle* — à comparer avec l'interrupteur *physique* de la carte 256 Ko du même fabricant, §3.2).
- Existe en deux capacités déclarées sur le **même PCB** (« 512/1024 »), le peuplement (nombre de puces mémoire montées) déterminant la capacité finale, la logique de commutation restant commune aux deux versions.

**Lecture technique — confirmée par le manuel du fabricant** (`Carte memoire 1024Ko FR.docx`, `Photos Carte memoire Sharp/`, voir `07-sources-et-bibliographie.md`) : l'hypothèse « 2 bits → 4 bancs de 256 Ko » est la bonne, et n'est plus une simple hypothèse. Le manuel décrit explicitement une carte existant en deux variantes sur le **même PCB** : « **2 (512 Ko) ou au maximum 4 (1024 Ko) zones mémoire de 256 Ko** (chacune généralement sur une carte) », chaque zone de 256 Ko étant sélectionnable librement par logiciel — confirmation directe que la variante 1024 Ko = **4 bancs de 256 Ko**, pas 2 bancs de 512 Ko.

Le registre de contrôle 4 bits (`POKE 65536,valeur` / `POKE &10000,valeur`, non lisible — *write only*) est documenté avec sa table de vérité complète :

| Bit | Poids | Rôle |
|---|---|---|
| `D0` | 1 | Sélection de banc, bit de poids faible (combiné à `D1`) |
| `D1` | 2 | Sélection de banc, bit de poids fort — `D0`+`D1` codent les 4 bancs de 256 Ko (0-3) ; sur une carte 512 Ko, `D1` est sans effet (banc supplémentaire absent) |
| `D2` | 4 | **Card OFF** — un `0` désactive la carte (retour à la RAM interne seule) |
| `D3` | 8 | **Protect** — un `1` active la protection en écriture, un `0` autorise l'écriture |

Exemples donnés par le manuel : `POKE 65536,0` = banc 0, carte activée, écriture autorisée (réglage par défaut à faire avant toute reprogrammation) ; `POKE 65536,11` (1+2+0+8) = banc 3, protection en écriture activée. Le réglage doit toujours repasser par `MEM$="S1"` avant un nouveau `POKE`, et un `POKE 65536,0` restaure l'accès en cas de mauvaise combinaison.

Procédure d'usage documentée : après un `POKE` de changement de banc, il faut **éteindre puis rallumer** l'ordinateur pour que le lecteur reconnaisse la carte 256 Ko sélectionnée, puis configurer avec `MEM$="S1"` (carte seule, taille = celle du banc) ou `MEM$="S2"` (recommandé pour numéroter plusieurs bancs) ; le mode `MEM$="B"` (fusion RAM interne + banc) est présenté par le fabricant lui-même comme **« interdit »**, limité par le système d'exploitation aux disques ≤ 128 Ko — cohérent avec la limite empirique « Out of memory » au-delà de ~256-288 Ko déjà observée indépendamment sur le forum silicium.org (`09-cartes-meres-ram-interne.md` §4).

### 3.4 « Böttcher Datentechnik » — 256 Ko, seconde génération du même concepteur « M.K. »

*Photos : `berlin1.jpg` (face composants, carte nue), `berlin2.jpg` (face arrière, étiquette revendeur).*

Carte nue (sans coque plastique), sérigraphie au dos : **« (C) M.K. 10.98 Vers. 4.1 »** + **« 128/256 KB Ramkarte fuer SHARP »** — soit le **même concepteur « M.K. »** que les cartes du §3.2 (« M. Kemper », 1993) et du §3.3 (Dynatech, 1024 Ko), mais une **révision ultérieure** (octobre 1998, version 4.1 du PCB) de la carte 256 Ko, ici revendue sous une étiquette collée d'un revendeur différent : **« Böttcher Datentechnik — 256 KB Speicherkarte für den PC-E500/S »**. Le PCB porte aussi, imprimé, « **Lithium-Zelle CR1216 / 3Volt** » (pile de sauvegarde) et « **Bottom L.** » (repère de face).

Composants côté face :

| Réf. | Marquage | Fonction |
|---|---|---|
| U1, U2 (mémoire) | 2× `V62C5181024L-70W` (marquage `0019DN`) | SRAM CMOS **128K×8 chacune** (1 Mbit), boîtier SOP, 70 ns. **2 puces → 256 Ko au total.** — **même référence exacte** que les puces de la carte mère interne 256 Ko documentée en `09-cartes-meres-ram-interne.md` §2, confirmation supplémentaire que ce composant était une source SRAM 128K×8 courante à la fin des années 1990. |
| U3 | Boîtier SOP-16, marquage partiel peu lisible (proche de `7MC1300`/`D9901PL`) | Très probablement un **décodeur 74HC138** (même rôle qu'en §3.2) : le boîtier, le nombre de broches et la position sur le PCB (entre les deux puces mémoire) sont cohérents avec ce rôle, mais le marquage n'a pas pu être confirmé avec certitude sur la photo. |
| — | Petits composants CMS non identifiés (résistances/condensateurs de découplage) | Découplage/logique annexe. |
| **Interrupteur/pont** | Repéré « **Prot.** » et « **R./W.** » avec flèches, near un plot métallique soudé | **Même terminologie exacte** que la carte M. Kemper 4.93 (§3.2) pour la protection en écriture — confirme qu'il s'agit bien de la même lignée de conception, cette fois-ci avec un **pont/plot soudé** plutôt qu'un DIP switch à glissière (variante d'implémentation entre versions du PCB, fonction identique). |
| Pile | **CR1216** (confirmé par la sérigraphie du PCB) | Sauvegarde hors appareil — ⚠ **différente** de la `CR-1616` des cartes Sharp officielles (§2) et de la `CR1220` de la carte Dynatech 1024 Ko (§3.3) : trois formats de pile bouton différents selon la carte/le fabricant, à bien vérifier avant tout remplacement. |
| Connecteur | Contacts dorés visibles en bord de carte (carte nue, sans coque) | Carte non protégée par un boîtier plastique sur cette photo — **utile pour un futur traçage de continuité** (§6 point 1), le connecteur étant entièrement visible et non masqué par une coque. |

**Lecture technique** : cette carte confirme, de façon indépendante et à 5 ans d'écart (1993 → 1998), le schéma déjà identifié en §3.2 pour une carte 256 Ko tierce : deux puces SRAM 128K×8 + un décodeur simple + une protection en écriture matérielle, sans registre de banque (256 Ko restant sous la limite native de 512 Ko, §1). Le même concepteur (« M.K. ») a donc fait évoluer sa carte 256 Ko sur au moins deux générations de composants (Samsung `KM681000ALG-7` en 1993 → `V62C5181024L-70W` en 1998) et l'a vue redistribuée sous au moins **deux marques revendeur différentes** (aucune étiquette revendeur visible sur l'exemplaire de 1993 du §3.2 ; « Böttcher Datentechnik » sur celui-ci ; « Dynatech » sur la carte 1024 Ko du §3.3) — cohérent avec un modèle où un concepteur/fabricant individuel produisait les cartes et laissait plusieurs revendeurs les distribuer sous leur propre étiquette.

### 3.5 « Becker & Partner » (Aachen) — 256 Ko, conception indépendante

*Photos : `s-l1600.png` (face composants, carte nue), `s-l1601.png` (face arrière, carte nue — vue complète et dégagée du connecteur à contacts dorés et de son routage).*

Carte nue, sérigraphie « **Becker & Partner — Aachen-Germany** ». Contrairement aux cartes des §3.2 et §3.4 (même concepteur « M.K. » sous deux étiquettes), il s'agit ici d'une **conception distincte, d'un troisième fabricant indépendant** — mais **toujours basé à Aix-la-Chapelle (Aachen)**, comme M. Kemper (§3.2). Recherche complémentaire : **Becker & Partner GmbH Mobile Datensysteme** a réellement existé comme entreprise à Aachen (Neuenhofstr. 110, 52078 Aachen — annuaires professionnels allemands, juillet 2026), confirmant qu'il s'agissait d'une société commerciale et non d'un simple particulier, à la différence de M. Kemper. Aachen apparaît ainsi comme un véritable **pôle allemand de fabrication de cartes mémoire tierces** pour le PC-E500(S) dans les années 1990, avec au moins trois acteurs distincts identifiés dans ce référentiel (M. Kemper, Böttcher Datentechnik en tant que revendeur, et Becker & Partner).

Composants côté face :

| Réf. | Marquage | Fonction |
|---|---|---|
| U1, U2 (mémoire) | `SRM20100LTM` et `SRM20100LRM` (marquage `JAPAN A32 2708` / `JAPAN A26 ...`) | SRAM CMOS **128K×8 chacune**, fabricant **Seiko Instruments Inc.** (confirmé par recherche : famille `SRM20100`, boîtier SOP) — **2 puces → 256 Ko au total**. Les suffixes `LTM`/`LRM` différents suggèrent des variantes de brochage miroir entre les deux puces, une astuce classique pour simplifier le routage entre composants adjacents. |
| U3 | Boîtier SOP-8, marquage partiel (`G3`/`63`) | Petit composant logique/support non identifié avec certitude — **pas de décodeur 16 broches visible comme le `74HC138` des cartes §3.2/§3.4**, ce qui suggère un mécanisme de sélection de puce plus minimal ou différent. **À vérifier avant fabrication.** |
| — | Composant CMS marqué `22R`/`228` + petits composants appairés (probablement diodes ou résistances) | Rôle non déterminé avec certitude à partir de la seule photo. |
| **Pile** | **Absente** — aucun support de pile bouton visible sur la carte (ni plot cruciforme comme §3.2/§3.4, ni support à ressort). Le petit plot métallique rond en bas du PCB est trop simple/petit pour être un contact de pile ; il s'agit plus probablement d'un point de test ou d'une masse. | **Aucune sauvegarde locale.** Voir « Conséquence pratique » ci-dessous. |

**Apport particulier de `s-l1601.png`** : cette photo montre la face arrière **nue et dégagée** (sans coque, sans composants) avec le **connecteur à contacts dorés entièrement visible et son routage individuel vers des vias** (environ 30-32 contacts dénombrables sur la photo, à confirmer par un comptage précis). C'est la vue la plus exploitable obtenue jusqu'ici pour la vérification recommandée en §6 point 1 (traçage de continuité connecteur ↔ signal) : elle permet de voir individuellement quel contact rejoint quel via, une étape préalable utile avant de relier ces vias aux broches des puces mémoire.

**Conséquence pratique de l'absence de pile** : les puces `SRM20100L` sont du SRAM **volatile** (confirmé par la fiche Seiko Instruments) — sans pile locale, leur contenu ne peut être maintenu que par une alimentation externe continue. Tant que la carte reste **installée** dans un PC-E500S dont les piles/l'accumulateur sont en bon état, elle est vraisemblablement alimentée en permanence via le connecteur par le même circuit de maintien que celui qui préserve la RAM interne `S1:` hors tension apparente de l'appareil (`09-cartes-meres-ram-interne.md`). Mais **au retrait de la carte, plus rien ne l'alimente : le contenu SRAM serait perdu quasi instantanément** — à la différence des cartes Sharp officielles (§2) et des cartes M. Kemper/Böttcher/Dynatech (§3.2-§3.4), toutes équipées d'une pile locale précisément pour survivre au retrait. Cette carte semble donc conçue pour une **extension mémoire semi-permanente** (laissée en place) plutôt que pour un usage de carte de données transportable/échangeable — une limitation fonctionnelle réelle, pas juste un détail de construction, et une différence à ne pas reproduire si l'objectif d'une nouvelle carte est la portabilité des données.

**Lecture technique** : cette carte confirme, avec un **troisième concepteur indépendant**, que le schéma « 2 puces SRAM 128K×8 = 256 Ko » est bien la solution dominante du marché tiers allemand pour cette capacité (aux côtés des §3.2 et §3.4). Elle introduit cependant deux incertitudes : le mécanisme exact de sélection entre les deux puces n'est pas visiblement un `74HC138` classique, et l'absence de pile en fait un cas particulier moins directement transposable — deux points à éclaircir avant de considérer cette carte comme un troisième schéma de référence équivalent aux précédents pour un usage de carte amovible classique.

## 4. Carte FRAM commerciale moderne (référence de faisabilité)

Une carte FRAM 256 Ko pour la série PC-E500/E650 est commercialisée par un vendeur japonais (produit trouvé chez `tmfg.jp`, également revendu sur eBay ; mentionnée par Arno Welzel, `07-sources-et-bibliographie.md`) : elle ne nécessite **aucune pile de sauvegarde**, se formate comme une carte RAM classique (`MEM$="S1"` puis `INIT "F:255K"`) et tient dans le même connecteur/format que les cartes SRAM d'origine. C'est la preuve par l'existant que le remplacement SRAM+pile → FRAM sans pile est mécaniquement et électriquement viable sur ce connecteur, ce qui valide l'approche recherchée par l'utilisateur.

## 5. Puces FRAM candidates pour fabriquer 256 Ko / 512 Ko / 1024 Ko

Famille **Fujitsu/Cypress/Infineon `MB85R`** (FRAM parallèle, interface asynchrone type SRAM — « pseudo-SRAM », remplacement fonctionnel direct de SRAM, sans pile, ≥ 10 ans de rétention, ≥ 10¹⁰ cycles d'écriture) :

| Capacité cible | Puce candidate | Organisation | Alimentation | Boîtier | Lignes d'adresse | Schéma carte à répliquer |
|---|---|---|---|---|---|---|
| 128 Ko | Famille `MB85R1001` / équivalent 1 Mbit parallèle (ex. gamme Cypress « 1 Mbit, 128K×8, 60 ns ») | 128K×8 | ≤ 3,6 V | TSOP-32 | A0-A16 | CC Sharp Card (§3.1), puce unique — remplacement direct de `M5M51008AFP`. |
| **256 Ko** | **2× puce 128 Ko** (même famille que ci-dessus) **+ décodeur simple** | 2×128K×8 | ≤ 3,6 V | TSOP-32 ×2 | A0-A17, décodage simple (pas de banc) | **Cartes M. Kemper 256 Ko (§3.2), Böttcher/M.K. v4.1 (§3.4) et Becker & Partner (§3.5)** — schéma confirmé par **trois exemplaires physiques indépendants** de fabricants différents (1993, 1998, non daté) : remplacer les 2 puces SRAM (`KM681000ALG-7`, `V62C5181024L-70W` ou `SRM20100L` selon le fabricant) par 2× FRAM 128 Ko. Les cartes §3.2/§3.4 utilisent un `74HC138` identifiable et un interrupteur/pont physique `Prot.` réutilisable tel quel ; la carte Becker & Partner (§3.5) utilise un mécanisme de sélection différent, non encore identifié avec certitude — à clarifier avant de s'en inspirer. |
| **512 Ko** | **`MB85R4001A`** (Fujitsu/Ramtron/Cypress, 4 Mbit) — **correspond exactement aux 19 lignes d'adresse natives du SC62015 (§1) : candidat idéal pour un remplacement à puce unique** | 512K×8 | 3,0-3,6 V | TSOP-48 | A0-A18 | Remplacement 1:1 d'une des deux puces `K6X4008C1F` de la carte 1024 Ko (§3.3), si le brochage est compatible (à vérifier). |
| 1024 Ko (1 Mo) | Famille `MB85R8Mxx` (8 Mbit, 1M×8) *(vérifier disponibilité en boîtier à broches, souvent en FBGA)*, sinon **2× `MB85R4001A` (512 Ko) + `74HC138` + registre de banque** | 1M×8 ou 2×512K×8 | 3,0-3,6 V | FBGA-48 ou 2×TSOP-48 | A0-A19 → **nécessite un bit de banc**, comme sur la carte M. Kemper/Dynatech (§3.3) | Carte M. Kemper/Dynatech 1024 Ko (§3.3) — schéma le plus proche : remplacer les 2× `K6X4008C1F` par 2× `MB85R4001A`, conserver `74HC138` + NOR + registre de banque piloté par `POKE`. |

Points de vigilance avant de finaliser un choix de puce :

1. **Vérifier le brochage exact** de la puce FRAM retenue contre celui de la SRAM qu'elle remplace (`KM681000ALG-7` pour un projet basé sur la carte M. Kemper 256 Ko, `K6X4008C1F` pour un projet basé sur la carte 1024 Ko, `M5M51008AFP` pour un projet basé sur la carte CC Sharp Card) — le marketing « pin-compatible » des FRAM `MB85R` vise en priorité les SRAM JEDEC standard de même organisation, mais le boîtier exact doit être confirmé sur les fiches techniques avant routage.
2. **Alimentation** : la carte fonctionne sur les 3 V fournis par l'appareil hôte quand elle est installée — vérifier que la puce FRAM choisie fonctionne bien dans la plage 2,7-3,6 V réellement disponible (les piles AAA du PC-E500S peuvent descendre sous 4×1,5 V nominal).
3. **Suppression de la pile de sauvegarde** : avec une FRAM, la pile bouton (CR-1616/CR1220/CR2032 selon la carte) devient **inutile et peut être omise** — de même que toute logique associée à sa surveillance (l'inverseur `TIF327ER` de la carte CC Sharp Card, par exemple). Sur la carte 256 Ko, l'**interrupteur physique de protection en écriture** (`Prot.`/`R/W.O`) reste en revanche pertinent et peut être conservé tel quel avec une FRAM.
4. **Bancs > 512 Ko** : conserver un décodeur `74HC138` + registre de banque, sur le modèle éprouvé de la carte 1024 Ko (§3.3), plutôt que d'inventer un nouveau mécanisme — mais **caractériser d'abord précisément** son fonctionnement (§6) pour rester compatible avec d'éventuels pilotes/logiciels existants qui s'attendraient au protocole `POKE 65536` de Dynatech.
5. **En dessous de 512 Ko (128/256 Ko)**, pas besoin de registre de banque du tout : le schéma à décodeur simple de la carte M. Kemper 256 Ko (§3.2) — nettement plus simple à répliquer — suffit.

## 6. Vérifications recommandées avant fabrication

Comme indiqué en préambule, plusieurs points mériteraient d'être confirmés par la mesure directe sur les cartes déjà en possession (`Photos Carte memoire Sharp/`) et sur l'appareil réel, avant de lancer un nouveau PCB :

1. **Brochage du connecteur d'extension** : tracer à l'ohmmètre/continuité, depuis les contacts dorés visibles sur le dos de la carte « CC Sharp Card » (`IMG_2412.jpg`) jusqu'aux broches des puces mémoire/logique, la correspondance contact ↔ signal (`A0`-`A18`, `DIO0`-`DIO7`, `CE`, `RD`/`WR`, `VCC`/`GND`, pile). Cela donne un brochage de référence directement réutilisable, sans dépendre d'un schéma Sharp introuvable en ligne. **C'est l'étape la plus simple et la plus rentable à faire en premier**, la carte 256 Ko de M. Kemper (§3.2) étant un excellent second point de comparaison (décodage simple, sans registre de banque, donc plus facile à lire). La photo `s-l1601.png` (carte Becker & Partner, §3.5) est la vue la plus exploitable obtenue à ce jour pour ce traçage : connecteur entièrement dégagé, routage individuel de chaque contact visible.
2. **Décodeur `74HC138` de la carte 256 Ko (M. Kemper §3.2, ou Böttcher/M.K. v4.1 §3.4 — deux exemplaires disponibles pour comparaison)** : relever ses entrées de sélection — c'est le cas le plus simple (pas de registre de banque), donc le plus rapide à caractériser complètement, et il valide le principe de décodage commun aux cartes tierces. La carte Böttcher (§3.4) étant nue (sans coque), son connecteur est directement accessible pour ce traçage.
3. ~~Mécanisme de commutation de banque de la carte 1024 Ko (M.K./Dynatech, §3.3)~~ — **confirmé par le manuel du fabricant** (§3.3) : `D0`+`D1` codent bien 4 bancs de 256 Ko, `D2` = Card OFF, `D3` = Protect. Reste néanmoins à vérifier physiquement (oscilloscope/analyseur logique) **quelle ligne d'adresse/CE est réellement activée en interne** par l'écriture à `65536` (`10000H`), le manuel documentant le comportement logiciel mais pas le schéma électronique interne du décodeur.
4. **Confirmer la capacité réellement reconnue par le firmware** : la capacité de la carte est `cpSlot1` en `BFC12` (`03-memoire-et-systeme-pc-e500s.md` §4, en unités de 2 Ko) — ⛔ une version antérieure citait `cpSlot0`, qui est la capacité de `S1:`, la RAM **interne**. Ce champ suffit largement à coder 1024 Ko (512 × 2 Ko) — donc la limite n'est pas côté table système, mais bien côté bus physique (§1), ce qui confirme que la logique active à bord de la carte est le bon niveau où intervenir.
5. **Prévoir des résistances de tirage (pull-up) sur `A15`-`A17`** sur toute nouvelle carte : un cas réel documenté (`09-cartes-meres-ram-interne.md` §4) montre qu'une carte 256 Ko sans ces résistances n'est pas reconnue sur un appareil dont ces lignes sont laissées flottantes — les intégrer sur la carte elle-même (plutôt que de compter sur l'hôte) maximise la compatibilité.
6. **Ne pas promettre le mode `MEM$="B"` (fusion RAM interne + carte) au-delà de ~256-288 Ko de total** : comportement `Out of memory` documenté (`09-cartes-meres-ram-interne.md` §4-5) sur une machine 256 Ko + carte, alors que les modes `S1`/`S2` (carte utilisée seule) restent fiables à toute capacité testée. Confirmé indépendamment par le manuel de la carte 1024 Ko (§3.3), qui présente ce mode comme explicitement interdit par le fabricant lui-même au-delà de 128 Ko.
7. **Utiliser l'autotest usine du PC-E500S pour valider une carte prototype** : combinaison `SHIFT` + réinitialisation au démarrage ouvre un menu de diagnostic dont la section **« ESDE »** teste la présence/le bon contact de la carte en slot (limité à 64 Ko car Sharp ne fournissait que des cartes de test de cette taille) — un « Pas de carte » signalé alors qu'une carte est en place indique un problème de contact ou une carte figée en protection en écriture totale (`POKE 65536,0` pour réinitialiser). Méthode rapide à essayer avant l'analyseur logique du point 3.

## 7. Voir aussi

- `01-architecture-cpu-sc62015.md` §3 — espace mémoire externe et pagination.
- `03-memoire-et-systeme-pc-e500s.md` §1bis-1ter, §4-5 — carte mémoire externe précise par ligne `CE`, tables système des emplacements de carte (`ldAdSlot`/`cpSlot`), modes `MEM$`.
- `09-cartes-meres-ram-interne.md` — carte mère 32 Ko vs 256 Ko, confirmation communautaire (forum silicium.org) du schéma 2×128 Ko et des limites réelles observées.
- `07-sources-et-bibliographie.md` — sources générales (Arno Welzel, sharppocketcomputers.com, spécification CPU d'Andrew Woods).
- Dossier `Photos Carte memoire Sharp/` — photos sources de ce fichier, **conservées hors du dépôt** (`NOTICE.md`). Les deux photos de la carte M. Kemper 256 Ko ont été reçues au format HEIC (conservées sous `531135.heic`/`594245.heic`) et converties en `531135.jpg`/`594245.jpg` pour une lecture directe. Les photos `berlin1.jpg`/`berlin2.jpg` (carte Böttcher/M.K. v4.1, §3.4) et `s-l1600.png`/`s-l1601.png` (carte Becker & Partner, §3.5) ont été ajoutées ultérieurement.
