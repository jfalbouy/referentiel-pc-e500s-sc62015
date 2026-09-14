# Étendre le BASIC — ajouter ses propres instructions et fonctions

L'interpréteur BASIC du PC-E500S expose **deux crochets** par lesquels un programme en langage machine installe ses propres mots-clés. Une fois installés, ils s'emploient exactement comme ceux de la ROM : `PRINT LPEEK &BF100` et non `CALL &BF000`.

Le mécanisme est **spécifié** (§7), mais aucun des projets de `C:\Claude` ne l'exploitait avant septembre 2026, et le seul exemple du corpus — `BASCOM` (TORO, 1994) — n'ajoute que des **instructions**. Les **fonctions** ont demandé une rétro-ingénierie complète, validée sur machine ; c'est l'objet principal de ce document.

> **Où est le code.** Les gabarits assemblables sont dans `SC62015Disassembler/Samples/LPEEK/` — `LSEPT.ASM` (instruction) et `LPEEK.ASM` (deux fonctions, `LPEEK` et `WPEEK`), avec `TEST.BAS` qui les exerce depuis un programme. Le détail commenté ligne à ligne est dans `SC62015Disassembler/Docs/Routines-ROM-PC-E500S.asm`. Ce document en donne le principe, pas la copie.

---

## 1. Les deux crochets

Ils vivent dans la zone de travail de l'interpréteur, atteinte par le pointeur `BASPTR` (`0D1H`, RAM interne, 3 octets) :

| Crochet | Contenu |
|---|---|
| `[(0D1H)+090H]` | pointeur vers la **liste des noms** (texte → token) |
| `[(0D1H)+093H]` | pointeur vers la **liste des adresses d'exécution** (token → routine) |

La ROM y écrit `0FFFFFH` au démarrage (routine `F93C3H`) : c'est la sentinelle « aucune extension ».

L'installation tient en une douzaine d'instructions. Elle sauvegarde les anciennes valeurs pour permettre une désinstallation — mais **seulement si elles sont encore la sentinelle** : sans ce garde-fou, un second appel detruit la sauvegarde (§12).

```asm
        mv    x,(basptr)          ; 0D1H
        mv    a,[x+092H]          ; poids fort du crochet des noms
        cmp   a,00FH              ; encore la sentinelle 0FFFFFh ?
        jrnz  deja_installe       ; non : NE PAS écraser la sauvegarde (§12)
        mv    y,[x+090H]          ; l'ancienne liste de noms
        mv    [old_kw],y          ;   sauvée
        mv    y,[x+093H]          ; l'ancienne liste d'adresses
        mv    [old_disp],y
deja_installe:
        mv    y,kw_table
        mv    [x+090H],y
        mv    y,disp_table
        mv    [x+093H],y
        retf
```

---

## 2. La liste des noms

Une entrée par mot-clé, terminée par un octet de longueur nul :

```asm
kw_table:
        db      5,'LPEEK',007H    ; longueur, lettres, token
        db      5,'WPEEK',00AH
        db      0                 ; fin de liste
```

Trois contraintes, dont la troisième est un apport de mesure :

1. **Les tokens doivent être distincts.** C'est ce qui distingue une liste de mots-clés d'une liste de même forme (les libellés de touches de fonction de la ROM, par exemple, ont tous le même dernier octet).
2. **Le token `000H` est exclu** : il termine la liste des adresses.
3. ⛔ **Un mot-clé ajouté ne doit ni commencer par un mot-clé existant, ni être le début de l'un d'eux.** Le tokeniseur consulte la table de la ROM, et `PEEKX` y est coupé en `PEEK` + la variable `X` — la ligne relue s'affiche littéralement `PRINT HEX$ PEEK X &BF100`. Vérifiable avant d'écrire une ligne de code, sur les 168 noms de `SC62015Disassembler/Data/BasicTokens.csv`.

**88 des 256 tokens sont libres** dans la table de la ROM. `BASCOM` occupe `04H`-`06H`.

> Un mot-clé d'extension est tokenisé **`0FEH` + token**, sur deux octets. Le préfixe `0FEH` n'est pas propre aux extensions : il précède *tous* les tokens dans un programme BASIC (*« Im Basicprogramm haben die Token noch den Vorcode FE »*, manuel système allemand).

---

## 3. La liste des adresses, et le drapeau qui décide de tout

Entrées de **quatre octets** — un token, puis trois octets d'adresse — terminées par un token nul :

```asm
disp_table:
        db      007H, LOW(rout), HIGH(rout), EXT(rout) | 080H   ; instruction
        db      00AH, LOW(f-3),  HIGH(f-3),  EXT(f-3)  | 040H   ; fonction
        db      0
```

**Le quartet haut du troisième octet d'adresse est un drapeau**, et les deux résolveurs de la ROM sont côte à côte, ne différant que par le bit testé :

| Résolveur | Bit | Routine |
|---|---|---|
| instructions (`F58E5H`) | `080H` | l'adresse **telle quelle** |
| fonctions (`F590BH`) | `040H` | l'adresse **+ 3** |

Chacun cherche d'abord dans la table de la ROM, puis, s'il n'y trouve pas son bit, dans la table d'extension — avec exactement le même test. Le PC ne faisant que 20 bits, les quatre bits de poids fort sont ignorés à l'exécution : le drapeau tient donc dans le même mot que l'adresse.

⚠️ **Le « + 3 » n'est pas un descripteur.** Les trois octets qui précèdent une fonction ne décrivent rien : devant `BAS_ASC` on lit `25 03 06` et devant `CHR$` `8B FB 07` — `06H` est `RET`, `07H` est `RETF`. C'est la **queue de la routine précédente**. La règle est uniforme parce que les six tokens qui sont à la fois instruction *et* fonction commencent par un `jp` de 3 octets ; pour une fonction seule, on range simplement `routine − 3` et on fait précéder la routine de trois octets quelconques.

---

## 4. Le contrat d'une routine

| | |
|---|---|
| `X` | **pointeur de programme, à l'entrée ET à la sortie** — il désigne l'octet suivant à traiter. Une routine qui consomme un argument doit le rendre **avancé** d'autant. |
| `I` | à préserver (`pushu`/`popu`) |
| `BP` | voir §5 |
| `U` | à sauver dans une zone à soi et à rendre **avant tout `popu`** : les services ROM allouent leurs chaînes en abaissant `U` |
| retenue | **claire** = pas d'erreur ; **armée** = `A` contient le code d'erreur BASIC |
| fin | **`RETF` obligatoire** |

⚠️ `X` est le piège principal, et il donne un symptôme trompeur : l'oubli ne plante pas. Le nom est reconnu, la routine s'exécute, et l'interpréteur reprend n'importe où — « Syntax error » sans autre indice. Le sauver **après** l'évaluation de l'argument, jamais avant.

---

## 5. Une fonction numérique — le gabarit

Le modèle est `BAS_PEEK` (`F9F44H`), qui fait exactement la même chose sur un octet. Trois différences imposées par la page :

> **Une extension vit en page `0B`, les services en `0E` et `0F`.** Un service terminé par `RET` **ne se laisse pas appeler** d'une autre page : son retour dépile 2 octets et reste dans *sa* page. Il faut donc, pour chaque service, l'entrée qui finit par `RETF`.

`BAS_PEEK` appelle `F5F9CH`, qui finit par `RET`. Mais ce n'est qu'une enveloppe : `callf EFBF3H` puis `call F5C9AH`, et `F5C9A` n'est elle-même que `callf EFAD4H` plus `mv y,(001h)`. Les deux services sous-jacents sont **appelables de partout** :

| Entrée | Rôle |
|---|---|
| `EFBF3H` | évalue **un terme** numérique et vérifie le type — ⚠️ pas une expression : voir §8 |
| `EFAD4H` | *decimal → binaire* : le nombre devient un entier 24 bits en `(bp+1)`..`(bp+3)` |
| `EFB6FH` | *binaire → decimal* : l'entier de `(bp+0)`..`(bp+3)` redevient un nombre BASIC |

