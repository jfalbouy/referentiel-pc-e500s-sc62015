# Mémoire et système du PC-E500S

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier détaille la carte mémoire du PC-E500S (interne + externe), les registres d'E/S, les vecteurs d'interruption et les points d'entrée IOCS. Sources principales : le manuel *ESR-L CPU Instruction Manual*, le *Technical Reference Manual PC-E500*, et la feuille de dépouillement `Docs/Doc technique/PCE500 Description mémoire.xls.xlsx` déjà constituée dans le projet `SC62015Disassembler` (adresses recoupées avec des listings XASM réels).

## 1. Vue d'ensemble

Rappel (détaillé dans `01-architecture-cpu-sc62015.md`) : 256 octets de mémoire **interne** (`(adresse)`) intégrés au CPU, et jusqu'à 1 Mo de mémoire **externe** (`[adresse]`, 20 bits) répartie entre ROM système (256 Ko), RAM interne utilisateur (32 Ko de base, extensible) et cartes mémoire optionnelles (slots S1/S2/S3, jusqu'à 256 Ko chacune sur PC-E500S).

## 1bis. Carte mémoire externe par ligne `CE` (répartition physique du 1 Mo)

Répartition des 8 lignes de sélection `CE0`-`CE7` (rappelées en `08-cartes-memoire.md` §1) sur l'espace externe de 1 Mo, reconstituée par la communauté (forum silicium.org, fil « PC-E500S 256K », 2015 — cf. `07-sources-et-bibliographie.md`) :

