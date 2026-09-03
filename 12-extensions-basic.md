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

L'installation tient en dix instructions, et sauvegarde les anciennes valeurs pour permettre une désinstallation :

```asm
        mv    x,(basptr)          ; 0D1H
        mv    y,[x+090H]          ; l'ancienne liste de noms
        mv    [old_kw],y          ;   sauvée
        mv    y,[x+093H]          ; l'ancienne liste d'adresses
        mv    [old_disp],y
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
| `EFBF3H` | évalue l'expression numérique et vérifie le type |
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
| `BASWRK` | `BFD0EH` | la **zone** de travail, en mémoire externe — nom repris du listing de E. Kako (`register.lst`, 1990), vérité terrain du corpus |
| `BASPTR` | `0D1H` | le **pointeur** vers cette zone, en RAM interne — celui qui porte les crochets |

Le générateur de `pce500.inc` écartait *en silence* l'entrée interne dont le nom existait aussi côté système. L'adresse `0D1H` devenait donc **innommable**, et `mv x,(baswrk)` prenait la version externe — que XASM **tronquait à 8 bits** pour un accès interne, sans le moindre avertissement : `0BFD0EH` devenait `0EH`. L'extension installait ses crochets à la mauvaise adresse, et le mot-clé restait un simple nom de variable.

Corrigé depuis à trois niveaux : les carnets ne peuvent plus porter deux fois le même nom (test verrouillé), le générateur **refuse** au lieu d'écarter, et `pce500.inc` expose les deux, chacun renvoyant à l'autre.

> **La leçon générale** : le contrôle se fait dans l'**objet**, jamais dans la source. On doit lire `30 84 D1` — et un `30 84 0E` signale le piège.

---

## 8. Sources

- **`Nx commandes BASIC.docx`** — SynologyDrive, `Sharp PC E500S\01- Manuels et Guides\01- Basic\`. Traduction française d'un document allemand. **Dix-sept lignes qui spécifient le mécanisme** : les deux crochets, le format des deux listes, les bits 7 et 6, l'obligation de `RETF`, la retenue comme statut d'erreur, `(BP+0)` comme statut BASIC, et `X` comme pointeur de programme en entrée/sortie. C'est la source de référence ; tout le reste de ce document la complète par la lecture de la ROM et la mesure.
- **`PC-E500 systemhandbuch.pdf`** et sa traduction `Systemhandbuch_PC-E500_traduction_FR.docx` (`Sharp Basic Converter/Documentation/`) — le préfixe `0FEH` des tokens, et une troisième confirmation de la doctrine PRE (*« Die übliche Adressierungsart ist (BP+n), sie erfordert keinen [PRE] »*).
- **`BASCOM`** (TORO, 1994) — `SC62015Disassembler/Samples/BASCOM/`. Seul exemple du corpus, et seul témoin de l'installation : douze mots-clés, mais **trois routines seulement**, toutes des instructions.
- **Rétro-ingénierie de `rom83.bin`** — les résolveurs `F58E5H` et `F590BH`, la recherche `F593DH`, le tokeniseur `F4413H`/`F448BH`, le gabarit `BAS_PEEK` (`F9F44H`) et les trois services de conversion. Détail dans `SC62015Disassembler/Docs/Routines-ROM-PC-E500S.asm`.
- **Mesures sur PC-E500S et PockEmul**, septembre 2026 — la contrainte de nommage, la limite des 20 bits, le contrat de `X`, et la validation finale (`LPEEK`, `WPEEK`, en ligne de commande et en programme, avec argument littéral puis variable).

---

## Statut

Établi et validé sur machine en septembre 2026. Les deux gabarits assemblent et fonctionnent ; le mécanisme est éprouvé pour les instructions **et** les fonctions, en direct et en programme.

**Reste ouvert** : le passage de plusieurs arguments à une fonction d'extension (non essayé), et le retour d'une **chaîne** plutôt que d'un nombre — les services de chaîne du device 9 allouent sur la pile `U` et consomment leur argument, ce qui change le contrat.
