# Formats de fichiers et assembleur XASM

*Rédigé le 2026-08-26 — mis à jour le 2026-09-25*

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
| **xasm2026-4** | `xasm2026-4/` | Réécriture C# (.NET 8), non-régression **bit-exacte** contre `xasm2026-1`. Reproduit en plus le **préprocesseur de l'assembleur A62 (N. Kon)** — `rel` et la table de relocation — ce qui assemble telles quelles les sources des drivers `ssfdc120` et `PLINKC`. Documentation complète : `xasm2026-4/Documentation/Documentation_XASM2026-4_PC-E500S.md`. ⚠️ Trois défauts d'encodage silencieux y ont été corrigés en septembre 2026 (§7bis) : **un objet antérieur au 2026-09-17 qui emploie les formes concernées est à réassembler.** |

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

> ⛔ **`IFDEF`/`IFNDEF` ne voient que les symboles posés par `DEF`, jamais ceux d'un `EQU`.** Mesuré le 2026-09-16 en rendant `BASEXT.ASM` incluable par `BASEXT-DRV` : avec `basext_pilote: equ 1`, l'include de `pce500.inc` semblait masqué, mais l'`org 0BF000H` passait (`Code: 0BE000h - 0BFA94h`) ; avec `def basext_pilote`, tout est masqué. C'est la bonne façon de rendre une source **à la fois autonome et incluable** :
>
> ```asm
>         ifndef  basext_pilote       ; pose par « def basext_pilote » chez l'includeur
>         org     0BF000H
>         pre_on
>         endif
> ```

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

⚠️ **`-K` masque, dans un fichier inclus, TOUTE ligne qui n'émet pas d'octet** — commentaires et étiquettes seules compris. Sans conséquence pour `pce500.inc` ; mais quand l'include porte du **code** (une source entière incluse par une autre, comme `BASEXT.ASM` dans `BASEXT-DRV`), le listing perd tous ses commentaires et des étiquettes qu'on vient y relire (`kw_table:`) : 816 lignes sur 2429 mesurées. Assembler alors **sans `-K`**.

### La commande canonique pour un programme destiné à la machine

```
xasm2026-4 NOM.ASM -ONOM.OBJ -L -S -B -K
```

C'est celle des sondes de `SC62015Disassembler/Samples/DEVICE9/` et du module BASEXT (`C:\Claude\BASEXT\src\`, anciennement `Samples/BASEXT/`).

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

### 5.1bis L'enveloppe texte — **quatre** formes, pas une

> **Référent : `C:\Claude\UUENCODE-UUDECODE\FORMATS.md`** (comparaison structure par structure,
> sommes de contrôle comprises) — document d'un **dépôt privé**, qui conserve les binaires de 1993 et
> les sources japonaises d'époque. Ce paragraphe se lit donc **sans lui** : il en porte la clé de
> lecture et les mesures, et c'est le détail octet par octet qui reste dans le référent.

⛔ **Deux fichiers portant tous deux l'extension `.UUE` peuvent ne pas avoir la même structure**,
selon qu'ils viennent du PC ou de la machine. C'est la source d'erreur principale du transport
d'objets. Les quatre formes partagent le **même alphabet uuencode** ; ce qui les sépare est la
nature du fichier et le contrôle d'intégrité :

| Forme | Produite par | Signe distinctif | Nature |
|---|---|---|---|
| `.UUE` **du PC** | `UUENCODE.EXE` 5.25 (R. Marks, 1993), ou sa réécriture C17 | en-tête `section 1 of…`, deux lignes `sum -r/size` (somme tournante BSD 16 bits), `1Ah` final en MS-DOS | fichier texte d'**échange** |
| `.UUE` **du Sharp** | `UUENC3.ASM` sur la machine | **une somme de contrôle par ligne** (6 bits), fin `end` + `size n` | fichier texte d'échange |
| `.uu` | `xasm2026-4 -B` | le précédent, chaque ligne préfixée `NNNN '`, précédé d'une **amorce BASIC** de 39 lignes | **programme BASIC exécutable** qui se décode lui-même : format de **déploiement** |
| `.uux` | `uuencode -b` (PC) ou `UUENC3 -B` (Sharp) | le `.uu` **sans** son amorce : inerte | bloc destiné au `MERGE` |

