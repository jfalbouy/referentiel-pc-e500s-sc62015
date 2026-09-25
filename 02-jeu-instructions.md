# Jeu d'instructions du SC62015 (ESR-L)

*Rédigé le 2026-08-26 — mis à jour le 2026-09-14*

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier synthétise en français le jeu d'instructions ; **pour l'encodage exact octet par octet, les cycles et les tailles, la référence détaillée reste** `SC62015Disassembler/Docs/Doc technique/README - PC-E500 Instruction Table.md` (anglais, exhaustif, table par table) **et** `SC62015Disassembler/Data/OpcodeTable.json` (256 entrées machine-readable, utilisées par le désassembleur du projet). Ce référentiel évite de dupliquer ces 256 lignes octet-par-octet et se concentre sur une vue d'ensemble et un sommaire par mnémonique.

## 1. Notation

- `(x)` : mémoire **interne** (adresse fixe ou modifiée par un octet PRE, voir `01-architecture-cpu-sc62015.md` §3.3).
- `[x]` : mémoire **externe**.
- `n`, `m`, `l`, `k` : octets immédiats/d'adresse ; `mn` = valeur/adresse 16 bits, `lmn` = valeur/adresse 20 bits (24 bits stockés).
- `r₁`…`r₄` : classes de registres (voir `01-architecture-cpu-sc62015.md` §2.1) ; `r'₃` = un de `X,Y,U,S`.
- Flags : seuls `C` (retenue) et `Z` (zéro) existent sur ce CPU.

## 2. Table des opcodes (vue 16×16)

> ⚠️ **Sens de lecture — `opcode = colonne × 16 + ligne`.** La **ligne** donne le quartet **bas**,
> la **colonne** le quartet **haut**. Exemple : `MV X,[lmn]` est en ligne `C`, colonne `8` — c'est
> donc **`08Ch`**, et non `0C8h` (qui est `MV (m),(n)`). La transposition est facile et
> silencieuse : elle a déjà fait conclure à tort qu'une adresse n'était lue nulle part. Pour
> construire un motif d'octets, prendre l'encodage dans `SC62015Disassembler/Data/OpcodeTable.json` ou le relever sur
> une instruction réelle d'un `.lst`.

