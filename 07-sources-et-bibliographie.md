# Sources et bibliographie

> Voir `00-index.md` pour la vue d'ensemble. Provenance de chaque affirmation du référentiel, classée par fiabilité : manuels Sharp d'origine > rétro-ingénierie vérifiée sur matériel/listings réels > sites communautaires > reconstructions tierces non vérifiées.

## 1. Sources primaires — manuels Sharp

| Document | Emplacement local | Provenance |
|---|---|---|
| *ESR-L CPU Instruction Manual* (Sharp, manuel interne) | `SC62015Disassembler/Docs/Doc technique/ESR-L_CPU_tech_manual.pdf` et `Mise en forme documents SHARP/ESR-L_CPU_tech_manual - CL.docx` (version nettoyée/OCR) | Manuel Sharp jamais officiellement publié ; circule depuis une numérisation retrouvée. La piste remonte à une copie archivée sur `sarnau.info`, elle-même sauvegardée par la Wayback Machine (lien retrouvé et documenté par la communauté japonaise, voir §2 *Electrelic*). |
| *Technical Reference Manual PC-E500* (Sharp Corporation, Information Systems Group) | `SC62015Disassembler/Docs/Doc technique/TechnicalReferenceManualPC-E500.pdf` et `Mise en forme documents SHARP/TechnicalReferenceManualPC-E500 - CL.docx` | Même origine que ci-dessus (`sarnau.info` / Wayback Machine). Couvre FCS, IOCS, drivers, interruptions, mémoire de travail. |
| `XASM - ENGLISH.DOC` / `XASM140 - Information.DOC` | `XASM Origine/01- Documentation/` et `02- xasm/` | Documentation d'origine de XASM 1.40 (E. Kako, distribué via 工学社/Kōgakusha, « Poke-Con Library 2 »). |

Ces deux manuels sont la source de la quasi-totalité de `01-architecture-cpu-sc62015.md`, `02-jeu-instructions.md`, `03-memoire-et-systeme-pc-e500s.md` et `04-fcs-iocs.md`.

## 2. Rétro-ingénierie interne au projet (vérifiée sur matériel/listings réels)

| Élément | Emplacement | Nature de la vérification |
|---|---|---|
| `Data/OpcodeTable.json`, `InternalRAMNames.json`, `SystemAddresses.json`, `FCSFunctions.json`, `SystemDataRegions.csv` | `SC62015Disassembler/Data/` | Recoupés sur listings XASM réels (`register.lst` — E. Kako 1990/1992, `TMAP2020.lst` — TORO 1994, `tycom.LST`/`tydos.LST` — J.-F. Albouy 2019) et sur une image ROM PC-E500S réelle (`rom83.bin`). |
| `README - PC-E500 Instruction Table.md` | `SC62015Disassembler/Docs/Doc technique/` | Reconstruction complète (table 16×16 + tables détaillées par catégorie) à partir du manuel ESR-L. |
| `PCE500 Description mémoire.xls.xlsx` | `SC62015Disassembler/Docs/Doc technique/` | Dépouillement en français, daté 2022-12-14, vérifié sur listings réels — base principale de `03-memoire-et-systeme-pc-e500s.md`. |
| Correction *exec_addr* / *reserved* de l'en-tête objet XASM | `SC62015Disassembler/CLAUDE.md` | Vérifiée directement sur `register.obj`/`tmap.obj`/`vogue.obj` réels. |
| 249 tests de non-régression du désassembleur, reproduction 100 % de `tmap`/`vogue` | `SC62015Disassembler/Tests/` | Comparaison automatisée contre les `.lst` produits par XASM lui-même. |

## 3. Sources communautaires — Japon

Le Japon est, avec l'Allemagne, le pays où le PC-E500/PC-E500S a été le plus documenté et exploité par la scène amateur. Sources consultées :