✅ **Mesuré le 2026-09-23** (`C:\Claude\UUENCODE-UUDECODE`, 26 essais) : la réécriture C17 des deux
outils de 1993 encode **à l'octet près** comme les binaires MS-DOS d'époque ; le `.uux` produit sur
PC par `uuencode -b` est **identique octet pour octet** à celui produit par `UUENC3 -B` **sur un
PC-E500S réel** ; et le bloc de données d'un `.uu` de `xasm2026-4 -B` est ce même `.uux`. Le dump
ROM `rom83_x.uue` (262 144 octets) se décode conforme à sa somme `#META`.

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
| `Undefined instruction` sur `cmp a,(n)`, `test a,(n)`, `test (m),(n)` | Ces formes **n'existent pas** dans le jeu SC62015 (§7bis). | Écrire `cmp (n),a` en **inversant la condition** qui suit ; voir `xasm2026-4/Exemples/TUTORIEL/10_cmp_test_formes.asm`. |
| `rel` : « l'instruction ne porte aucune adresse absolue » | `rel` devant une instruction qui ne porte pas **exactement une** adresse `mn` ou `lmn` (`mv a,05H`, `jr`, `db`, `dw` à plusieurs valeurs). | Retirer `rel` : il n'y a rien à reloger. |

## 7bis. Trois défauts silencieux de `xasm2026-4`, corrigés en septembre 2026

Aucun ne produisait de message : l'objet sortait, faux. Tous trois ont été trouvés en confrontant `xasm2026-4` au **moteur C** (`xasm2026-1-2`) ou à une **mesure**, jamais par la relecture du listing — c'est la méthode à garder.

| Défaut | Ce qui sortait | Corrigé | Rapport |
|---|---|---|---|
| **`cmp a,(n)` accepté** (et `cmp a,(BP+n)`) | `62 nn` : `62h` est `CMP [lmn],n`, **5 octets** ; le CPU avalait les 3 octets suivants et l'exécution se désynchronisait (retour au MENU sous PockEmul). Même défaut sur `test a,(n)` (`6Bh` = `XOR (n),A`, qui **écrit** en mémoire) et `test (m),(n)` (`6Ah`, tronqué). Le moteur C les refuse | 2026-09-15 : refusées (`Undefined instruction`) | `xasm2026-4/RAPPORT-BUG-cmp-a-parentheses.md` |
| **Octet PRE omis, déplacé ou dédoublé** | Sous `pre_on` : `mvp [r3],(n)` **sans PRE** (`EA 05 0B` au lieu de `30 EA 05 0B` : l'écriture part en `(BP+0Bh)`), `mvw [r3],(n)` et `mvl (n),[lmn]`/`[lmn],(n)` avec le PRE **après** l'opcode, famille `F0h`-`FBh` avec deux PRE simples au lieu d'un combiné, `jp (n)` assemblé en saut direct, `mv s,[(n)]` accepté. Mesuré sur **2520 formes** : 224 divergences sous `pre_on`, 65 sous `pre_off`, neuf défauts | 2026-09-17 : **0 divergence** sur 2520 formes contre le moteur C | `xasm2026-4/RAPPORT-BUG-octet-pre.md` |
| **Préfixe `rel` : champ d'adresse mal situé** | La position du champ était **déduite de la fin** de l'instruction : fausse quand un octet suit l'adresse (`cmp/test/and/or/xor [lmn],n`, `mv`..`mvl [lmn],(n)`), pour `jpz/jpnz/jpc/jpnc` (3 octets en +0 au lieu de 2 en +1 : l'**opcode** aurait été relogé) et pour `dw` — 14 formes sur 29. `rel` devant une instruction sans adresse encodait un écart négatif en `0FFh`, le **terminateur** de la table. Mesuré sur BASEXT : 12 entrées décalées, **48 octets faux** après relocation | 2026-09-17 : le champ est **relevé à l'émission** ; sur les 144 sites de BASEXT, table identique à la mesure de `BASEXT-DRV/outils/reloc.py`, **0 octet faux** | `xasm2026-4/RAPPORT-BUG-rel-champ-adresse.md` |

Réponse de `xasm2026-4` et vérification : `xasm2026-4/REPONSE-RAPPORTS-BUG-octet-pre-et-rel.md` ; contre-vérification indépendante dans `C:\Claude\BASEXT-DRV\CONCEPTION.md` §3.4. Les objets de référence (`REGISTER`, `TMAP2020`, `VOGUE`, `PLINKC.OBJ`, BASEXT) sont **inchangés à l'octet** : aucun n'employait les formes fautives.

