# Routines réutilisables — ce qu'on ne devrait plus jamais réécrire

*Rédigé le 2026-09-28 — mis à jour le 2026-09-29*

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

Le tri est donc mécanique, et il est sévère : **71 des 81 routines du catalogue finissent par
`ret` seul**. Il en reste huit, dont voici les contrats, relevés dans le désassemblage :

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

### 2.5 Le balayage complet — 280 points d'entrée, et la méthode pour les trouver

Les huit du §2.4 sont les seules **du catalogue documenté**. La ROM en offre bien davantage, et il
n'est pas nécessaire de les lire une par une pour les trouver : **une routine que la ROM
elle-même appelle par `CALLF` rend forcément la main par `RETF`**. Le critère se renverse donc en
une recherche mécanique.

Balayage de `rom83` (2026-09-28, `e500dasm` en mode flot, 76 329 instructions décodées) :

| Mesure | Valeur |
|---|---|
| sites `CALLF` dans la ROM | **744** |
| cibles distinctes | **295** |
| dont les portes officielles (`0FFFD8h`, `0FFFDCh`, `0FFFE4h`, `0FFFE8h`) | 4 |
| cibles atteignant un `RETF` avant tout `ret` | **280** |

**280 points d'entrée appelables depuis un programme utilisateur** — la liste complète, avec le
nombre d'appelants de chacun, est dans `Documentation/rom-callf.txt`. Les plus sollicités sont
déjà connus de ce référentiel, ce qui valide la méthode :

| Adresse | Appels | Ce que nous en savons |
|---|---|---|
| `0EF0DDh` | 23 | `alloc` — réserve `BA` octets sur `U` (§2.1) |
| `0EF26Eh` | 21 | `eval` — évalue une expression complète (§2.1) |
| `0EFB6Fh` | 7 | `bin2dec` (§2.1) |
| `0EFBF3h` | 5 | `chknum` (§2.1) |
| `0F0063h` | 9 | le traitement IOCS `41h` `search_phys` du device 6 (`03` §7bis) |
| `0F00C3h` | 5 | IOCS `42h` `block_resize` — celui qui dimensionne un bloc |
| `0F01C1h` | 5 | IOCS `43h` `block_transfer` — le déplacement de mémoire |
| `0F0E9Bh` | 6 | `NEXTBLK` (§2.4) |
| `0FB334h` | 8 | ⚠️ calcule l'**adresse d'une variable simple** du BASIC — `(datbas)` + `[+32h]` + 5 + (lettre − `41h`) × 12 (`12` §17) |
| `0FE8B7h` | 10 | le traitement d'erreur vers lequel `PUTBLK` saute (`jpf`) |

⚠️ **Ce que le balayage ne dit pas** : ce que fait chacune des 280, ni ce qu'elle attend. Il donne
des **points d'entrée légitimes**, pas des contrats — et un point d'entrée sans contrat n'est pas
utilisable. Les identifier reste un travail de lecture, routine par routine ; les 81 du catalogue
en sont le début, et le nombre d'appelants indique par où commencer.

⛔ Et la réserve du §2.4 vaut entière : ces 280 adresses sont celles de **`rom83`**. Le même
balayage sur `rom53` ou `rom75` donnerait d'autres adresses — c'est d'ailleurs une mesure à faire,
car recouper les trois listes dirait lesquelles de ces routines sont **stables d'une révision à
l'autre**, donc sur lesquelles un programme portable peut s'appuyer.

**Les 71 autres ne sont pas perdues pour autant** : elles restent précieuses à la *lecture* — pour
comprendre ce que fait la ROM, retrouver un algorithme, ou nommer une adresse dans un
désassemblage. Elles ne sont simplement pas *appelables*.

### 2.6 ✅ Lesquelles survivent d'une ROM à l'autre — la mesure qui rend un programme portable

Le même balayage a été fait sur les **trois images du corpus** : `rom53` (5.3, série ancienne),
`rom75` (7.5, PC-E500/PC-E550) et `rom83` (8.3, PC-E500S). Le résultat commence par une régularité
frappante :

| ROM | Instructions décodées | Sites `CALLF` | Points d'entrée `RETF` |
|---|---|---|---|
| `rom53` | 76 076 | 740 | **280** |
| `rom75` | 76 117 | 743 | **280** |
| `rom83` | 76 329 | 744 | **280** |

**Exactement 280 dans chacune** — les trois ROM sont la même architecture, réarrangée. Car les
adresses, elles, ne suivent pas : **28 seulement sont communes aux trois**, et une adresse commune
ne suffit pas à faire une routine commune. En comparant les **octets** à chacune de ces adresses :

- ✅ **24 portent exactement le même code** — même début, mêmes appelants (`10/10/10`, `4/4/4`…) ;
- ⛔ **4 sont de pures coïncidences d'adresse** : `0DF9A7h`, `0DF9C4h`, `0DF9D7h`, `0E0043h`. Le
  code y diffère. Les appeler parce qu'« elles sont dans les trois ROM » serait exactement l'erreur
  que cette vérification évite.

**Et les 24 stables se répartissent en deux familles, ce qui est tout l'enseignement :**

| Plage | Ce qui s'y trouve |
|---|---|
| `0DF8B8h` – `0E230Fh` | le **système de fichiers** et les services de device — dont `NAMEMTCH` (`0E0CCBh`) et `DIRNAME` (`0E0D53h`) |
| `0F0063h` – `0F0A19h` | les **traitements du device 6** : `41h` `search_phys`, `42h` `block_resize`, `43h` `block_transfer`, `44h` `block_rename`, `45h` `block_create`, `46h` `block_delete`, `47h` `condense` |

⛔ **Aucun service du BASIC n'est dans la liste.** `alloc`, `eval`, `chknum`, `dec2bin`, `bin2dec` —
les cinq du §2.1, les plus utiles à une extension — **n'existent qu'à leur adresse de `rom83`**. Ils
ont bougé d'une révision à l'autre.

> **La règle qui en découle, et elle est nette :**
> **la couche système est stable, la couche BASIC ne l'est pas.**
> Un programme qui n'appelle que le device 6 et le système de fichiers tourne sur les trois
> machines sans rien vérifier. Dès qu'il touche à l'interpréteur, il doit **lire la version de la
> ROM** en `0FFFF0h`/`0FFFF1h` et refuser ce qu'il ne connaît pas — c'est ce que fait `HISTDRV`,
> et c'est aussi pourquoi l'éditeur du BASIC range sa touche en `(BP+2Ah)` sur les ROM 5.x-7.x
> contre `(BP+2Bh)` sur la 8.3 (`03` §3bis).