`EFAD4H` et `EFB6FH` sont, accessoirement, les commandes `07EH` et `07FH` du **Function Driver** (device 9), documenté en `04-fcs-iocs.md`.

```asm
lpeek:  callf   0EFBF3H         ; l'argument ; BP descend d'un cadre de 15
        jrc     dehors
        callf   0EFAD4H         ; -> entier 24 bits en (bp+1)..(bp+3)
        jrc     dehors
        ...                     ; lire, calculer, DANS LE MÊME CADRE
        pushu   x               ; X sauvé APRÈS l'évaluation
        mv      (BP+0),000H     ; positif, simple précision
        callf   0EFB6FH         ; -> nombre BASIC, rendu à l'appelant
        rc
        popu    x
dehors: retf
```

**Le cadre `BP`.** L'évaluateur descend `BP` de 15 pour y rendre la valeur ; la fonction lit et écrit **dans ce même cadre**, et n'a donc aucun `pmdf` à faire sur le chemin de succès. `BAS_PEEK` en fait deux (`+15` dans `F5C9A`, `−15` ensuite) parce qu'il passe par l'enveloppe : c'est un aller-retour net nul. Sur le chemin d'**erreur**, en revanche, `F5C9A` fait toujours le `+15`, y compris pour son erreur 33 — il faut le reproduire.

⚠️ **`(BP+0)` à l'entrée n'est pas un emplacement libre : c'est le statut BASIC** (bit 2 : exécution programme/manuelle ; bit 4 : mode PRO/RUN). Ce n'est qu'après l'évaluation, dans le cadre de la valeur, qu'il devient l'octet de signe et de précision.

---

## 6. Deux limites mesurées