| Ligne | Plage d'adresses | Polarité | Rôle |
|---|---|---|---|
| `CE6` | `10000H`-`1FFFFH` | Négative | Carte graphique (logique d'extension). |
| `CE3` | `20000H`-`3FFFFH` | Négative | Non identifié précisément (« comme DELTA » selon la source). |
| `CE1` | `40000H`-`7FFFFH` | Positive | **Carte mémoire d'extension, slot `S2:`** — sous-découpée selon la capacité de la carte (64 Ko/32 Ko/16 Ko/8 Ko occupent des sous-plages différentes à partir de `40000H`). |
| `CE0` | `80000H`-`BFFFFH` | Négative | **RAM interne intégrée, `S1:`/lecteur `E:`** — fenêtre de 256 Ko (`80000H`-`BFFFFH`), dont seule la partie haute est peuplée sur les machines à faible capacité (ex. PC-E500 32 Ko : `B8000H`-`BFFFFH` ; PC-E550 64 Ko : `B0000H`-`BFFFFH`). Voir §1ter et `09-cartes-meres-ram-interne.md`. |
| `CE7` | `BC000H`-`BFFFFH` | Positive | Réservation réseau/extension (chevauche la fin de `CE0`). |
| `CE2` | `C0000H`-`FFFFFH` | Négative | **ROM système, slot `S3:`/lecteur `G:`**. |
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
| `EFH` | `AMC` | Address Modify Control | `AME`(7) active la jonction virtuelle CE1/CE0 · `AM5-0`(6-1) taille CE0 (000000=2 Ko … 111111=128 Ko). |
| `ECH`/`EDH`/`EEH` | `BP`/`PX`/`PY` | Pointeurs d'adressage interne | Voir `01-architecture-cpu-sc62015.md` §3.3. |

## 3. Adresses système en mémoire interne (zone RAM générale, `00H`-`E3H`)

Ces adresses ne sont pas documentées dans le manuel CPU (qui ne décrit que `ECH`-`FFH`) : elles ont été identifiées par rétro-ingénierie du firmware/BASIC (listings XASM réels, cf. `07-sources-et-bibliographie.md`) et consolidées dans le tableur `PCE500 Description mémoire.xls.xlsx` et `Data/InternalRAMNames.json`.

| Addr | Sigle | Description |
|---|---|---|
| `CBH` | `TXTBAS` | Adresse (3 octets) où réside `TEXT.BAS` courant. |
| `CEH` | `DATBAS` | Adresse où réside `DATA.BAS` courant. |
| `D1H` | `BASPTR` | **Pointeur** (3 octets) vers la zone de travail de l'interpréteur BASIC. Nommé `BASWRK` jusqu'en septembre 2026 — même nom que la zone externe `BFD0EH`, ce qui rendait l'adresse interne innommable dans une source et a coûté une dizaine d'essais : voir `12-extensions-basic.md` §7. Porte les crochets d'extension du BASIC en `[(0D1H)+090H]` et `[(0D1H)+093H]`. |
| `D4H`/`D5H` | `BL`/`BH` | Registre `B` étendu (paire haute/basse, usage interne interpréteur). |
| `D6H`/`D7H` | `CL`/`CH` | Registre `C` étendu. |
| `D8H`/`D9H` | `DL`/`DH` | Registre `D` étendu. |
| `DAH` | `SI` | Pointeur source, 3 octets. |
| `DDH` | `DI` | Pointeur destination, 3 octets. |
| `E6H` | `IOCSW` | Zone de travail IOCS (interne). |

## 4. Mémoire externe : zones système principales

| Adresse | Sigle | Description |
|---|---|---|
| `BFC15`–`BFCDE` | `ldAdSlot0`/… | Table des slots de cartes mémoire : `ldAdSlot2`/`cpSlot2` (S3, défaut `C0000`/`&40`×2Ko), `ldAdSlot1`/`cpSlot1` (S2, défaut `40000`/`&80`×2Ko = 256 Ko sur E500S), `ctrlCRAM`, `ldAdSlot0`/`cpSlot0` (S1, défaut `80000`/`&80`×2Ko = 256 Ko sur E500S, `&B8000` sur E500 d'origine). |
| `BFC27`/`BFC28` | `SCRNX`/`SCRNY` | Prochaine coordonnée d'affichage sur `STDO:`/`SCRN:`. |
| `BFC2A` | `LINPTN` | Cadre de points affiché dans une boîte 16 points. |
| `BFC2D`–`BFC41` | — | Table de conversion des codes clavier (normal / SHIFT / CTRL), 1 et 2 octets, + *hook* de la routine de traitement clavier (`&F1B4D` par défaut). |
| `BFC6D` | `FHANDLE` | Table des handles de fichiers ouverts. |
| `BFC7D` | `DEFDRV` | Nom du lecteur par défaut (7 octets, ex. `F:____0`). |
| `BFC84`–`BFC96` | — | Adresses (3 octets chacune) des polices de caractères par plage de codes (0-1F, 20-7F, 80-9F, A0-DF, E0-FF) + `DOTSOP` (mode d'affichage précédent). |
| `BFC9B`–`BFCA1` | `CSRX`/`CSRY`/`WIDTH`/`HEIGHT`/`LCDMOD` | Curseur LCD, largeur (`&28`=40 col.), hauteur (4 lignes), mode vidéo (normal/inversé). |
| `BFCA2` | `IOCSH` | Adresse du prochain en-tête de chaîne IOCS (voir §7). |
| `BFCBA`–`BFCBF` | — | Paramètres clavier : délai avant répétition, cadence de répétition, délai d'extinction automatique (`&B0`×0,5s), drapeaux *break*/pile faible, répétition/clic activés. |
| `BFCC0`/`BFCC3` | — | Table principale de conversion code matériel → code logiciel clavier, *hook* de traitement. |
| `BFCC6`–`BFCDB` | voir §6 | Vecteurs RAM des 8 sources d'interruption. |
| `BFCDE` | `UWORK` | Dernière adresse du slot S1 + 1. |
| `BFCE1` | `SWORK` | Zone de la pile système (`S`). |
| `BFD0E` | `BASWRK` | Zone de travail BASIC (externe). Ce nom est celui du listing de E. Kako (`register.lst`, 1990), vérité terrain du corpus. ⚠️ À ne pas confondre avec `BASPTR` = `0D1H`, le pointeur en RAM interne : un `mv x,(baswrk)` serait tronqué à 8 bits par l'assembleur, sans avertissement. |
| `BFD17` | `IOCSWRK` | Zone de travail IOCS (externe). |
| `BFD1A` | `USRWRK` | **Zone langage machine** — début de la zone utilisateur pour les programmes en code machine (`CALL &BFD1A` typique après `LOADM`/assemblage). ⚠️ Elle ne fait que **23 octets** : les paramètres SIO commencent en `BFD31H`. Un programme plus long assemblé là écrase la configuration de la liaison série, et la panne se manifeste au transfert *suivant*. Au-delà, charger en `BF000H`. |
| `BFD31`–`BFD62` | — | Paramètres SIO (temporisation, vitesse, parité, fin de ligne `&1A`, délais d'ouverture/fermeture). |
| `BFD42`–`BFD53` | — | Constantes de codage cassette (`CAS:`) : longueurs et seuils des niveaux logiques 0/1, blocs d'en-tête. |
| `DF820`–`DF8A9` | — | Chaîne des en-têtes de drivers IOCS (voir §7). |

*(Toutes les adresses ci-dessus proviennent de `PCE500 Description mémoire.xls.xlsx`, elles-mêmes vérifiées sur listings XASM réels ; les valeurs marquées « défaut » correspondent aux réglages ROM standard et peuvent différer selon la variante S1/E500/E500S.)*

## 5. Cartes mémoire — modes S1 / S2 / B

D'après le manuel PC-E500S et confirmé par la documentation d'Arno Welzel (Allemagne, cf. `07-sources-et-bibliographie.md`) : la commande BASIC `MEM$` lit/positionne le mode d'utilisation de la carte insérée dans le logement au dos de l'appareil :

| Mode | Effet |
|---|---|
| `S1` | Mémoire interne = programmes ; la carte RAM est accessible en lecteur `F:`. |
| `S2` | La carte RAM contient les programmes ; la mémoire interne devient accessible en lecteur `E:`. |
| `B` | La carte RAM étend la mémoire interne (pas de lecteur séparé). |

Formatage d'une carte en lecteur : `MEM$="S1"` puis `INIT "F:31K"` (taille selon capacité de la carte). Chaque carte RAM embarque sa propre pile de sauvegarde (maintien ~1 an hors alimentation). Des cartes **FRAM** modernes (ferroélectriques, jusqu'à 256 Ko, non volatiles sans pile) sont utilisables en remplacement — voir `07-sources-et-bibliographie.md`.

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

À partir de `DF820H` (`IOCSH`, §4), une chaîne d'en-têtes décrit chaque driver installé (nom de lecteur(s), point d'entrée). Table des drivers standard :

| En-tête | Lecteur(s) | Point d'entrée |
|---|---|---|
| `DF820` | `STDI:` `KYBD:` (clavier) | `F16F2` |
| `DF833` | `STDO:` `SCRN:` (écran) | `F21E7` |
| `DF846` | `COM:` (liaison série) | `EAA71` |
| `DF853` | `STDL:` `PRN:` (imprimante) | `EA4BD` |
| `DF865` | `CAS:` (cassette) | `E96E8` |
| `DF872` | `S1:` `S2:` `S3:` (carte mémoire) | `E493E` |
| `DF884` | `E:` `F:` `G:` (fichier mémoire) | `E493E` |
| `DF893` | `X:` `Y:` (lecteur de disquette) | `EF029` |
| `DF89C` | `SYSTM:` | `B66C0` |
| `DF8A9` | fonction système | `FFFFF` (fin de chaîne) |

Ce format d'en-tête chaîné (adresse du suivant sur 3 octets + identifiant + attributs + adresse d'entrée 3 octets + nom ASCII) permet d'installer un nouveau driver résident sans modifier la ROM : c'est le mécanisme qu'utilise `PLINKC162` (lecteur `L:`, voir `06-ecosysteme-outils.md`) et que documente `Data/SystemDataRegions.csv` sous la clé `iocs_hdr_tbl` (`DF820`, 152 octets, jusqu'à sentinelle `FFFFF`).