Les deux listes complètes sont dans `Documentation/` : `rom-callf.txt` (les 280 de `rom83`, avec
leur nombre d'appelants) et `rom-callf-stables.txt` (les 24, avec leur identification).

⚠️ **Ce que cette mesure ne dit pas** : une routine peut très bien exister dans les trois ROM **à
des adresses différentes**. Les 24 sont celles qui n'ont pas bougé, pas toutes celles qui existent
partout. Les retrouver par leur **empreinte de code** plutôt que par leur adresse donnerait une
liste plus large — et une table de correspondance par version, qui est ce qu'il faudrait pour
écrire un programme portable qui appelle l'interpréteur.

### 2.7 ✅ La table de correspondance entre les trois ROM — retrouvées par leur code

Le §2.6 disait que 24 routines n'avaient pas bougé. **Il ne disait pas où étaient passées les
autres.** La question se règle en cherchant les routines par leur **code** plutôt que par leur
adresse.

**La méthode**, et ses deux difficultés :

1. **Masquer les opérandes d'adresse.** Une routine déplacée garde ses instructions mais voit
   changer tous ses `callf`, `jpf`, `call`, `jp` — et aussi `jpz`, `jpnz`, `jpc`, `jpnc`, que
   j'avais oubliés au premier essai : le rendement est passé de **105 à 205** routines retrouvées
   quand je les ai masqués à leur tour. Une empreinte de 24 octets suffit.
2. **Suivre les trampolines.** Les services du BASIC ne sont pas des routines mais des **thunks de
   quatre octets** — `call <cible>` puis `retf`. Leur empreinte ne compare que deux octets, ce qui
   ne cherche rien. Il faut donc chercher la **cible**, puis, dans la ROM visée, retrouver le
   trampoline qui l'appelle.

**Résultat** : sur les 280 points d'entrée de `rom83`, **205 sont retrouvés dans `rom75` et 207
dans `rom53`**. ✅ Contrôle : **chacune des adresses trouvées tombe sur une frontière
d'instruction** dans le désassemblage de sa propre ROM — zéro faux positif sur 412 correspondances.

**Et voici ce que le §2.6 ne pouvait pas donner : les services du BASIC, sur les trois machines.**

| Service | `rom83` (8.3) | `rom75` (7.5) | `rom53` (5.3) |
|---|---|---|---|
| `chknum` | `0EFBF3h` | `0EFECDh` | `0EFEAFh` |
| `dec2bin` | `0EFAD4h` | `0EFDAEh` | `0EFD90h` |
| `bin2dec` | `0EFB6Fh` | `0EEF99h` | `0EEF7Eh` |
| `eval` | `0EF26Eh` | `0EF548h` | `0EF52Dh` |
| `alloc` | `0EF0DDh` | `0EF3B7h` | `0EF39Ch` |
| adresse d'une variable simple | `0FB334h` | `0FBA02h` | `0FB981h` |
| `PUTBLKF` | `0F227Fh` | `0F29E1h` | `0F2991h` |
| `CRLFF` | `0FBEBFh` | `0FC589h` | `0FC508h` |

Les cinq premières ont été **vérifiées une à une** dans le désassemblage de leur ROM : à chacune de
ces adresses on lit bien `call <cible>` suivi de `retf`.

> **Ce que cela ouvre** : une extension du BASIC **portable sur les trois machines**. Jusqu'ici la
> seule conduite sûre était de lire la version en `0FFFF0h` et de **refuser** ce qu'on ne
> connaissait pas (`HISTDRV`). Avec cette table, on peut lire la version et **choisir le jeu
> d'adresses** — trois `equ` conditionnels, et `BASEXT` tournerait sur un PC-E500 comme sur un
> PC-E500S.
>
> ✅ **Éprouvée le 2026-09-29**, et par une voie meilleure que prévu : `eval` et `dec2bin` ont été
> appelés sur un PC-E500 porteur d'une ROM **7.3** — une révision dont nous n'avons aucun dump —
> avec des adresses **trouvées par empreinte de code à l'exécution**. Voir la fin de cette section.
> La table reste utile comme **trace**, elle n'est plus le moyen.

⚠️ **Les 67 à 75 routines non retrouvées** ne sont pas forcément absentes : une empreinte trop
courte (un trampoline dont la cible bouge aussi), une routine réellement réécrite, ou un décalage
d'instruction suffisent à la manquer. La table dit ce qu'elle a trouvé, pas ce qui existe.

Table complète des 280 lignes : `Documentation/rom-correspondance-3roms.txt`.

#### La sonde qui met la table à l'épreuve — `Documentation/T2BIN.ASM` — ✅ éprouvée

Elle **lit la version** en `0FFFF0h`, **réécrit l'opérande de ses deux `callf`**
(l'idiome de `MEMCHECK`, déjà employé par le filtre de `XCONSOLE`), puis évalue l'argument du
`CALL` et le convertit :

```asm
        mv      a,[rom_majeure]
        cmp     a,008H
        jrz     v83
        ...
v75:    mv      x,chknum75              ; 0EFECDh
        mv      y,dec2bin75             ; 0EFDAEh
pose:
        mv      [!ap_chknum+1],x        ; l'operande du callf, reecrit
        mv      [!ap_dec2bin+1],y
        popu    x                       ; le texte de l'argument du CALL
ap_chknum:
        callf   000000H                 ; <- reecrit
```

Une version inconnue n'est pas devinée : la sonde refuse (état 1) — ⛔ **ce n'était vrai qu'à partir du 2026-09-29, voir plus bas**. Elle signe ses octets (`0D2h`),
comme l'exige la leçon des sondes précédentes, et rend toujours la main **retenue claire**.

**L'essai décisif est le dernier** : `CALL &BF000 1048576`, soit 2²⁰. `dec2bin` **doit** le
refuser avec l'erreur 33 — c'est sa limite mesurée (§2.1). Une adresse fausse ne produirait pas un
refus propre à cette valeur exacte : **ce test ne vérifie pas seulement que ça marche, il vérifie
qu'on appelle bien `dec2bin`.**

⛔ **Ce que les trois premiers relevés ont appris, qui vaut mieux que ce qu'ils cherchaient.**
Mesures sur PC-E500S, 2026-09-28 :

| Essai | Résultat | Ce qu'il apprend |
|---|---|---|
| `X` passé tel quel à `chknum` | erreur **90**, type | `X` désigne le **guillemet ouvrant** : `chknum` évaluait une chaîne |
| six octets relevés en `[X]` | `22 31 32 33 34 35` pour `"12345"`, `22 39 22 3A FE 62` pour `"9"` | `X` pointe dans le **texte de programme tokenisé** — on voit le `:` et le token suivant |
| guillemet sauté, chiffres présentés à `chknum` | erreur **10**, syntaxe | ⛔ **les chiffres ASCII ne sont pas un nombre pour l'interpréteur** |

**La cause est dans le format du texte tokenisé** : une constante numérique n'y est pas de l'ASCII,
elle s'écrit **`1Dh` + attributs + exposant + chiffres BCD** (`Codes_BASIC`, et la skill
`references/basic.md` §4). `chknum` lit du texte de programme et n'y cherche que cette forme-là ;
les chiffres d'une **chaîne littérale** ne sont pas de sa grammaire, et son erreur 10 est juste.

#### ⛔ La correction : l'argument s'écrit **sans guillemets**

Le 2026-09-28 j'ai conclu de ces trois relevés qu'« on ne nourrit pas `chknum` depuis un
`CALL &adr` ». **Cette conclusion était trop large, et elle était fausse.** Ce qu'ils établissent,
c'est qu'on ne le nourrit pas depuis une *chaîne littérale* — parce que le tokeniseur laisse en
ASCII ce qui est entre guillemets. Hors guillemets, il écrit une vraie constante. Et les deux
faits qui rendent la route praticable ont été **mesurés dans ces mêmes relevés**, pas supposés :

1. `X` pointe dans le **texte de programme tokenisé vivant** — le `3A FE 62` relevé après `"9"` est
   le `:` et le token de l'instruction suivante ;
2. `CALL` **laisse intact tout ce qui suit son adresse** — le guillemet était encore là.

Donc :

```basic
CALL &BF000 "12345"      ' [X] = 22 31 32 33 34 35   -> erreur 10, et c'est normal
CALL &BF000 12345        ' [X] = 20 1D ...           -> la grammaire de chknum
```

C'est exactement le mécanisme d'une extension du BASIC : lire ses arguments dans le texte qui suit,
et rendre `X` là où on s'est arrêté. Le `CALL` sert seulement de point d'entrée.

⚠️ **Faute de discipline, payée sur la machine.** La première version ne rendait `BP` que sur le
chemin de succès. Six appels en erreur ont donc laissé `BP` décalé de six fois quinze octets, et la
machine s'est mise à refuser ce qu'elle acceptait — « `CALL` n'accepte plus les chaînes ».
**`BP` se rend en ABSOLU, sur tous les chemins**, comme le fait BASEXT (`12` §5).

✅ **Un acquis déjà solide, quoi qu'il advienne de la suite** : la mécanique de choix des adresses
fonctionne. La sonde a lu `8.3`, posé `0EFAD4h` dans l'opérande de son `callf`, appelé, et reçu une
erreur **propre** — 90 puis 10, jamais un plantage. L'appel à une adresse choisie **à l'exécution**
est donc éprouvé ; c'est l'argument qui était mal formé.

#### T2BIN v3 — ce qu'elle fait de plus

329 octets. Trois différences avec la v2 :

- **elle saute les espaces** puis **refuse d'emblée** un argument commençant par `022h` (état 4,
  « argument entre guillemets ») : inutile d'appeler `chknum` pour s'entendre dire non ;