> **Le format de table de relocation Kon** (celui de `rel`, de `PLINKC` et de `BASEXT-DRV`) : un octet par champ, écart depuis le champ précédent (le premier depuis l'origine) ; bit `080h` = champ de 3 octets, absent = 2 octets ; `07Eh` = écart long sur 2 octets qui suivent ; `0FFh` = fin. **Une relocation doit préserver le quartet haut** d'un champ de 3 octets : c'est là que la table de répartition du BASIC porte son drapeau instruction/fonction (`12-extensions-basic.md` §3). La boucle de PLINKC le garantit en calculant **en RAM interne** (`mvp`/`sbcl`/`mvp`), sans passer par un registre de 20 bits.

## 7ter. Convertir une source d'un autre assembleur — l'exemple de History 1.11

`C:\Claude\HIS111\hist111.asm` (TORO, 1994, assembleur de *Katsuyō Kenkyū* 活用研究, Shift-JIS)
a été converti en `history.asm`, qui redonne `history.bin` **à l'octet**, sauf les trois octets de
la somme de contrôle, calculée et rangée par le premier `CALL` avant diffusion. Ce que la conversion
a demandé, et qui vaut pour toute source de cette famille :

| Original | `xasm2026-4` | Pourquoi |
|---|---|---|
| `ORG 0E0000H,0BE300H` (adresse logique, adresse physique) | `org 0BE300h` puis `phase 0E0000h` … `dephase` | les labels à l'adresse logique, les octets à leur place. 📖 **C'est la convention de toute cette famille de sources** : l'installateur `INSTd` 1.05 écrit de même `ORG 0E0000H,0BC340H`, et sa relocation *par analyse* repose précisément sur cette base `0E0000h` — d'où son interdiction d'écrire une valeur de 3 octets en `0Exxxxh` (`03` §7bis) |
| un `ORG` qui **revient en arrière** (TSR en `0BE300h`, puis installateur en `0BE000h`) | **remettre les segments dans l'ordre croissant**, combler par `ds cible-*,0` | ⛔ `xasm2026-4` **concatène** les segments dans l'ordre du source et prend l'adresse la plus basse pour l'en-tête : l'objet sort **faux, sans aucun message** (mesuré le 2026-09-24) |
| `PRE 30H MV Y,[(BASWRK)+0BH]` sur une ligne | `pre 30h` sur sa propre ligne, **sans** `pre_on` | les accès `(n)` sans PRE de l'original sont voulus : ce sont des `(BP+n)` |
| `HISIZE EQU 110H` | `hisize: equ 110h` | le `:` est obligatoire, même devant `EQU` (§2) |
| labels `@TIT`, `@@START`, `@` | acceptés tels quels | — |
| `DB 0FDH,35H ;MV I,Y` | `mv i,y` | mêmes octets (`FD 35`, et `FD 24` pour `mv ba,x`) |
| une **tabulation** dans une chaîne `DM` | des espaces | l'assembleur d'origine la convertissait : `history.bin` contient 8 espaces là où la source a une tabulation et 7 espaces |

Vérification : l'objet réassemblé est comparé à `history.bin`, puis **désassemblé** par
`e500dasm`, et l'export `asm` réassemblé redonne les deux parties à l'octet.

## 8. Voir aussi

- `06-ecosysteme-outils.md` — comment XASM Origine/xasm2026-1-2/xasm2026-4 et `SC62015Disassembler` se répondent (assembleur ↔ désassembleur, non-régression bit-exacte).
- `02-jeu-instructions.md` — mnémoniques reconnus par XASM (liste extraite de sa table de hachage), recoupée avec la table d'opcodes.
- `Docs/Doc technique/Documentation_XASM_PC-E500S.md` (dans `SC62015Disassembler`) — version complète, avec exemples de macros/structures/sections et le détail du workflow VS Code.
- `xasm2026-4/README.md` et `xasm2026-4/Documentation/Documentation_XASM2026-4_PC-E500S.md` — la référence de la génération maintenue, dont les options `-U`, `-K` et le dialecte A62.
- `12-extensions-basic.md` §14 — le format des programmes BASIC d'essai (`CRLF`, majuscules, noms 8.3), qui accompagnent un objet sur la machine.