| L\H | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **0** | NOP | JP(n) | | PRE(BP+n) | ADD A,n | ADC A,n | CMP A,n | AND A,n | MV A,(n) | MV A,[r3] | MV (n),A | MV [r3],A | EX (m),(n) | MV (k),[lmn] | MV (m),[r3] | MV (m),[(n)] |
| **1** | RETI | JP r3 | PRE(BP+m) | PRE(BP+PY) | ADD (m),n | ADC (m),n | CMP (m),n | AND (m),n | MV IL,(n) | MV IL,[r3] | MV (n),IL | MV [r3],IL | EXW(m),(n) | MVW (k),[lmn] | MVW (m),[r3] | MVW (m),[(n)] |
| **2** | JP mn | JR +n | PRE(BP+m) | PRE(m) | ADD A,(n) | ADC A,(n) | CMP[klm],n | AND[klm],n | MV BA,(n) | MV BA,[r3] | MV (n),BA | MV [r3],BA | EXP(m),(n) | MVP (k),[lmn] | MVP (m),[r3] | MVP (m),[(n)] |
| **3** | JPF lmn | JR -n | PRE(BP+m) | PRE(PY+n) | ADD (n),A | ADC (n),A | CMP (n),A | AND (n),A | MV I,(n) | MV I,[r3] | MV (n),I | MV [r3],I | EXL(m),(n) | MVL (k),[lmn] | MVL (m),[r3++] | MVL (m),[(n)] |
| **4** | CALL mn | JPZ mn | PRE(BP+PX)(BP+n) | PRE(PX+m)(BP+n) | ADD r1,r2/r2,r2 | ADCL(m),(n) | TEST A,n | MV A,B | MV X,(n) | MV X,[r3] | MV (n),X | MV [r3],X | DADL(m),(n) | DSBL(m),(n) | ROR A | SHR A |
| **5** | CALLF lmn | JPNZ mn | PRE(BP+PX)(BP+PY) | PRE(PX+m)(BP+PY) | ADD r3,r | ADCL(m),A | TEST (m),n | MV B,A | MV Y,(n) | MV Y,[r3] | MV (n),Y | MV [r3],Y | DADL(n),A | DSBL(n),A | ROR (n) | SHR (m) |
| **6** | RET | JPC mn | PRE(BP+PX)(n) | PRE(PX+m)(n) | ADD r1,r1 | MVL(m),[X+n] | TEST[klm],n | AND (m),(n) | MV U,(n) | MV U,[r3] | MV (n),U | MV [r3],U | CMPW(m),(n) | CMPW(m),r2 | ROL A | SHL A |
| **7** | RETF | JPNC mn | PRE(BP+PX)(PY+n) | PRE(PX+m)(PY+n) | PMDF(m),n | PMDF(m),A | TEST (n),A | AND A,(n) | MV S,(n) | SC (n) | MV (n),S | CMP (m),(n) | CMPP(m),(n) | CMPP(m),r3 | ROL (n) | SHL (n) |
| **8** | MV A,n | JRZ +n | PUSHU A | POPU A | SUB A,n | SBC A,n | XOR A,n | OR A,n | MV A,[lmn] | MV A,[(n)] | MV [lmn],A | MV [(n)],A | MV (m),(n) | MV [lmn],(n) | MV [r3],(n) | MV [(n)],(n) |
| **9** | MV IL,n | JRZ -n | PUSHU IL | POPU IL | SUB (m),n | SBC (m),n | XOR (m),n | OR (m),n | MV IL,[lmn] | MV IL,[(n)] | MV [lmn],IL | MV [(n)],IL | MVW(m),(n) | MVW[lmn],(n) | MVW[r3],(n) | MVW[(n)],(n) |
| **A** | MV BA,mn | JRNZ +n | PUSHU BA | POPU BA | SUB A,(n) | SBC A,(n) | XOR[klm],n | OR[klm],n | MV BA,[lmn] | MV BA,[(n)] | MV [lmn],BA | MV [(n)],BA | MVP(m),(n) | MVP[lmn],(n) | MVP[r3],(n)¹ | MVP[(n)],(n) |
| **B** | MV I,mn | JRNZ -n | PUSHU I | POPU I | SUB (n),A | SBC (n),A | XOR (n),A | OR (n),A | MV I,[lmn] | MV I,[(n)] | MV [lmn],I | MV [(n)],I | MVL(m),(n) | MVL[lmn],(n) | MVL[r3++],(n) | MVL[(n)],(n) |
| **C** | MV X,lmn | JRC +n | PUSHU X | POPU X | SUB r2,r2 | SBCL(m),(n) | INC r | DEC r | MV X,[lmn] | MV X,[(n)] | MV [lmn],X | MV [(n)],X | MV (m),n | MVP(n),[lmn] | DSLL(n) | DSRL(n) |
| **D** | MV Y,lmn | JRC -n | PUSHU Y | POPU Y | SUB r3,r | SBCL(n),A | INC (n) | DEC (n) | MV Y,[lmn] | MV Y,[(n)] | MV [lmn],Y | MV [(n)],Y | MVW(l),mn | EX A,B | EX r2,r2/r3,r3 | MV r2,r2/r3,r3 |
| **E** | MV U,lmn | JRNC +n | PUSHU F | POPU F | SUB r1,r1 | MVL[X+n],(m) | XOR(m),(n) | OR(m),(n) | MV U,[lmn] | MV U,[(m)] | MV [lmn],U | MV [(n)],U | TCL | HALT | SWAP A | IR |
| **F** | MV S,lmn | JRNC -n | PUSHU IMR | POPU IMR | PUSHS F | POPS F | XOR A,(n) | OR A,(n) | MV S,[lmn] | RC | MV [lmn],S | | MVLD(m),(n) | OFF | WAIT | RESET |