- **sur succès elle rend `X` tel que `chknum` l'a laissé** — lui seul sait où finit l'expression ;
- **sur échec elle rebalaye depuis l'original en respectant les longueurs** — `1Dh` vaut 8 ou 13
  octets selon le bit 0 de son octet d'attributs, `0FEh` en vaut 2, une chaîne court jusqu'à son
  guillemet fermant. Un balayage naïf chercherait un `03Ah` et buterait sur un octet d'exposant ou
  un code de token qui lui ressemble : c'est le piège de désynchronisation de la skill §4,
  rencontré ici pour de bon.

Emploi, sur le PC-E500S (8.3) puis sur l'émulateur PC-E500 (7.5) :

```basic
POKE &BFE03,&1A,&FD,&B,0,&C,0 : CALL &FFFD8
LOAD M "X:T2BIN.OBJ"
RUN                                    ' T2BIN.BAS : six essais, dont 2^20
```

#### ✅ Mesure sur PC-E500S, 2026-09-29 — la sonde est éprouvée

**T2BIN v4**, six essais sur six, `ROM 8.3` :

| Essai | Rendu |
|---|---|
| `CALL &BF000 0` | `0 = 0 VIA EF26E EFAD4` |
| `CALL &BF000 9` | `9 = 9 VIA EF26E EFAD4` |
| `CALL &BF000 65535` | `65535 = 65535 VIA EF26E EFAD4` |
| `CALL &BF000 100*3+45` | `345` — ✅ **l'expression complète, opérateurs compris** |
| `CALL &BF000 1048575` | `1048575 = 1048575 VIA EF26E EFAD4` |
| `CALL &BF000 1048576` | `ETAT 3 ERR 33`, `RECU 1D 0 6 10 48 57` — ✅ **`dec2bin` refuse 2²⁰** |

