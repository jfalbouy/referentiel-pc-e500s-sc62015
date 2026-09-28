# Routines réutilisables — ce qu'on ne devrait plus jamais réécrire

*Rédigé le 2026-09-28 — mis à jour le 2026-09-28*

> Voir `00-index.md` pour la vue d'ensemble. Ce chapitre rassemble des routines prêtes à l'emploi
> pour écrire un nouveau programme : celles que la **ROM offre déjà** — et qu'il serait absurde de
> réécrire —, puis celles que le corpus a produites.

Trois origines, et le chapitre les distingue toujours :

| Marque | Ce que cela veut dire |
|---|---|
| ✅ **éprouvé** | la routine, ou l'appel, a tourné sur un PC-E500S et a rendu le résultat attendu |
| ⚙️ **assemblé** | le code assemble sans erreur et son encodage a été relu au désassembleur, mais **il n'a pas tourné** |
| 📖 **lu** | contrat relevé dans la ROM ou dans un manuel, jamais exécuté |

⚠️ Une routine ⚙️ n'est pas une routine fausse — c'est une routine **dont personne ne peut encore
dire qu'elle est juste**. Le §5 dit comment la faire passer en ✅ ; c'est une demi-heure de machine,
et le référentiel préfère l'attendre plutôt que de promettre.

---

## 1. Le contrat, et pourquoi il vient avant le code

Une routine de ce chapitre se décrit toujours par **quatre lignes** : ce qu'elle attend, ce qu'elle
rend, ce qu'elle **détruit**, et ce qu'elle exige de l'appelant. La troisième est celle qu'on
oublie, et c'est celle qui coûte une séance.

```asm
; ---------------------------------------------------------------------------
; nom -- ce qu'elle fait, en une ligne.
;   Entree : X = ..., (cl) = ...
;   Sortie : carry clair si ..., Y = ...
;   Detruit: A, IL, X, (000H)-(002H)
;   Exige  : pre_on, et la zone langage machine reservee
; ---------------------------------------------------------------------------
```

Deux exigences valent pour **tout** ce chapitre, et les rappeler ici évite de les répéter :

- **`include pce500.inc` et `pre_on` juste après l'`org`.** Sans octet PRE, le mode par défaut d'un
  opérande de RAM interne est `(BP+n)`, pas `(n)` : un programme dont `BP` n'est pas nul lit
  ailleurs qu'il ne croit (`01` §3.3, `05` §2).
- **`(n)` est la RAM interne, `[n]` la mémoire externe.** Les deux espaces ne se mélangent pas.

---

## 2. Ce que la ROM fait déjà — ✅ mesuré

**C'est la section la plus rentable du chapitre.** Ces points d'entrée sont ceux qu'emploie
`C:\Claude\BASEXT`, dont les quatorze mots-clés tournent sur machine : leurs contrats ne sont pas
déduits, ils sont éprouvés.

### 2.1 Nombres — la conversion que personne ne devrait réécrire

| Service | Adresse | Contrat |
|---|---|---|
| `chknum` | `0EFBF3h` | évalue l'expression numérique pointée par `X`, vérifie le type, pousse un cadre et laisse le nombre BASIC en `(bp+0)`. Carry armé = erreur |
| `dec2bin` | `0EFAD4h` | le nombre BASIC de `(bp+0)` → **entier 24 bits** en `(bp+1)`..`(bp+3)`. ⛔ **Refuse au-delà de 2²⁰** (erreur 33) : `X` étant un registre de 20 bits, `0EFAFBh` fait `add x,a` / `jrc` |
| `bin2dec` | `0EFB6Fh` | l'entier de `(bp+0)`..`(bp+3)` → nombre BASIC en `(bp+0)`. ⛔ **Tronque à 20 bits sans rien dire** : `&345678` revenait `&045678`. Tester le quartet haut soi-même et refuser |
| `eval` | `0EF26Eh` | évalue une **expression complète** (pas seulement un terme) → `(bp+0)` |
| `alloc` | `0EF0DDh` | réserve `BA` octets sur la pile `U` (`U -= BA`) ; carry = erreur 54 |

✅ Les cinq sont mesurés : `LPEEK`, `WPEEK`, `LPOKE` et `MOD` les enchaînent, et les deux pièges
notés `⛔` ont été payés puis corrigés (`12` §5 et §8).

