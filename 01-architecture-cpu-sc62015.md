# Architecture du CPU SC62015 (ESR-L)

*Rédigé le 2026-08-26 — mis à jour le 2026-09-14*

> Fichier du référentiel PC-E500S / SC62015. Voir `00-index.md` pour la vue d'ensemble et la liste des sources.

## 1. Identité du CPU

Le **SC62015** est un microprocesseur CMOS 8 bits propriétaire Sharp, désigné en interne **ESR-L** (« new-SC », par opposition à l'ancien **SC61860 / ESR-H** utilisé sur les PC-1350, PC-1360, PC-1403...). On le trouve dans :

- la famille **Sharp PC-E500 / PC-E500S / PC-E550** (pocket computers, cadence 2,304 MHz sur PC-E500S) ;
- les organiseurs **OZ-7000 / IQ-7000** (Wizard) ;
- une variante étendue **ESR-P** équipe les organiseurs **DB-Z** et **PV-F1** ainsi que certains **Zaurus** — elle ajoute un mode de compatibilité capable d'exécuter du code ESR-H (SC61860), sans que l'inverse soit vrai.

Sharp n'a jamais publié de manuel CPU largement diffusé : la documentation qui circule aujourd'hui provient de deux manuels internes retrouvés et numérisés (*ESR-L CPU Instruction Manual* et *Technical Reference Manual PC-E500*, cf. `07-sources-et-bibliographie.md`), complétés par la rétro-ingénierie de la scène ordinateur de poche japonaise depuis les années 1990.

Une description japonaise (wizforest.com, « PC-E500 CPU ») résume bien le caractère composite du jeu d'instructions : une page interne rapide de 256 octets à la 6502, des registres pointeurs larges (20 bits) façon 68000, et une richesse de modes d'adressage et de paires de registres qui rappelle le 8086 — « le meilleur (et le pire) des trois » selon l'auteur.

## 2. Registres

| Registre | Taille | Rôle | Remarques |
|---|---|---|---|
| `A` | 8 bits | Accumulateur | Octet bas de `BA`. La plupart des opérations arithmétiques/logiques passent par `A`. |
| `B` | 8 bits | Registre auxiliaire | Octet haut de `BA`. |
| `BA` | 16 bits | Paire `B:A` | Registre général 16 bits. |
| `IL` | 8 bits | Compteur de boucle, octet bas | Décrémenté automatiquement par les instructions de boucle (`MVL`, `ADCL`, `WAIT`...). |
| `IH` | 8 bits | Octet haut de `I` | Remis à 0 par les `MV r1,...` quand `r1=IL`. |
| `I` | 16 bits | Paire `IH:IL` | Compteur de boucle 16 bits. |
| `X` | 20 bits utiles (stocké sur 3 octets) | Registre pointeur | Accès direct à tout l'espace mémoire externe (1 Mo). |
| `Y` | 20 bits utiles (3 octets) | Registre pointeur | Idem `X`. |
| `U` | 20 bits utiles (3 octets) | Pointeur de pile utilisateur | Empile/dépile via `PUSHU`/`POPU` ; utilisable aussi comme pointeur générique. |
| `S` | 20 bits utiles (3 octets) | Pointeur de pile système | Utilisé par `CALL`/`RET`/`CALLF`/`RETF`/`RETI` et les interruptions. |
| `PC` | 16 bits + page | Compteur de programme | Voir §3 « Pagination ». |
| `PS` | 4 bits | Registre de page (*Page Segment*) | Sélectionne la page de 64 Ko d'où sont exécutées les instructions ; combiné à `PC` il forme une adresse effective 20 bits. |
| `F` | 8 bits | Registre d'état | Seuls deux bits sont documentés : `C` (retenue) et `Z` (zéro). Pas de flags signe/débordement séparés. |

**Point d'attention (concilié entre les deux sources) :** le manuel CPU officiel (*ESR-L Instruction Manual*) présente `X`, `Y`, `U`, `S` comme des registres **20 bits**, cohérent avec l'espace externe adressable de 1 Mo (2²⁰ octets). La table de reconstruction utilisée par le désassembleur du projet (`README - PC-E500 Instruction Table.md`) les qualifie de « 24 bits » : il s'agit du **format de stockage physique** (3 octets = 24 bits), dont seuls les 20 bits de poids faible sont significatifs comme adresse. Les deux descriptions sont donc compatibles ; ce référentiel retient « 3 octets stockés, 20 bits d'adresse utile ».

### 2.1 Table d'encodage des registres dans les opcodes

Les post-bytes et certains octets d'opcode encodent un registre sur 3 bits, avec un regroupement par « classe de taille » `r₁`…`r₄` :

| Classe | Registre | Valeur | Binaire |
|---|---|---|---|
| r₁ (8 bits) | `A` | 0 | `000` |
| r₁ (8 bits) | `IL` | 1 | `001` |
| r₂ (16 bits) | `BA` | 2 | `010` |
| r₂ (16 bits) | `I` | 3 | `011` |
| r₃ (adressage externe) | `S` | 7 | `111` |
| r₄ (20 bits) | `X` | 4 | `100` |
| r₄ (20 bits) | `Y` | 5 | `101` |
| r₄ (20 bits) | `U` | 6 | `110` |

`r'₃` (utilisé pour l'adressage externe indirect `[r'₃]`, `[r'₃++]`, `[--r'₃]`, `[r'₃±n]`) désigne l'un de `X`, `Y`, `U`, `S`.

## 3. Espaces mémoire

Le SC62015 distingue strictement deux espaces adressables, avec une syntaxe d'écriture différente en assembleur (héritée de XASM et reprise partout dans l'écosystème) :

- **`(adresse)`** — mémoire **interne** : les 256 octets de RAM rapide intégrée au CPU.
- **`[adresse]`** — mémoire **externe** : jusqu'à 1 Mo (ROM + RAM utilisateur + cartes mémoire), adressée sur 20 bits.

### 3.1 Mémoire interne (256 octets)

| Zone | Adresses | Contenu |
|---|---|---|
| RAM générale | `00H`–`EBH` (236 octets) | Libre, utilisée par le système et les programmes (variables BASIC, buffers...). |
| Pointeurs d'adressage | `ECH` `BP`, `EDH` `PX`, `EEH` `PY` | Registres utilisés par les modes d'adressage internes composés (§3.3). |
| Contrôle carte mémoire | `EFH` `AMC` | *Address Modify Control* : permet de rendre virtuellement contiguës deux zones d'adresses de carte RAM discontinues (CE1/CE0). |
| Clavier / port E | `F0H`–`F6H` | `KOL`/`KOH` (sortie clavier), `KIL` (entrée clavier), `EOL`/`EOH`/`EIL`/`EIH` (port E, extension). |
| UART | `F7H` `UCR`, `F8H` `USR`, `F9H` `RXD`, `FAH` `TXD` | Contrôle/état UART, tampons réception/émission (liaison série `COM:`). |
| Interruptions | `FBH` `IMR`, `FCH` `ISR` | Masque et statut des 8 sources d'interruption (voir §5). |
| Système | `FDH` `SCR`, `FEH` `LCC`, `FFH` `SSR` | Contrôle système (horloges, LCD), contraste LCD, statut système (touche ON, reset...). |

Le détail bit-à-bit de `UCR`/`USR`/`IMR`/`ISR`/`SCR`/`LCC`/`SSR` est donné dans `03-memoire-et-systeme-pc-e500s.md` (repris du manuel ESR-L et de la table d'instructions).

Au-delà de cette zone documentée par Sharp (`ECH`-`FFH`), la rétro-ingénierie du projet (listings XASM réels : `register.lst`, `TMAP2020.lst`, `tycom.LST`, `tydos.LST`) a identifié des adresses internes utilisées par le firmware/BASIC à l'intérieur de la zone RAM générale, notamment `CBH` (`txtbas`, pointeur TEXT.BAS courant), `CEH` (`datbas`), `D6H`/`D7H` (`cl`/`ch`), `DAH` (`si`, pointeur source 3 octets) et `E6H` (`iocsw`, zone de travail IOCS). Voir `Data/InternalRAMNames.json` dans `SC62015Disassembler` et `03-memoire-et-systeme-pc-e500s.md`.

### 3.2 Mémoire externe (1 Mo) et pagination

`PC` s'incrémente de 16 bits (donc un « segment » = 64 Ko de code), et le registre **`PS`** (4 bits, pages `0H`–`FH`) sélectionne la page de 64 Ko active — l'adresse effective d'exécution est `PS:PC` sur 20 bits. Les instructions de saut/appel courtes (`JP mn`, `JR ±n`, `CALL mn`, sauts conditionnels) restent dans la page courante ; les formes *far* (`JPF lmn`, `CALLF lmn`) chargent une adresse 20 bits complète (`l`= octet de page/poids fort, `m`,`n` = poids faible) et positionnent `PS`.

Les registres `X`, `Y`, `U`, `S` portent chacun une adresse 20 bits complète et peuvent donc adresser directement n'importe quel octet des 1 Mo, sans dépendre de `PS`.

### 3.3 Modes d'adressage internes composés (octet PRE)

Certaines instructions combinent deux accès à la mémoire interne dont chacun peut utiliser un mode différent : direct `(n)`, relatif à `BP` (`(bp+n)`), relatif à `PY` (`(py+n)`), ou combiné `(bp+px)` / `(bp+py)`. Cette combinaison est signalée par un **octet PRE** placé avant l'opcode réel (valeurs `21H`–`27H` et `30H`–`37H`) :

| 1er opérande \ 2e opérande | `(n)` | `(BP+n)` | `(PY+n)` | `(BP+PY)` |
|---|---|---|---|---|
| **`(n)`** | 32H | 30H | 33H | 31H |
| **`(BP+n)`** | 22H | — | 23H | 21H |
| **`(PX+n)`** | 36H | 34H | 37H | 35H |
| **`(BP+PX)`** | 26H | 24H | 27H | 25H |

`BP` (`ECH`), `PX` (`EDH`) et `PY` (`EEH`) sont eux-mêmes situés en mémoire interne et chargés au préalable par le programme.

### 3.4 Modes d'adressage externes

| Écriture | Signification |
|---|---|
| `[lmn]` | Adresse externe directe 20 bits (`l`,`m`,`n` = octets de poids fort à faible). |
| `[r'₃]` | Indirect via registre (`X`, `Y`, `U` ou `S`). |
| `[r'₃++]` / `[--r'₃]` | Indirect avec post-incrément / pré-décrément (taille = celle de l'opérande transféré). |
| `[r'₃±n]` | Indirect avec décalage 8 bits signé. |
| `[(n)]` | Indirect via un pointeur 20 bits rangé en mémoire interne à l'adresse `(n)` (mémoire interne → externe). |
| `[(m)±n]` | Comme ci-dessus avec décalage signé `n` appliqué au pointeur lu en `(m)`. |

## 4. Pile système et pile utilisateur

Le CPU dispose de **deux piles indépendantes**, toutes deux en mémoire externe :

- **Pile système (`S`)** : utilisée automatiquement par `CALL`/`CALLF`/`RET`/`RETF`/`RETI`, ainsi que par les interruptions matérielles et `IR`/`RESET`. `PUSHS F` / `POPS F` permettent d'y empiler explicitement les flags.
- **Pile utilisateur (`U`)** : gérée uniquement par le programme via `PUSHU`/`POPU` (registres `A`, `IL`, `BA`, `I`, `X`, `Y`, `F`, `IMR`). Sert typiquement à la sauvegarde de registres dans les sous-routines applicatives, sans interférer avec la pile d'appels système.

## 5. Interruptions

Le SC62015 possède **8 sources d'interruption**, mais un **unique vecteur matériel** (`FFFFAH`-`FFFFCH`, cf. `03-memoire-et-systeme-pc-e500s.md`) : la routine pointée par ce vecteur doit lire `ISR` (`FCH`) pour déterminer la cause et distribuer vers 8 adresses de traitement rangées en RAM (modifiables par l'utilisateur), selon le manuel *Technical Reference Manual PC-E500*.

| Source | Bit `IMR`/`ISR` | Déclenchement |
|---|---|---|
| Timer rapide (MTI) | bit 0 | Toutes les 4 ms ou 16 ms selon `SCR` bit 1 (utilisé pour le scan clavier, le clignotement du curseur...). |
| Timer lent (STI) | bit 1 | Toutes les ~0,5 s ou ~2 s selon `SCR` bit 2. |
| Touche (KEYI) | bit 2 | Une touche du matriçage clavier (`KI0`-`KI7`) passe à 1. |
| Touche ON (ONKI) | bit 3 | Appui sur la touche `ON`. |
| Émission SIO (TXRI) | bit 4 | Fin de transmission d'un octet UART. |
| Réception SIO (RXRI) | bit 5 | Réception d'un octet UART. |
| Externe (EXI) | bit 6 | Signal externe (contrôleur de batterie). |
| Logicielle | — | Déclenchée par l'instruction `IR`. |

`RETI` restaure `IMR`, `F`, `PC` et `PS` depuis la pile système dans cet ordre (voir `02-jeu-instructions.md`).

## 6. Comportement HALT / OFF / RESET

| | `HALT` | `OFF` | `RESET` |
|---|---|---|---|
| Registres | Tous conservés | Tous conservés | `PC` recharge le vecteur reset ; les autres registres sont conservés |
| Flags C/Z | Indéfinis | Indéfinis | Conservés |
| Mémoire interne | `USR` (F8H) bits 0-2/5 remis à 0 ; `SSR` (FFH) bit 2 et `USR` bits 3-4 mis à 1 ; le reste est conservé | Identique à HALT | `AMC` bit 7 (`AME`)/`UCR`/`USR` bits 0-2/5/`IMR`/`SCR` remis à 0 ; `SSR` bit 2 et `USR` bits 3-4 mis à 1 ; le reste conservé |

> ⚠️ **Le manuel écrit « `ACM` (FEH) bit 7 »** à la page 34 : c'est une double coquille, lettres
> **et** chiffres transposés, pour `AMC` (`EFH`) bit 7 — le nom `ACM` n'apparaît nulle part
> ailleurs dans le manuel, et `FEH` est `LCC`. Tranché dans
> `SC62015Disassembler/Docs/Synthese/Registres-materiels.md` (§ `AMC`), avec la routine de la ROM
> qui dimensionne la carte (`0F0E7Bh`).

## 7. Voir aussi

- `02-jeu-instructions.md` — jeu d'instructions complet, table 16×16, catégories.
- `03-memoire-et-systeme-pc-e500s.md` — carte mémoire externe complète du PC-E500S, détail des registres `F0H`-`FFH`, vecteurs.
- `07-sources-et-bibliographie.md` — provenance de chaque affirmation (manuels Sharp, sites japonais/allemands, dépôts de rétro-ingénierie).