Relevés intermédiaires de la mise au point, conservés parce qu'ils portent les contrats :
`CALL &BF000 "100*3+45"` → `ARGUMENT ENTRE GUILLEMETS`, `RECU 22 31 30 30 2A 33` (la chaîne
littérale reste de l'ASCII, §ci-dessus) ; et, en v3, `CALL &BF000 100*3+45` → `Syntax error`
(`chknum` s'arrêtait sur le `*`).

**Trois choses sont acquises d'un coup.**

1. ✅ **`eval` et `dec2bin` s'appellent depuis un `CALL &adr`**, à la seule condition que
   l'argument soit écrit **sans guillemets**. La correction ci-dessus est confirmée sur machine —
   et `345` prouve que l'expression est évaluée en entier, priorité des opérateurs comprise.
2. ✅ **C'est bien `dec2bin` qu'on appelle, et pas une adresse voisine qui marcherait par hasard** :
   `1048575` passe, `1048576` est refusé avec l'erreur **33**. Le refus tombe exactement sur 2²⁰,
   la limite mesurée au §2.1 — aucune autre routine ne produirait cette frontière-là.
3. ✅ **Le format de la constante tokenisée est vérifié à l'œil**, sur le relevé `RECU` du dernier
   essai :

   ```
   1D  00  06  10 48 57 …
   |   |   |   +-- BCD : 1 048 576
   |   |   +------ exposant 10^6
   |   +---------- attributs : bit 3 = 0 (positif), bit 0 = 0 (simple precision)
   +-------------- constante reelle
   ```

   Soit `1,048576 × 10⁶`. La table de la skill `references/basic.md` §4 se lit ici en clair.

#### ⛔ Le cas qui résistait : `chknum` ne lit **qu'un terme**

`CALL &BF000 100*3+45` rendait `Syntax error` **avant tout affichage**, alors que les constantes
isolées passaient. La cause n'était pas à chercher : elle était déjà écrite, et déjà payée une
fois, dans `BASEXT.ASM` (`12` §5) —

> `chknum` (`0EFBF3h`) ne lit qu'**UN TERME** : un nombre, une variable, un appel de fonction, ou
> une expression entre parenthèses — mais il **s'arrête au premier OPERATEUR**. C'est ce
> qu'emploie `PEEK` (`0F5F9Ch`), et c'est pourquoi `PEEK &BFD1C*&100` se lit `(PEEK &BFD1C)*&100`.
> `eval` (`0EF26Eh`) évalue une **EXPRESSION COMPLETE**, opérateurs compris.
>
> *Mesure d'alors : `MOD (17,5)` et `MOD (A,B)` passaient, `MOD (A+1,B)` et `MOD (I*3571,997)`
> rendaient une erreur — `chknum` lisait « A », puis butait sur le « + ».*

`chknum` avait donc consommé `100` et rendu `X` sur le `2Ah` — que l'interpréteur lit comme
l'ouverture d'une **référence de label**, d'où la syntaxe refusée. Le contrat de `chknum` sur `X`
est juste ; c'est la sonde qui appelait le mauvais service.

**T2BIN v4 appelle `eval`**, suivi du test de type que font les appelants de la ROM
(`0F5F6Bh`) puisque `eval` ne le fait pas lui-même :

```asm
ap_eval:
        callf   000000H         ; 0EF26Eh sur 8.3, reecrit a l'execution
        jrc     ec_eval
        test    (BP+0),080H     ; bit 7 : c'est une chaine -> etat 5
        jrnz    ec_type
        mv      [!sv_x1],x      ; X tel qu'eval l'a laisse
ap_dec2bin:
        callf   000000H
```

#### Le mot-clé `EVAL` du BASIC ne prend qu'une **chaîne**

Mesure en mode direct, 2026-09-29 :

| Frappé | Rendu |
|---|---|
| `A$="100*3+45"` puis `EVAL A$` | `345` |
| `EVAL 100*3+45` | `Type mismatch` |
| `EVAL(100*3+45)` | `Type mismatch` |

⛔ **Ne pas confondre les deux.** Le mot-clé `EVAL` du BASIC attend une **chaîne de caractères**,
qu'il fait tokeniser puis évaluer. Le service `eval` de la ROM (`0EF26Eh`) attend au contraire du
**texte de programme déjà tokenisé** désigné par `X`. Même nom, contrats opposés — et c'est le
second que `BASEXT` et `T2BIN` appellent. Le premier est la voie BASIC pour évaluer une expression
construite à l'exécution ; il n'a pas d'équivalent en machine sans passer par le tokeniseur.


#### ⛔ Sur PC-E500 (7.5), la sonde plante — et ce n'est pas la table

Essai du 2026-09-29 sur l'émulateur PC-E500 : `RUN`, la version s'affiche, puis **la machine se
bloque** (écran brouillé, `RESET` nécessaire). Trois causes étaient possibles. **Deux se sont
écartées sans toucher à la machine** — c'est tout l'intérêt d'avoir les dumps et les essais
précédents sous la main.

**1. « L'adresse d'`eval` serait fausse » — écartée.** Les trois ROM portent à l'adresse attendue
la même vignette, relue octet à octet dans les dumps :

| ROM | Adresse | Octets | Décodé | Début du corps |
|---|---|---|---|---|
| 8.3 | `0EF26Eh` | `04 72 F2` `07` | `call 0F272h` + `retf` | `72 5C FC 0B…` |
| 7.5 | `0EF548h` | `04 4C F5` `07` | `call 0F54Ch` + `retf` | `72 5C FC 0B…` |
| 5.3 | `0EF52Dh` | `04 31 F5` `07` | `call 0F531h` + `retf` | `72 5C FC 0B…` |

Chaque vignette appelle **`soi + 4`** : c'est un enrobage `retf` posé juste devant la routine, et
le corps commence par les mêmes octets dans les trois. Même constat pour `dec2bin` (`0EFAD4h`,
`0EFDAEh`, `0EFD90h`, corps `65 00 80 15…`).

**2. « La réservation ou l'adresse de chargement ne conviendraient pas au PC-E500 » — écartée.**
`HISTDRV` emploie **exactement** la même ligne — `POKE &BFE03,&1A,&FD,&B,0,&C,0 : CALL &FFFD8` —
et le même `0BF000h`, et il tourne sur cet émulateur (essai du 2026-09-24, `12` §17).

**3. Le chemin d'appel lui-même.** C'est ce qui reste, et cela recouvre au moins trois choses
qu'aucune mesure ne sépare encore : le `popu` de l'argument (le `CALL` de la 7.5 ne laisse
peut-être pas `X` où celui de la 8.3 le laisse), le `callf` et son cadre, et l'écriture de la zone
de résultat **à 32 octets du plafond** `0BFC00h` — `HISTDRV` n'était jamais monté au-delà de
`0BF937h`.

⚠️ **À noter, parce que cela change la lecture du plantage** : `BASEXT` non plus n'a jamais tourné
sur un PC-E500. Appeler un service du BASIC depuis `0BF000h` sur une 7.5 est un terrain **entièrement
neuf** — l'échec n'accuse pas la table, il dit qu'on y arrive pour la première fois.

#### `Documentation/T2DRY.ASM` — la sonde à blanc

140 octets. Elle **lit tout et n'appelle rien** : pas de `popu` (elle s'appelle sans argument, la
pile `U` reste intacte), pas de `callf`, et sa zone de résultat est en `0BF700h`, loin du plafond.
Elle fait une chose de plus que T2BIN, et c'est celle qui compte :

```asm
pose:   mv      [res_ev],x              ; l'adresse choisie
        mv      y,res_oev
        mv      il,004H
cp_ev:  mv      a,[x++]                 ; RELIRE LA ROM DE LA MACHINE
        mv      [y++],a
        dec     il
        jrnz    cp_ev
```

**Elle relit dans la ROM de la machine les quatre octets du service qu'elle aurait appelé.** Si
l'émulateur rend `04 4C F5 07` pour `eval`, sa ROM est bien celle de notre dump et **la table est
confirmée sur place** — le problème est alors entièrement dans le chemin d'appel. Si elle rend
autre chose, c'est une autre révision, et la table ne la couvre pas.

Un seul essai, qui ne peut rien casser :

```basic
POKE &BFE03,&1A,&FD,&B,0,&C,0 : CALL &FFFD8
LOAD M "X:T2DRY.OBJ"
RUN                                    ' T2DRY.BAS
```


#### ⛔ La cause : l'émulateur PC-E500 porte une **ROM 7.3**, que nous n'avons jamais vue

Essai du 2026-09-29 : l'écran affiche **`ROM 7 . 3`**. Notre collection de dumps ne contient que
trois versions, et 7.3 n'en fait pas partie :

| Dump | Version |
|---|---|
| `rom53.bin` | 5.3 |
| `rom75.bin` | 7.5 |
| `rom83.bin` | 8.3 |

**Et la sonde ne comparait que le chiffre majeur** — `cmp a,007H` — donc elle a rangé la 7.3 avec
la 7.5 et posé `0EF548h` dans l'opérande de son `callf`. Un appel lointain au milieu d'une ROM
inconnue : la machine s'est bloquée, et c'est la seule fin possible.

⛔ **La faute est double, et la seconde est la plus grave.** Le chapitre affirmait, quatre
paragraphes plus haut : « *Une version inconnue n'est pas devinée : la sonde refuse (état 1)* ».
**C'était faux du code que j'avais écrit** : le refus n'existait que pour un chiffre majeur autre
que 5, 7 ou 8. Une prose qui décrit l'intention plutôt que le code est pire qu'une prose absente —
elle fait porter la confiance sur une garantie qui n'est pas là.

Correction appliquée aux deux sondes : **les deux octets sont comparés**, et seules `5.3`, `7.5` et
`8.3` sont reconnues.

```asm
        mv      a,[rom_majeure]
        cmp     a,007H
        jrnz    ck53
        mv      a,[rom_mineure]         ; ⛔ LE SECOND OCTET AUSSI
        cmp     a,005H
        jrz     v75
        jr      inconnue                ; 7.3, 7.4, 7.6 : inconnues, on refuse
```

✅ **Ce que l'essai a appris de positif, et ce n'est pas rien** : `T2DRY` a été chargée et appelée
sur le PC-E500 **sans bloquer la machine**. Le `Syntax error in 50` qui a suivi est exactement ce
qu'elle doit produire — elle s'appelle sans argument et ne consomme donc pas le ` 0` de la ligne de
`T2BIN.BAS` restée en mémoire. **Réserver, charger en `0BF000h`, appeler et revenir : tout cela
fonctionne sur une 7.3.** Seul l'appel lointain à une adresse devinée ne fonctionnait pas.

⚠️ **La table de correspondance ne couvre donc pas la machine dont nous disposions pour l'éprouver.**
Ce n'est pas un échec de la table : c'est la découverte d'une quatrième révision. Deux suites
possibles, et elles ne s'excluent pas :

1. **Faire parler la 7.3 avec `T2DRY`** — elle lit, elle n'appelle pas. Sur une version inconnue
   elle choisit désormais le jeu du chiffre majeur et **relit quand même la ROM** : on verra donc
   noir sur blanc ce qu'une 7.3 porte en `0EF548h`. Si c'est `04 4C F5 07`, les deux révisions
   partagent l'adresse et la table s'étend d'une colonne. Sinon, l'écart se mesure.
2. **Obtenir un dump de la 7.3** et lui appliquer la recherche par empreinte du §2.7, comme aux
   trois autres. C'est la voie complète, mais elle suppose le dump.

> **La leçon de méthode, et elle vaut au-delà de cette sonde** : `HISTDRV` a réussi là où `T2BIN` a
> échoué parce qu'il adaptait par **intervalle** (« 5.x–7.x contre 8.x », un décalage d'un octet
> dans un champ), là où `T2BIN` avait besoin d'une **adresse exacte**. Une adaptation par intervalle
> tolère une révision inconnue ; une table d'adresses, jamais. **Le degré de précision dont on a
> besoin dicte la sévérité du refus.**