- **[Seiji Kako — kako.com](http://www.kako.com/neta/1999-016/1999-016.html)** (加古静司) — bibliothèque logicielle de l'auteur historique de **XASM** (`xasm140.lzh`, versions DOS et Win95/98) et du désassembleur **ESR-L `DISASM12`** (« Now all we need is a DIS-ASSEMBLER » — la remarque d'Andrew Woods en 1995, voir plus bas, trouve ici sa réponse d'époque). On y trouve aussi `disbacon`/`bacon` (les outils repris par le projet `disbacon` de ce référentiel) et une version traduite en anglais de la doc XASM.
- **[Takayuki Mizuno — kt.rim.or.jp/~tmizuno](http://www.kt.rim.or.jp/~tmizuno/pocket/library/sharp01.html)** — index de bibliothèque logicielle SHARP incluant XASM/DISASM12 (miroir des fichiers de Kako), l'assembleur alternatif **A62**/**MASSE** (Nmasu), le compilateur **VOGUE** (契約 稔 / N. Kon — le même « N. Kon » crédité comme co-auteur de XASM et dont le programme `VOGUE` sert de jeu de test au désassembleur du projet), et **COMPO-System**/**SG** (TORO / 高橋 良和).
- **[TORO's Library — toro.d.dooo.jp](https://toro.d.dooo.jp/sle500.html)** — bibliothèque logicielle PC-E500 de TORO (高橋良和), dont **COMPO-System** (débogueur commande pour développement en langage machine sur E500), **Tmap** (« memory map creation », affiche l'information des slots système et des drivers — la source directe du nom du fichier de test `tmap.obj`/`TMAP2020.lst`) et **BASCOM** (compilateur BASIC autonome).
- **[Electrelic — SC61860 & SC62015 (その1)](https://electrelic.com/electrelic/node/4232)** (H. Asano, 2024) — article de reconstruction pour le désassembleur *Macroassembler AS*, incluant la piste qui a permis de retrouver les deux manuels Sharp cités en §1 (liens Wayback Machine vers `sarnau.info`), et des références à d'autres bases documentaires (*ポケコン・マシン語ブック*, *ポケコンうらわざ大事典*).
- **[wizforest.com — 魔法使いの森, « PC-E500のCPU »](https://www.wizforest.com/OldGood/PC-E500/PC-E5002.html)** (1996-1999) — analyse comparative du SC62015 avec le 6502, le 68000 et le 8086 ; source de la description « page interne rapide façon 6502 + pointeurs larges façon 68000 » reprise en `01-architecture-cpu-sc62015.md` §1.
- **[GitHub — gikonekos/sc62015-opcode-reference](https://github.com/gikonekos/sc62015-opcode-reference)** — projet de reconstruction indépendant et actif (2026), à partir de matériel imprimé historique ; utile en **contre-vérification** de la table d'opcodes, bien qu'explicitement présenté par son auteur comme non définitif.
- **[andrewwoods3d.com — PC-E500 Instruction Table](http://www.andrewwoods3d.com/pce500/insttabl.html)** — la plus ancienne table d'opcodes ESR-L connue en ligne (1995, Andrew Woods), initialement publiée pour accompagner « DUMPTOOL for SHARP PC-E500 ». Intérêt historique : c'est cette table qui appelait déjà à l'écriture d'un désassembleur.
- **[Forth500 — Robert van Engelen](https://github.com/Robert-van-Engelen/Forth500)** — système Forth complet pour PC-E500(S), inclut une traduction anglaise de la documentation XASM et des ressources techniques PC-E500 ; illustre un autre exemple de toolchain XASM + PC-E500S indépendant du projet.

## 4. Sources communautaires — Allemagne

- **[Arno Welzel](https://arnowelzel.de/en/projects/technology-museum/pocket-computers/sharp-pc-e500s)** — fiche technique et d'usage détaillée du PC-E500S (specs, cartes mémoire, modes `S1`/`S2`/`B` via `MEM$`, cartes FRAM modernes, interfaces série et adaptateur USB) — source de `03-memoire-et-systeme-pc-e500s.md` §5.
- **[Anton Thimet — CalcCollection](https://www.thimet.de/CalcCollection/Calculators/Sharp-PC-E500/Contents.htm)** — fiche de collection PC-E500/PC-E500S (specs précises, manuel allemand scanné 341 pages) avec une liste de liens vers d'autres ressources (Pocketcom Journal d'Andrew Woods, Silrun Systems, pages de Daisuke Mizobata, Johannes Heimansberg, Simon Lehrmayr, sharp-pc-1600.de).
- **[Peil & Partner — PocketTools](https://www.peil-partner.de/ifhe.de/sharp/)** — suite d'outils en ligne de commande (`Wav2bin`, `Bin2wav`, `Bas2img`) pour la communication PC ↔ pocket computers Sharp, utilisée notamment par le projet `Forth500` (§3) pour le chargement de programmes.

## 5. Autres sources

- **[Wikipedia — Sharp PC-E500S](https://en.wikipedia.org/wiki/Sharp_PC-E500S)** — repères généraux (date de sortie 1995, cadence CPU 2,304 MHz, mémoire).
- **[Andrew Woods — Sharp PC-E500 Pocket Computer Specifications](http://www.andrewwoods3d.com/pce500/e500spec.html)** (1995) — fiche technique complète incluant le brochage du SC62015 (`A0`-`A18`, `CE0`-`CE7`, `DIO0`-`DIO7`...) et la liste des cartes RAM officielles d'origine (CE-212M à 128 Ko) ; source principale de `08-cartes-memoire.md` §1.
- **[sharppocketcomputers.com — RAM Memory Cards](https://sharppocketcomputers.com/ram_cards.htm)** et **[fiche PC-E500/E500S](https://sharppocketcomputers.com/pce500.htm)** — gamme complète des cartes demi-format Sharp (CE-210M à CE-2H64M), caractéristiques électriques et mécaniques ; composants internes du PC-E500S (ROM `LH5320XD`, RAM interne `HM62256`). Source principale de `08-cartes-memoire.md` §2.
- Manuels utilisateur BASIC (hors périmètre strict SC62015, utiles pour le contexte) : `Renum Basic Sharp/Documentation/PC-E500 manual_EN.pdf`, `PC-E500S-DE.pdf` (allemand), `Codes_BASIC_PC-E500S.md`.
- **Photos personnelles de cartes mémoire** (`Referentiel PC-E500S SC62015/Photos Carte memoire Sharp/`) — trois cartes tierces démontées et photographiées par l'utilisateur : « CC Sharp Card » (1992, 128 Ko), carte **M. Kemper** (Aix-la-Chapelle, Allemagne, avril 1993, 256 Ko, sérigraphie complète avec adresse/téléphone), et carte « (C) M.K. » revendue sous la marque **Dynatech** (« Aktive 512/1024 KB », probablement le même M. Kemper). Source primaire de `08-cartes-memoire.md` §3.
- **Fiches techniques FRAM Fujitsu/Cypress/Ramtron** (`MB85R4001A` notamment, `www.fujitsu.com/us/Images/MB85R4001A-DS501-00005-3v0-E.pdf`) — famille de composants FRAM parallèles compatibles SRAM utilisée pour les pistes de fabrication de `08-cartes-memoire.md` §5.
- **[Forum silicium.org — « PC-E500S 256K »](https://forum.silicium.org/viewtopic.php?t=39164)** (2015) — fil de discussion très détaillé : démontage d'une carte mère PC-E500S 256 Ko (confirmation de 2 puces SRAM `Sony CXK581000AM` 128 Ko), carte mémoire externe précise par ligne `CE` (`00000`-`FFFFF`), comportement réel de `MEM$="B"`/`FRE0` avec RAM interne 256 Ko + carte, et incompatibilité d'une carte 256 Ko avec le PC-1360 (lignes `A15`-`A17` sans résistances de tirage). Source principale de `03-memoire-et-systeme-pc-e500s.md` §1bis-1ter et de `09-cartes-meres-ram-interne.md`.
- **Photos de cartes mère** (`Referentiel PC-E500S SC62015/Photos carte mère Sharp/`) — carte mère PC-E500S 32 Ko (photos personnelles de l'utilisateur) et carte mère 256 Ko (photos de type petites annonces, nommage `s-l1600-*`, probablement une référence externe). Source primaire de `09-cartes-meres-ram-interne.md`.
- **Photos `berlin1.jpg`/`berlin2.jpg`** (`Referentiel PC-E500S SC62015/Photos Carte memoire Sharp/`) — carte 256 Ko « Böttcher Datentechnik », en réalité une révision 1998 (v4.1) de la carte « M.K. »/M. Kemper de 1993. Source primaire de `08-cartes-memoire.md` §3.4.
- **Photos `s-l1600.png`/`s-l1601.png`** (`Referentiel PC-E500S SC62015/Photos Carte memoire Sharp/`) — carte 256 Ko « Becker & Partner » (Aachen), fabricant indépendant. Source primaire de `08-cartes-memoire.md` §3.5.
- **Puce Seiko Instruments `SRM20100`** — identifiée par recherche web (juillet 2026) comme SRAM 128K×8, confirmant l'identité des puces de la carte Becker & Partner (§3.5).
- **Annuaires professionnels allemands** (`gelbeseiten.de`, `goyellow.de`) — confirmation de l'existence de « Becker & Partner GmbH Mobile Datensysteme », Neuenhofstr. 110, 52078 Aachen, comme entreprise réelle. Source de `08-cartes-memoire.md` §3.5.
- **[高松製作所 Online Shop (tmfg.jp)](https://tmfg.jp/products/list?category_id=53)** — vendeur japonais de pièces et services pour ordinateurs de poche Sharp. Trois fiches produit consultées directement (juillet 2026) : [carte FRAM 256 Ko](https://tmfg.jp/products/detail/202) (¥14 850, dimensions 42×54×3 mm, protection par interrupteur), [carte FRAM 128 Ko](https://tmfg.jp/products/detail/203) (¥14 300), et [service de transformation RAM interne 32/64 Ko → 256 Ko](https://tmfg.jp/products/detail/104) (¥6 380). Source principale de `11-fabrication-carte-256ko-fram.md` §0 ; la troisième fiche recoupe commercialement le témoignage forum de `09-cartes-meres-ram-interne.md`.
- **Octopart** (agrégateur de distributeurs, `octopart.com`) — utilisé pour établir des fourchettes de prix indicatives des puces FRAM `MB85R4001A` (1024 Ko) et `MB85R1001A` (256 Ko), composants de niche peu présents chez les distributeurs de premier rang (Digikey/Mouser). Source de `10-fabrication-carte-1024ko-fram.md` et `11-fabrication-carte-256ko-fram.md` §1.
- **`Carte memoire 1024Ko FR.docx`** (`Referentiel PC-E500S SC62015/Photos Carte memoire Sharp/`) — manuel utilisateur du fabricant de la carte 1024 Ko (§3.3 de `08-cartes-memoire.md`), traduction française (probablement d'un original allemand, style parfois maladroit) couvrant : structure du registre de banque (`POKE 65536`, table de vérité `D0`-`D3`), procédure d'installation/retrait physique de la carte, remplacement de la pile de sauvegarde (CR1220, seuil 2,5 V), et procédure d'autotest usine (combinaison `SHIFT`+réinitialisation, menu « ESDE »). Source qui **confirme** l'hypothèse de banking « 4 × 256 Ko » précédemment déduite par simple lecture des composants.

## 6. Notes de méthode

- Quand une même donnée (ex. largeur des registres `X`/`Y`/`U`/`S`) est formulée différemment selon la source (« 20 bits » dans le manuel officiel vs « 24 bits » dans une table de reconstruction), ce référentiel le signale explicitement plutôt que de trancher silencieusement (voir `01-architecture-cpu-sc62015.md` §2).
- Les adresses système et fonctions FCS/IOCS de `03` et `04` proviennent en priorité du manuel officiel ; les compléments issus de rétro-ingénierie (adresses non documentées comme `TXTBAS`/`DATBAS`/`SI`/`DI`...) sont marqués comme tels.
- Les sites communautaires japonais historiques utilisent l'encodage Shift-JIS ; certains ont été récupérés via récupération automatique de contenu et peuvent comporter des artefacts d'encodage sur les caractères non-ASCII — les informations factuelles (noms de fichiers, adresses, versions) restent fiables, les commentaires narratifs ont été reformulés en français plutôt que traduits mot à mot.

## 7. Voir aussi

Tous les autres fichiers de ce référentiel (`01` à `06`) renvoient vers ce fichier pour le détail de leurs sources.