*(Table de reconstruction reprise et traduite de `README - PC-E500 Instruction Table.md`, elle-même issue du manuel ESR-L. Croisée avec le projet indépendant [gikonekos/sc62015-opcode-reference](https://github.com/gikonekos/sc62015-opcode-reference), voir `07-sources-et-bibliographie.md`.)*

¹ Corrigé par rapport à la table source, qui indique par erreur `MVW[r3],(n)` à cette case (`0xEA`) — `Data/OpcodeTable.json` confirme `MVP` (mnémonique cohérent avec la colonne, opérandes 24 bits). Cette table est vérifiée depuis à trois niveaux par `SC62015Disassembler` (972 tests en septembre 2026) : réencodage de chaque instruction à l'octet près, reproduction de 25 listings XASM (18 764 instructions, 0 divergence), et réassemblage par XASM de 34 programmes **identiques à l'octet** ; le manuel ESR-L de Sharp la confirme case pour case (`Docs/Synthese/Jeu-d-instructions.md`).

## 3. Sommaire par mnémonique (65 mnémoniques, 256 opcodes)

| Mnémonique | Formes d'opérandes | Effet |
|---|---|---|
| `NOP` | — | Aucune opération. |
| `RETI` | — | Retour d'interruption : restaure `IMR`, `F`, `PC`, `PS` depuis la pile système. |
| `JP` | `(n)` / `mn` / `r3` | Saut absolu (page courante), via adresse en mémoire interne, immédiate ou registre `X/Y/U/S`. |
| `JPF` | `lmn` | Saut absolu 20 bits (change de page, `PS←l`). |
| `CALL` | `mn` | Appel de sous-routine (page courante), empile `PC` sur la pile système. |
| `CALLF` | `lmn` | Appel de sous-routine 20 bits, empile `PC`+`PS`. |
| `RET` / `RETF` | — | Retour de sous-routine (page courante / avec restauration de page). |
| `JR` | `+n` / `-n` | Saut relatif court (avant/arrière). |
| `JPZ`/`JPNZ`/`JPC`/`JPNC` | `mn` | Saut absolu conditionnel (Z=1/Z=0/C=1/C=0). |
| `JRZ`/`JRNZ`/`JRC`/`JRNC` | `±n` | Saut relatif conditionnel. |
| `MV` | (voir §4) | Transfert de données — la famille la plus riche (79 variantes), voir détail ci-dessous. |
| `MVW`/`MVP`/`MVL`/`MVLD` | idem `MV` | Transferts 16 bits / 24 bits / en boucle (compteur `I`) croissant / décroissant. |
| `EX`/`EXW`/`EXP`/`EXL` | mémoire interne, registres | Échange de données (au lieu de copie), tailles 8/16/24 bits ou en boucle. |
| `ADD`/`SUB` | immédiat, mémoire, registres | Addition/soustraction, affecte `C` et `Z`. |
| `ADC`/`SBC` | idem + retenue | Addition/soustraction avec retenue entrante. |
| `ADCL`/`SBCL` | mémoire, en boucle | Addition/soustraction avec retenue, propagée octet par octet sur `I` itérations. |
| `DADL`/`DSBL` | mémoire, en boucle | Addition/soustraction **décimale** (BCD) avec retenue, multi-octets. |
| `PMDF` | `(m),n` / `(n),A` | Ajoute `n` (ou `A`) à `(m)` **sans toucher aux drapeaux** (`C`, `Z` inchangés). ⚠️ Le relevé communautaire le dit « BCD packed » ; la ROM l'emploie **en binaire** pour déplacer `BP` d'un cadre — `pmdf (bp_ram),0F1h` retire 15, et c'est mesuré (`BP` 150 → 135, `12-extensions-basic.md` §10). |
| `CMP`/`CMPW`/`CMPP` | 8/16/24 bits | Comparaison (soustraction sans stockage du résultat), affecte `C`/`Z`. |
| `TEST` | 8 bits | ET logique sans stockage, affecte `Z`. |
| `AND`/`OR`/`XOR` | immédiat, mémoire, registre A | Opérations logiques bit à bit, affectent `Z`. |
| `SWAP` | `A` | Échange les deux nibbles de `A`. |
| `INC`/`DEC` | registre ou `(n)` | Incrémente/décrémente de 1, affecte `Z`. |
| `ROR`/`ROL` | `A` ou `(n)` | Rotation **circulaire** : huit rotations rendent l'octet intact. ⛔ Une version antérieure écrivait « à travers `C` », comme pour `SHR`/`SHL` — ce qui a fait rendre `0` à une extension pour toutes les valeurs. La ROM tranche, preuves en `12-extensions-basic.md` §11. |
| `SHR`/`SHL` | `A` ou `(n)` | Décalage **à travers `C`** — c'est ce qui les rend chaînables octet par octet pour un décalage multi-octets (la ROM enchaîne cinq `shr` en `0EEBB8H`). Ils ne consultent pas `I`. |
| `DSRL`/`DSLL` | `(n)`, en boucle | Décalage décimal (BCD) multi-octets, adresses croissantes/décroissantes. |
| `PUSHU`/`POPU` | `A,IL,BA,I,X,Y,F,IMR` | Empilement/dépilement sur la pile **utilisateur** `U`. |
| `PUSHS`/`POPS` | `F` | Empilement/dépilement des flags sur la pile **système** `S`. |
| `SC`/`RC` | — | Positionne/efface le flag `C`. |
| `TCL` | — | Efface le diviseur de temporisation (*timer clear*). |
| `HALT` | — | Arrêt CPU (horloge système stoppée), réveil sur interruption. |
| `OFF` | — | Extinction (horloges système et sous-horloge stoppées). |
| `WAIT` | — | Boucle d'attente, `I` itérations. |
| `IR` | — | Interruption logicielle. |
| `RESET` | — | Reset logiciel. |
| `PRE` | 15 combinaisons | Préfixe d'adressage interne composé (voir `01-architecture-cpu-sc62015.md` §3.3) — n'est pas une instruction exécutable en soi. |
| `DB` | — | Pseudo-entrée pour les deux opcodes non définis, **`20H` et `BFH`** (les deux cases vides de la table §2) ; illégal à l'exécution. Les 254 autres sont définis. |

## 4. Détail de la famille `MV`/`MVW`/`MVP`/`MVL`/`MVLD` (transferts de données)

C'est la famille la plus dense (à elle seule elle occupe la moitié de la table 16×16). Elle décline systématiquement les mêmes combinaisons source/destination pour 4 tailles d'opérande :

- **`MV`** — 8 bits (`r₁` : `A`, `IL`)
- **`MVW`** — 16 bits (`r₂` : `BA`, `I`)
- **`MVP`** — 24 bits (`r₃`/`r₄` : `X`, `Y`, `U`, `S`)
- **`MVL`** — en boucle, `I` itérations, adresses croissantes
- **`MVLD`** — en boucle, `I` itérations, adresses décroissantes

Combinaisons source/destination disponibles (present pour tout ou partie de ces 5 tailles selon la variante) :

1. Registre ↔ immédiat (`MV r,n` / `MV r,mn` / `MV r,lmn`)
2. Registre ↔ mémoire interne (`MV r,(n)` / `MV (n),r`)
3. Registre ↔ mémoire externe directe (`MV r,[lmn]` / `MV [lmn],r`)
4. Registre ↔ mémoire externe indirecte via `r'₃` : simple, post-incrément `++`, pré-décrément `--`, décalage `±n`
5. Registre ↔ mémoire externe via pointeur en mémoire interne : `[(n)]`, `[(m)±n]`
6. Mémoire interne ↔ mémoire interne (`MV (m),(n)`)
7. Mémoire interne ↔ mémoire externe directe (`MV (k),[lmn]` / `MV [lmn],(n)` etc.)
8. Mémoire interne ↔ mémoire externe indirecte via `r'₃`
9. Mémoire interne ↔ immédiat (`MV (m),n`)
10. Registre ↔ registre (`MV A,B` / `MV r2,r2'` / `MV r3,r3'` via post-byte)

`EX`/`EXW`/`EXP`/`EXL` reprennent le sous-ensemble « mémoire interne ↔ mémoire interne » et « registre ↔ registre » de cette même grille, en mode échange plutôt que copie.

Pour l'encodage précis (quel opcode, quels octets suivants, combien de cycles) de chacune des ~140 combinaisons `MV*`, se référer à la table détaillée de `README - PC-E500 Instruction Table.md` (section « Memory Transfer Instructions ») et à `OpcodeTable.json`.

## 5. Voir aussi

- `01-architecture-cpu-sc62015.md` — registres, modes d'adressage, PRE-bytes.
- `05-format-fichiers-et-xasm.md` — comment ces mnémoniques sont écrites en syntaxe XASM.
- `SC62015Disassembler/Data/OpcodeTable.json` — table exploitable par programme (256 entrées : `op`, `mn`, `opr`, `len`, `kind`, `flow`...).
- [gikonekos/sc62015-opcode-reference](https://github.com/gikonekos/sc62015-opcode-reference) — reconstruction indépendante, utile en contre-vérification.
