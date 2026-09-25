# Mémoire et système du PC-E500S

*Rédigé le 2026-08-26 — mis à jour le 2026-09-25*

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier détaille la carte mémoire du PC-E500S (interne + externe), les registres d'E/S, les vecteurs d'interruption et les points d'entrée IOCS. Sources principales : le manuel *ESR-L CPU Instruction Manual*, le *Technical Reference Manual PC-E500*, et la feuille de dépouillement `Docs/Doc technique/PCE500 Description mémoire.xls.xlsx` déjà constituée dans le projet `SC62015Disassembler` (adresses recoupées avec des listings XASM réels).

## 1. Vue d'ensemble

Rappel (détaillé dans `01-architecture-cpu-sc62015.md`) : 256 octets de mémoire **interne** (`(adresse)`) intégrés au CPU, et 1 Mo de mémoire **externe** (`[adresse]`, 20 bits). Trois zones de cette mémoire externe portent un nom de lecteur mémoire — **et une seule est une carte** :

| Lecteur | Adresse de tête | Nature | Disque RAM qu'elle héberge |
|---|---|---|---|
| `S1:` | `80000H` | **RAM interne soudée** — 256 Ko sur PC-E500S, 32 Ko calés en haut (`B8000H`) sur PC-E500 | `E:` |
| `S2:` | `40000H` | **l'unique emplacement de carte amovible** | `F:` |
| `S3:` | `C0000H` | ROM système, qui porte les programmes préinstallés du menu | `G:` |

⛔ **`S1:`, `S2:`, `S3:` ne sont pas trois slots de carte** (voir `08-cartes-memoire.md` §1bis). Et **`E:`, `F:`, `G:` ne sont pas des synonymes de `S1:`, `S2:`, `S3:`** : ce sont les disques RAM du driver MEMORY FILE (device 5, §7), chacun logé **dans** la mémoire correspondante. `E:` est toujours dans la mémoire interne, `F:` toujours dans la carte — `MEM$` ne les échange jamais (§5).

*Sources : les adresses de tête sont lues dans la System Data Area d'un PC-E500S réel, et la ROM les consulte par numéro de lecteur (§4) ; l'appartenance de `E:` et `F:` est donnée par le manuel utilisateur, en anglais et en allemand, qui concordent (§5) ; `G:` pour les programmes de la ROM par le manuel allemand (PDF p. 123).*

## 1bis. Carte mémoire externe par ligne `CE` (répartition physique du 1 Mo)

Répartition des 8 lignes de sélection `CE0`-`CE7` (rappelées en `08-cartes-memoire.md` §1) sur l'espace externe de 1 Mo, reconstituée par la communauté (forum silicium.org, fil « PC-E500S 256K », 2015 — cf. `07-sources-et-bibliographie.md`) :