> **La leçon de méthode** : avant d'écrire une conversion décimale, regarder si la ROM ne la fait
> pas déjà. Ici elle la fait, en deux appels, avec le domaine complet du BASIC — et l'écrire
> soi-même n'aurait donné qu'un sous-ensemble de 20 bits.

### 2.2 Arithmétique complète — le device 9

Pour dépasser les 20 bits, ou travailler sur les **nombres BASIC** eux-mêmes (dix chiffres,
décimales comprises), la bibliothèque mathématique est accessible par l'IOCS : `add` `047h`,
`sub` `048h`, `mul` `049h`, `div` `04Ah`, `int` `056h`, et une trentaine d'autres. Contrat,
sens des opérandes (« `Y op X` → `X` ») et pièges : **`04` §2.5** et **`12` §13**.

⛔ **Qui appelle `div` teste son diviseur lui-même** : une division par zéro déplace `BP` de +30,
en silence, et cela s'accumule.

### 2.3 Écrire, lire, chercher — FCS et IOCS

| Besoin | Appel | Où |
|---|---|---|
| écrire un bloc / un octet sur un handle | FCS `04h` / `06h`, `callf fcs_call` | `04` §1 |
| trouver un bloc par son nom dans un lecteur | IOCS device 6, `41h` `search_phys` | `03` §7bis |
| compacter, créer un bloc, le dimensionner | IOCS device 6, `47h` / `48h` / `42h` | `03` §7bis |
| écran : effacer, défiler, lire/écrire un motif | IOCS device 0, `51h`, `47h`/`48h`, `55h`/`56h` | `04` §2.4 |

### 2.4 Les routines internes de la ROM — ⛔ huit sur quatre-vingt-une sont appelables

Le désassembleur a produit un catalogue de **81 routines de `rom83`** avec leur contrat, leur
taille et leur nombre d'appelants (`SC62015Disassembler/Docs/Routines-ROM-PC-E500S.asm`). La
tentation est grande d'y puiser. **Elle se heurte à une règle qui ne se négocie pas :**

> Une routine de la ROM n'est appelable depuis la zone langage machine que si elle finit par
> **`RETF`**. Celles qui finissent par `ret` rendent la main **dans la page où elles s'exécutent**,
> c'est-à-dire en pleine ROM : `callf` empile trois octets, leur `ret` n'en dépile que deux, et le
> programme part où personne ne l'attend.

Le tri est donc mécanique, et il est sévère : **71 des 81 finissent par `ret` seul**. Il en reste
huit, plus les cinq services du §2.1 et les quatre portes officielles (`0FFFD8h`, `0FFFDCh`,
`0FFFE4h`, `0FFFE8h`). Les voici, contrats relevés dans le désassemblage :

| Routine | Adresse | Entrée → Sortie | Détruit |
|---|---|---|---|
| **`PUTBLKF`** | `0F227Fh` | `X` = texte, `Y` = **nombre d'octets** → affiché à l'écran ; `cy` = erreur | `A`, `X`, `Y`, `IL`, `(cl)` |
| **`CRLFF`** | `0FBEBFh` | rien → `CR`+`LF` sur le device **courant du BASIC** | `A`, `BA`, `X`, `Y`, `I` |
| **`DIRNAME`** | `0E0D53h` | `X` = entrée de répertoire (8+3), `Y` = destination (12 o) → `NOM.EXT` écrit | `A`, `BA`, `X`, `Y`, `IL` |
| **`NAMEMTCH`** | `0E0CCBh` | `X` = motif (8+3), `Y` = nom → `cy` = 0 si correspondance | `A`, `IL`, `(000h)` |
| **`NEXTBLK`** | `0F0E9Bh` | `X` = bloc de départ, `Y` = zone de 3 o → `X` sur le premier octet qui n'est pas `0FBh` | `A`, `X`, `Y`, `IL` |
| **`PUTBLK`** | `0EA480h` | `X` = source, **`IL`** = nombre d'octets → émis sur le **driver courant** | `A`, `X`, `IL` |
| `CALHEX` / `CALDECI` | `0EEAFEh` / `0EEAEEh` | les tokens `HEX` et `DECI` : une **bascule d'affichage**, ⚠️ pas une conversion | — |