#### ✅ Ce que la 7.3 porte vraiment à ces adresses — et le décalage qui s'en déduit

Relevé en mode direct, 2026-09-29, sur l'émulateur PC-E500 :

```
D3  07 03  48 F5 0E  AE FD 0E  F5 1C 02 90  00 F7 9F 06  07
|   |      |         |         |            |
|   |      |         |         |            +-- lus en 0EFDAEh
|   |      |         |         +--------------- lus en 0EF548h
|   |      |         +------------------------- dec2bin choisi : 0EFDAEh
|   |      +----------------------------------- eval choisi : 0EF548h
|   +------------------------------------------ version 7.3
+---------------------------------------------- signature T2DRY
```

Ni `F5 1C 02 90` ni `00 F7 9F 06` n'est une vignette `04 xx xx 07` : `callf 0EF548h` exécutait donc
du code quelconque, et la machine ne pouvait que se bloquer. **La mesure confirme le diagnostic.**

Mais elle donne bien davantage. Ces octets ne sont pas quelconques — **ils sont dans le corps des
routines que l'on cherchait**, au même endroit pour les deux :

| Service | Octets lus sur la 7.3 | Où ils sont dans le corps de la 7.5 | Adresse 7.5 correspondante |
|---|---|---|---|
| `eval` | `F5 1C 02 90` | offset **17** | `0EF55Dh` |
| `dec2bin` | `00 F7 9F 06` | offset **17** | `0EFDC3h` |

Le même point de code est donc lu en `0EF548h` là où la 7.5 l'a en `0EF55Dh` : **un décalage de
−21 octets — et exactement le même pour les deux services.** Deux mesures indépendantes qui
tombent sur le même écart ne sont pas une coïncidence.

⚠️ **D'où une prédiction, qui n'est pas une mesure** : sur 7.3, `eval` serait en `0EF533h` et
`dec2bin` en `0EFD99h`. C'est en la vérifiant qu'on la croira — et surtout pas en la posant dans un
`callf`, ce qui serait refaire exactement la faute qui a bloqué la machine.

#### `Documentation/T2FIND.ASM` — ne plus deviner : **faire chercher la machine**

248 octets. La bonne réponse au problème n'est pas une quatrième colonne dans la table : c'est de
**ne plus avoir besoin de table**.

> ⛔ **Le principe** : les adresses changent d'une révision à l'autre ; **le code, non.** Le corps
> d'`eval` commence par les mêmes six octets dans `rom53`, `rom75` et `rom83`, et ces six octets
> n'apparaissent **qu'une seule fois** dans chacune des trois — vérifié sur les dumps. Il suffit
> donc de les chercher.

| Service | Empreinte cherchée | Position | Vignette |
|---|---|---|---|
| `eval` | `72 5C FC 0B BF 04` | offset 0 du corps | trouvé − 4 |
| `dec2bin` | `65 01 80 18 0A` | offset 6 du corps | trouvé − 10 |

Vérification sur les trois dumps : la recherche rend `0EF52Dh`/`0EFD90h` sur `rom53`,
`0EF548h`/`0EFDAEh` sur `rom75`, `0EF26Eh`/`0EFAD4h` sur `rom83` — **les adresses exactes de la
table, retrouvées sans la table.**

La sonde balaie `0C0000h`-`0FFFFFh` octet par octet. Le test de fin tient au fait que `X` n'a que
**20 bits** : passé `0FFFFFh` il reboucle à `000000h`, et le quartet de poids fort redescend sous
`0Ch` :