| Ligne | Plage d'adresses | Polarité | Rôle |
|---|---|---|---|
| `CE6` | `10000H`-`1FFFFH` | Négative | Carte graphique (logique d'extension). |
| `CE3` | `20000H`-`3FFFFH` | Négative | Non identifié précisément (« comme DELTA » selon la source). |
| `CE1` | `40000H`-`7FFFFH` | Positive | **Carte mémoire d'extension, slot `S2:`** — sous-découpée selon la capacité de la carte (64 Ko/32 Ko/16 Ko/8 Ko occupent des sous-plages différentes à partir de `40000H`). |
| `CE0` | `80000H`-`BFFFFH` | Négative | **RAM interne intégrée, `S1:`** (qui héberge le disque RAM `E:`, §1) — fenêtre de 256 Ko (`80000H`-`BFFFFH`), dont seule la partie haute est peuplée sur les machines à faible capacité (ex. PC-E500 32 Ko : `B8000H`-`BFFFFH` ; PC-E550 64 Ko : `B0000H`-`BFFFFH`). Voir §1ter et `09-cartes-meres-ram-interne.md`. |
| `CE7` | `BC000H`-`BFFFFH` | Positive | Réservation réseau/extension (chevauche la fin de `CE0`). |
| `CE2` | `C0000H`-`FFFFFH` | Négative | **ROM système, `S3:`** (programmes préinstallés, lus par `G:`). |
| `CE4`, `CE5` | — | — | Non attribués/non identifiés dans les sources disponibles. |

Adresses `00000H`-`000FFH` : accessibles par `PEEK`/`POKE` depuis BASIC mais ne correspondent **pas** à de la mémoire externe — c'est un alias vers la RAM **interne** du CPU (`(adresse)`, §2-3 ci-dessous), exposé ainsi pour rester cohérent avec l'espace d'adressage 20 bits vu depuis BASIC.

## 1ter. Fenêtre de RAM interne intégrée par modèle

La fenêtre `CE0` (`80000H`-`BFFFFH`, 256 Ko) est dimensionnée pour la configuration maximale ; chaque modèle ne peuple que la partie haute correspondant à sa RAM interne réellement installée :

| Modèle | RAM interne | Fenêtre peuplée |
|---|---|---|
| PC-E500 (32 Ko) | 32 Ko | `B8000H`-`BFFFFH` |
| PC-E550 (64 Ko) | 64 Ko | `B0000H`-`BFFFFH` |
| PC-E500S (256 Ko, variante haut de gamme) | 256 Ko | `80000H`-`BFFFFH` (fenêtre `CE0` entière) |

Voir `09-cartes-meres-ram-interne.md` pour l'inventaire des puces observées sur une carte mère 32 Ko et une carte mère 256 Ko réelles.

**Comportement observé (BASIC, non documenté par Sharp) :** combiner la RAM interne 256 Ko avec une carte en mode `MEM$="B"` (extension) échoue avec `Out of memory`, alors que la même manipulation fonctionne avec 32 Ko de RAM interne + carte 32 Ko. Le mode `S1`/`S2` (carte utilisée seule, sans fusion) fonctionne dans tous les cas testés par la source. À vérifier avant de concevoir une carte visant à être combinée en mode `B` avec une machine 256 Ko : la limite semble se situer autour de 256-288 Ko de total contigu, pas aux 512 Ko théoriques de l'espace `CE0`+`CE1`.

## 2. Registres d'E/S en mémoire interne (`F0H`–`FFH`)

| Addr | Sigle | Nom | Détail des bits |
|---|---|---|---|
| `F0H` | `KOL` | Key Output Buffer L | Commande `KO0`-`KO7` (matrice clavier, sortie). |
| `F1H` | `KOH` | Key Output Buffer H | Commande `KO8`-`KO15`. |
| `F2H` | `KIL` | Key Input Buffer | Lecture `KI0`-`KI7`. Non nul ⇒ déclenche l'interruption clavier. |
| `F3H`/`F4H` | `EOL`/`EOH` | E Port Output Buffer L/H | Port d'extension E, sortie (`E0`-`E15`). |
| `F5H`/`F6H` | `EIL`/`EIH` | E Port Input Buffer L/H | Port d'extension E, entrée. |
| `F7H` | `UCR` | UART Control Register | `BOE`(7) *break output enable* · `BR2-0`(6-4) vitesse (300 à 19200 bps) · `PA1-0`(3-2) parité · `DL`(1) longueur (7/8 bits) · `ST`(0) stop bits (1/2). |
| `F8H` | `USR` | UART Status Register | `RXR`(5) réception prête · `TXE`(4) émetteur vide · `TXR`(3) émetteur prêt · `FE`(2) erreur de trame · `OE`(1) dépassement · `PE`(0) erreur de parité. |
| `F9H` | `RXD` | UART Receive Buffer | Octet reçu. |
| `FAH` | `TXD` | UART Transmit Buffer | Octet à émettre. |
| `FBH` | `IMR` | Interrupt Mask Register | `IRM`(7) masque global · bits 6-0 masquent chaque source (externe, UART Rx/Tx, ON, touche, timer lent, timer rapide). |
| `FCH` | `ISR` | Interrupt Status Register | Bits 6-0 = drapeau « interruption survenue » par source (mêmes rangs que `IMR`). |
| `FDH` | `SCR` | System Control Register | `ISE`(7) autorise le démarrage IRQ · `BZ2-0`(6-4) contrôle broches CO/CI · `VDDC`(3) · `STS`(2) sélection timer lent (0=0,5s / 1=2s) · `MTS`(1) sélection timer rapide (0=4ms / 1=16ms) · `DISC`(0) contrôle pilote LCD. |
| `FEH` | `LCC` | LCD Contrast Control | `LCC4-0`(7-3) niveau de contraste (0-31) · `KSD`(2) désactive le balayage clavier · `STCL`(1)/`MTCL`(0) clear timers. |
| `FFH` | `SSR` | System Status Register | `ONK`(3) état touche ON · `RSF`(2) *reset-start flag* · `CI`(1) entrée CMT · `TEST`(0) entrée test. |
| `EFH` | `AMC` | Address Modify Control | `AME`(7) active la jonction virtuelle CE1/CE0 · bit 6 inutilisé · `AM5-0`(**5-0**) capacité de la carte côté `CE0`, en code thermomètre : `000000`=2 Ko, `000001`=4, `000011`=8, `000111`=16, `001111`=32, `011111`=64, `111111`=128 Ko. Confirmé par la routine de dimensionnement de la ROM (`0F0E7Bh`), qui produit exactement cette suite par `shr a`. ⛔ Une version antérieure écrivait « bits 6-1 ». |
| `ECH`/`EDH`/`EEH` | `BP`/`PX`/`PY` | Pointeurs d'adressage interne | Voir `01-architecture-cpu-sc62015.md` §3.3. |

## 3. Adresses système en mémoire interne (zone RAM générale, `00H`-`E3H`)

Ces adresses ne sont pas documentées dans le manuel CPU (qui ne décrit que `ECH`-`FFH`) : elles ont été identifiées par rétro-ingénierie du firmware/BASIC (listings XASM réels, cf. `07-sources-et-bibliographie.md`) et consolidées dans le tableur `PCE500 Description mémoire.xls.xlsx` et `Data/InternalRAMNames.json`.

| Addr | Sigle | Description |
|---|---|---|
| `CBH` | `TXTBAS` | Adresse (3 octets) où réside `TEXT.BAS` courant. |
| `CEH` | `DATBAS` | Adresse où réside `DATA.BAS` courant. ⚠️ Ces deux blocs **bougent** quand un pilote s'insère avant eux : l'installateur doit recaler `TXTBAS`/`DATBAS` (§7bis). |
| `D1H` | `BASPTR` | **Pointeur** (3 octets) vers la zone de travail de l'interpréteur BASIC. Nommé `BASWRK` jusqu'en septembre 2026 — même nom que la zone externe `BFD0EH`, ce qui rendait l'adresse interne innommable dans une source et a coûté une dizaine d'essais : voir `12-extensions-basic.md` §7. Porte les crochets d'extension du BASIC en `[(0D1H)+090H]` et `[(0D1H)+093H]`, et en `[(0D1H)+00BH]` le **vecteur d'extension de l'éditeur** du mode direct (§3bis). |
| `D4H`/`D5H` | `BL`/`BH` | Registre `B` étendu (paire haute/basse, usage interne interpréteur). |
| `D6H`/`D7H` | `CL`/`CH` | Registre `C` étendu. |
| `D8H`/`D9H` | `DL`/`DH` | Registre `D` étendu. |
| `DAH` | `SI` | Pointeur source, 3 octets. |
| `DDH` | `DI` | Pointeur destination, 3 octets. |
| `E6H` | `IOCSW` | Zone de travail IOCS (interne). |

## 3bis. L'éditeur du BASIC : ses variables sont relatives à `BP`, et le PC-E500S les a décalées

L'éditeur de ligne du mode direct appelle, **à chaque touche**, le vecteur `[(0D1H)+00BH]` :
`mv x,[(basptr)+0Bh]` puis `callf` vers un `jp x` (`0F2D61h` → `0F1786h` dans `rom83`). L'appel est
**inconditionnel** : le vecteur désigne toujours une routine valide, et un programme qui le
détourne peut rendre la main à l'ancien par `jpf`. C'est le crochet de **History** (TORO, 1994),
devenu le pilote `HISTDRV` en septembre 2026 (`12-extensions-basic.md` §17quater).

Le code appelé partage les variables de l'éditeur, que la ROM lit **sans octet PRE** : ce sont des
`(BP+n)` (`01` §3.3). Celles qu'emploie History :

| `(BP+n)` | Rôle |
|---|---|
| `00h` | pointeur (3 o) de la ligne saisie |
| `07h` | longueur de la ligne |
| `0Ah` | bit 7 : mode PRO (édition d'un texte BASIC) |
| `08h`, `0Eh`, `10h`, `12h` | position et état d'édition, remis à jour en fin de commande étendue |
| `17h` | colonne du curseur |
| `2Ah`/`2Bh`, ou `2Bh`/`2Ch` | **code étendu / caractère de la touche** — selon la ROM, voir ci-dessous |

⛔ **Le code de la touche n'est pas au même endroit selon la ROM.** Mesuré à l'appel du vecteur
dans les trois images, le 2026-09-24 :

| ROM | Machine | Instruction avant l'appel | Code étendu | Caractère |
|---|---|---|---|---|
| 5.3 | PC-E500, série ancienne | `0F3714h` `mv (2Ah),ba` | `(BP+2Ah)` | `(BP+2Bh)` |
| 7.5 | PC-E500 BL / E550 | `0F3764h` `mv (2Ah),ba` | `(BP+2Ah)` | `(BP+2Bh)` |
| **8.3** | **PC-E500S** | `0F2D61h` **`mv (2Bh),ba`** | **`(BP+2Bh)`** | **`(BP+2Ch)`** |

Les offsets `00h` à `22h` n'ont **pas** bougé : les deux éditeurs, 5.3 et 8.3, alignés instruction
par instruction (1 896 paires), emploient les mêmes, et `17h` (colonne) a les mêmes usages dans les
deux ROM. Le code de la touche, lui, **est le même** : CTRL + ← donne le code étendu `5Dh`,
CTRL + → `5Ch`, dans les deux (§4, tables du clavier). Un programme écrit pour l'E500 lit donc,
sur l'E500S, **l'octet d'à côté** — c'est pourquoi History 1.11 « ne fait rien » sur un PC-E500S
(mesuré sur émulateur). La seule façon, pour un programme, de savoir où lire est la **version de
la ROM** (§8) : `HISTDRV` la lit à l'installation.

## 4. Mémoire externe : zones système principales

| Adresse | Sigle | Description |
|---|---|---|
| `BFC09`–`BFC19` | `ldAdSlot2`/… | **Table des trois lecteurs mémoire** — adresse de tête (3 octets) puis capacité en blocs de 2 Ko : `BFC09` `ldAdSlot2` / `BFC0C` `cpSlot2` = **`S3:`** (ROM, `C0000`, `&40`) · `BFC0F` `ldAdSlot1` / `BFC12` `cpSlot1` = **`S2:`** (carte, `40000`, `&80` = 256 Ko) · `BFC14` `ctrlCRAM` (carte insérée) · `BFC15` `ldAdSlot0` / `BFC18` `cpSlot0` = **`S1:`** (RAM interne, `80000`, `&80` = 256 Ko sur E500S ; `B8000` sur PC-E500). ✅ La ROM les lit **par numéro de lecteur** — `0F02B7h` `mv y,[0BFC15h]` pour le lecteur 0, `0F02CBh` `[0BFC0Fh]` pour le 1, `0F02D5h` `[0BFC09h]` pour le 2 : c'est ce qui nomme `S1:` la RAM interne. ⛔ Une version antérieure faisait commencer la table en `BFC15` et la prolongeait jusqu'à `BFCDE`. |
| `BFC27`/`BFC28` | `SCRNX`/`SCRNY` | Prochaine coordonnée d'affichage sur `STDO:`/`SCRN:`. |
| `BFC2A` | `LINPTN` | Cadre de points affiché dans une boîte 16 points. |
| `BFC2D`–`BFC41` | — | Table de conversion des codes clavier (normal / SHIFT / CTRL), 1 et 2 octets, + *hook* de la routine de traitement clavier (`&F1B4D` par défaut). ✅ Six pointeurs, dans l'ordre de `pce500.inc` : `keytbl_1b`, `keytbl_2b`, `keytbl_1b_shift`, `keytbl_2b_shift`, `keytbl_1b_ctrl`, `keytbl_2b_ctrl`, déposés par la commande clavier `3Fh` depuis `0F1C6Dh` (`rom83`) : `0307A4h`, `030806h`, `03092Ch`, `03098Eh`, `030868h`, `0308CAh` — **l'ordre en mémoire (S3EXT) n'est pas celui des pointeurs**. Dans la table CTRL à 2 octets, ← (code matriciel `1Dh`) donne `5Dh` et → (`1Ch`) `5Ch` ; les six tables se retrouvent à l'identique dans `rom53` (à partir de `0F3422h`). |
| `BFC6D` | `FHANDLE` | Table des handles de fichiers ouverts. |
| `BFC7D` | `DEFDRV` | Nom du lecteur par défaut (7 octets, ex. `F:____0`). |
| `BFC84`–`BFC96` | — | Adresses (3 octets chacune) des polices de caractères par plage de codes (0-1F, 20-7F, 80-9F, A0-DF, E0-FF) + `DOTSOP` (mode d'affichage précédent). |
| `BFC9B`–`BFCA1` | `CSRX`/`CSRY`/`WIDTH`/`HEIGHT`/`LCDMOD` | Curseur LCD, largeur (`&28`=40 col.), hauteur (4 lignes), mode vidéo (normal/inversé). ✅ **`HEIGHT` = `BFC9E` (`lcd_height` dans `pce500.inc`) borne la console** : le pilote d'écran défile quand la ligne l'atteint (`0F2B6Ah`), `clear_disp` n'efface que les lignes `0`..`HEIGHT−1`, et `LOCATE` refuse un `Y` au-delà. Mesuré sur machine (J.-F. Albouy, septembre 2026) : `POKE &BFC9E,2` confine affichage, défilement **et** `CLS` aux lignes 0-1, et `CLS` **ne remet pas** la hauteur à 4. ⚠️ L'état **survit à la fin du programme** (l'invite du mode direct reste dans la fenêtre) : finir par `POKE &BFC9E,4:CLS`, dans cet ordre ; après une erreur ou un BREAK, MENU puis BASIC. ⚠️ `CLS` avec une hauteur ≠ 4 affiche les **libellés des touches de fonction** sur la ligne 3 (`0F931Ah` → `fkey_display` `0F1CFAh`) : ce n'est pas un plantage. La ROM ne connaît **aucune ligne de début** : son défilement part toujours de la ligne 0 (`0F27F1h`) — pour figer le haut de l'écran, voir `XCONSOLE` (`12-extensions-basic.md` §17bis). |
| `BFCA2` | `IOCSH` | Adresse du prochain en-tête de chaîne IOCS (voir §7) — `d_link` dans `pce500.inc`. 📖 La ROM la remet à `DF820H` en `0F0D9Eh` (séquence d'amorçage) et en `0F1665h` (initialisation complète). ✅ Un pilote chaîné en tête y **reste** après `OFF`/`ON`, un petit reset `CALL &FFFD8` et un soft RESET (bouton seul, sans initialisation) : mesuré avec `BASEXT-DRV` le 2026-09-16 (§7bis). |
| `BFCBA`–`BFCBF` | — | Paramètres clavier : délai avant répétition, cadence de répétition, délai d'extinction automatique (`&B0`×0,5s), drapeaux *break*/pile faible, répétition/clic activés. |
| `BFCC0`/`BFCC3` | — | Table principale de conversion code matériel → code logiciel clavier, *hook* de traitement. |
| `BFCC6`–`BFCDB` | voir §6 | Vecteurs RAM des 8 sources d'interruption. |
| `BFCDE` | `UWORK` | Dernière adresse du slot S1 + 1. |
| `BFCE1` | `SWORK` | Zone de la pile système (`S`). |
| `BFD0E` | `BASWRK` | **Pointeur** (3 octets) vers la zone de travail BASIC : il **contient** l'adresse, il **n'est pas** la zone. La ROM le recopie en RAM interne par `mvp (0D1h),[0BFD0Eh]` en `0F98CAh`. Pour lire la zone depuis le BASIC : `W=LPEEK &BFD0E` puis `PEEK (W+n)`. Nom de E. Kako (`register.lst`, 1990). ⛔ Décrit « zone de travail » jusqu'en septembre 2026 — corrigé à la source dans `SC62015Disassembler/Data/SystemAddresses.json`, puis `pce500.inc` **régénéré** (`12-extensions-basic.md` §7). ⚠️ À ne pas confondre avec `BASPTR` = `0D1H`, sa copie en RAM interne : un `mv x,(baswrk)` serait tronqué à 8 bits par l'assembleur, sans avertissement. |
| `BFD17` | `IOCSWRK` | Zone de travail IOCS (externe). |
| `BFD1A` | `USRWRK` | **Pointeur** (3 octets) vers le début de la **zone langage machine**, qui s'étend de `[BFD1A]` jusqu'au plafond `BFC00H` (défaut `BFC00H` : zone vide). Il **contient** l'adresse, il n'est pas la zone — comme `BASWRK` et `IOCSWRK`. Lire : `PEEK &BFD1A+PEEK &BFD1B*256+PEEK &BFD1C*65536`. La zone se réserve par `CALL &FFFD8` (§8, `12-extensions-basic.md` §14, et le référent `C:\Claude\BASEXT\MODE-EMPLOI.md` §9.2), puis un module s'assemble en `BF000H`. ⛔ **Ne jamais assembler EN `BFD1AH`** : les paramètres SIO commencent 23 octets plus loin, en `BFD31H` ; un programme placé là écrase le pointeur puis la configuration de la liaison série, et la panne se manifeste au transfert *suivant*. ⛔ Une version antérieure décrivait `USRWRK` comme une « zone de 23 octets » avec « `CALL &BFD1A` typique » : ces 23 octets sont la distance au premier paramètre SIO, pas une zone. Sources : la ROM (`mv x,[usrwrk]`, `Docs/Synthese/BASIC-en-ROM.md`), `Docs/Synthese/Drivers-IOCS.md` (la zone IOCS va « jusqu'à l'adresse écrite en `0BFD1Ah` »), et la réservation de VOGUE en 1992, qui passe `&1A,&FD,&B` à `CALL &FFFD8` comme adresse d'un pointeur. |
| `BFD31`–`BFD62` | — | Paramètres SIO (temporisation, vitesse, parité, fin de ligne `&1A`, délais d'ouverture/fermeture). |
| `BFD42`–`BFD53` | — | Constantes de codage cassette (`CAS:`) : longueurs et seuils des niveaux logiques 0/1, blocs d'en-tête. |
| `BFE00`–`BFE04` | — | **Zone d'échange** des portes `CALL &FFFDC` et `CALL &FFFD8` (§8) : `[BFE00]` = `(cl)`, `[BFE01]` = `(ch)`, `[BFE02]` = `IL`, puis `[BFE03]` et `[BFE04]` pour deux paramètres (lettre A–Z des matrices, numéros de séquence des statistiques — `04-fcs-iocs.md` §2.5). ⚠️ **C'est un brouillon, pas une variable** : `RENUM` et `DELETE` s'en servent aussi comme octet de drapeaux (`SC62015Disassembler/Docs/Synthese/BASIC-en-ROM.md`). |
| `BFFD0` / `BFFD8`–`BFFF7` | — | Sauvegardes des familles matrices et statistiques du device 9 : `BP` de l'appelant en `BFFD0`, RAM interne `0A0h`–`0BFh` en `BFFD8` (`04-fcs-iocs.md` §2.5). |
| `DF820`–`DF8A9` | — | Chaîne des en-têtes de drivers IOCS **de la ROM** (voir §7). |

*(Toutes les adresses ci-dessus proviennent de `PCE500 Description mémoire.xls.xlsx`, elles-mêmes vérifiées sur listings XASM réels ; les valeurs marquées « défaut » correspondent aux réglages ROM standard et peuvent différer selon la variante S1/E500/E500S.)*

## 5. Cartes mémoire — `MEM$`, et les disques RAM `E:` / `F:`

⛔ **Deux notions indépendantes, qu'une version antérieure de ce tableau juxtaposait au point de se lire à l'envers** — la ligne « `S1` » y citait `F:`, la ligne « `S2` » citait `E:`, ce qui se lisait « S1 = F, S2 = E ». La correspondance réelle est l'inverse de cette lecture, et elle **ne dépend pas du mode** :

- **`E:` est TOUJOURS le disque RAM de la mémoire interne** (`S1:`) ;
- **`F:` est TOUJOURS le disque RAM de la carte** (`S2:`).

`MEM$` choisit **où vivent les programmes et les variables** ; il ne déplace aucun disque RAM, il dit seulement lequel reste utilisable.

| `MEM$` | Programmes et variables | `E:` (mémoire interne) | `F:` (carte) |
|---|---|---|---|
| `"S1"` | mémoire interne seule, la carte n'est pas utilisée pour eux | utilisable | **utilisable** — « *RAM disk F within the RAM card remains available* » |
| `"S2"` | carte seule — variables fixes comprises ; formules (`AER`) et touches de fonction restent en mémoire interne | **utilisable** — « *RAM disk E and AER memory area within the computer memory remain available* » | non cité par le manuel : la carte porte les programmes |
| `"B"` | mémoire interne **et** carte fusionnées | utilisable | **indisponible** — « *RAM disk F becomes unavailable, while RAM disk E remains available* » |

*Source : manuel utilisateur, pp. 20-23 du livre — en anglais (`SC62015Disassembler/Docs/Doc technique/PC-E500 manual_EN.pdf`, PDF pp. 28-31) et en allemand (`PC-E500S-DE.pdf`, PDF pp. 29-31), qui disent la même chose mot pour mot. La page `MEM$` du manuel anglais (livre p. 295) : « "S1" means that the external RAM card is not being used. "S2" means that only the external RAM card is being used. "B" means that the RAM card is being used as an extension of the internal memory space. »*

Ce que le manuel ajoute, et qui compte en pratique :

- **Pour un disque RAM plus grand, Sharp recommande `F:` avec `MEM$="S1"`** plutôt que `E:` avec `MEM$="B"` : une carte ainsi formatée passe d'un PC-E500 à l'autre, et plusieurs cartes s'emploient comme des disquettes.
- **Une carte en `"S2"`** ne passe sur une autre machine que si celle-ci n'emploie ni `E:` ni l'`AER`.
- **Une carte en `"B"`** ne fonctionne qu'avec la mémoire interne qui l'accompagne, et doit être vidée avant le passage en `"B"`.
- **Les bascules permises** : `"S1"` ↔ `"S2"` et `"S1"` ↔ `"B"` — jamais `"S2"` ↔ `"B"` directement.

Formatage d'une carte en disque RAM : `MEM$="S1"` puis `INIT "F:31K"` (taille selon capacité de la carte). Chaque carte RAM embarque sa propre pile de sauvegarde (maintien ~1 an hors alimentation). Des cartes **FRAM** modernes (ferroélectriques, jusqu'à 256 Ko, non volatiles sans pile) sont utilisables en remplacement — voir `07-sources-et-bibliographie.md`.

## 6. Interruptions — vecteurs RAM par défaut

Rappel : un seul vecteur matériel (§8) redirige vers 8 adresses modifiables en RAM, une par source (`Technical Reference Manual PC-E500`, chapitre *Interrupt*) :

| Addr RAM | Sigle | Source | Valeur par défaut (ROM E500S) |
|---|---|---|---|
| `BFCC6` | `FASTV` | Timer rapide (4/16 ms) | `&F2102` |
| `BFCC9` | `SLOWV` | Timer lent (0,5/2 s) | `&F219E` |
| `BFCCC` | `KEYV` | Touche | `&F20EF` |
| `BFCCF` | `ONV` | Touche ON | `&F20F8` |
| `BFCD2` | `SIOSV` | Émission SIO | `&F2101` |
| `BFCD5` | `SIORV` | Réception SIO | `&F59D7` |
| `BFCD8` | — | Externe (contrôleur batterie) | `&F211D` |
| `BFCDB` | `EXV` | Logicielle (`IR`) | `&F2101` |

Un programme peut réécrire une de ces adresses pour intercepter l'interruption correspondante (technique standard des TSR/drivers de l'écosystème, cf. `06-ecosysteme-outils.md`).

## 7. Chaîne des en-têtes de drivers IOCS

`IOCSH` (`BFCA2`, §4) désigne la tête de la chaîne ; celle de la ROM commence en `DF820H`. Chaque en-tête vaut `suivant 3 o / device 1 o / attributs 1 o / entrée 3 o / nom(s) ASCII`, et la chaîne s'arrête sur `suivant = FFFFFH`. Relevé **octet à octet dans `rom83.bin`** (PC-E500S 8.3) :

| En-tête | Device | Driver | Lecteur(s) | Entrée | Suivant |
|---|---|---|---|---|---|
| `DF820` | 0 | DISPLAY | `STDO:` `SCRN:` | `F21E7` | `DF833` |
| `DF833` | 1 | KEY | `STDI:` `KYBD:` | `F16F2` | `DF846` |
| `DF846` | 2 | SIO | `COM:` | `EAA71` | `DF853` |
| `DF853` | 3 | PRINTER | `STDL:` `PRN:` | `EA4BD` | `DF865` |
| `DF865` | 4 | TAPE | `CAS:` | `E96E8` | `DF872` |
| `DF872` | 6 | MEMORY CARD | `S1:` `S2:` `S3:` | `F0000` | `DF884` |
| `DF884` | 5 | MEMORY FILE | `E:` `F:` `G:` | `E493E` | `DF893` |
| `DF893` | 9 | FUNCTION | *(aucun)* | `EF029` | `DF89C` |
| `DF89C` | 7 | FDD | `X:` `Y:` | `EB66C` | `DF8A9` |
| `DF8A9` | 8 | SYSTEM | `SYSTM:` | `E0183` | `FFFFF` |

⛔ **La version antérieure de ce tableau était fausse sur sept lignes sur dix** : les deux premiers en-têtes intervertis (clavier et écran), `S1:`/`S2:`/`S3:` et `E:`/`F:`/`G:` rendus au même point d'entrée, `X:`/`Y:` placés sur l'en-tête du Function Driver, `SYSTM:` décalé d'un cran avec une entrée inventée (`B66C0`), et le device 8 donné comme « fonction système » en fin de chaîne. Le relevé ci-dessus est celui de `SC62015Disassembler/Docs/Synthese/Memoire-et-SDA.md` et `Drivers-IOCS.md`, où il **fait autorité**.

> ⚠️ **Les en-têtes ne sont pas rangés par numéro de device** : 6 précède 5, et 9 précède 7 puis 8. C'est le numéro de **device** — et non la position dans la chaîne — qui choisit le jeu de commandes `41H`-`7FH`.
>
> ⚠️ **La colonne « Entrée » vaut pour `rom83` et pour elle seule.** La ROM du PC-E500 (`rom53`) porte les mêmes en-têtes dans le même ordre, mais **sept entrées sur dix** y sont à d'autres adresses : une entrée de driver se lit dans la chaîne de la machine, jamais dans un relevé.

Ce format d'en-tête chaîné (adresse du suivant sur 3 octets + identifiant + attributs + adresse d'entrée 3 octets + nom ASCII) permet d'installer un nouveau driver résident sans modifier la ROM : c'est le mécanisme qu'utilise `PLINKC162` (lecteur `L:`, voir `06-ecosysteme-outils.md`) et que documente `Data/SystemDataRegions.csv` sous la clé `iocs_hdr_tbl` (`DF820`, 152 octets, jusqu'à sentinelle `FFFFF`).

## 7bis. La chaîne des blocs de `S1:` — où loger un pilote résident

Un pilote résident vit dans un **bloc** de `S1:` (en-tête `0FBh` + nom 8.3 ; attribut relevé `25h` pour `BASEXT.SYS`, `20h` pour les blocs du BASIC — l'**attribut** est en `+0Ch`, et PLINKC y reconnaît un pilote par `mv a,[x+0Ch]` / `test a,00Ch`), protégé comme un fichier, et publie son en-tête IOCS dans la chaîne du §7. **Où** l'insérer dans la chaîne des blocs n'est pas un détail : c'est ce qui décide s'il bougera. Mesuré sur émulateur PC-E500S par J.-F. Albouy le 2026-09-16 (`C:\Claude\BASEXT-DRV\essais\BLOCS.BAS`, `PEEK` seulement ; `CONCEPTION.md` §5bis et §5ter).

✅ **Après un RESET complet, sans pilote** :

| Bloc | Adresse | Taille du bloc (`+11h`) | `FILES` |
|---|---|---|---|
| `DATA    BAS` | **`080018h`**, le premier | **251 597** | 411 |
| `TEXT    BAS` | `0BD6E5h` | 650 | 616 |
| `FUNCKEY` | `0BD96Fh` | 110 | 76 |
| `AER` | `0BD9DDh` | 61 | 27 |

La chaîne est jointive et se termine en `0BDA1Ah`, contre `[s1_btm]` − 1. Ce qui en découle :

1. ✅ **`DATA.BAS` est le premier bloc et contient toute la mémoire libre.** Il n'y a pas de place « après la chaîne » : elle est **dans** `DATA.BAS`, et `TXTBAS`/`DATBAS` (§3) désignent ces deux blocs.
2. ⛔ **Un bloc ajouté en fin de chaîne bouge.** Au premier besoin de place du BASIC, `DATA.BAS` regrossit et **pousse vers le haut** tout ce qui le suit ; le bloc est déplacé **sans relocation**, et les pointeurs extérieurs (maillon IOCS, crochets du BASIC) désignent l'ancienne adresse. Mesuré : bloc copié en `0804E5h`, relu en `0BCF30h` au `RUN` suivant ; la ligne tapée ensuite a arrêté PockEmul (FACTORY RESET). C'est ce que faisait le `DRIVER_TEMPLATE` de `xasm2026-4`, validé seulement jusqu'à « apparaît dans `FILES` ».
3. ✅ **Le modèle qui tient est celui de `PLINKC` 1.62**, validé sur matériel : compacter (IOCS device 6, `47h`), insérer **avant le premier bloc qui n'est pas un pilote** (donc avant `DATA.BAS`), décaler les blocs suivants vers le haut, puis **recaler `TXTBAS`/`DATBAS`** (`linkbas` : IOCS `41h` sur les noms rangés en `[baswrk]+72h`/`+7Eh`) — sur **tous** les chemins qui suivent le compactage, erreurs comprises. Mesuré avec `BASEXT-DRV` 0.2 : bloc en `080018h`, **immobile** après une ligne tapée, `DATA.BAS` et `TEXT.BAS` regrossissant derrière lui.

📖 **La ROM sait elle-même créer un bloc en tête** : la commande IOCS `48h` du device 6,
`block_create_top`, « création d'un bloc mémoire **en tête** des blocs, propre au PC-E500 »
(`Data/FCSFunctions.json` ; `(ch)` = lecteur, `X` = nom, `Y` = taille). Nos installateurs
n'en usent pas — ils compactent par `47h` puis insèrent à la main —, et un pilote d'époque,
`EXTSLOT`, **étend** cette commande plutôt que de la contourner (`07` §3bis). **Piste à
mesurer** : `48h` rend-elle inutile le décalage manuel des blocs du point 3 ? 📖 La lecture du
traitement (`0F034Bh`) va dans ce sens — il contrôle la place, refuse un nom déjà pris (`41h`),
**efface le bit du lecteur** dans `[(iocsw)+3Ah]` (`0F0330h` ; c'est l'octet que l'installateur de
`BASEXT-DRV` remet à zéro) puis **déplace la mémoire** par la commande `43h` `block_transfer`, avant
de copier un gabarit d'en-tête de `22h` octets. La commande `45h` est la même routine **sans** le
déplacement. Sonde, protocole et relevés :
`C:\Claude\BASEXT-DRV\sondes\` (`T48.ASM`, `T48.BAS`, `README.md`).

✅ **Première mesure, PC-E500S réel, 2026-09-25 (J.-F. Albouy) : `48h` REFUSE, erreur `0Ch`, et ne
touche à rien.** Chaîne, `TXTBAS`/`DATBAS` et `[(iocsw)+3Ah]` identiques avant et après. La ROM
explique le refus : le traitement appelle d'abord `SUB_F0244`, qui rend dans `Y` **l'espace libre
après la chaîne**, puis fait `sub y,22h` et part en erreur `0Ch` si la soustraction emprunte. Or
`S1BTM` − `FIN` valait **1 octet** — la chaîne est jointive, et c'est le cas normal (point 1
ci-dessus). Le carnet confirme le sens du code : pour la commande voisine `45h`, « `00Ch` mémoire
insuffisante ».

**Ce que cela apprend :**

- ✅ **`47h` `condense` avant `48h` n'est pas une précaution, c'est une condition.** L'installateur
  de `BASEXT-DRV` compacte déjà en premier ; la mesure lui donne raison.
- ✅ Un refus de `48h` est **sans effet de bord** : la commande vérifie avant d'agir.
- ⚠️ **Le carnet prête à `48h` un paramètre que la ROM n'emploie pas.** Il annonce « `Y` = taille » ;
  `Y` est écrasé dès la deuxième instruction, la chaîne n'est décalée que de `22h` octets, et le
  gabarit écrit une taille de `22h` avec l'attribut `20h`. 📖 `48h` créerait donc un bloc **vide**,
  la taille venant ensuite de `42h` `block_resize` (`(ch)`, `a` = 0/1, `X` = nom, `Y` = taille ;
  rend `Y` = taille possible). La voie ROM complète serait `47h` → `48h` → `42h`, plus la pose de
  l'attribut. **Non mesurée** : l'installateur ne bouge pas tant qu'elle ne l'est pas.

### Trois façons de reloger un pilote — dont une d'époque

Un bloc copié à une adresse choisie à l'installation doit voir ses adresses absolues corrigées.
Trois modèles existent, et le troisième est le plus surprenant :

| Modèle | Comment | Où |
|---|---|---|
| **Table mesurée** | double assemblage à deux origines, chaque octet qui change est rangé dans un champ de 2 ou 3 octets, table émise au format Kon et vérifiée à une troisième origine | `BASEXT-DRV/outils/reloc.py` ✅ |
| **Table déclarée** | préfixe `rel` devant chaque instruction à reloger ; l'assembleur émet la table après le code | A62 (N. Kon), `PLINKC`, `xasm2026-4` (`05` §7bis) ✅ |
| **Analyse du programme** | **aucune table** : l'installateur *analyse* le code et reconnaît lui-même les adresses à corriger | `INSTd`/`INSTt` 1.05 (TORO, 1994) 📖 |

📖 Le troisième impose des règles à la source, que sa notice énonce (`07` §3bis) et qui disent
bien ce qu'une analyse peut et ne peut pas faire :

- **code et données séparés** — le code avant `@@PEND`, les données entre `@@PEND` et `@@DEND` ;
- **aucune valeur de 3 octets en `0Exxxxh`** : elle serait prise pour une adresse du programme
  (écrire `1Exxxxh`). C'est le revers de la convention d'assemblage de cette famille, qui donne
  aux labels une adresse **logique** en `0E0000h` (`ORG 0E0000H,adresse physique`, `05` §7ter) ;
- **aucune référence d'adresse par table** (`JP [X]`) : seul le programme est relogé, pas les
  données ;
- une **somme de contrôle** à calculer en exécutant le programme une fois après l'assemblage.

Il installe sur `S1:` **ou `S2:`** et cherche un numéro de device IOCS libre — la même conduite
que PLINKC. ⚠️ Rien de tout cela n'a été assemblé ni mesuré ici.

Autres faits mesurés au passage :

- ✅ **`FILES` affiche la taille du bloc moins `22h`** (34 octets d'en-tête) pour `TEXT`, `FUNCKEY`, `AER`, `BASEXT.SYS` et `PLINK.SYS` — mais **pas** pour `DATA.BAS`, dont le bloc est plus grand que le fichier. Hypothèse : `FILES` affiche `[+16h]` − `22h`, la taille du fichier. Non vérifié.
- ✅ **`s1_btm` suit la réservation de la zone langage machine octet pour octet** : réserver 6144 octets l'abaisse de 6144, revenir à 16 le remonte de 6128, et `DATA.BAS` reprend la place.
- ✅ Après `KILL` d'un bloc, **la ROM recale elle-même** `TXTBAS`/`DATBAS`.
- 📖 Un bloc **`ENG     $$$`** peut apparaître : c'est un fichier de travail de la ROM (`"S1:ENG     .$$$"` en `0DF995h`, voisin du catalogue de formules), pas un pilote.
- ✅ **Un deuxième pilote au même modèle** : `HISTORY.SYS` (`HISTDRV`, 851 octets, `FILES` 817) s'insère en **`080018h`** sur PC-E500S et en **`0B8018h`** sur PC-E500, en tête de `S1:`, devant `DATA.BAS` — émulateurs, 2026-09-24 (`12-extensions-basic.md` §17quater).
- ⛔ **Limite mesurée, commune avec PLINKC** : un pilote inséré **sous** un autre puis supprimé par
  `KILL` fait descendre celui du dessus **sans relocation**, et la machine tombe. ✅ Mesuré le
  2026-09-25 sur PC-E500S réel (J.-F. Albouy) : `PLINK.SYS` en `080018h` sous `BASEXT.SYS` en
  `080938h`, `KILL "S1:PLINK.SYS"` → arrêt, hard RESET. Trois pointeurs extérieurs deviennent faux
  d'un coup : la tête de `d_link` et les deux crochets du BASIC. **La conduite sûre est de détacher
  le pilote du dessus d'abord** (rendre les crochets, délier le maillon), puis de retirer le bloc,
  puis de rattacher à la nouvelle adresse — procédure et essai dans
  `C:\Claude\BASEXT-DRV\essais\KILLSOUS.BAS`.
- ⚠️ **`SET` et `KILL` sont des commandes de mode direct** : un programme BASIC qui les contient
  rend `Direct command error` sur la ligne fautive. Toute procédure de retrait de bloc se termine
  donc par des commandes **tapées**, jamais par un programme qui ferait tout.

## 8. Zone haute fixe (`FFFD8H`–`FFFFFH`)

| Adresse | Contenu |
|---|---|
| `FFFD8H` | `secure_work_call` — **réservation d'une zone** depuis le BASIC : `POKE &BFE03,` adresse (3 o) `,` valeur (3 o) puis `CALL &FFFD8`. La ROM (`0F964Fh`) relit ces six octets et appelle **IOCS `042h` `secure_work`** du device 8. Usage détaillé, et la réservation de la zone langage machine : `12-extensions-basic.md` §14. |
| `FFFDCH` | `iocs_call3` — appel IOCS **depuis le BASIC**, sans code machine : `POKE &BFE00,cl,ch,il` puis `CALL &FFFDC`. La ROM (`0EF01Ah`) fait `mv il,[0BFE02h]` / `mvw (cl),[0BFE00h]` / `callf iocs_call`. Exemple du manuel : `POKE &BFE00,8,0,&41 : CALL &FFFDC` éteint la machine. |
| `FFFE4H` | Point d'entrée **FCS** (`callf fcs_call`) — voir `04-fcs-iocs.md`. |
| `FFFE8H` | Point d'entrée **IOCS** (`callf iocs_call`), entrée principale du BIOS. |
| `FFFF0H` | Version majeure ROM (`8` = famille PC-E500S, dont le PC-U6000 ; `7` = PC-E500 et PC-E550 ; `5` = série ancienne de PC-E500). |
| `FFFF1H` | Version mineure ROM — ✅ **c'est elle qui dit où l'éditeur range sa touche** (§3bis) : `HISTDRV` la lit à l'installation et adapte son image. **Elle varie** : `8.3` PC-E500S, `8.4` PC-U6000, `7.2` PC-E500 japonais, `7.3` PC-E500, `7.5` PC-E500-BL et PC-E550, `5.3` série ancienne (`SC62015Disassembler/Data/RomVersions.csv`). ⛔ Une version antérieure écrivait « `3` pour les deux modèles ». |
| `FFFF2H`–`FFFF9H` | Réservé / non documenté. |
| `FFFFAH`–`FFFFCH` | **Vecteur d'interruption matériel** (3 octets, pointeur). |
| `FFFFDH`–`FFFFFH` | **Vecteur RESET** (3 octets) — saute vers le lancement du menu BASIC. |

## 9. Voir aussi

- `01-architecture-cpu-sc62015.md` — registres CPU, pagination, modes d'adressage.
- `04-fcs-iocs.md` — catalogue des fonctions FCS/IOCS accessibles via `FFFE4H`/`FFFE8H`.
- `06-ecosysteme-outils.md` — outils du projet qui exploitent ces adresses (PLINKC162, `BASEXT-DRV`, désassembleur...).
- `C:\Claude\BASEXT-DRV\CONCEPTION.md` — les mesures du §7bis, et un installateur de pilote complet (relocation, insertion, recalage du BASIC, désinstallation).
- `07-sources-et-bibliographie.md` — provenance détaillée (manuels Sharp, xlsx interne, site d'Arno Welzel).