**La plus utile des huit tient en quatre instructions** — c'est la réponse la plus courte à
« afficher un texte de longueur connue » :

```asm
0F227Fh  mv    (cl),000H       ; handle 0 = l'ecran
         mv    il,004H         ; fonction FCS 004h : write_block
         callf fcs_call
         retf
```

⚠️ **Trois pièges, tous relevés dans le désassemblage :**

- `PUTBLKF` : **`IL` n'est pas le compte**, c'est le numéro de fonction FCS ; la longueur est dans
  `Y`. Les confondre émet un nombre d'octets arbitraire. (`PUTBLK`, lui, prend bien le compte dans
  `IL` — les deux se ressemblent et ne s'emploient pas pareil.)
- `CRLFF` : le device vient de `[(baswrk)+06Ch]`, **celui que le BASIC a sélectionné**, pas d'un
  paramètre.
- `NAMEMTCH` : elle écrit dans `(000h)` et manipule `BP` par `PMDF` — à n'employer qu'avec la zone
  de travail du système en place. Son joker est le **point d'interrogation** (`03Fh`), pour un
  caractère ; **il n'y a pas d'étoile**.

⛔ **Et la réserve qui vaut pour toute cette section** : ces adresses sont celles de **`rom83`
(PC-E500S 8.3)**. Rien ne garantit qu'elles soient les mêmes sur une autre révision — c'est même
l'inverse qui est établi : **sept des dix entrées de drivers sur dix diffèrent entre `rom53` et
`rom83`** (`03` §7), et l'éditeur du BASIC range sa touche en `(BP+2Ah)` sur les ROM 5.x-7.x contre
`(BP+2Bh)` sur la 8.3 (`03` §3bis). Un programme qui appelle la ROM **lit d'abord sa version** en
`0FFFF0h`/`0FFFF1h` et refuse ce qu'il ne connaît pas — c'est ce que fait `HISTDRV`.

**Les 71 autres ne sont pas perdues pour autant** : elles restent précieuses à la *lecture* — pour
comprendre ce que fait la ROM, retrouver un algorithme, ou nommer une adresse dans un
désassemblage. Elles ne sont simplement pas *appelables*.

---

## 3. Les routines du référentiel

Le code qui suit est dans `Documentation/routines-2026.asm`, **assemblé par `xasm2026-4`** et
relu au désassembleur. Les numéros d'encodage cités sont ceux que produit l'assembleur.

### 3.1 Écrire une chaîne — ⚙️ assemblé

```asm
; wr_str -- ecrit une chaine terminee par 0 sur un handle ouvert.
;   Entree : X = adresse de la chaine, (cl) = handle (0 = ecran)
;   Sortie : carry clair ; X sur l'octet 0 terminal
;   Detruit: A, IL, X
wr_str:
        mv      a,[x]
        cmp     a,000H
        jrz     ws_fin
        inc     x
        mv      il,006H                 ; FCS : ecrire un octet
        callf   fcs_call
        jrc     ws_fin                  ; erreur FCS : rendre la main telle quelle
        jr      wr_str
ws_fin:
        rc
        ret
```

⚠️ Octet par octet, donc lent. Pour une longueur connue, **un seul** `fcs_write_block` (`04h`,
`X` = tampon, `Y` = taille) vaut mieux — c'est ce que fait le gabarit du §3 de la skill.

### 3.2 Hexadécimal — ⚙️ assemblé, idiome ✅ éprouvé

`hex_20` reprend l'idiome de `bd_hexa` (`BASEXT-DRV`), qui, lui, **tourne sur machine** : c'est
ainsi que l'installateur affiche `Hooks: CALL &xxxxx`.

