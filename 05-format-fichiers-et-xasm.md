# Formats de fichiers et assembleur XASM

> Voir `00-index.md` pour la vue d'ensemble. Synthèse de `SC62015Disassembler/Docs/Doc technique/Documentation_XASM_PC-E500S.md` (déjà très complet) et du `README.md`/`CLAUDE.md` de `SC62015Disassembler` pour le point de vue « lecture » des mêmes formats.

## 1. Généalogie de l'outil

XASM est un **assembleur croisé absolu** (pas d'édition de liens : les adresses sont fixées par `ORG`) pour le CPU ESR-L/SC62015, écrit à l'origine par **加古静司 / Seiji Kako** (auteur également du désassembleur ESR-L historique `DISASM12` et de l'outil `disbacon`/`bacon`, cf. `06-ecosysteme-outils.md` et `07-sources-et-bibliographie.md`), à partir d'une version Turbo Pascal, puis amélioré (hachage des symboles). Trois générations coexistent dans l'écosystème du projet :

| Génération | Dossier | Nature |
|---|---|---|
| **XASM 1.40** | `XASM Origine/` | Original historique en C (DOS), syntaxe et mnémoniques de référence. |
| **xasm2026-1 / xasm2026-1-2** | `xasm2026-1-2/` | Port maintenu en C11 moderne (GCC/CMake), sorties étendues (Intel HEX, S-Record, MAP, dépendances, UU, dump HxD), compatibilité **octet par octet** vérifiée contre l'original. |
| **xasm2026-4** | `xasm2026-4/` | Réécriture C# (.NET 8), non-régression **bit-exacte** contre `xasm2026-1`. |

## 2. Syntaxe source

```asm
LABEL:  INSTRUCTION  OPERANDE1,OPERANDE2   ; commentaire
        INSTRUCTION  OPERANDE
        END
```

- Label : lettres/chiffres/`_`, ne commence pas par un chiffre, 16 caractères max, insensible à la casse. Le `:` après un label est **obligatoire**, y compris devant `EQU`.
- `END` obligatoire en fin de fichier (source principal et chaque fichier inclus).
- Nombres : suffixe de base `B`/`O`/`D`/`H` (`0E000h`) ou préfixe `$` hexadécimal ; `_` séparateur visuel. Sans suffixe : décimal. Expressions calculées sur 20 bits.
- Chaînes : `DB 'texte'` développe caractère par caractère ; apostrophe doublée pour l'échapper (`'I don''t know'`).
- Opérateurs (priorité croissante) : `|` (OR) < `&` (AND) < `%` (modulo) < `+`/`-` < `*`/`/`, plus le signe unaire.
- `*` = compteur de position courant (utile pour `TAILLE: EQU *-DEBUT` ou `DS cible-*,0`).

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

| Option | Fichier | Contenu |
|---|---|---|
| `-O[f]` | `.obj` | Objet principal (voir §5 pour les formats). |
| `-L[f]` | `.lst` | Listing assemblé (adresses, octets, messages). |
| `-E` | — | Listing d'erreurs seul (sans partie objet). |
| `-S` | — | Ajoute la table des symboles au listing. |
| `-T[type]` | — | Choix du format objet historique : `Z` (ZSH texte), `F` (FTX), `B` (binaire brut), `H` (hex ASCII), absent = en-tête XASM 16 octets. |
| `-I[f]` | `.hex` | Intel HEX. |
| `-M[f]` | `.s19` | Motorola S-Record (S1/S9). |
| `-P[f]` | `.map` | Sections et symboles. |
| `-D[f]` | `.d` | Dépendances façon *make*. |
| `-B[f]` | `.uu` | Encodage BASIC auto-décodable pour transfert vers PC-E500S. |
| `-X[f]` | `.txt` | Dump hexadécimal façon HxD. |
| `-C`/`-W`/`-V`/`-R` | console | Compteur de lignes / warnings / colonne d'erreur / rapport de taille par `SECTION`. |

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

Le désassembleur du projet (`SC62015Disassembler`) reconnaît en entrée un sur-ensemble de formats rencontrés dans l'écosystème PC-E500 :

| Format | Option | Détection automatique | Taille en-tête | Adresse de base |
|---|---|---|---|---|
| IOCS (BLOAD) | `--format iocs` | Signature `FF 00 06 01 10` | 16 octets | Dans l'en-tête. |
| Binaire compact | `--format tb` | Cohérence taille (3o) + adresse (3o) | 6 octets | Dans l'en-tête. |
| Hex ASCII | `--format hex` | Contenu ASCII hexadécimal pur | 6 octets décodés | Dans l'en-tête. |
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
| `Location counter wandered` | `ORG` répété/instable. | Préférer `DS` pour combler un espace. |

## 8. Voir aussi

- `06-ecosysteme-outils.md` — comment XASM Origine/xasm2026-1-2/xasm2026-4 et `SC62015Disassembler` se répondent (assembleur ↔ désassembleur, non-régression bit-exacte).
- `02-jeu-instructions.md` — mnémoniques reconnus par XASM (liste extraite de sa table de hachage), recoupée avec la table d'opcodes.
- `Docs/Doc technique/Documentation_XASM_PC-E500S.md` (dans `SC62015Disassembler`) — version complète, avec exemples de macros/structures/sections et le détail du workflow VS Code.