## 8. Zone haute fixe (`FFFF0H`–`FFFFFH`)

| Adresse | Contenu |
|---|---|
| `FFFE4H` | Point d'entrée **FCS** (`callf fcs_call`) — voir `04-fcs-iocs.md`. |
| `FFFE8H` | Point d'entrée **IOCS** (`callf iocs_call`), entrée principale du BIOS. |
| `FFFDCH` | Appel IOCS alternatif par registres (`il`,`cl`,`ch`) : `POKE &BFE00,cl,ch,il` puis `CALL &FFFDC`. |
| `FFFF0H` | Version majeure ROM (`8` = PC-E500S, `7` = PC-E500). |
| `FFFF1H` | Version mineure ROM (`3` pour les deux modèles). |
| `FFFF2H`–`FFFF9H` | Réservé / non documenté. |
| `FFFFAH`–`FFFFCH` | **Vecteur d'interruption matériel** (3 octets, pointeur). |
| `FFFFDH`–`FFFFFH` | **Vecteur RESET** (3 octets) — saute vers le lancement du menu BASIC. |

## 9. Voir aussi

- `01-architecture-cpu-sc62015.md` — registres CPU, pagination, modes d'adressage.
- `04-fcs-iocs.md` — catalogue des fonctions FCS/IOCS accessibles via `FFFE4H`/`FFFE8H`.
- `06-ecosysteme-outils.md` — outils du projet qui exploitent ces adresses (PLINKC162, désassembleur...).
- `07-sources-et-bibliographie.md` — provenance détaillée (manuels Sharp, xlsx interne, site d'Arno Welzel).
