# Formats de fichiers et assembleur XASM

> Voir `00-index.md` pour la vue d'ensemble. Synthèse de `SC62015Disassembler/Docs/Doc technique/Documentation_XASM_PC-E500S.md` (déjà très complet) et du `README.md`/`CLAUDE.md` de `SC62015Disassembler` pour le point de vue « lecture » des mêmes formats.

## 1. Généalogie de l'outil

XASM est un **assembleur croisé absolu** (pas d'édition de liens : les adresses sont fixées par `ORG`) pour le CPU ESR-L/SC62015. Sa bannière dit « *(c)1990-1996 N.Kon and E.Kako* », et la généalogie donnée par `xasm2026-4/README.md` est :

| Version | Auteur | Langage | Années |
|---|---|---|---|
| XASM 1.0 | N. Kon | Turbo Pascal | 1990-1993 |
| XASM 1.40 | E. Kako | C (ANSI) | 1995-1996 |

> ⚠️ **Discordance non tranchée.** `07-sources-et-bibliographie.md` §3 rattache le site `kako.com` à « 加古静司 / Seiji Kako » ; la bannière et les sources écrivent « **E.** Kako ». Une version antérieure de ce paragraphe donnait « Seiji Kako » pour auteur de XASM, sans source qui le fonde. Ce référentiel retient **E. Kako**, l'initiale que portent le programme et ses listings.
>
> **N. Kon est très probablement Narihito Kon** (Tokyo Institute of Technology), l'auteur du compilateur VOGUE (1991-1992), dont le nom complet est établi par quatre documents d'époque (`07` §3). Le rapprochement avec le « N. Kon » de XASM 1.0 et de `PLINK` (1990-1994) tient au même milieu, aux mêmes années, et à ce que son courrier d'août 1992 cite les travaux de « Mr. KAKO » — un pilote de kanji pour PC-E500, adresse à l'université de Gifu. **Aucune source ne l'écrit en toutes lettres** : c'est une inférence forte, pas un fait établi.

Quatre générations coexistent dans l'écosystème du projet :

| Génération | Dossier | Nature |
|---|---|---|
| **XASM 1.40** | `XASM Origine/` | Original historique en C (DOS), syntaxe et mnémoniques de référence. |
| **xasm2026-1 / xasm2026-1-2** | `xasm2026-1-2/` | Port maintenu en C11 moderne (GCC/CMake), sorties étendues (Intel HEX, S-Record, MAP, dépendances, UU, dump HxD), compatibilité **octet par octet** vérifiée contre l'original. C'est le « moteur C » dont les bornes sont décrites au §2. |
| **xasm2026-4** | `xasm2026-4/` | Réécriture C# (.NET 8), non-régression **bit-exacte** contre `xasm2026-1`. Reproduit en plus le **préprocesseur de l'assembleur A62 (N. Kon)** — `rel` et la table de relocation — ce qui assemble telles quelles les sources des drivers `ssfdc120` et `PLINKC`. Documentation complète : `xasm2026-4/Documentation/Documentation_XASM2026-4_PC-E500S.md`. |

## 2. Syntaxe source

```asm
LABEL:  INSTRUCTION  OPERANDE1,OPERANDE2   ; commentaire
        INSTRUCTION  OPERANDE
        END
```

- Label : lettres/chiffres/`_`, ne commence pas par un chiffre, 16 caractères max, insensible à la casse. Le `:` après un label est **obligatoire**, y compris devant `EQU`.
- `END` obligatoire en fin de fichier (source principal et chaque fichier inclus).
- **Ligne : 253 caractères au plus** — voir l'avertissement ci-dessous, c'est la seule des trois
  bornes qui ne se signale pas où il faut.
- Nombres : suffixe de base `B`/`O`/`D`/`H` (`0E000h`) ou préfixe `$` hexadécimal ; `_` séparateur visuel. Sans suffixe : décimal. Expressions calculées sur 20 bits.
- Chaînes : `DB 'texte'` développe caractère par caractère ; apostrophe doublée pour l'échapper (`'I don''t know'`).
- Opérateurs (priorité croissante) : `|` (OR) < `&` (AND) < `%` (modulo) < `+`/`-` < `*`/`/`, plus le signe unaire.
- `*` = compteur de position courant (utile pour `TAILLE: EQU *-DEBUT` ou `DS cible-*,0`).

> ⛔ **UNE LIGNE TROP LONGUE N'EST PAS REFUSÉE : ELLE EST COUPÉE EN DEUX.** Le moteur C
> historique lit ses lignes par `fgets(asmtext, 255, …)` (`src/genop.c`). Au-delà de
> **253 caractères**, la fin de la ligne devient **une ligne à part entière**, que l'assembleur
> tente de lire comme du source — d'où un `Label format error` signalé sur la ligne
> **SUIVANTE**, alors que la fautive est celle d'avant. Mesuré : 253 passent, 254 cassent.
>
> ⚠️ Les deux autres bornes du moteur C — labels ≤ 16 caractères, `END` dans chaque include —
> sont dites plus haut, et celles-là **se signalent proprement**. Seule la longueur de ligne
> ment sur l'emplacement de la faute.
>
> Ce n'est pas théorique : la mise aux normes de `pce500.inc` en septembre 2026 a dû renommer
> `sio_open_port_ctrl` (18 caractères) en `sio_open_ctrl` **à la source**, dans
> `SC62015Disassembler/Data/SystemAddresses.json` — c'est un symbole d'adresse que le
> désassembleur émet dans ses opérandes, et le renommer ailleurs aurait fait diverger le carnet
> et la sortie. Les descriptions longues du même carnet, elles, sont ce qui produisait les
> lignes de plus de 253 caractères.

## 3. Directives

| Directive | Effet |
|---|---|
| `ORG expr` | Fixe/avance l'adresse d'assemblage. |
| `EQU` | Constante symbolique. |
| `DB` / `DM` | Émet des octets. |
| `DW` | Émet des mots 16 bits (bas puis haut). |
| `DP` | Émet des pointeurs 3 octets. |
| `DS taille[,valeur]` | Réserve/remplit une zone. |
| `PRE expr` | Émet un octet PRE manuellement. |
| `PRE_ON` / `PRE_OFF` | Génération automatique des PRE-bytes (par défaut off, pour compatibilité historique). |
| `INCLUDE fichier[,arg]*` | Inclusion avec arguments `@0`-`@9`. |
| `MACRO nom,arg... / ENDM` | Macro simple (substitution textuelle). |
| `DEF`/`UNDEF`, `IFDEF`/`IFNDEF`/`ELSE`/`ENDIF` | Symboles et assemblage conditionnel. |
| `LOCAL` / `ENDL` | Blocs de labels hiérarchiques (voir §3.1). |

Extensions `xasm2026-1`/`xasm2026-1-1` : `REPEAT`/`ENDR`, `IFEQ`/`IFNE`/`IFGT`/`IFLT`, `STRUCT`/`ENDS` (calcule `NOM_SIZE`), `SECTION` (regroupement pour rapport `-R`/MAP).

### 3.1 Labels hiérarchiques

| Notation | Portée |
|---|---|
| `label` | Bloc courant. |
| `bloc!label` | Bloc enfant nommé. |
| `..!label` | Bloc parent. |
| `...!label` | Deux niveaux au-dessus. |
| `!bloc!label` | Référence absolue depuis la racine. |

`SCOPE_ON` (extension) rapproche la portée des labels de celle du C : les labels des blocs englobants deviennent visibles depuis l'intérieur.

## 4. Ligne de commande et sorties

Table de `xasm2026-4` (`xasm2026-4/README.md`, analysée dans `src/CommandLineOptions.cs`). Chaque sortie est **optionnelle et explicite** ; le nom peut être collé à l'option (`-Lfoo.lst`) ou séparé par une espace, et s'il est omis le nom du source est repris avec la nouvelle extension.

| Option | Fichier | Contenu |
|---|---|---|
| `-O[f]` | `.obj` | Objet principal (voir §5 pour les formats). |
| `-L[f]` | `.lst` | Listing assemblé (adresses, octets, messages). |
| `-E` | `.err` | Rapport d'erreurs. |
| `-S` | — | Ajoute la table des symboles au listing (avec `-L`). |
| `-U` | — | Ajoute la table des **références croisées** au listing (avec `-L`). |
| **`-K`** | — | **Listing : ne garde, des fichiers inclus, que les constantes `EQU` réellement utilisées** (avec `-L`). Voir ci-dessous. |
| `-H` | — | Désactive le hachage pour le listing des symboles. |
| `-T[type]` | — | Format objet historique : `Z` (ZSH texte), `F` (FTX), `B` (binaire), `H` (hexadécimal), autre = binaire + en-tête XASM 16 octets. |
| `-I[f]` | `.hex` | Intel HEX. |
| `-M[f]` | `.s19` | Motorola S-Record (S1/S9). |
| `-P[f]` | `.map` | Sections et symboles. |
| `-D[f]` | `.d` | Dépendances façon *make*. |
| `-B[f]` | `.uu` | Programme BASIC auto-décodable pour transfert vers PC-E500S. |
| `-X[f]` | `.txt` | Dump hexadécimal façon HxD. |
| `-C` | console | Compteur de lignes. |
| `-W` | console | Avertissements. |
| `-V` | console | Mode verbeux : ajoute la colonne aux diagnostics. |
| `-R` | console | Rapport de taille par `SECTION`. |
| `-?` | console | Aide. |

### `-K` — un listing lisible malgré `pce500.inc`

Inclure `pce500.inc` (près de 280 `EQU`) noie le `.lst`. **`-K`** (avec `-L`) n'y conserve, **des fichiers inclus**, que les constantes `EQU` effectivement **référencées** par le programme ; tout ce qui n'émet pas d'octet et n'est pas une constante utilisée est masqué, **la source principale restant intégrale**. Sur `example.asm`, le listing passe de 357 à 36 lignes.

C'est un **filtre de listing pur** : l'objet et toutes les autres sorties sont identiques avec ou sans `-K`. Les sorties de référence des tests de `xasm2026-4` étant produites sans `-K`, elles ne bougent pas.

### La commande canonique pour un programme destiné à la machine

```
xasm2026-4 NOM.ASM -ONOM.OBJ -L -S -B -K
```

C'est celle des sondes de `SC62015Disassembler/Samples/DEVICE9/` et du module `Samples/BASEXT/`.

> ⚠️ **Le nom de fichier VOYAGE DANS l'enveloppe uuencode — et deux noms distincts sont en jeu.**
>
> | Ce qui est nommé | D'après quoi | Exemple, `xasm2026-4 MATTEST.ASM -OMATTEST.OBJ -B` |
> |---|---|---|
> | la ligne `begin 644 NOM.EXT` **dans** le `.uu` | **`-O`** | `begin 644 MATTEST.OBJ` |
> | le `FNAME$` du décodeur BASIC embarqué | `-O`, cadré en 8.3 majuscules | `FNAME$="MATTEST .OBJ"` |
> | le **fichier** `.uu` lui-même | le **source**, extension en **minuscules** | `MATTEST.uu` |
>
> Un décodeur PC lit la ligne `begin` **pour choisir le fichier à écrire** : renommer le `.uu` ne change donc rien à ce qui sortira à l'autre bout. Sans `-O` explicite, cette ligne porte le nom du source en minuscules (`begin 644 TYDOS.obj`), alors que le Sharp et l'émulateur nomment en **majuscules 8.3**. D'où les deux gestes :
>
> 1. **reconstruire avec `-ONOM.EXT` en majuscules** — jamais renommer un `.uu` mal nommé ;
> 2. **renommer ensuite le fichier `.uu` en `.UU`**, et sous Windows **en deux temps** (`NOM.uu` → `NOM.tmp` → `NOM.UU`) : un changement de casse seul y est ignoré.
>
> Établi en septembre 2026 sur les douze artefacts de TY-DOS, puis sur les sondes de `DEVICE9` (commit « les cinq autres .uu portaient un nom en minuscules »).

## 5. Formats de fichier objet

### 5.1 Écriture (XASM) — option `-T`

| Type `-T` | Format |
|---|---|
| *(aucun)* | **En-tête XASM 16 octets** + code, format « machine language » standard pour PC-E500. |
| `B` | Binaire brut : taille (3 octets) + adresse de départ (3 octets) + corps. |
| `H` | Représentation hexadécimale ASCII du format `B`. |
| `Z` | Texte ZSH (historique PC-E500). |
| `F` | Format FTX. |

En-tête par défaut (exemple réel, programme VOGUE) :

```
FF 00 06 01 10 61 38 00 00 98 0B FF FF FF 00 0F
```

**Point de vigilance vérifié sur fichiers réels** (`register.obj`/`tmap.obj`/`vogue.obj`) : le champ *exec address* (décalage `0x0B`, 3 octets) vaut en pratique `FF FF FF` — une sentinelle « pas de point d'entrée distinct de l'adresse de chargement » plutôt qu'une adresse réelle — et le champ *reserved* (décalage `0x0E`) vaut `00 0F`. Un lecteur qui validerait strictement `exec ≤ 0xFFFFF` rejetterait donc les objets XASM réels ; il faut traiter `0xFFFFFF` comme cas spécial (`EntryPoint = load_addr`).

### 5.2 Lecture (désassembleur) — formats reconnus

Le désassembleur du projet (`SC62015Disassembler`) reconnaît en entrée un sur-ensemble de formats rencontrés dans l'écosystème PC-E500.

> ⚠️ **L'hex ASCII est un transport, pas un format de contenu.** Une version antérieure de ce tableau lui donnait un en-tête de 6 octets, comme si le texte décodé était toujours du `-TB` : `Samples/tred111/tred.hex` décode en un objet **IOCS** de 16 octets d'en-tête, identique à l'octet à `TRED.obj`. Le format est donc redétecté après décodage.

| Format | Option | Détection automatique | Taille en-tête | Adresse de base |
|---|---|---|---|---|
| IOCS (BLOAD) | `--format iocs` | Signature `FF 00 06 01 10` | 16 octets | Dans l'en-tête. |
| Binaire compact | `--format tb` | Cohérence taille (3o) + adresse (3o) | 6 octets | Dans l'en-tête. |
| Hex ASCII | `--format hex` | Contenu ASCII hexadécimal pur | celui du contenu **décodé**, redétecté | Dans l'en-tête. |
| Image ROM | `--format rom` | Signature `10 12 40 …` ou taille caractéristique | 32 octets (0 si image plate) | `--base` (défaut `C0000`). |
| Binaire brut | `--format raw` | Aucune | 0 | `--base` (défaut `0`). |

## 6. Modes d'adressage — rappel syntaxe XASM

| Mode | Exemple | |
|---|---|---|
| Immédiat | `MV A,34h` | |
| Interne directe | `MV A,(23h)` | |
| Interne relative `BP` | `MV A,(BP+10h)` | |
| Interne composée | `MV A,(BP+PX)` | |
| Externe directe | `MV A,[89AB3h]` | |
| Externe par registre | `MV A,[X]` | |
| Post-incrément / pré-décrément | `MV A,[X++]` / `MV A,[--X]` | |
| Externe registre+offset | `MV A,[X+23h]` | |
| Externe via pointeur interne | `MV A,[(7Bh)]` | |

## 7. Diagnostics courants

| Message | Cause | Action |
|---|---|---|
| `Prebyte error` | PRE incohérent avec `PRE`/`PRE_ON`. | Vérifier l'adressage RAM interne composé. |
| `Bad internal RAM addressing` | `(BP+n)`/`(BP+PX)` mal formé. | Vérifier parenthèses/offsets. |
| `Bad external MEMORY addressing` | Crochets/pointeur incorrects. | Vérifier `X`/`Y`/`U`/`S`, adresse 20 bits. |
| `Branch too far` | Saut relatif hors portée. | Utiliser une forme longue/far. |
| `Duplicate label` | Label déjà défini dans le bloc. | Renommer ou `LOCAL`/`ENDL`. |
| `EOF comes before END` | `END` manquant. | Ajouter `END`. |
| `Label format error` **sur une ligne saine** | ⛔ La ligne **précédente** dépasse 253 caractères et a été coupée en deux par le moteur C (voir §2). | Raccourcir la ligne d'**avant**, pas celle que le message désigne. |
| `Location counter wandered` | `ORG` répété/instable. | Préférer `DS` pour combler un espace. |

## 8. Voir aussi

- `06-ecosysteme-outils.md` — comment XASM Origine/xasm2026-1-2/xasm2026-4 et `SC62015Disassembler` se répondent (assembleur ↔ désassembleur, non-régression bit-exacte).
- `02-jeu-instructions.md` — mnémoniques reconnus par XASM (liste extraite de sa table de hachage), recoupée avec la table d'opcodes.
- `Docs/Doc technique/Documentation_XASM_PC-E500S.md` (dans `SC62015Disassembler`) — version complète, avec exemples de macros/structures/sections et le détail du workflow VS Code.
- `xasm2026-4/README.md` et `xasm2026-4/Documentation/Documentation_XASM2026-4_PC-E500S.md` — la référence de la génération maintenue, dont les options `-U`, `-K` et le dialecte A62.
- `12-extensions-basic.md` §14 — le format des programmes BASIC d'essai (`CRLF`, majuscules, noms 8.3), qui accompagnent un objet sur la machine.