```asm
boucle: mv      (000H),x                ; les trois octets de X en RAM interne
        mv      a,(002H)                ; bits 16-19
        cmp     a,00CH
        jrc     fini                    ; sorti de la ROM : termine
        mv      a,[x]
        cmp     a,072H                  ; ancre d'eval
        jrz     t_ev
        cmp     a,065H                  ; ancre de dec2bin
        jrz     t_d2
```

Elle **compte toutes les occurrences** (un `x1` à l'affichage est aussi un résultat : il dit que
l'empreinte est restée discriminante), et **relit les quatre octets** à l'adresse trouvée — on doit
y voir `04 xx xx 07`.

⚠️ Le balayage couvre 256 Ko : compter quelques secondes. La sonde **n'appelle rien** et n'écrit
que dans sa zone ; elle ne peut pas bloquer la machine.

```basic
POKE &BFE03,&1A,&FD,&B,0,&C,0 : CALL &FFFD8
LOAD M "X:T2FIND.OBJ"
RUN
```

Attendu sur la 7.3, si la prédiction du décalage est juste : `EVAL EF533 x1 : 04 37 F5 07` et
`D2B EFD99 x1 : 04 9D FD 07`.

> **Ce que cela change pour toute extension du BASIC portable.** Une table d'adresses vieillit :
> elle ne connaît que les ROM qu'on a dumpées, et elle est muette — donc dangereuse — sur les
> autres. Une **recherche par empreinte à l'installation** ne connaît que du code, et le code est
> ce qui ne bouge pas. Elle coûte quelques secondes une fois, au chargement du pilote, et elle
> **échoue proprement** quand elle ne trouve pas : c'est exactement ce qui manquait à `T2BIN`.


#### ✅ La recherche par empreinte, éprouvée sur la 7.3 — et la prédiction vérifiée

Essai du 2026-09-29, émulateur PC-E500, `T2FIND` :

```
VUE 7 . 3   ETAT 8
EVAL EF533 x 1  :  4 37 F5 7
D2B  EFD99 x 1  :  4 9D FD 7
```

**Les deux adresses prédites, à l'octet près.** Et les deux vignettes sont bien formées : `04 37 F5`
appelle `0F537h`, soit `0EF533h + 4` ; `04 9D FD` appelle `0FD9Dh`, soit `0EFD99h + 4`. Le `x1` dit
de surcroît que chaque empreinte n'apparaît **qu'une fois** dans les 256 Ko : elle est restée
discriminante sur une quatrième révision, qui n'avait pas servi à la construire.

La table de correspondance gagne donc une colonne — **mesurée**, pas déduite :

| Service | 8.3 | 7.5 | **7.3** | 5.3 |
|---|---|---|---|---|
| `eval` | `0EF26Eh` | `0EF548h` | **`0EF533h`** | `0EF52Dh` |
| `dec2bin` | `0EFAD4h` | `0EFDAEh` | **`0EFD99h`** | `0EFD90h` |

Le décalage 7.5 → 7.3 est de **−21 octets** pour les deux services, ce qui confirme après coup la
lecture faite des quatre octets relevés par `T2DRY`.

#### T2BIN v5 — la sonde cherche elle-même

630 octets. Elle ne consulte plus aucune table : à son premier appel elle **balaie la ROM**, garde
les deux adresses, et les **revalide** à chaque appel suivant — signature, version, et les deux
vignettes toujours à leur place — pour ne balayer qu'une fois.

⚠️ **Un détail d'assembleur qui vaut d'être noté** : `cmp a,[adresse]` **n'existe pas** dans
`xasm2026-4` (ni `cmp a,(n)`). Pour comparer `A` à un octet de mémoire externe, la sonde **réécrit
l'opérande immédiat** du `cmp` — le même idiome que pour ses deux `callf` :

```asm
        mv      a,[rom_majeure]
        mv      [!cv_maj+1],a           ; poser l'octet attendu dans l'operande
        mv      a,[res_ver]
cv_maj: cmp     a,000H                  ; OPERANDE REECRIT
        jrnz    rebal
```

⛔ **Et une chausse-trape côté BASIC, évitée de justesse** : le programme de lecture ne doit **pas**
effacer la signature après l'avoir lue, comme le faisaient les versions précédentes — c'est elle
qui dit à la sonde que son cache d'adresses est bon. Il marque désormais l'**état**. Sans cela,
chaque appel rebalayait 256 Ko.


#### Les trois sondes écrivent leur relevé sur `F:`

Relire un écran de 4 lignes à la loupe est une source d'erreur en soi — et le relevé se perd au
`RESET` suivant. Les trois programmes de conduite écrivent donc **à l'écran *et* dans un fichier**,
l'idiome des sondes `T48x` (`BASEXT-DRV/sondes/T484.BAS`) :

| Sonde | Fichier |
|---|---|
| `T2BIN.BAS` | `F:T2BINRES.TXT` |
| `T2FIND.BAS` | `F:T2FINDRE.TXT` |
| `T2DRY.BAS` | `F:T2DRYRES.TXT` |

```basic
50 OPEN "F:T2BINRES.TXT" FOR OUTPUT AS #1
...
940 PRINT L$:PRINT #1,L$:RETURN
```

⛔ **Et un garde-fou, parce que ces fichiers passent par le CE-140F.** Le défaut mesuré le
2026-08-27 est là : **une ligne dont la somme des caractères vaut `03h`** fait `1Ah` une fois `CR`
et `LF` ajoutés — le marqueur de fin de fichier — et **le fichier est tronqué là, sans aucune
erreur signalée**. Les lignes écrites ici portent des valeurs mesurées : leur somme n'est pas
connue à l'avance. Le sous-programme d'écriture la calcule donc et décale d'une espace :

```basic
920 S9=0:FOR I9=1 TO LEN (L$):S9=S9+ASC (MID$ (L$,I9,1)):NEXT I9
930 IF S9-INT (S9/256)*256=3 THEN L$=" "+L$
```

⚠️ **L'espace se met en TÊTE, jamais en fin** : la machine supprime l'espace terminale quand elle
relit une ligne, et le correctif disparaîtrait au premier aller-retour.

#### ✅ L'épreuve complète : `eval` et `dec2bin` appelés sur une ROM dont nous n'avons aucun dump

`F:T2BINRES.TXT`, émulateur PC-E500, 2026-09-29 — le fichier rendu tel quel :

```
T2BIN V5 -- ROM 7. 3
0 = 0
ADRESSES EN CACHE
EVAL EF533 X 1
  VIGNETTE 4 37 F5 7
D2B  EFD99 X 1
  VIGNETTE 4 9D FD 7
9 = 9
65535 = 65535
100*3+45 = 345
1048575 = 1048575
1048576 : ETAT 3 ERR 33
  RECU 1D 0 6 10 48 57
```

**Cinq résultats d'un coup, et le dernier est le plus important.**

1. ✅ **Les six essais passent sur la 7.3**, expression composée comprise (`100*3+45 = 345`).
2. ✅ **`1048576` est refusé avec l'erreur 33** : la frontière tombe exactement sur 2²⁰, la limite
   mesurée de `dec2bin` au §2.1. C'est bien lui qu'on appelle, à une adresse que personne n'avait
   écrite nulle part.
3. ✅ **Les adresses ont été trouvées par empreinte de code**, pas lues dans une table :
   `0EF533h` et `0EFD99h`, vignettes `04 37 F5 07` et `04 9D FD 07`, **une seule occurrence
   chacune** dans les 256 Ko.
4. ✅ **Le cache et sa revalidation fonctionnent** — `ADRESSES EN CACHE` : la sonde a retrouvé la
   signature, la version et les deux vignettes en place, et n'a pas rebalayé.
5. ✅ **La sortie sur `F:` n'est pas tronquée** : le fichier finit sur son `1Ah` d'EOF après la
   dernière ligne, garde-fou compris.

> ⛔ **Ce que cela établit, et qui dépasse la sonde.** Nous avons appelé deux services du BASIC sur
> une machine dont **nous ne possédons pas la ROM**, sans table d'adresses, sans deviner, et sans
> la bloquer. La conjecture du §2.7 — « une extension du BASIC portable sur les trois machines » —
> est non seulement vraie, mais **trop timide** : la recherche par empreinte ne connaît pas de
> liste de révisions. Elle connaît du code, et elle **échoue proprement** quand elle ne le trouve
> pas.
>
> Le coût est une poignée de secondes **une seule fois**, à l'installation d'un pilote. C'est le
> prix d'un programme qui ne se trompera pas de machine.

⚠️ **Ce que cela n'établit pas.** L'empreinte a été construite sur trois ROM et vérifiée sur une
quatrième : c'est un échantillon, pas une preuve. Une révision où le corps d'`eval` différerait de
ses six premiers octets ne serait pas trouvée — et la sonde le **dirait** (`x0`, état 1) au lieu de
sauter dans le vide. C'est toute la différence avec la v4.

⚠️ **La ligne `chknum` de la table reste sans épreuve** (`0EFECDh` en 7.5, `0EFEAFh` en 5.3) :
depuis la v4 la sonde ne l'appelle plus. Son empreinte se construirait de la même façon ; il
faudrait une sonde jumelle, ou le premier mot-clé calqué sur `PEEK` qu'on portera.

##### ✅ Et le même objet sur PC-E500S (8.3) — `F:T2BINRES.TXT`, même jour

```
T2BIN V5 -- ROM 8. 3
0 = 0
BALAYAGE FAIT A CET APPEL
EVAL EF26E X 1
  VIGNETTE 4 72 F2 7
D2B  EFAD4 X 1
  VIGNETTE 4 D8 FA 7
9 = 9
65535 = 65535
100*3+45 = 345
1048575 = 1048575
1048576 : ETAT 3 ERR 33
  RECU 1D 0 6 10 48 57
```

⛔ **C'est la contre-épreuve, et elle vaut mieux que l'essai sur la 7.3.** Le balayage retrouve
`0EF26Eh` et `0EFAD4h` — **exactement les deux adresses que `BASEXT` avait mesurées
indépendamment**, par une tout autre voie (les mots-clés `LPEEK`, `WPEEK`, `MOD` qui tournent sur
machine, `12` §5). La recherche par empreinte est donc confrontée à une vérité de terrain établie
ailleurs, et elle tombe dessus.

✅ **`BALAYAGE FAIT A CET APPEL`** : cet essai-ci a réellement exercé le chemin de balayage, ce que
l'essai sur la 7.3 n'avait pas fait — il avait trouvé ses adresses en cache, laissées par un
passage antérieur. Les deux chemins sont donc éprouvés, chacun sur une machine.

**Un même objet de 630 octets, sans une ligne de différence, tourne sur les deux machines** : un
PC-E500S en 8.3 et un PC-E500 en 7.3 dont nous n'avons pas la ROM. C'était l'objectif énoncé plus
haut ; il est atteint, et par un moyen plus solide que celui qui avait été envisagé.

##### ✅ Et sur PC-E550 (7.5) — la colonne qui n'avait jamais été exécutée

```
T2BIN V5 -- ROM 7. 5
0 = 0
BALAYAGE FAIT A CET APPEL
EVAL EF548 X 1
  VIGNETTE 4 4C F5 7
D2B  EFDAE X 1
  VIGNETTE 4 B2 FD 7
9 = 9
65535 = 65535
100*3+45 = 345
1048575 = 1048575
1048576 : ETAT 3 ERR 33
  RECU 1D 0 6 10 48 57
```

**C'est la question posée au début de cette section, et elle est close.** Il y était écrit : « il
faudrait appeler `dec2bin` à `0EFDAEh` sur une vraie 7.5 et voir ce qui revient ». Voilà ce qui
revient — et deux fois plutôt qu'une, car l'épreuve est double :

1. ✅ **Le balayage trouve `0EF548h` et `0EFDAEh`**, les adresses que la table avait déduites par
   identité de code, **jamais exécutées jusqu'ici** ;
2. ✅ **Les vignettes rendues par la machine sont `04 4C F5 07` et `04 B2 FD 07`** — **octet pour
   octet** ce que porte notre `rom75.bin` à ces deux adresses. Le dump et la machine disent la
   même chose.

Et les six essais passent comme ailleurs, `100*3+45 = 345` compris.

##### ✅ Et sur PC-U6000 (8.4) — une cinquième révision, sans dump elle non plus

```
T2BIN V5 -- ROM 8. 4
0 = 0
BALAYAGE FAIT A CET APPEL
EVAL EF2AE X 1
  VIGNETTE 4 B2 F2 7
D2B  EFB13 X 1
  VIGNETTE 4 17 FB 7
9 = 9
65535 = 65535
100*3+45 = 345
1048575 = 1048575
1048576 : ETAT 3 ERR 33
  RECU 1D 0 6 10 48 57
```

`0EF2AEh` et `0EFB13h`, vignettes bien formées (`04 B2 F2` appelle `0F2B2h` = `0EF2AEh + 4`), une
seule occurrence chacune, et les six essais passent.

⛔ **Et c'est l'essai qui condamne définitivement l'extrapolation.** La 7.3 s'était laissé décrire
par un **décalage uniforme** de −21 octets pour les deux services, ce qui donnait envie de
généraliser. La 8.4 ne se laisse pas faire :

| Service | 8.3 | 8.4 | Écart |
|---|---|---|---|
| `eval` | `0EF26Eh` | `0EF2AEh` | **+64** |
| `dec2bin` | `0EFAD4h` | `0EFB13h` | **+63** |

**Deux écarts différents dans la même ROM.** Une extrapolation à partir d'un seul service aurait
donc placé l'autre à un octet près — c'est-à-dire au milieu d'une instruction, et la machine se
serait bloquée comme le 2026-09-29. Aucun raccourci arithmétique ne remplace la recherche.

##### ⛔ Et sur PC-E500JP (7.2) — l'argument se referme par l'autre bout

```
T2BIN V5 -- ROM 7. 2
0 = 0
BALAYAGE FAIT A CET APPEL
EVAL EF548 X 1
  VIGNETTE 4 4C F5 7
D2B  EFDAE X 1
  VIGNETTE 4 B2 FD 7
...
1048576 : ETAT 3 ERR 33
```

Une **sixième** révision — et elle porte `eval` et `dec2bin` **aux adresses de la 7.5**,
`0EF548h` et `0EFDAEh`, avec les vignettes `04 4C F5 07` et `04 B2 FD 07`, celles-là mêmes que
notre `rom75.bin`. ⚠️ Cela dit que **ces deux points de code coïncident**, pas que les deux ROM
soient identiques : nous n'avons mesuré que deux adresses.

**Les deux façons dont une table de versions échoue sont maintenant mesurées, et elles sont
opposées :**

| Révision | Ce que fait une table indexée par la version | Ce qui arrive |
|---|---|---|
| **7.3** | même chiffre majeur que 7.5 → **devine** ses adresses | ⛔ `callf` au milieu d'une instruction, **machine bloquée** (mesuré) |
| **7.2** | version absente de la table → **refuse** | ⛔ une machine parfaitement utilisable est écartée, alors que les adresses de la 7.5 y marchent |

> ⛔ **Le numéro de version n'est ni suffisant ni nécessaire.** Pas suffisant : deux ROM qui le
> partagent presque peuvent différer (7.3 contre 7.5). Pas nécessaire : deux ROM qui l'ont
> différent peuvent coïncider là où ça compte (7.2 et 7.5). Toute discipline fondée sur lui est
> donc fautive dans un sens ou dans l'autre — trop laxiste, elle plante ; trop stricte, elle
> refuse ce qui marche. **Seul le code dit la vérité sur le code.**

C'est aussi la correction, mesurée, de ce que `HISTDRV` fait encore : il adapte par **intervalle**
(« 5.x–7.x contre 8.x ») et refuse le reste. Cela l'a protégé du plantage — mais cela lui aurait
fait refuser le PC-U6000 en 8.4, qui fonctionne.

##### Où l'on sait maintenant quelle machine porte quelle ROM

| Machine | ROM | Dump en notre possession | `eval` | `dec2bin` |
|---|---|---|---|---|
| PC-U6000 | **8.4** | ⛔ aucun | `0EF2AEh` | `0EFB13h` |
| PC-E500S | **8.3** | ✅ `rom83.bin` | `0EF26Eh` | `0EFAD4h` |
| PC-E550 | **7.5** | ✅ `rom75.bin` | `0EF548h` | `0EFDAEh` |
| **PC-E500-BL** | **7.5** | ✅ `rom75.bin` | `0EF548h` | `0EFDAEh` |
| PC-E500 | **7.3** | ⛔ aucun | `0EF533h` | `0EFD99h` |
| **PC-E500JP** | **7.2** | ⛔ aucun | `0EF548h` | `0EFDAEh` |
| — | 5.3 | ✅ `rom53.bin` | `0EF52Dh` | `0EFD90h` |

⚠️ **Le PC-E500-BL porte la même 7.5 que le PC-E550** (essai du 2026-09-29, relevé identique au
sien ligne pour ligne). La correspondance machine → révision **n'est donc pas bijective** : deux
modèles distincts peuvent partager une ROM, et — comme le montrent le PC-E500 en 7.3 et le PC-E550
en 7.5 — **un même modèle peut en porter plusieurs**. C'est une raison de plus de lire la version
plutôt que de déduire du modèle, et de chercher plutôt que de lire la version.

⚠️ **Ni la 7.3 ni la 8.4 ne sont des curiosités** : la première est la révision du PC-E500 de
première génération, la seconde celle du PC-U6000 — que `HISTDRV` avait justement **refusée** faute
de l'avoir mesurée (`12` §17). Ce sont les deux dont nous n'avons pas le dump, et ce sont
exactement celles que la table n'aurait jamais pu couvrir.

Les relevés sont conservés : `Documentation/T2BINRES-84.TXT`, `-83`, `-75`, `-75-BL`, `-73`, `-72`.

⚠️ `T2BINRES-75.TXT` (PC-E550) est une **transcription** : l'outil de copie était indisponible à ce
moment-là et le fichier a été retapé depuis sa lecture. Son contenu est identique à celui du
PC-E500-BL, aux espaces terminales et au `1Ah` final près — `T2BINRES-75-BL.TXT` est, lui, l'octet
pour octet rendu par la machine. Le dire plutôt que de laisser croire que les deux ont la même
provenance.

#### La table de correspondance, après l'épreuve

| Service | 8.4 | 8.3 | 7.5 | 7.3 | 5.3 | Établie par |
|---|---|---|---|---|---|---|
| `eval` | `0EF2AEh` | `0EF26Eh` | `0EF548h` | `0EF533h` | `0EF52Dh` | ✅ empreinte, **balayage exécuté** sur 8.4, 8.3, 7.5, 7.3 et 7.2 |
| `dec2bin` | `0EFB13h` | `0EFAD4h` | `0EFDAEh` | `0EFD99h` | `0EFD90h` | ✅ idem |
| `chknum` | — | `0EFBF3h` | `0EFECDh` | — | `0EFEAFh` | ⚙️ identité de code, exécutée sur 8.3 seulement |

⚠️ La colonne **5.3** reste la seule jamais exécutée : nous en avons le dump, pas la machine.

> **Le bilan de la section, en une phrase.** Un objet de 630 octets, sans une ligne de différence,
> a appelé `eval` et `dec2bin` sur **cinq révisions de ROM** — dont **trois dont nous ne possédons
> aucun dump** — en trouvant leurs adresses lui-même, et il n'a bloqué aucune machine. Une table
> indexée par la version aurait planté sur l'une (7.3) et refusé une autre (7.2). La table n'est
> plus le moyen : elle est la trace.

**La table n'est plus le moyen : elle est la trace.** Un programme qui a besoin de ces adresses les
cherche ; la table sert à vérifier qu'il a trouvé ce qu'il fallait.

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