```asm
; hex_byte -- l'octet A en deux chiffres hexadecimaux ASCII, ecrits en [Y++].
;   Detruit: A, IL
hex_byte:
        mv      il,a                    ; FD 10 -- garder l'octet
        shr     a                       ; F4 x4 : le quartet haut
        shr     a
        shr     a
        shr     a
        call    hb_chiffre
        mv      a,il
        and     a,00FH
        call    hb_chiffre
        ret
hb_chiffre:
        cmp     a,00AH
        jrc     hb_09
        add     a,007H                  ; 'A' - '9' - 1
hb_09:
        add     a,030H
        mv      [y++],a
        ret

; hex_20 -- X (20 bits) en cinq chiffres hexadecimaux ASCII en [Y].
;   Detruit: A, IL, (000H)-(002H)
hex_20:
        mv      (000H),x                ; 30 A4 00 -- les 3 octets de X
        mv      a,(002H)
        and     a,00FH
        call    hb_chiffre              ; le quartet de poids fort
        mv      a,(001H)
        call    hex_byte
        mv      a,(000H)
        call    hex_byte
        ret
```

### 3.3 Décimal sans la ROM — ⚙️ assemblé, **non éprouvé**

Quand `bin2dec` ne convient pas — par exemple dans un **pilote**, où l'on ne veut pas dépendre du
cadre `BP` de l'interpréteur —, voici la conversion par soustractions répétées. L'algorithme vient
du recueil de J.-F. Albouy (2022, `Documentation/routines asm.asm`) ; il est réécrit ici en syntaxe
`xasm2026`, sans dépendance à l'écran, et avec la suppression des zéros de tête.

```asm
; dec_u24 -- X (entier non signe 24 bits) en decimal ASCII, zeros de tete otes.
;   Entree : X = la valeur, Y = tampon (8 octets suffisent)
;   Sortie : les chiffres en [Y], Y sur l'octet suivant le dernier
;   Detruit: A, IL, X, Y, (000H)-(006H)
dec_u24:
        mv      (000H),x                ; la valeur, en RAM interne
        mv      x,tb_dix                ; X = pointeur sur la table
        mv      (006H),000H             ; aucun chiffre significatif encore ecrit
        mv      (005H),008H             ; 8 rangs : 10^7 .. 10^0
dc_rang:
        mvp     (003H),[x++]            ; la puissance de 10 courante
        mv      a,0FFH                  ; le chiffre, incremente avant chaque essai
dc_sous:
        inc     a
        rc                              ; pas d'emprunt entrant
        mv      il,003H
        sbcl    (000H),(003H)           ; valeur -= puissance, sur 3 octets
        jrnc    dc_sous
        rc
        mv      il,003H
        adcl    (000H),(003H)           ; une soustraction de trop : la rendre
        cmp     a,000H
        jrnz    dc_ecrit
        cmp     (006H),000H
        jrz     dc_suite                ; zero de tete : supprime
dc_ecrit:
        mv      (006H),001H
        add     a,030H
        mv      [y++],a
dc_suite:
        dec     (005H)
        jrnz    dc_rang
        cmp     (006H),000H
        jrnz    dc_fin
        mv      a,030H                  ; la valeur etait nulle : un seul '0'
        mv      [y++],a
dc_fin:
        ret

tb_dix: dp      10000000,1000000,100000,10000,1000,100,10,1
```

⚠️ **`sbcl` et `adcl` consomment `IL` comme compteur d'octets** : il faut le **recharger avant
chaque appel**, et c'est ce que fait la boucle. Le `rc` qui les précède efface l'emprunt entrant.

### 3.4 Texte et caractères — ⚙️ assemblé

```asm
; skip_spc -- avance X tant qu'il pointe une espace. Sortie : A = l'octet trouve.
skip_spc:
        mv      a,[x]
        cmp     a,020H
        jrnz    sk_fin
        inc     x
        jr      skip_spc
sk_fin: ret

; is_digit -- carry CLAIR si '0' <= A <= '9'. A est preserve.
is_digit:
        cmp     a,030H
        jrc     id_non
        cmp     a,03AH
        jrnc    id_non
        rc
        ret
id_non: sc
        ret

; upcase -- A en majuscule si c'est une minuscule ASCII.
upcase:
        cmp     a,061H
        jrc     up_fin
        cmp     a,07BH
        jrnc    up_fin
        sub     a,020H
up_fin: ret

; str_len -- Y = longueur de la chaine [X] terminee par 0 ; X inchange.
str_len:
        pushu   x
        mv      y,000000H
sl_boucle:
        mv      a,[x++]
        cmp     a,000H
        jrz     sl_fin
        inc     y
        jr      sl_boucle
sl_fin: popu    x
        ret

; str_cpy -- copie [X] vers [Y], le 0 terminal compris.
str_cpy:
        mv      a,[x++]
        mv      [y++],a
        cmp     a,000H
        jrnz    str_cpy
        ret
```