**`EFB6FH` ne convertit que 20 bits.** Son code fait trois passes de décalage : 8 bits, 8 bits, puis **quatre** seulement sur le troisième octet. `8 + 8 + 4 = 20` — c'est un convertisseur d'**adresse**, et une adresse fait 20 bits sur cette machine. Une valeur de 24 bits perd son quartet haut **sans que rien ne le signale** : `&345678` revient en `&045678`. Une extension qui lit trois octets doit donc refuser plutôt que tronquer (`LPEEK` rend l'erreur 33, *Data out of range*, celle que `F5C9A` emploie pour le même cas).

**`USRWRK` ne fait que 23 octets.** La zone langage machine commence en `BFD1AH`, mais les paramètres SIO commencent en `BFD31H`. Un programme plus long assemblé là écrase la configuration de la liaison série — vitesse, parité, contrôle des lignes, code de fin de fichier — c'est-à-dire, sur un poste qui transfère par série, le canal lui-même. La panne se manifeste au transfert **suivant**. Charger en `BF000H`, l'adresse qu'emploient `PLINK` et `PLINKC`.

---

## 7. Le piège `BASWRK` / `BASPTR`

⛔ Deux adresses ont porté le même nom, et cela a coûté une dizaine d'essais sur machine :

| | Adresse | Ce que c'est |
|---|---|---|
| `BASWRK` | `BFD0EH` | nom repris du listing de E. Kako (`register.lst`, 1990), vérité terrain du corpus. ⚠️ **Il ne désigne pas la zone mais un POINTEUR vers elle** : la ROM fait `mvp (0D1h),[0BFD0EH]` en `0F98CAH`, c'est-à-dire qu'elle y lit trois octets pour en charger `BASPTR`. Les deux sont donc des pointeurs, l'un en mémoire externe, l'autre sa copie en RAM interne. ✅ Corrigé le 2026-09-12 **à la source** — dans `Data/InternalRAMNames.json` et `SystemAddresses.json`, dont `pce500.inc` est **généré** (`xasm2026-4/tools/generate_pce500_inc.py`) : le `.inc` porte en tête « NE PAS EDITER A LA MAIN ». Les sept copies sont réalignées |
| `BASPTR` | `0D1H` | le **pointeur** vers cette zone, en RAM interne — celui qui porte les crochets |

Le générateur de `pce500.inc` écartait *en silence* l'entrée interne dont le nom existait aussi côté système. L'adresse `0D1H` devenait donc **innommable**, et `mv x,(baswrk)` prenait la version externe — que XASM **tronquait à 8 bits** pour un accès interne, sans le moindre avertissement : `0BFD0EH` devenait `0EH`. L'extension installait ses crochets à la mauvaise adresse, et le mot-clé restait un simple nom de variable.

Corrigé depuis à trois niveaux : les carnets ne peuvent plus porter deux fois le même nom (test verrouillé), le générateur **refuse** au lieu d'écarter, et `pce500.inc` expose les deux, chacun renvoyant à l'autre.

> **La leçon générale** : le contrôle se fait dans l'**objet**, jamais dans la source. On doit lire `30 84 D1` — et un `30 84 0E` signale le piège.

---

## 8. Deux évaluateurs d'argument, et ils ne lisent pas la même chose

C'est la distinction la plus coûteuse à ignorer, et elle n'apparaît nulle part dans la
spécification.

| Entrée | Ce qu'elle lit |
|---|---|
| `EFBF3H` (`chknum`) | **un TERME** : un nombre, une variable, un appel de fonction, ou une expression **entre parenthèses** — mais elle **s'arrête au premier opérateur** |
| `EF26EH` (`eval`) | une **EXPRESSION COMPLÈTE**, opérateurs compris |

**La preuve est dans la ROM, et les deux familles y sont côte à côte** — cinq enveloppes
consécutives, qui ne diffèrent que par l'évaluateur appelé puis par la largeur de conversion :

```
0F5F65  callf 0EF26EH   puis test (000h),080h  puis call F5C79H   <- expression
0F5F74  callf 0EF26EH   puis test (000h),080h  puis call F5C86H   <- expression
0F5F83  callf 0EF26EH   puis test (000h),080h  puis call F5C9AH   <- expression
0F5F92  callf 0EFBF3H                          puis call F5C58H   <- terme
0F5F9C  callf 0EFBF3H                          puis call F5C58H   <- terme
```

`POINT (x,y)` emprunte la première (`F5F65H`), `PEEK` la seconde (`F5F9CH`). D'où le
comportement que tout le monde connaît sans l'avoir formulé : **`PEEK &BFD1C*&100` se lit
`(PEEK &BFD1C)*&100`**, parce que `PEEK` ne lit qu'un terme et rend la main avant le `*`.

⛔ **La mesure qui l'a établi.** `MOD` employait `chknum`. Sur machine :

| saisie | réponse |
|---|---|
| `MOD (17,5)` | `2` — juste |
| `MOD (A,B)` | juste |
| `MOD (A+1,B)` | **erreur** |
| `MOD (I*3571,997)` | **erreur** |

`chknum` lisait `A`, puis butait sur le `+` — le code appelant attendait une virgule et
trouvait un opérateur. Le symptôme désigne la cause dès qu'on réunit les quatre lignes : ce
qui échoue, ce sont **les arguments qui sont des expressions**.

⚠️ **`eval` ne vérifie pas le type.** Ses appelants de la ROM font le
`test (000h),080h` **à sa suite** (`0F5F6BH`) et rendent l'erreur eux-mêmes. Une extension
doit faire de même : bit 7 armé = chaîne, erreur **90** *Type mismatch* — le code que `chknum`
emploie pour ce cas (`0EFC04H`).

> ✅ **Le témoin décisif**, ajouté par J.-F. Albouy hors programme d'essai :
> **`MOD (ASC "A",5)` rend `0`**, et c'est juste — `ASC "A"` vaut 65, `65 MOD 5` vaut 0. Un
> **appel de fonction** passe donc en argument. Il ne serait jamais passé avec `chknum`, qui
> se serait arrêté au premier caractère non numérique.

**La règle de conception, en une phrase :** *choisir l'évaluateur d'après la syntaxe que l'on
imite*. Sans parenthèses, à la manière de `PEEK` — `chknum`. Avec une liste parenthésée, à la
manière de `POINT` — `eval`. `LPEEK` et `WPEEK` gardent donc `chknum`, `MOD` prend `eval`.

---

## 9. Plusieurs arguments — la question est close

Le §« Reste ouvert » de la version précédente donnait ce point pour non essayé. Il l'est
désormais : `MOD (A,B)` fonctionne, expressions comprises.

**Le modèle est `SUB_FD5E7`**, le lecteur d'arguments de `POINT (x,y)`, et il exige
**trois** caractères, chacun vérifié à une adresse précise :

| | |
|---|---|
| `(` | `0FD5ECH` |
| `,` | `0FD5F9H` |
| `)` | `0FD60AH` |

⛔ **La parenthèse ouvrante doit être consommée par VOTRE code, pas laissée à l'évaluateur.**
Une première version de `MOD` n'en vérifiait aucune : l'évaluateur recevait `(17,5)` tout
entier, tentait d'évaluer une expression parenthésée, et butait sur la virgule. Réponse de la
machine : *Syntax error*, sans autre indice. La séquence juste est :

```asm
md_sp:  mv    a,[x++]
        cmp   a,020H        ; sauter les espaces
        jrz   md_sp
        cmp   a,028H        ; '(' obligatoire
        jrnz  erreur
        callf 0EF26EH       ; premier argument
        ...
        mv    a,[x++]
        cmp   a,02CH        ; ',' obligatoire
        jrnz  erreur
        callf 0EF26EH       ; second argument
        ...
        mv    a,[x++]
        cmp   a,029H        ; ')' obligatoire
        jrnz  erreur
```

⚠️ Sur `(` manquante, la ROM **remet le caractère en place** avant de se plaindre
(`dec x`, `LOC_FD619`) : l'interpréteur doit pouvoir le relire pour composer son message.

> 📄 **La ROM saute les espaces autrement, et la nuance compte.** `SUB_FD5E7` commence par
> `call SUB_FB2D9` (`0FB2D9H`), qui **regarde** `[x]` sans le consommer, avance tant qu'il lit
> `020H`, puis fait `dec x` : il laisse donc `X` **sur** le premier caractère non-espace, et
> c'est l'appelant qui le consomme ensuite par `mv a,[x++]`. La boucle ci-dessus consomme
> directement — l'effet net est le même ici, puisque le caractère attendu est aussitôt testé,
> mais les deux ne sont pas interchangeables devant un caractère qu'on voudrait **relire**.

---

## 10. Le compte des cadres — établi par la mesure, jamais par le raisonnement

C'est l'endroit où l'on se trompe, et le seul remède est de **lire les octets sur la machine**.
Trois raisonnements successifs s'y sont fourvoyés en septembre 2026 ; les trois mesures qui
suivent ont tranché.

### Ce qui est mesuré

✅ **`chknum` (`EFBF3H`) empile un cadre de 15 octets.** Le module a laissé `BP` en RAM avant
et après son appel ; relevé sur machine : **150 puis 135**.

✅ **`eval` (`EF26EH`) en empile un aussi, et de la même taille.** Même méthode, même séance :
**135 puis 120**. Ce n'était pas supposé — les deux évaluateurs auraient pu différer.

✅ **Une fonction rend son résultat en laissant UN cadre**, celui du résultat, en `(bp+0)`.
C'est ce que fait `LPEEK`, qui ne touche pas à `BP` sur son chemin de succès.

### Le piège que rien n'annonce

⛔ **`EFAD4H` (`dec2bin`) RESTITUE DÉJÀ SON CADRE quand il échoue.** Sa sortie d'erreur
commune, `LOC_ECB5C`, est :

```asm
0ECB5C  sc
0ECB5D  pmdf  (bp_ram),00Fh
0ECB61  ret
```

Après un `dec2bin` en échec, on tient donc **un cadre de moins** qu'on ne le croit. Une
extension qui rendrait « son » cadre en plus décalerait `BP` de 15 et corromprait
l'interpréteur. `LPEEK` avait raison depuis le début de renvoyer directement, sans rien
rendre ; c'est mon raisonnement qui était faux.

En **erreur**, la règle est de rendre tous les cadres tenus — la ROM le fait
(`0ECD63H` rend `+15` avant de signaler l'erreur 21, *Division by zero*).

### La conduite recommandée

⚠️ **Sauver et restaurer `BP` en ABSOLU plutôt que de compter des `pmdf`.** Deux octets de
RAM, deux instructions, et le compte ne peut plus être faux :

```asm
        mv    [!sv_bp0],(bp_ram)   ; à l'entrée -- l'état à rendre en erreur
        ...
        mv    [!sv_bp1],(bp_ram)   ; après la 1re évaluation -- le cadre du résultat
        ...
        mv    (bp_ram),[!sv_bp1]   ; pour rendre le résultat
        mv    (bp_ram),[!sv_bp0]   ; sur tout chemin d'erreur
```

C'est ce que fait `MOD`. Le coût est dérisoire ; le bénéfice est qu'aucune supposition sur la
taille d'un cadre ne peut plus faire de dégât, y compris si un service futur en poussait un
autre.

> 📄 **Une question restée sans réponse, et il faut le dire.** Après la **seconde**
> évaluation, le premier opérande devrait se relire en `(bp+16)`..`(bp+18)` — le cadre faisant
> 15, et l'entier de `dec2bin` étant en `(bp+1)`..`(bp+3)`. `MOD` ne s'y fie pas : il met son
> premier opérande **à l'abri dans sa propre RAM** dès que `dec2bin` l'a converti. Ce choix a
> été fait quand `MOD` rendait `0` et que j'avais conclu, à tort, que `(bp+16)` était en cause
> — la vraie cause était le `ROL` du §11. **`(bp+16)` n'a donc jamais été ni confirmé ni
> infirmé.** La mise à l'abri reste de toute façon la conduite sûre : elle ne dépend d'aucune
> hypothèse.

---

## 11. `SHL`/`SHR` passent par la retenue, `ROL`/`ROR` sont circulaires

⛔ Cette confusion a fait rendre `0` à `MOD` **pour toutes les valeurs**, et elle venait de
notre propre documentation, qui décrivait les deux familles « via C » — ce qui ne peut pas
être vrai des deux.

**La ROM tranche, et dans les deux sens :**

| | |
|---|---|
| elle **chaîne `shr`** sur des octets consécutifs | `F5 12 F5 13 F5 14 F5 15 F5 16` en `0EEBB8H` — un décalage de **cinq** octets. Seul un décalage *à travers* la retenue peut se chaîner ainsi. Sur les 256 Ko, `ROL`/`ROR` ne sont **jamais** chaînés : 4 cas pour `SHR`, **0** pour les rotations |
| elle emploie **`ror`** là où il faut **lire** les bits sans détruire l'octet | `EFB6FH` fait `ror (00Bh)` / `jrnc` **huit fois par octet** (`0EFBA3H`) : huit rotations circulaires le rendent intact |

Les deux usages sont cohérents entre eux, et ils s'excluent. **Pour un décalage multi-octets,
c'est donc `SHL`/`SHR`**, du poids faible vers le poids fort pour un décalage à gauche.

⚠️ Ces décalages **ne consultent pas `I`** : `EFB6FH` enchaîne son `ror` alors que `IL` vaut
encore 4 d'un `dadl` précédent, et n'en fait tourner qu'un seul octet.

✅ Les instructions **bloc**, elles, bouclent **`I` fois et non `I+1`** : `mv il,00Fh` puis
`mvl (000h),(00Fh)` recopie bien les **15** octets d'un cadre.

> `SC62015_InstructionSet_Reference.md` est corrigé depuis, avec les deux preuves ROM.

---

## 12. L'installation doit être idempotente

✅ **Le crochet vierge n'est pas un pointeur, c'est une sentinelle**, et la ROM l'écrit en
`0F93C3H` :

```asm
0F93C3  mv  x,(basptr)
0F93C6  mv  y,0FFFFFh
0F93CA  mv  [x+090h],y
0F93CD  mv  [x+093h],y
```

⛔ **Sans garde-fou, un second `CALL` détruit la sauvegarde.** La première version de `BASEXT`
sauvait sans regarder : au deuxième appel elle écrasait `old_kw` et `old_disp` avec **ses
propres tables**, et les valeurs d'origine étaient perdues — la désinstallation avec. Constaté
sur machine : le vidage montrait `old_kw = 0BF1B1H`, l'adresse de notre propre liste de noms.

Le garde-fou tient en trois instructions :

```asm
        mv    a,[x+092H]    ; poids fort du crochet des noms
        cmp   a,00FH        ; encore la sentinelle 0FFFFFh ?
        jrnz  deja_installe ; non : une extension est déjà là, ne rien écraser
```

**L'argument est celui-ci**, et il faut l'énoncer exactement : l'octet de poids fort d'un
crochet **vierge** vaut `0Fh`, puisque la sentinelle est `0FFFFFh`. Une **extension**, elle,
vit forcément en RAM, page `0B` : son crochet ne peut pas valoir `0Fh`. Le test distingue donc
bien les deux états.

⚠️ Il pourrait être plus strict — comparer les **trois** octets à `0FFFFFh` plutôt que le seul
poids fort. En l'état il accepterait n'importe quel `0Fxxxx`, ce qu'aucune extension ne peut
produire, mais qu'une ROM future pourrait.

⚠️ **Conséquence à retenir :** si le module est installé par-dessus une autre extension,
`old_kw` reste **nul** — et c'est voulu, mieux vaut zéro, qui se voit, qu'une valeur fausse en
silence. **La désinstallation n'est alors pas disponible.** Pour l'avoir, installer sur une
machine dont les crochets sont encore la sentinelle : après un RESET, avant toute autre
extension.

---

## 13. Le device 9 — la route est établie depuis le 2 septembre, et je l'ai ignorée

⛔ **La première version de cette section s'intitulait « l'impasse du device 9 ». C'était faux,
et le dépôt contenait déjà la preuve du contraire.**

### La route qui fonctionne

`SC62015Disassembler/Samples/DEVICE9/SQR.ASM` appelle la bibliothèque mathématique par
l'**IOCS**, et calcule `√2 = 1.414213562`. Il a été mis au point sur machine les 2 et
3 septembre 2026, en **dix étapes** dont plusieurs corrigent une faute d'emploi constatée à
l'essai. La séquence est celle-ci, et il n'y a rien de plus :

```asm
d9:     equ     00009H          ; device 9, famille 0 -- (cl) ET (ch) d'un coup
        ...
        mvw     (cl),d9
        mv      il,079H         ; val
        callf   iocs_call
        mvw     (cl),d9
        mv      il,059H         ; sqr
        callf   iocs_call
        mvw     (cl),d9
        mv      il,078H         ; str$
        callf   iocs_call
```

Le `mvw` écrit `(cl)` **et** `(ch)` en une fois : `09H` dans le premier, `00H` dans le second —
et `(ch)` choisit la famille (`0` numérique et chaîne, `1` matrices, `2` statistiques).
`vogue` procède déjà ainsi, en `0BA788H`. La famille `1` a son contrat propre — `BP` ≥ `0A0h`,
retenue significative, opérandes dans des tableaux BASIC — et elle est mesurée depuis le
2026-09-14 : voir §17.

**La recette canonique est écrite**, dans `SC62015Disassembler/Docs/Routines-ROM-PC-E500S.asm`
(section « APPELER LE DEVICE 9 ») :

```
mvw (cl),00009h    device 9 et famille 0 en une instruction
mv  il,<commande>  041h-05Bh, 070h-079h, 07Eh-07Fh
callf iocs_call    les trois instructions JOINTES

nombre   : operande X en (bp+0)..(bp+14), resultat au meme endroit
chaine   : sur la pile U, longueur en (bp+4) -- et la commande la CONSOMME
resultat chaine : alloue sur U, adresse et longueur rendues dans le moule
retenue  : n'indique RIEN (la sortie d'erreur du guichet fait rc/retf)
```

### Ce que j'avais conclu, et pourquoi c'était faux

J'avais écrit : « **la ROM n'écrit jamais 9 dans `(cl)`** — mon appel était donc faux ».

**L'observation est vraie** : sur les 256 Ko de `rom83`, on ne trouve **aucune** écriture
immédiate de `9` dans `(cl)`. Elle y met `00H`, `01H`, `02H`, `04H`, `05H`, `06H`, `14H`, `15H`
ou `5BH`, jamais `09H`.

⛔ **Mais l'inférence est invalide.** La ROM n'a pas besoin de l'IOCS pour atteindre son propre
driver : l'interpréteur l'appelle **directement**, par la table de répartition de
`drv_function`. L'IOCS est la porte du code **utilisateur** — celle que le manuel documente, et
celle que `SQR` emprunte avec succès. *L'absence d'un idiome dans la ROM ne prouve pas qu'il
soit invalide* ; elle prouve seulement que la ROM n'en a pas l'usage.

### ✅ Le point qui manquait est maintenant MESURE

La même section portait cet avertissement, écrit le 2 septembre :

> ⚠ **RESTE NON VERIFIE : l'operande Y en (bp+15)..(bp+29).** Toute la chaine eprouvee ici
> est unaire. **La premiere operation binaire — « add », « div », une comparaison — le
> tranchera.**

`Samples/DEVICE9/ADDTEST.ASM` est cette opération, et la machine a répondu le **2026-09-06** :

| opérandes | résultat |
|---|---|
| `3` et `2` | **`ADD 5`**   **`SUB 1`** |
| `9` et `4` | **`ADD 13`**   **`SUB 5`** |

✅ **`Y` est bien lu en `(bp+15)`..`(bp+29)`, et le sens est bien « `Y op X -> X` ».** Les deux
essais concordent, et le second departage : `4-9` aurait donné `-5`. Le montage est celui-ci —
deux `VAL`, séparés par une recopie de `X` vers `Y` :

```asm
        callf iocs_call     ; 079h val  -> le 1er operande arrive en X
        mv    i,00015
        mvl   (BP+15),(BP+0) ; X -> Y : quinze octets, la taille d'un operande
        callf iocs_call     ; 079h val  -> le 2nd operande arrive en X
        mvw   (cl),00009h
        mv    il,047H       ; add : Y+X -> X
        callf iocs_call
```

⚠️ **`add` seul ne pouvait PAS trancher le sens** — il est commutatif. C'est `sub` qui le
fait, et c'est pourquoi la sonde enchaîne les deux en refaisant le montage entre les deux :
on ne suppose pas que la commande laisse `Y` intact.

✅ **Et `div` aussi**, mesuré le lendemain par `Samples/DEVICE9/DIVTEST.ASM` :

| opérandes | `04Ah div` | `+ 056h int` | `04Bh pow` |
|---|---|---|---|
| `7` et `2` | **3.5** | **3** | **49** |
| `6` et `3` | **2** | **2** | **216** |

**Les quatre opérations suivent donc la même règle, sans exception** — `add`, `sub`, `div`,
`pow` sont toutes « `Y op X -> X` ». Et l'essai intermédiaire établit le **chaînage** : `int`
travaille bien sur le résultat que `div` vient de laisser en `X`, ce dont un `MOD` flottant a
précisément besoin.

⛔ **J'avais déduit le contraire, et la déduction était DOUBLEMENT fausse.** J'avais écrit que
« l'évaluateur d'expression de la ROM, en `0E8559H`, fait `exl (000h),(00Fh)` avant `div` et pas
avant `sub`, donc `div` prend ses opérandes à l'envers ». La conclusion est démentie par la
mesure — et **la prémisse elle-même était fausse** : *cette routine n'est pas l'évaluateur
d'expression*.

Elle lit son code d'opération dans `[0BFEE3H]`, et cet emplacement n'est écrit que par **deux**
endroits de la ROM — `0E7CEBH` et `0E8F76H`. Or ce sont exactement les deux branches de famille
du répartiteur du device 9 :

```asm
0EF03D  jpz  LOC_E7CE9    ; (ch)=1 -> MATRICES
0EF040  jpnc LOC_E8F74    ; (ch)=2 -> STATISTIQUES
```

La routine de `0E8530H` appartient donc à la famille **matrices** ou **statistiques**, où les
opérandes sont disposés autrement — pas au chemin numérique ordinaire `(ch)=0`. Son `exl` a un
sens local ; il n'en avait aucun pour un appel simple.

> **La leçon :** j'avais **nommé** cette routine « l'évaluateur d'expression » sans l'établir,
> puis raisonné sur ce nom. *Une étiquette posée de soi-même n'est pas une source* — c'est la
> même faute que le `--code BEFBC-BF039` de `register`, où la doctrine avait été écrite d'après
> notre propre sortie au lieu du listing XASM.

### Ce qui a réellement fait planter ma première tentative n'est pas établi

Le symptôme — `CLS` puis MENU principal — n'a jamais été instruit, puisque j'avais cru tenir la
cause. L'en-tête de `SQR.ASM` liste **trois fautes d'emploi** trouvées à l'essai machine, et la
première est un candidat sérieux :

> ⛔ **Sous `pre_on`, `(000H)` est une adresse DIRECTE, pas `(BP+0)`.** Les opérandes partaient
> en RAM interne 0-4 au lieu de `(bp+0)`-`(bp+4)`, et le Function Driver ne les voyait jamais.
> On écrit donc `(BP+n)` **explicitement**.

Les deux autres valent d'être connues : ne pas relire trois octets là où l'on en a écrit deux,
et **ne rien intercaler entre deux commandes** — un appel FCS peut se servir de la zone
`(bp+n)` pour son propre compte et détruire l'opérande que la commande suivante doit lire.

⚠️ Et une quatrième, propre au guichet : **la retenue n'est pas un indicateur d'erreur ici**.
La sortie d'erreur du répartiteur (`0EF076H`) fait `rc` / `retf`, donc carry **clair**. Un `jrc`
après l'appel serait inerte ; c'est le **résultat** qui dit si la commande a travaillé.

### Ce qui reste vrai de mes mesures

La **seconde** tentative n'employait pas l'IOCS : elle sautait directement aux entrées de la
table de répartition, relevées dans `DB_EF078` (entrées de 2 octets, index = commande − `041H`) :

| commande | entrée | |
|---|---|---|
| `047H` | `0ECBA9H` | `add` |
| `048H` | `0ECBB4H` | `sub` |
| `049H` | `0ECBBFH` | `mul` |
| `04AH` | `0ECBD8H` | `div` |
| `056H` | `0EE4D5H` | `int` |

Ces cinq adresses sont exactes, et chacune a bien la forme `call <travail>` puis `retf`. **Mais
c'est une route différente de l'IOCS, et son contrat n'est pas établi** : la ROM a deux
diviseurs, `SUB_ECCEA` et `SUB_ECD85`, l'entrée `04AH` ne mène qu'au second, et le seul
appelant connu — l'évaluateur d'expression, en `0E8559H` — écrit :

```asm
mv   i,0000Fh
exl  (000h),(00Fh)     ; il ÉCHANGE X et Y
call SUB_ECBDC         ; ... puis seulement divise
```

Il échange les opérandes avant de diviser, ce que ne fait aucune des trois autres opérations.
**Pour court-circuiter l'IOCS, il faudrait donc établir ce contrat ; pour passer par l'IOCS,
il n'y a rien à établir — `SQR` l'a fait.**

### La leçon, et c'est la plus utile du document

Deux fois dans la même campagne, j'ai reconstruit — mal — ce que le dépôt établissait déjà :
le device 9 ici, et le cadre de 15 octets que le §5 donnait depuis le début. **Avant d'ouvrir
un chantier, lire ce qui a été fait.** `git log` sur le dossier concerné, et les en-têtes des
sources voisines : `SQR.ASM` porte trente lignes de commentaire qui auraient épargné deux
plantages et une conclusion fausse.

> ⚠️ **Conséquence pour `MOD` :** il calcule en **entiers de 20 bits** parce que je le croyais
> coupé de la bibliothèque mathématique. Il ne l'est pas. Le refaire sur le device 9 lui
> donnerait le **domaine complet du BASIC** — voir la piste 12 du §16.

---

## 14. Les pièges d'usage — ils coûtent plus cher que le code

Aucun n'est un défaut du mécanisme, et tous ont coûté au moins une séance.

### Le BASIC tokenise À LA SAISIE, pas à l'exécution

⛔ Si le texte d'un programme entre dans la machine **avant** le `CALL` d'installation, un
mot-clé d'extension est tokenisé comme un **nom de variable** — et il le reste. Installer
ensuite n'y change rien : la ligne est figée. `MOD (17,5)` devient une référence de tableau, et
la machine répond `Array specified without DIM`, ce qui ne désigne évidemment pas la cause.

L'ordre est donc imposé : **charger le module, installer, vérifier, puis seulement charger le
programme.** Cela vaut pour tout texte **ré-importé** — un `.BAS` ressorti de l'émulateur est
retokenisé. Un `.BSA` déjà tokenisé, lui, garde son jeton.

### Le contrôle d'installation, en deux lignes

Il est sourcé : la ROM fait `mvp (0D1h),[0BFD0EH]` en `0F98CAH`, c'est-à-dire qu'elle recopie
dans `BASPTR` les trois octets rangés en `BFD0EH`. Le crochet est donc lisible depuis le BASIC :

```basic
W=LPEEK &BFD0E
PRINT HEX$ LPEEK (W+&90)      ' l'adresse de votre liste de noms
```

> 📄 Cela corrige au passage la description de `BASWRK` : `BFD0EH` **contient** le pointeur, il
> n'est pas la zone elle-même. Voir §7.

### La zone langage machine doit être RÉSERVÉE

Un module chargé en `BF000H` n'est **pas** protégé par défaut : le BASIC a le droit d'y écrire.
Constaté sur machine — après avoir tapé une ligne, on lisait en `0BF1E7H`, un octet après la
fin du module, `4D 59 41 44 52 20 3D 20 FE A4` = **`MYADR = PEEK`**, du texte de programme
tokenisé.

La zone s'étend de **`[BFD1AH]`** (le pointeur `USRWRK`) jusqu'à **`BFC00H`**, où commence la
System Data Area. `A_AREA.BAS` (J.-F. Albouy, racine de `C:\Claude`) la lit et la fixe :

```basic
POKE &BFE03,&1A,&FD,&B,TL,TM,TH : CALL &FFFD8
'          └ l'adresse DU POINTEUR   └ la taille voulue
```

⛔ **`BFC00H` est un PLAFOND, pas une base.** Une réservation demandée avec un début à
`&BFE00` donne une taille de `&BFC00 - &BFE00` = **−512** : la zone ne couvre rien, et tout ce
qui est en dessous reste au BASIC. Pour protéger un module en `BF000H`, il faut donc **3072
octets** (`&BFC00 - &BF000`). Le menu doit ensuite afficher `[BF000 - BFC00] -> 3072`.

✅ **Un auteur tiers l'écrivait déjà en 1992.** La notice de VOGUE (Narihito Kon, `VOGUE.DOC`)
réserve la zone de son compilateur, chargé en `$B9800`, par la même porte et le même pointeur :

```basic
poke &bfe03,&1a,&fd,&b,0,&64,0
call &fffd8
```

La taille demandée vaut `&006400` = 25 600 octets, et **`&BFC00 − &6400 = &B9800`** : c'est
exactement la règle du plafond, retrouvée ici par une source indépendante de la mesure.

⚠️ `CALL &FFFD8` provoque un petit reset : l'extension est à réinstaller ensuite.

### Le format des programmes d'essai

Deux règles, toutes deux payées :

- ⛔ **`CRLF` obligatoire.** Un `.BAS` en `LF` seul fait répondre **`Line buffer overflow`** :
  sans les `CR`, l'import ne voit qu'**une seule ligne** de plus de mille caractères.
- ⛔ **MAJUSCULES ASCII.** Les minuscules sont **mangées** à l'import :
  `'A lancer APRES avoir installe BASEXT` ressort `'A  APRES   BASEXT`. Mots-clés et variables
  survivent, les commentaires deviennent illisibles.

---

## 15. Méthode — comment mettre au point une extension

Le fond de ce document est là. Sur cinq défauts successifs de `MOD`, **le raisonnement s'est
trompé trois fois** et la mesure a tranché à chaque fois.

**Poser des témoins dans la RAM du module.** Trois mots de 3 octets, écrits par un
`mvp [!dbg_x],(BP+n)` aux points clés — les arguments tels que la conversion les rend, le
résultat avant sa remise. Ils ne coûtent que 5 octets chacun, et ils sont **relisibles depuis
le BASIC** :

```basic
PRINT HEX$ LPEEK &BF216       ' l'argument, tel que dec2bin l'a converti
```

**Vider la mémoire dans un fichier.** C'est ce qui a débloqué deux fois. Un `PEEK` en boucle
sur la plage du module, écrit dans `F:RESULT.BAS`, donne l'image exacte à comparer à l'objet —
et distingue les octets écrits à l'exécution de ceux du code.

⚠️ **Comparer à la BONNE version.** Un vidage comparé à l'image d'une autre version fait
conclure n'importe quoi : les adresses bougent à chaque assemblage. Relire `kw_table` dans le
`.lst`, et faire vérifier au programme d'essai qu'il parle bien au module qu'il croit.

> **Trois symptômes et leur lecture**, tous rencontrés :
>
> | symptôme | ce qu'il désigne |
> |---|---|
> | `0` pour **toutes** les valeurs | un opérande perdu, ou un calcul qui ne calcule pas — jamais un cas particulier |
> | le cas **simple** échoue aussi | la cause n'est pas dans ce qui distingue les cas compliqués |
> | un témoin **refusé** par `LPEEK` (erreur 33) | la valeur n'est pas un entier 20 bits : ce n'est pas un nombre converti, mais des octets bruts |

---

## 16. Pistes à vérifier

Aucune n'est bloquante ; toutes sont à portée d'une séance de mesure.

### Sur le mécanisme

1. **`(bp+16)` porte-t-il le premier opérande après la seconde évaluation ?** Les deux cadres
   faisant 15, il le devrait. Jamais confirmé ni infirmé (§10). Un témoin suffirait à trancher,
   et une réponse positive économiserait la mise à l'abri en RAM.
2. **La désinstallation.** Jamais essayée. Restaurer les deux crochets depuis `old_kw` et
   `old_disp` — donc sur une machine où ils portent la sentinelle `0FFFFFh` (§12). Vérifier
   qu'un mot-clé retiré redevient bien un nom de variable, et ce que devient un programme déjà
   tokenisé avec ce jeton.
3. **Un mot-clé à la fois instruction ET fonction.** Six tokens de la ROM le sont, et leur
   troisième octet d'adresse porte le quartet `11` — les deux bits armés. Ils commencent tous
   par un `jp` de 3 octets, ce qui donne une seconde entrée à `+3`. Reproductible pour une
   extension ?
4. **Le chaînage de deux modules.** Il n'existe pas : un second module écrase le premier. Mais
   rien n'interdirait à un module de **balayer d'abord la liste qu'il a sauvée** avant la
   sienne. À éprouver — c'est ce qui permettrait de composer plusieurs extensions.
5. **Le RESET.** `F93C3H` réécrit la sentinelle dans les deux crochets. Est-elle appelée à
   chaque RESET, ou seulement au démarrage à froid ? La réponse dit si une extension survit à
   un RESET (le code reste en RAM ; seuls les crochets sont perdus).

### Sur les arguments et les types

6. **Trois arguments ou plus.** `MOD` en prend deux, sur le modèle de `POINT`. `MID$` en prend
   trois : lire son lecteur d'arguments donnerait le gabarit.
7. **Retourner une CHAÎNE.** Déjà noté comme ouvert. Les services de chaîne du device 9 allouent
   sur la pile `U` et consomment leur argument : le contrat de `U` (§4) change, et il faudra
   l'établir avant d'écrire une ligne.
8. **Accepter les deux types.** `eval` rend le bit 7 de `(bp+0)` armé pour une chaîne. Une
   extension pourrait s'en servir pour offrir deux comportements sous un même nom, comme la ROM
   le fait pour ses six tokens doubles.
9. **Le domaine au-delà de 20 bits.** `EFAD4H` refuse au-delà (`add x,a` / `jrc` en `0EFAFBH`),
   `X` étant un registre de 20 bits. Une extension qui voudrait 24 bits devrait travailler sur
   les nombres BASIC eux-mêmes — donc résoudre le contrat du device 9 (§13), ou faire sa propre
   arithmétique décimale.

### Sur l'outillage

10. **Les 88 tokens libres.** La table de la ROM en laisse 88 inoccupés, mais rien ne garantit
    qu'une révision de ROM n'en emploie aucun. Vérifiable en confrontant les tables lues dans
    `rom83`, `rom75` et `rom53`.
11. ✅ **~~La position du second opérande, et le sens de `div`~~** — **mesurés le 2026-09-06**
    par `ADDTEST.ASM` et `DIVTEST.ASM` : `Y` en `(bp+15)`..`(bp+29)`, et `add`, `sub`, `div`,
    `pow` toutes « `Y op X -> X` ». Le chaînage de deux commandes est établi lui aussi.
12. **Refaire `MOD` sur le device 9** — la voie est désormais **entièrement mesurée**. Il calcule aujourd'hui en entiers de 20 bits parce que
    je le croyais coupé de la bibliothèque mathématique (§13). La route IOCS étant établie, un
    `MOD` écrit dessus travaillerait sur les **nombres BASIC** eux-mêmes — domaine complet,
    décimales comprises. `A - B * INT (A/B)` demande quatre commandes : `04AH` div, `056H` int,
    `049H` mul, `048H` sub — les quatre sont mesurées « `Y op X -> X` », et le chaînage aussi.
    ⚠️ Reste à trancher le **domaine** : dix chiffres significatifs, et la troncature du
    dixième n'est établie que sur un échantillon. Un `MOD` flottant ne serait donc pas
    *strictement* meilleur que l'actuel, exact par construction : les faire **coexister** reste
    le choix raisonnable.

> ⛔ **La division par zéro déplace `BP` de +30, en silence** — éprouvé le 2026-09-06 avec
> `op1 = 7`, `op2 = 0` : la machine a rendu `DIV 0`, `INT -1`, `POW 1`. Seul `POW 1` est un
> résultat (`7⁰ = 1`, le montage fonctionne même avec `X = 0`) ; les deux autres sont la
> signature d'un échec muet.
>
> Le chemin se lit dans la ROM : `SUB_ECD85` teste `(003h),0F0h`, saute en `LOC_ECD63` qui fait
> `pmdf +0Fh`, puis en `LOC_ECB5C` qui refait `pmdf +0Fh`. **+30 sur `BP`** — et le code
> d'erreur 21 que la ROM avait posé dans `A` est perdu en chemin, la sortie du guichet
> (`0EF076H`) faisant `rc / retf`.
>
> **Et cela s'accumule.** Depuis `BP = 070h` : un échec → `08Eh`, deux → `0ACh`, trois →
> `0CAh` — et l'opérande `Y` occuperait alors `0D9h`–`0E7h`, **par-dessus `si`, `di` et `bp`
> lui-même**. La sonde n'a survécu à ses deux échecs que parce qu'elle **restaure `BP` en
> absolu** à la sortie.
>
> ✅ **La règle qui en découle, et elle vaut pour toute extension : qui appelle `div` doit
> tester son diviseur LUI-MÊME, avant l'appel.** C'est ce que fait déjà le `MOD` entier de
> `Samples/BASEXT`, qui rend l'erreur 21 sur `B = 0` sans jamais laisser la ROM s'en charger —
> choix qui se trouve validé après coup.
>
> ⚠️ Cette lecture explique les trois affichages mais reste une **inférence** : `BP` n'a pas
> été relevé. Ce qui la soutient est que `INT` aurait dû rendre `0` — `int(0)`, `div` ayant
> laissé `X` inchangé — et rend `-1`. Pour la **mesurer**, ranger `(bp_ram)` dans une variable
> du module après chaque essai et la lire au `PEEK`.
13. **Un `.BSA` d'essai plutôt qu'un `.BAS`.** Un programme livré déjà tokenisé échapperait au
    piège du §14 — le jeton d'extension y est figé. `Sharp Basic Converter` sait le produire ;
    reste à vérifier qu'il accepte un token hors des 168 de la ROM.

### Sur les matrices (§17) — chantier suspendu le 2026-09-14, à reprendre

14. **Les matrices depuis le BASIC pur, sans code machine.** `POKE &BFE00,9,1,&4B : CALL &FFFDC`.
    `BP` vaut `0BEh` au moment d'un `CALL` (mesuré), donc le contrôle `BP ≥ 0A0h` passerait.
    Mais le calcul écrit alors `080h`-`09Fh` **sans les sauver**. Ces octets sont sous le `BP`
    du BASIC, donc libres par discipline de pile — c'est une **inférence**. Sauvegarder son
    travail avant l'essai. Une réponse positive ouvrirait les 29 matrices à tout programme BASIC.
15. **MA-MZ et les opérations binaires.** `52H`/`53H` avec la lettre en `[BFE03H]` ; `41H`,
    `42H`, `43H` sur X et Y ; et ce que rend une **erreur de dimension** (le mode MATRIX affiche
    « IMPOSSIBLE CALCULATION » — quel code dans `A` ?).
16. **La commande `56H`.** Présente dans la table de répartition, classe « matrice et scalaire »,
    mais **dans aucun des deux manuels**. Le manuel utilisateur cite cinq opérations scalaires
    (k·X, k·X⁻¹, X/k, k+X, k−X), le manuel technique n'en numérote que quatre : `56H` = X/k est
    l'hypothèse naturelle, à mesurer avec `X` = 25 et un tableau connu.
17. **Les statistiques par le tableau `MD`.** Le manuel allemand (livre p. 138) dit que la
    matrice `MD` partage son tableau avec les données d'échantillon des statistiques. Ce serait
    la réponse à la question laissée ouverte sur la famille `(ch)=2` : comment ses données sont
    fournies. À vérifier dans le code de `0E8F74h` — y chercher le nom `M`,`D` fabriqué comme
    en `SUB_E813E` — avant toute sonde.

---

## 17. Les matrices — device 9, famille `(ch)=1`

Ouvert le 2026-09-14 et **suspendu volontairement après une première mesure** : le contrat est
lu sur trois sources indépendantes, deux commandes sont éprouvées, et le reste est inventorié
pour la reprise (pistes 14 à 17 du §16).

### Ce que sont les matrices : des tableaux BASIC

| Fait | Le code (`rom83`) | Les manuels |
|---|---|---|
| X, Y, M et MA-MZ **sont** les tableaux BASIC `X(*,*)`, `Y(*,*)`, `M(*,*)`, `MA(*,*)`-`MZ(*,*)`, en simple précision | `SUB_E813E` fabrique le nom — lettre(s), `21h` `!`, `28h` `(` — et le fait chercher dans la table des variables par `SUB_FACEB` | utilisateur allemand, livre p. 147 : « *im gleichen Bereich wie die Feldvariablen X (\*,\*), Y (\*,\*), M (\*,\*) und MA (\*,\*) - MZ (\*,\*)* » |
| l'élément X(i,k) est le BASIC `X(i-1,k-1)` | — | idem |
| le **scalaire** k des commandes `46H`-`49H` est **lu** dans la **variable simple** `X` | `0E7DB6h` : `mv a,058h` / `callf SUB_FB334`, qui calcule l'adresse d'une variable fixe : `(datbas)` + `[+32h]` + 5 + (lettre − `41h`) × 12 | technique p. 81 : « *CALL after entering scholar value in X* » |
| le **déterminant** (`4CH`) est **écrit** dans la variable `X` | `0E7D2Ch`, chemin propre à cette seule commande : `call SUB_E3235` / `mv a,058h` / `callf SUB_FB334` / `callf SUB_FD65F` | technique p. 81 : « *Answer enters in x* » |
| la lettre A-Z de `52H`/`53H` se pose en `[BFE03H]` | `0E7D9Dh` n'accepte que `41h`-`5Ah`, et la range en `(0B9h)` | technique p. 82, colonne « A-Z » |
| le mode MATRIX de la ROM **ne passe pas** par l'IOCS | `0E135Dh` saute directement dans les classes d'après `[0BFEE3h]` | — |

### Les six classes, lues dans la table de répartition

L'entrée `0E7CE9h` range `IL − 41h` en `[0BFEE3h]`, puis une table d'octets (`DB_E7D87`) donne la
classe de chacune des 22 commandes `41H`-`56H` :

| Classe | Nature | Commandes |
|---|---|---|
| `31h` | sur X seule | `45H` inverse · `4BH` transposée · `4CH` déterminant · `4DH` changement de signe · `4EH` carré |
| `32h` | X avec Y ou M | `41H` X+Y · `42H` X−Y · `43H` X·Y · `44H` X·Y⁻¹ (« Division ») · `51H` X+M→M · `54H` système d'équations · `55H` résidu |
| `53h` | X avec le scalaire k | `46H` k+X · `47H` k−X · `48H` k·X · `49H` k·X⁻¹ · `56H` *(non documentée)* |
| `54h` | X ↔ M | `4FH` X→M · `50H` M→X |
| `52h` | X ↔ MA-MZ | `52H` X→MA-MZ · `53H` MA-MZ→X |
| `58h` | X ↔ Y | `4AH` échange |

*Recoupement : les numéros et les libellés viennent du manuel technique p. 81-82, le sens exact des
opérations du manuel utilisateur p. 142-144, la classe du code.*

### Le contrat d'appel

```asm
        mv      x,sauve
        mv      i,00040H
        mvl     [x++],(080H)    ; SAUVER 080h-0BFh : le calcul y travaille
        mv      (bp_ram),0A0H   ; au moins 0A0h, sinon erreur 60
        mvw     (cl),00109H     ; device 9 ET famille 1, en une instruction
        mv      il,04BH         ; transposee
        callf   iocs_call
        ; retenue ARMEE = erreur, et A porte alors un code BASIC
        ; ... puis rendre 080h-0BFh, U et BP
```

| Règle | Pourquoi |
|---|---|
| **`BP` ≥ `0A0h`** | `0E7CEFh` : `cmp (bp_ram),0A0h` → erreur 60 |
| **sauver `080h`-`0BFh` soi-même** | le calcul pose `BP` = `080h`, emploie `080h`-`0BFh`, et ne sauve que `0A0h`-`0BFh` (en `BFFD8H`, et `BP` en `BFFD0H`). À l'entrée d'une fonction, `BP` vaut `096h` : le cadre vivant du BASIC tombe dans la zone non sauvée |
| **la retenue fait foi** | la queue d'erreur (`0E7D46h`-`0E7D6Bh`) traduit les codes internes en codes BASIC 60, 21, 20 ou 22, puis `sc` / `retf` — l'inverse de la famille 0, dont le guichet fait `rc` / `retf` |
| **`A` ne se lit que si la retenue est armée** | la sortie de succès ne pose pas `A` : c'est un reste du calcul |

### ✅ Mesuré — `Samples/DEVICE9/MATTEST.ASM`, 2026-09-14

La sonde relève retenue, `A` et `BP` à des **adresses fixes** en tête du code (`&BF002`-`&BF005`) ;
`MATTEST.BAS` fournit les tableaux.

| Essai | Affiché | Ce que cela établit |
|---|---|---|
| transposée `4BH` de `DIM X(1,2)` = 1 2 3 / 4 5 6 | `CY 0  A 0  BP 190` puis `T 4  6` | la famille 1 répond par l'IOCS ; `DIM X(1,2)` est bien le tableau que la ROM cherche sous le nom `X!(` ; `X(0,1)` = 4 et `X(2,1)` = 6 existent : le tableau a été **recréé en 3×2** |
| déterminant `4CH` de 2 1 / 1 3 | `CY 0  A 20` puis `DET 5` | le résultat va dans la variable simple `X`, **qui coexiste** avec le tableau `X(,)` ; `ERASE` puis `DIM` redonnent un tableau que la ROM retrouve |

⚠️ **« `A 20` » avec `CY 0` n'est pas une erreur** — c'est la règle du tableau précédent, vérifiée
sur la machine.

✅ **`BP` vaut 190 (`0BEh`) au moment d'un `CALL`**, contre 150 (`096h`) à l'entrée de `MOD` : une
fonction est appelée **pendant** l'évaluation d'une expression, une instruction `CALL` non.

### Ce qu'il faut savoir avant d'écrire une extension matricielle

- **`RUN`, `CLEAR`, `NEW`, `ARUN` et `ERASE` effacent les matrices** (manuel utilisateur p. 147) :
  ce sont des variables. Un programme qui les emploie doit les dimensionner **après** son `RUN`,
  et le manuel imprime ses matrices par `GOTO`, jamais par `RUN`.
- **La mémoire** : (lignes × colonnes × 7 + 10) octets par matrice, et le calcul en crée une
  **intermédiaire** — deux pour l'inverse et les systèmes (manuel utilisateur p. 147).
- **Simple précision seulement**, et des résultats approchés sur une matrice presque singulière :
  le manuel en donne un exemple, et recommande `RESID` pour évaluer l'erreur d'un système.
- **`MD` est aussi le tableau des données statistiques** (p. 138) — piste 17.

---

## 18. Sources

- **`Nx commandes BASIC.docx`** — SynologyDrive, `Sharp PC E500S\01- Manuels et Guides\01- Basic\`. Traduction française d'un document allemand. **Dix-sept lignes qui spécifient le mécanisme** : les deux crochets, le format des deux listes, les bits 7 et 6, l'obligation de `RETF`, la retenue comme statut d'erreur, `(BP+0)` comme statut BASIC, et `X` comme pointeur de programme en entrée/sortie. C'est la source de référence ; tout le reste de ce document la complète par la lecture de la ROM et la mesure.
- **`PC-E500 systemhandbuch.pdf`** et sa traduction `Systemhandbuch_PC-E500_traduction_FR.docx` (`Sharp Basic Converter/Documentation/`) — le préfixe `0FEH` des tokens, et une troisième confirmation de la doctrine PRE (*« Die übliche Adressierungsart ist (BP+n), sie erfordert keinen [PRE] »*).
- **`BASCOM`** (TORO, 1994) — `SC62015Disassembler/Samples/BASCOM/`. Seul exemple du corpus, et seul témoin de l'installation : douze mots-clés, mais **trois routines seulement**, toutes des instructions.
- **Rétro-ingénierie de `rom83.bin`** — les résolveurs `F58E5H` et `F590BH`, la recherche `F593DH`, le tokeniseur `F4413H`/`F448BH`, le gabarit `BAS_PEEK` (`F9F44H`) et les trois services de conversion. Détail dans `SC62015Disassembler/Docs/Routines-ROM-PC-E500S.asm`.
- **Mesures sur PC-E500S et PockEmul**, septembre 2026 — la contrainte de nommage, la limite des 20 bits, le contrat de `X`, et la validation de `LPEEK` et `WPEEK`.
- **`SC62015Disassembler/Samples/DEVICE9/`** — les sondes du Function Driver : `SQR`, `ADDTEST`, `DIVTEST` (famille 0, §13) et `MATTEST` (famille 1, §17). Chaque source consigne en en-tête le résultat observé et sa lecture.
- **Manuels Sharp, pour les matrices** — technique, livre pp. 79-82 (PDF pp. 83-86) : les tables des trois familles ; utilisateur allemand `PC-E500S-DE.pdf`, livre pp. 138-148 (PDF pp. 146-156) : le mode MATRIX, le rangement dans les tableaux BASIC, la mémoire et les erreurs.
- **`SC62015Disassembler/Samples/BASEXT/`** — le module qui a servi à établir les §§8 à 16 : quatre mots-clés (`LPEEK`, `WPEEK`, `LPOKE`, `MOD`), 530 octets, validés sur machine le 2026-09-05. Son `README.md` porte le journal des cinq défauts successifs de `MOD` et de ce que chacun a coûté.

---

## Statut

Établi et validé sur machine. Le mécanisme est éprouvé pour les **instructions** et pour les
**fonctions**, en direct et en programme, avec un argument (`LPEEK`, `WPEEK`), une liste
d'arguments (`LPOKE`) et **deux arguments parenthésés** (`MOD`).

`MOD (A,B)` a été validé le 2026-09-05 : arguments littéraux, variables, **expressions**
(`MOD (A+1,B)`, `MOD (I*3571,997)`) et **appels de fonction** (`MOD (ASC "A",5)`) ; bornes du
domaine ; contrôle croisé par le BASIC lui-même sur **40 valeurs sans un écart** ; et les cinq
refus attendus — erreurs 21, 33, 33, 10 et 90.

**Ce que cette campagne a ajouté au document** : les deux évaluateurs (§8), le passage de
plusieurs arguments (§9, qui était le premier point resté ouvert), le compte des cadres mesuré
(§10), la distinction `SHL`/`ROL` (§11), l'idempotence de l'installation (§12), la route IOCS du device 9 (§13), les pièges d'usage (§14) et la méthode de mise au point (§15).

**Le 2026-09-14** : la famille **matrices** du device 9 (§17) — contrat lu sur le code et les deux
manuels, transposée et déterminant mesurés par `MATTEST`, puis chantier suspendu à la demande.

**Reste ouvert** : le retour d'une **chaîne** plutôt que d'un nombre, et les quinze autres pistes
du §16, dont quatre sur les matrices et les statistiques (14 à 17).

⛔ **Une correction du 2026-09-06, et c'est la plus instructive.** La première rédaction du
§13 concluait à une « impasse du device 9 ». Elle était fausse : `Samples/DEVICE9/SQR.ASM`
appelait déjà la bibliothèque mathématique par l'IOCS, et le faisait depuis le 2 septembre,
après dix étapes de mise au point sur machine. J'avais reconstruit — mal — ce que le dépôt
établissait déjà, exactement comme pour le cadre de 15 octets que le §5 donnait depuis
toujours. **Lire ce qui a été fait avant d'ouvrir un chantier** est la première règle, et elle
n'a pas été respectée deux fois dans la même campagne.