---

## 4. Trois contrats qui ne sont pas des routines, et qu'on paie si on les ignore

1. ⛔ **Un `CALL` depuis le BASIC rend la main *carry clair* (`rc`) et rééquilibre la pile `U`.**
   Carry armé au retour ⇒ « Syntax error » côté BASIC, sans autre explication. Si le `CALL` porte
   un argument entre guillemets : `popu x` à l'entrée, `pushu x` (pointeur avancé au-delà de
   l'argument) à la sortie, et reconnaître les **quatre** terminateurs `0`, `0Dh`, `1Ah`, `0FFh`.
   ✅ Mesuré (`12` §14).
2. ⛔ **Tout pointeur absolu interne à un pilote doit être relogé à l'installation.** Un pointeur
   oublié s'installe puis plante. ✅ `BASEXT-DRV/outils/reloc.py` **mesure** les champs par double
   assemblage plutôt que de les déclarer à la main (`03` §7bis).
3. ⛔ **`HALT` et `OFF` ne terminent pas le flot ; `RESET` oui.** Une analyse de flot qui s'arrête
   sur `HALT` perd tout le code qui suit.

---

## 5. Comment une routine passe de ⚙️ à ✅

Le chapitre est fait pour grandir, et la règle d'entrée est simple : **on n'y promeut rien sans
mesure**. La marche à suivre, pour n'importe laquelle des routines du §3 :

1. l'assembler dans un module d'essai en `0BF000h`, avec un `start:` qui l'appelle sur des valeurs
   connues et écrit le résultat en `0BFBF0h` (hors du code, dans la zone réservée) ;
2. **signer le résultat** — un identifiant écrit dès la première instruction : la zone langage
   machine n'est pas remise à zéro, et sans signature on lit de la mémoire quelconque en croyant
   lire un résultat (`03` §7bis, trois mesures perdues ainsi) ;
3. relire depuis un programme BASIC en `PEEK`, et **écrire le relevé dans un fichier** — sous
   PockEmul, `F:` se lit directement depuis le PC ;
4. reporter ici le résultat, avec sa date et ce qu'il établit.

**Valeurs d'essai pour `dec_u24`** : `0` (doit donner `0`, et non une chaîne vide), `9`, `10`,
`100`, `1000000`, `10000000`, et `16777215` = `0FFFFFFh`, le maximum sur 24 bits.

⛔ **J'avais écrit ici que ce dernier « débordait la table à huit rangs » et que c'était le cas le
plus douteux. C'est faux, et la vérification prend une minute** : la table monte à 10⁷, donc
jusqu'à 99 999 999, quand 24 bits s'arrêtent à 16 777 215 — huit rangs sont exactement ce qu'il
faut. L'algorithme a été simulé sur ces neuf valeurs : toutes justes, zéro compris.

Ce qui **reste** à éprouver n'est donc pas l'arithmétique mais **la sémantique des instructions** :
que `sbcl` arme bien la retenue sur emprunt (c'est ce que `jrnc` suppose), qu'il consomme `IL`
comme compteur d'octets, et que le `rc` préalable soit nécessaire. Trois points que seule la
machine tranche — et c'est exactement pour cela que la routine reste ⚙️.

---

## 6. Voir aussi

- `04-fcs-iocs.md` — le catalogue complet des appels FCS et IOCS, et le device 9.
- `12-extensions-basic.md` — les services de la ROM en contexte : c'est là qu'ils ont été mesurés.
- `05-format-fichiers-et-xasm.md` — la syntaxe, les directives et les pièges de l'assembleur.
- `Documentation/routines-2026.asm` — le fichier qui assemble tout ce chapitre.
- `Documentation/routines asm.asm` — le recueil de J.-F. Albouy (2022), source de plusieurs
  algorithmes repris ici : affichage décimal et hexadécimal, classification de caractères, appels
  FCS, pilotage de l'écran. Il est écrit en syntaxe A62 (`$` hexadécimal, `local`/`endl`) et avec
  les anciens noms d'étiquettes ; les routines reprises ici ont été portées aux conventions du
  corpus.
