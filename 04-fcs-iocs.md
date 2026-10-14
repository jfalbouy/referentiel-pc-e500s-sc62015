# FCS et IOCS — appels système du PC-E500S

*Rédigé le 2026-08-26 — mis à jour le 2026-10-14*

> Voir `00-index.md` pour la vue d'ensemble. Source principale : *Technical Reference Manual PC-E500* (chapitres 1 « File Control System », 2 « Outline of IOCS », 3 « How to use each device »), recoupé avec `SC62015Disassembler/Data/FCSFunctions.json`. Le manuel documente 17 fonctions FCS (`00H`-`10H`). Lors de la première rédaction de ce fichier (juillet 2026), le JSON du désassembleur n'en couvrait que 9 et ce fichier comblait les 8 autres ; **le carnet les porte désormais toutes les 17**, avec leurs registres d'entrée et de sortie, ainsi que les commandes IOCS communes, celles du device 0 et les 39 du device 9 (`SC62015Disassembler/CLAUDE.md` §8).

Le PC-E500S offre trois niveaux d'entrée/sortie, du plus haut niveau (portable, simple) au plus bas (rapide, dépendant du matériel) :

| Niveau | Nom | Description |
|---|---|---|
| 2 | **FCS** (File Control System) | Manipulation de fichiers/périphériques comme des fichiers : `OPEN`, lecture, écriture, `CLOSE`. Point d'entrée unique `FFFE4H`. |
| 1 | **IOCS** (I/O Control System) | Fonctions propres à chaque driver, plus riches que FCS. Point d'entrée unique `FFFE8H`, dispatché vers la routine du driver via la chaîne d'en-têtes (`03-memoire-et-systeme-pc-e500s.md` §7). |
| 0 | Matériel direct | Accès direct aux registres/E-S (`03-memoire-et-systeme-pc-e500s.md` §2). Rapide mais non portable ; le manuel Sharp lui-même déconseille son usage hors cas de performance critique. |

En BASIC, `OPEN`/`PRINT#`/`INPUT#`/`CLOSE` utilisent le niveau FCS ; les commandes plus spécialisées (contrôle LCD, cassette...) descendent au niveau IOCS.

## 0. ⛔ Un numéro d'`IL` ne veut rien dire tant qu'on ne sait pas QUI le consomme

C'est la première chose à savoir avant de lire — ou d'annoter — du code qui appelle le système.
`IL` porte un numéro, et **le même numéro désigne des choses différentes selon le vecteur qui
le reçoit** :

```asm
callf ptr_fcs      ; 0FFFE4h -> fonction FCS        (carnet : fcs_*)
callf ptr_iocs     ; 0FFFE8h -> commande IOCS       (carnet : iocs_* sous 041h, sinon propre au device)
callf ptr_<propre> ;         -> service du programme lui-meme
```

Le troisième n'est pas une curiosité : un résident qui expose ses propres services publie son
vecteur et l'appelle avec la même instruction. **TY-DOS** en a un (`ptr_runil`, huit `IL`
documentés à son manuel), et ses sources mêlent les trois.

⚠️ **Nommer un `mv il,n` sans avoir établi son vecteur produit des noms faux ET
vraisemblables** — le pire des résultats, puisque rien ne les dénonce à la relecture. Douze
noms de `tyed.asm` ont été pris ainsi, et débusqués par contrôle croisé avec le carnet FCS.

**La méthode qui tient** : partir du `mv il,n` et **descendre jusqu'au `callf ptr_*`**, en
traversant au plus une routine intermédiaire, et **s'arrêter** dès que `IL` est détruit, qu'un
autre chemin peut arriver sur le site, ou que la fenêtre est épuisée. Ce qui n'est pas établi
**reste littéral**.

> Mesuré sur les 304 sites des six programmes de TY-DOS (septembre 2026) : **174** nommés
> depuis le carnet (164 FCS, 10 IOCS), **52** depuis les services de TY-DOS, et **78 laissés
> littéraux** faute d'avoir pu établir leur vecteur — compteur de boucle, longueur d'un `mvl`,
> valeur chargée loin en amont. Un quart de non-réponses assumées vaut mieux qu'un quart
> d'inventions.

## 1. FCS — appel `CALLF FFFE4H`

Convention d'appel : numéro de fonction dans `I` (`IL`), paramètres additionnels dans `A`, `(CL)`, `X` ou `Y` selon la fonction. Retour : `C=0` succès, `C=1` erreur avec code dans `A`.

| N° (IL) | Fonction | Détail |
|---|---|---|
| `00H` | Créer un fichier | Crée (ou réinitialise à taille 0) le fichier nommé sur le lecteur ; pointeur de fichier positionné à `000000H`. |
| `01H` | Ouvrir un fichier | `open_file`. |
| `02H` | Fermer un fichier | `close_file` (`CX` = handle). |
| `03H` | Lire un bloc | `read_block` — `X` pointe le tampon, `Y` contient la taille. |
| `04H` | Écrire un bloc | `write_block` — `X` pointe le tampon, `Y` contient la taille. |
| `05H` | Lire un octet | `read_byte` (`CX` = handle). |
| `06H` | Écrire un octet | `write_byte` (`A` = caractère, `CX` = handle). |
| `07H` | Vérifier un fichier | `verify_file`. |
| `08H` | Lecture non destructive | Relit sans avancer le pointeur de fichier. |
| `09H` | Déplacer le pointeur de fichier | `seek`. |
| `0AH` | Lire les informations d'un fichier | Taille, attributs... |
| `0BH` | Modifier les informations du répertoire du lecteur | |
| `0CH` | Rechercher un nom de fichier correspondant | (jokers) |
| `0DH` | Renommer un fichier | |
| `0EH` | Supprimer un fichier | |
| `0FH` | Lire la capacité libre du lecteur | Équivalent BASIC `DSKF`. |
| `10H` | Initialiser le système de fichiers | `init_fcs` — libère les FCB. |

### Table des handles de fichiers (`BFC6DH`)

Table système faisant correspondre un *handle* de fichier (le `#n` de BASIC) à un numéro de FCB. Les handles `00H`-`0FH` correspondent à un numéro de FCB, `FFH` = inutilisé. Les handles `0`, `1`, `2` sont réservés et déjà ouverts au démarrage de BASIC : `0` = écran (`stdo:`), `1` = clavier (`stdi:`), `2` = imprimante/liste (`stdl:`).

### ⛔ Le FCS n'écrit pas par la chaîne IOCS : il mémorise l'entrée du pilote à l'ouverture

Chaque handle ouvert a un **bloc de contrôle** (`1Fh` octets), dont l'octet 0 est le device (`0FFh` = libre) et dont **`+2` porte l'adresse d'entrée du pilote, relevée à l'OUVERTURE** (`0E07F1h`). À l'écriture, le FCS **saute directement** à cette adresse (`0E0907h`), sans repasser par `iocs_call` ni par la chaîne des en-têtes.

```
bloc = [(iocsw)] + [(iocsw)+25h] + numero * 1Fh     numero = [FHANDLE + handle], borne [(iocsw)+27h]
releve sur un PC-E500S reel (s1-1.bin) -- handle 0 en 0BEE48h :  00 00 E7 21 0F A2 ...
                                                                 |  |  \______/
                                                            device lecteur  0F21E7h = entree DISPLAY
```

**Conséquence** : un filtre ajouté **en tête de la chaîne** (la méthode de `MEMCHECK`, valable pour les appels IOCS) **ne voit jamais un `PRINT`**. Pour intercepter l'écran, il faut remplacer l'adresse `[bloc du handle 0 + 2]` et transmettre à l'ancienne. ✅ Éprouvé sur machine par le filtre de `XCONSOLE` (`C:\Claude\BASEXT`, septembre 2026 ; `12-extensions-basic.md` §17bis). Si la ROM rouvre `STDO:`, le filtre disparaît simplement. ⚠️ Seul le handle concerné est filtré : un `OPEN "SCRN:"` ouvre un autre handle, qui garde l'adresse de la ROM.

### ⛔ Le FCS refuse le lecteur de disquettes : `X:` et `Y:` n'existent pas pour lui

Relevé dans `rom83` le 2026-10-08, en cherchant pourquoi `DIR X:` de TY-COM répond `error` alors que
`FILES "X:"` liste la disquette (`C:\Claude\TYCOM2\README.md`).

**Le test.** Les fonctions FCS qui partent d'un nom de lecteur — `00H` création (`0E0840h`), `01H`
ouverture (`0E070Bh`), `0BH`-`0FH` (`0E0A97h` : informations, **`0CH` recherche**, renommage,
suppression, capacité libre) — appellent toutes `0E07D0h` :

```asm
0E07D0  pushu i
0E07D1  mv    il,000h
0E07D3  callf iocs_call          ; IOCS 00h find_drive : Y = en-tete du lecteur
0E07D7  popu  i
0E07D8  jrc   LOC_E07E2
0E07DA  mv    a,[y+004h]         ; attribut de l'en-tete (§2.1)
0E07DD  test  a,080h             ; bit 7 : gerable par le FCS
0E07DF  jrz   LOC_E07E2
0E07E1  ret
0E07E2  mv    a,00Ah             ; "lecteur inexistant"
0E07E4  sc
0E07E5  ret
```

C'est le bit 7 du tableau de §2.1, **vérifié dans le code** : le FCS ne se contente pas de le documenter,
il l'exige.

**Le lecteur de disquettes ne l'a pas.** Attribut des en-têtes de la ROM (§2.1, chaîne en `03` §7) :

| En-tête | Lecteurs | Attribut | Bit 7 |
|---|---|---|---|
| `DF872h` MEMORY CARD | `S1:` `S2:` `S3:` | `0C3h` | ✅ |
| `DF884h` MEMORY FILE | `E:` `F:` `G:` | `083h` | ✅ |
| `DF89Ch` FDD | `X:` `Y:` | **`00h`** | ⛔ |

Et son guichet IOCS (`0EB66Ch`) n'accepte que la commande `40h` (`SC62015Disassembler/Docs/Synthese/
Drivers-IOCS.md` §12). **Toute fonction FCS sur `X:` ou `Y:` rend donc `0AH`**, sur machine comme sur
émulateur : un programme qui passe par le FCS — TY-DOS, TR-DOS, leurs applications, un pilote — ne lit ni
n'écrit la disquette.

**Le BASIC contourne le FCS.** `0FE986h` reconnaît un nom `X:` ou `Y:` (`58h`/`59h` suivi de `:`) ; `FILES`
arme alors le bit 2 de `(016h)` et appelle **directement** les routines du pilote FDD, qui parlent au
CE-140F (protocole : `SC62015Disassembler/Docs/Synthese/Drivers-IOCS.md` §13) :

| Routine (ROM 8.3) | Commande CE-140F | Entrée | Sortie |
|---|---|---|---|
| `0EB792h` ouverture du catalogue | `05h` | `X` = **motif**, forme `"X:    NOMFICHI.EXT"` (`0EC22Ch` envoie les 2 caractères du lecteur puis les 12 du nom) | `A` = nombre de fichiers **conformes au motif** ; carry si erreur |
| `0EB702h` entrée suivante | `06h` | `Y` = tampon de 19 octets | `"X:    "` + `"NOMFICHI.EXT"` + attribut (bit 0 = protégé, le `P` de `FILES`) |
| `0EB6F7h` entrée précédente | `07h` | idem | idem |

Les 19 octets ont la forme exacte du champ que rend `0CH` `find_file` pour les autres lecteurs. Ces
routines s'appellent par `CALLF` avec le `BP` courant : elles prennent leur cadre **sous** `BP`
(`pmdf (bp_ram),0FEh` / `0EBh`, soit 2 et 21 octets) et le rendent ; la RAM interne sous `BP` doit donc
être libre. ⚠️ Leurs adresses dépendent de la ROM ; relevées par leur
code (identique à l'adresse du premier `CALL` près) :

| ROM | `[0FFFF0h]` | Ouverture | Suivante | Précédente |
|---|---|---|---|---|
| 8.3 | `08 03` | `0EB792h` | `0EB702h` | `0EB6F7h` |
| 7.5 | `07 05` | `0EBA91h` | `0EBA01h` | `0EB9F6h` |
| 5.3 | `05 03` | `0EBA76h` | `0EB9E6h` | `0EB9DBh` |

**Le motif est appliqué par le lecteur.** Le manuel d'utilisation du CE-140F (`Manuels CE-140F`,
pp. 11 et 21) le dit : `?` vaut un caractère, espace de bourrage comprise (`FILES "X:SHARP??.BAS"` trouve
`SHARP`, `SHARP2`, `SHARP55`, pas `SHARP100`), `*` complète le champ, et ce qui suit un `*` jusqu'au point
est ignoré (`A*Z.*` = `A*.*`). ⚠️ **Les substituts du lecteur ne le font pas** : le CE-140F de PockEmul
est un **simulateur**, écrit sans le matériel ni sa ROM (de l'aveu de son auteur, 2026-10), et n'y
applique pas un joker partiel ; le manuel de l'émulateur matériel de F. Fumi (`CE-140F-EMU-manual.pdf`)
note que `*` et `?` n'y marchent avec aucune commande. Un programme qui veut marcher partout envoie
`????????.???` et filtre lui-même.

✅ **Mis en œuvre** dans TY-COM 2.03 (`dir_fdd`, filtre `fdd_match`), qui choisit la ligne par la
version puis vérifie l'empreinte avant d'appeler. ⛔ La 2.01 passait l'adresse de la routine dans `X` :
le lecteur recevait le code de la ROM comme motif et répondait 0 fichier. La 2.02 envoyait le motif
tapé : tout s'affichait avec `????????.???`, rien avec un joker partiel (PockEmul, 2026-10-08).
⚠️ La 2.03 n'est **pas encore essayée** à la date de rédaction.

> ✅ `Drivers-IOCS.md` §12 concluait qu'« un fichier sur disquette s'ouvre par les fonctions FCS
> ordinaires ». **Corrigé le 2026-10-08** : les appels de `0DFCAAh` qu'il citait sont l'instruction
> BASIC `INIT` (token `1Dh`, `0DFC87h`), qui formate la disquette par les commandes `08h` et `09h` du
> CE-140F (`0EB7EBh`, `0EB81Fh`) — pas le FCS.

### Codes d'erreur FCS (`C=1`, code retourné dans `A`)

| Code | Signification |
|---|---|
| `00H` | Erreur du périphérique, opération abandonnée. |
| `01H` | Paramètre hors limites. |
| `02H` | Le fichier spécifié n'existe pas. |
| `03H` | Le chemin d'accès spécifié n'existe pas. |
| `04H` | Nombre de fichiers ouverts simultanément dépassé. |
| `05H` | Traitement non autorisé sur ce fichier. |
| `06H` | Handle de fichier invalide. |
| `07H` | Opération non prévue par l'instruction `OPEN` utilisée. |
| `08H` | Le fichier est déjà ouvert. |
| `09H` | Nom de fichier dupliqué. |
| `0AH` | Le lecteur spécifié n'existe pas. |
| `0BH` | Erreur de vérification de données. |
| `0CH` | Le traitement du nombre d'octets demandé ne s'est pas terminé. |
| `FEH` | Batterie critique. |
| `FFH` | Traitement interrompu (touche BREAK). |

## 2. IOCS — appel `CALLF FFFE8H`

### 2.1 Structure d'un en-tête de driver

Chaque driver installé publie un en-tête (chaîné, voir `03-memoire-et-systeme-pc-e500s.md` §7) :

> ✅ **Ce format n'est pas qu'une structure du manuel : c'est une norme de fait, elle a une source
> datée, et un auteur.** La notice du pilote `EXTSLOT` déclare se conformer aux règles de pilotes
> décrites dans l'article **« Pocket Link », *Pocket Computer Journal*, juin 1990** (工学社)
> — soit deux ans avant les pilotes que le corpus conserve.
>
> ⛔ Une version antérieure de ce paragraphe attribuait cet article à « M. Chikashige », d'après
> la traduction française de la notice d'EXTSLOT, et ajoutait que rien ne le reliait à `PLINKC`.
> **Les deux points sont faux** : la bibliographie de `PLINKC.DOC` (D. Mizobata, 1999) cite
> « 近 成人：ポケット・リンク、ポケコン・ジャーナル1990年 6月号、工学社 » — l'article est de
> **近成人, c'est-à-dire N. Kon**, l'auteur de PLINK, de XASM 1.0 et de VOGUE (`07` §3).
> « Chikashige » est une lecture erronée des mêmes kanji. La norme de fait des pilotes remonte
> donc à celui-là même dont le corpus conserve l'assembleur et le compilateur.

| Décalage | Champ | Contenu |
|---|---|---|
| `+0` | Adresse du prochain en-tête | 3 octets ; `FFFFFH` pour le dernier de la chaîne. |
| `+3` | Numéro de périphérique | Identifiant propre à chaque device. |
| `+4` | Attributs du périphérique | Voir ci-dessous. |
| `+5` | Adresse d'entrée IOCS | 3 octets — point d'entrée de la routine du driver. |
| `+8` | Nom(s) de lecteur | Chaîne ASCII, max 5 octets par nom, séparés par `:`, terminée par `00H`. |

Bits d'attribut (`+4`) :

| Bit | Signification si à 1 |
|---|---|
| 7 | Périphérique gérable par le contrôle de fichiers (FCS). |
| 6 | Périphérique de fichier spécial (non gérable par le traitement standard FCS). |
| 5 | 0 = périphérique bloc (géré par cluster) · 1 = périphérique caractère. |
| 4 | Le périphérique traite par défaut en code ASCII. |
| 2 | Lecture et écriture non simultanées. |
| 1 | Périphérique autorisant l'écriture. |
| 0 | Périphérique autorisant la lecture. |

*(Exemple donné par le manuel : `C7H` = périphérique spécial lecture/écriture, non simultané.)*

### 2.2 Plages de numéros de commande IOCS

| Plage | Usage |
|---|---|
| `00H`-`07H` | Commandes de la routine IOCS principale (ne saute pas vers l'entrée d'un driver) — voir §2.3. |
| `08H`-`0FH` | Traitement fichier des périphériques caractère standard (utilisé depuis FCS). |
| `10H`-`1FH` | Traitement fichier des périphériques bloc standard (utilisé depuis FCS) — lecture/écriture secteur, etc. |
| `20H`-`3FH` | Traitement fichier des périphériques spéciaux (utilisé depuis FCS) ; `3FH` = formatage. |
| `40H` | Initialisation, commande commune à tous les drivers. 📖 La commande `05H` de la routine principale (`0E0159h`) l'envoie à **tous** les pilotes chaînés, en **ignorant la retenue** : un pilote qui répond « commande non gérée » (`sc`/`retf`) y est inoffensif. |
| `41H`-`7FH` | Fonctions propres à chaque périphérique, **choisies par le numéro de device en `(cl)`** : `43H` vaut `key_read` sur le clavier, `printer_check` sur l'imprimante, `block_transfer` sur la carte mémoire, `basic_exec` sur le driver système ; `47H` vaut `condense` (compactage) sur la carte mémoire, device 6 (cf. `Data/FCSFunctions.json`). |
| `80H`-`FFH` | Réservé. |

Codes d'erreur (registre `A`, `C=1`) selon la plage de commande :

**Commandes `00H`-`1FH`, `40H`** — `00H` écriture protégée, `01H` lecteur absent, `02H` lecteur non prêt, `03H` commande non gérée, `04H` support changé, `05H` erreur d'écriture, `06H` erreur de lecture, `07H` erreur de vérification, `08H` périphérique en écriture seule, `09H` périphérique en lecture seule, `FEH` batterie critique, `FFH` interruption BREAK.

**Commandes `20H`-`3EH`** — mêmes codes que FCS (§1), avec en plus `0AH` nom de fichier dupliqué, `0BH` lecteur inexistant, `0CH` erreur de vérification.

**Commande `3FH`** (formatage) — mêmes codes d'erreur que le BASIC.

**Commandes `41H`-`7FH`** — propres à chaque périphérique (voir §2.4).

### 2.3 Commandes de contrôle des drivers (`00H`-`05H`)

Utilisent l'entrée IOCS principale (indépendantes d'un driver particulier) :

| Cmd | Fonction | Entrée | Sortie |
|---|---|---|---|
| `00H` | Rechercher le numéro de périphérique/lecteur à partir du nom | `X` = adresse du nom de lecteur | `CL`=numéro de périphérique, `CH`=numéro de lecteur |
| `01H` | Chercher l'adresse d'en-tête (1) | `CL` = numéro de périphérique | `X` = adresse de l'en-tête |
| `02H` | Chercher l'adresse d'en-tête (2), recherche à partir d'un point donné | `CL` = numéro de périphérique, `X` = en-tête de départ | `X` = adresse de l'en-tête trouvé |
| `03H` | Récupérer le nom de lecteur | `CL`=périphérique, `CH`=lecteur, `X`=tampon (≥6 octets) | nom + `:` écrits en `X`, complétés par `20H` |

*(Erreur commune aux 4 commandes : `C=1`, `A=01H`.)*

### 2.4 Exemple de fonctions propres à un driver — écran (`STDO:`/`SCRN:`)

Illustration des fonctions « `41H`-`7FH` propres au périphérique », ici pour le driver LCD (n° 00), qui recouvre `STDO:`/`SCRN:` :

| Cmd | Fonction |
|---|---|
| `3FH` | Formatage — spécifique au PC-E500. |
| `40H` | Initialisation du driver LCD. |
| `41H` | Sortie d'un caractère à une position arbitraire. |
| `42H` | Sortie d'une chaîne à une position arbitraire : `(bl)`/`(bh)`, `X` = chaîne, `Y` = longueur. ✅ Mesuré (2026-10-10, `*PSTR` de `ROUTINES.BAS`) : **ne déplace pas le curseur et ne fait pas défiler** — 40 caractères tiennent sur la dernière ligne ; coupe au bord droit (retenue armée s'il reste des caractères). |
| `44H` | Positionnement du curseur. |
| `45H` | Type d'affichage du curseur : `A` bit 5 = affiché, bit 3 = clignotant, bits 2-0 = forme (0 souligné, 1 double souligné, 2 *full mark*, 3 *full space*, 4 marque d'insertion) ; `A = 0` le cache. ✅ Pavé affiché (mesuré 2026-10-10). |
| `46H` | Affichage d'un symbole. |
| `47H`/`48H` | Défilement de `n` lignes vers le haut/bas. |
| `49H` | Effacement d'une ligne. |
| `4AH` | Affichage d'un motif 8 points. |
| `52H` | Effacement à partir d'une position : `(bl)`/`(bh)` = départ, `A` = nombre de caractères. ✅ Lu (`0F2B85h`) et mesuré : **passe à la ligne suivante** en fin de ligne (`(bl)` ← 0, `(bh)` + 1) et s'arrête à la ligne 4 ; `A = 0` ne fait rien. |
| `51H` | `clear_disp` — effacement de l'affichage ; ✅ n'efface en fait que les lignes `0`..`lcd_height−1` (`BFC9EH`, `03` §4 ; `0F2771h`), mesuré sur machine par `CLS`. |
| `53H` | Absente du manuel, jamais émise par `rom83` ; un relevé antérieur disait « suppression d'une ligne », rien ne l'étaye. |
| `55H`/`56H` | `pattern_read`/`pattern_write` — motif de points d'une ligne (`(bh)` = ordonnée) vers/depuis la mémoire externe (`X`, **240 octets**). ✅ Employées avec `49H` par le filtre de `XCONSOLE` pour défiler une partie seulement de l'écran. |
| `58H` | `guide_line` — chaînes encadrées d'un guide ; c'est par elle que `fkey_display` (clavier, `46H`) dessine les libellés des touches de fonction, et la barre `[BASIC ][ CAL  ][MATRIX][ STAT ][ ENG  ]` de la touche `MENU` (libellés en `0F92AFh`). Voir ci-dessous. |

*(Noms et descriptions de `41H`-`58H` : `SC62015Disassembler/Data/FCSFunctions.json`, qui porte toute la liste.)*

**`58H` lue dans la ROM (`0F2C90h`), mesurée le 2026-10-10** — le manuel (p. 41) est inexact sur deux points :

| Entrée | Rôle |
|---|---|
| `(bl)`, `(bh)` | colonne et ligne de la première case |
| `A` | nombre de chaînes |
| `B` bit 7 | `0` = efface d'abord la ligne `(bh)`, `1` = la garde. Le manuel écrit « bit 8 » : c'est le bit 8 de **`BA`** |
| `B` bit 0 | `0` = crochets, `1` = parenthèses — glyphes spéciaux `B7h`/`B8h` ou `C2h`/`C3h` de la police en `0306C0h` |
| `X` | les chaînes, chacune terminée par `FFh` ; un `FEh` en tête est sauté |

Chaque case = un glyphe ouvrant, **6** caractères (un libellé plus court est **complété par des espaces** par la routine elle-même, `LOC_F2CE1` ; un plus long est **tronqué**, elle saute jusqu'au `FFh` suivant en `LOC_F2CCF` — lu puis ✅ **mesuré** le 2026-10-14, `DWGL 0,3,0,"A,TROPLONGPOURSIX,C"` donnant `[A     ][TROPLO][C     ]` ; le manuel dit 7), un glyphe fermant : 8 colonnes, 5 cases par ligne. Le guide de la ROM finit sur `retf` sans fixer la retenue : l'appelant fait `rc` (comme `fkey_display` en `0F1D3Eh`). Emploi depuis le BASIC : `*MNU` de `C:\Claude\ROUTINES\ROUTINES\ROUTINES.BAS`.

**Clavier (device 1), `46H` `fkey_display`, lue en `0F1CFAh`** : le manuel ne cite aucun paramètre, la ROM teste `A` — `A = 1` affiche les libellés des touches de fonction en ligne 3 (fichier `FUNCKEY.`, `BFC45H`), toute autre valeur efface la ligne 3 (`49H`). Retenue armée si l'écran est en 4 lignes (`lcd_height` = 4) ou si le bit 2 de `BFC3FH` interdit l'affichage. ✅ Mesuré (`*FKD`, 2026-10-10).

#### 2.4quater ⛔ « Display with guide line » n'a **qu'un seul émetteur** dans toute la ROM

Relevé dans `rom83` le 2026-10-12 : `mv il,058h` suivi d'un `callf iocs_call` n'apparaît **qu'une
fois** — en `0F1D2Eh`, **à l'intérieur de `fkey_display`**, c'est-à-dire de la commande `46H` du
pilote **clavier**.

> ⛔ **La « ligne de guide » du manuel et la barre des touches de fonction sont la même chose.** Il
> n'existe pas, dans la machine, de « barre de menu » distincte : l'écran `MENU` affiche cette
> barre-là, par ce chemin-là. C'est ce qui rend la commande exploitable — ce que la ROM se passe à
> elle-même, un programme peut se le passer aussi.

Ce que `fkey_display` fait, désassemblé :

```asm
        cmp     [lcd_height],004h       ; ⛔ refus si l'ecran est en 4 lignes
        jrnc    LOC_F1D56
        test    [funckey_flag],004h     ; ⛔ refus si le bit 2 de 0BFC3Fh est arme
        jrnz    LOC_F1D56
        cmp     a,001h                  ; A <> 1 : efface la ligne 3 (049h clear_line)
        jrnz    LOC_F1D40
        ...
        mv      a,[0BFC40h]
        mv      b,a                     ; B <- le selecteur de jeu de libelles
        mv      a,005h                  ; A = 5 cases
        mvw     (bl),00300h             ; colonne 0, ligne 3
        mv      il,058h
        mvw     (cl),00000h
        callf   iocs_call
        or      [funckey_flag],001h     ; bit 0 = « la barre est affichee »
        rc
        retf
```

**Deux bits d'état et un sélecteur**, que le carnet ne portait pas :

| Adresse | Ce qu'elle fait |
|---|---|
| `0BFC3Fh` bit 2 | **interdit** l'affichage — `fkey_display` refuse, retenue armée |
| `0BFC3Fh` bit 0 | ✅ **posé** quand la barre vient d'être affichée, **effacé** quand la ligne 3 est effacée ou le refus prononcé : c'est l'état « la barre est là » |
| `0BFC40h` | ✅ **sélecteur du jeu de libellés** (`< 1` → `013h`, sinon `033h`, passés à `0F6402h`) **et** passé tel quel en `B` à `58h` — donc son bit 0 choisit aussi crochets ou parenthèses. Sans nom dans `pce500.inc` |

**Le chemin depuis l'écran `MENU`** : `0F931Ah` est le secours « afficher la barre sauf si l'écran
est à 4 lignes » (`cmp [lcd_height],004h`, puis `A = 1`, commande `46h` par `IOCSKEY 0F3EB9h`). Il a
deux appelants, `0F9343h` et `0FA980h` ; `0F9343h` est lui-même appelé de trois sites de la région
du MENU PRINCIPAL et, **quand la barre a été refusée, force l'écran à 4 lignes** et remet `0BFC40h`
à zéro.

⚠️ **Ce qui n'est pas établi** : le chemin pas à pas depuis la frappe de la touche `MENU` (code
matriciel `06h`, §2.4bis) jusqu'à ces appels — il traverse le MENU PRINCIPAL (`0F9AA6h`) et je ne
l'ai pas suivi. Ce qui est mesuré, c'est **l'unicité de l'émetteur** et **les paramètres**.

##### L'exploiter : en ASM, et en BASIC malgré la porte

⛔ **`CALL &FFFDC` ne suffit pas.** La porte ne transporte que `(cl)`, `(ch)` et `IL` (§2.2), quand
`58h` veut en plus `(bl)`, `(bh)`, `A`, `B` et `X`. Il faut donc un module en langage machine — et
les deux voies existantes le font de la même façon, avec **les octets mêmes de la ROM** :

```asm
        mv      x,libelles              ; les chaines, chacune terminee par 0FFh
        mvw     (bl),00300H             ; colonne 0, ligne 3  -- 30 CD D4 00 03
        mv      ba,00005H               ; A = 5 cases, B = 0 : crochets, ligne effacee
        mvw     (cl),00000H             ; device 0 = ecran
        mv      il,058H                 ;                     -- 09 58
        callf   iocs_call
        rc                              ; ⛔ 058h ne fixe pas la retenue
        retf
```

| Voie | Où | Comment |
|---|---|---|
| **ASM** | `Documentation/GUIDE.ASM`, 60 octets | ✅ **éprouvé sur PC-E500S le 2026-10-12** ; `LOAD M` puis `CALL &BF000` |
| **BASIC** | `*MNU` de `C:\Claude\ROUTINES\ROUTINES\ROUTINES.BAS` | pose **26 octets** en `0BEF60h` et les libellés en `0BEF80h`, puis `CALL &BEF60` ; repli en `LOCATE`/`PRINT` si la zone langage machine n'est pas réservée |
| **mot-clé** | `DWGL colonne,ligne,tag,chaine$` de `C:\Claude\BASEXT` | la commande devenue **instruction du langage**, dans le module comme dans le pilote résident `BASEXT-DRV` : ni `POKE`, ni zone à réserver. Les libellés sont séparés par des virgules ; `DWGL` refuse ce que la ROM perdrait en silence (ligne ≥ 4, plus de 5 cases, débordement de la ligne → erreur 33). Écrite **et ✅ éprouvée sur PC-E500S le 2026-10-14** (`BASEXT/essais/DWGLTEST.BAS`) ; étude dans `BASEXT/CONCEPTION-DWGL.md`, référentiel `12-extensions-basic.md` §17quinquies |

✅ **Les trois encodages concordent à l'octet** — `30 CD D4 00 03`, `09 58`, `05 E8 FF 0F` — entre la
ROM (`0F1D29h`), `GUIDE.OBJ` et les `POKE` de `*MNU`. ⚠️ `GUIDE.ASM` appelle `58h` **directement** :
il ne refait aucun des trois contrôles de `fkey_display`, donc **l'écran doit être en 3 lignes**
(`CONSOLE 0`), faute de quoi la barre s'écrit par-dessus une ligne de texte.

✅ **Essai sur PC-E500S réel, 2026-10-12** — `LOAD M "GUIDE.OBJ"` puis `CALL &BF000` rend :

```
[BASIC ][CAL   ][MATRIX][STAT  ][ENG   ]
```

Trois choses s'y vérifient d'un coup : **une commande du device 0 s'appelle depuis un simple `CALL`
du BASIC** sans passer par le pilote clavier ni par aucun des contrôles de `fkey_display` ; les
cases font bien **six** caractères et non sept, `MATRIX` les occupant tous ; et **cinq cases de huit
colonnes remplissent exactement la ligne de 40**. La barre de la machine est donc reproductible à
l'identique par un programme — c'est le même appel, aux mêmes octets, avec d'autres libellés.

**Clavier : ce que voit le BASIC** — ✅ mesuré le 2026-10-10 (émulateur ; `C:\Claude\ROUTINES\MEMCHK\KEYCODE.BAS`) :

| Lecture | `PF4` | `CTRL`+`PF4` | `PF5` | `CTRL`+`PF5` | flèches, `C-CE` |
|---|---|---|---|---|---|
| `INKEY$ &1` | `F4h` | `F4h` | `F5h` | `F5h` | `05h` bas, `04h` haut, `0Eh` droite, `0Fh` gauche ; `02h` |
| `INPUT$ (1)` | texte programmé (`SAVE `) | `00h` puis `56h` | texte programmé (`LOAD `) | `00h` puis `57h` | `1Fh`, `1Eh`, `1Ch`, `1Dh` ; `0Ch` |

- `INKEY$` passe par sa propre table (`0F9C5Dh`) et **ignore `CTRL`**. `INPUT$` lit le pilote `STDI:` (FCS) : codes à deux octets `00h` + code de la table du manuel technique p. 47 (`CTRL`+`PF1`…`PF5` = `53h`…`57h`), mais les PF seules y rendent le **texte qui leur est programmé** (`RUN `, `LIST `, `CONT `, `SAVE `, `LOAD ` ici) : inutilisable pour un menu PF. `INPUT$` (token `INPUT` + `$`) est réaffiché `INPUT $ (1)` par `sharpbasic`, octets identiques.
- Lire `CTRL` dans la matrice depuis le BASIC (`POKE &F0,1` puis `PEEK &F2` bit 7) n'a **pas** marché pendant l'appui d'une autre touche, même en 10 lectures : `CTRL` seul donne bien `80h`, mais la ROM rebalaie le clavier (au repos elle laisse `KO` à 0, `0F1D7Dh`). Piste restante : IOCS clavier `43H` (lecture avec `SHIFT`/`CTRL` traités) depuis un code machine.

#### 2.4bis ✅ Pourquoi aucune des deux fonctions ne voit tout — les quatre tables, relevées

Le relevé ci-dessus disait **ce que** les deux fonctions rendent. Les tables disent **pourquoi**, et
elles se lisent dans les images (2026-10-12).

**Le manuel technique distingue trois représentations** d'une touche (chapitre 3, « *No. 01 Key
Driver (STDI: KYBD:)* », pp. 43-48) :

| Représentation | Où elle vit |
|---|---|
| **position matérielle** | les ports `KOL` `(0F0h)`, `KOH` `(0F1h)`, `KIL` `(0F2h)` — une ligne strobée, une ligne lue |
| **code matriciel** | rendu par IOCS `041h matrix_read`, bits 0-6 de `A` (bit 7 = touche relâchée) |
| **code caractère** | ce que voit le programme, après conversion **par une table** |

C'est le troisième passage qui sépare les deux fonctions — **et il n'y a pas une table, il y en a
sept** :

- le **pilote** en a **six**, déclarées dans la System Data Area en `0BFC2Dh`-`0BFC3Eh` (six
  pointeurs de 3 octets vers des tables de `062h` = 98 octets) : code d'**1 octet** et de **2
  octets**, chacun en **normal / `SHIFT` / `CTRL`** (`03` §3) ;
- **`INKEY$` n'en emploie aucune.** Il a la sienne, `0F9C5Dh`, en ROM, **citée nulle part ailleurs**
  — et **sans variante `SHIFT` ni `CTRL`**.

**Les quatre tables à un octet, au même index** (PC-E500S 8.3 ; pilote dans `S3EXT`) :

| Code matriciel | `INKEY$` `0F9C5Dh` | pilote `0307A4h` | `SHIFT` `03092Ch` | `CTRL` `030868h` |
|---|---|---|---|---|
| `31h` — `A` | `41h` `A` | `41h` `A` | `5Bh` `[` | `01h` |
| `16h` — `PF4` | **`F4h`** | `00h` | `00h` | `00h` |
| `1Dh` — flèche ← | **`0Fh`** | `1Dh` | `00h` | `00h` |
| `30h` — espace | `20h` | `20h` | `20h` | `00h` |

⛔ **Trois choses s'en déduisent, et elles répondent chacune à une observation du relevé :**

1. **`INKEY$` ignore `CTRL` par construction, pas par choix** : sa table n'a tout simplement pas de
   variante. Une seule table, un seul résultat.
2. **La table `CTRL` à un octet est vide sous `30h`.** Sous `CTRL`, une touche de fonction ou une
   flèche ne peut donc rien rendre par cette voie — elle passe par la table **à deux octets**, d'où
   le `00h` + code mesuré. Et la table `0308CAh` donne, pour `PF1`…`PF5` (codes matriciels
   `13h`-`17h`), exactement **`53h 54h 55h 56h 57h`** : ✅ le `00h`+`56h` mesuré sur machine le
   2026-10-10 **se retrouve dans l'image**, à l'index près.
3. **Les PF seules sous `INPUT$` rendent leur texte programmé** parce que le code à deux octets
   `00h`+`16h` que rend le pilote est intercepté **au-dessus de lui**, par l'éditeur du BASIC — qui
   lit le clavier *comme un fichier* (`0F3972h` : `(cl) = 1` = `STDI:`, FCS `005h read_byte`).

> ⚠️ **La régularité « `SHIFT` = +`20h`, `CTRL` = +`40h` » n'est vraie que par endroits** : elle
> tient sur les touches de fonction et les flèches (`13h`-`17h` → `33h`-`37h` / `53h`-`57h` ;
> `1Ch`-`1Fh` → `3Ch`-`3Fh` / `5Ch`-`5Fh`), mais **20 des 41** entrées `SHIFT` et **36 des 60**
> entrées `CTRL` seulement la suivent — le bloc katakana `51h`-`61h` répond à une autre logique.
> Ce sont des **tables**, pas une formule.

#### 2.4ter Le partage, en une ligne : il faut les deux fonctions

| | `INKEY$` | `INPUT$` |
|---|---|---|
| voie | IOCS `041h matrix_read`, device 1 (`0F9FC4h`) | FCS `005h read_byte` sur `STDI:` — le clavier **comme un fichier** |
| table | **la sienne**, `0F9C5Dh`, une seule | **les six** du pilote, dans la SDA |
| `SHIFT` / `CTRL` | ⛔ jamais | ✅ tables dédiées, codes à deux octets |
| touches `PF` | ✅ `0F1h`-`0F5h` | ⛔ texte programmé (sauf avec `CTRL` : `00h`+`53h`-`57h`) |
| flèches | `0Eh` `0Fh` `04h` `05h` | `1Ch` `1Dh` `1Eh` `1Fh` |
| attente | **non bloquant** (`INKEY$ &H1`) | **bloquant** |

> ⛔ **Un menu qui réagit aux `PF` *et* aux combinaisons `CTRL` a besoin des deux fonctions.** Ce
> n'est pas une maladresse de l'une ou de l'autre : elles ne lisent pas la même chose au même
> endroit. `INKEY$` lit **la matrice**, `INPUT$` lit **le pilote**.

✅ **Programme de démonstration** : `C:\Claude\ROUTINES\MEMCHK\CLAVIER.BAS`. Il lit les six pointeurs
dans la SDA, affiche pour un même code matriciel les quatre réponses, puis fait lire **la même
touche** par les deux fonctions. ⚠️ Il **vérifie la version de ROM** : l'adresse `0F9C5Dh` de la
table d'`INKEY$` n'est établie que sur la **8.3**, et il le dit au lieu de deviner. `KEYCODE.BAS`,
dans le même dossier, reste l'explorateur libre de codes de touches.

Le manuel documente de la même façon chaque autre driver standard (clavier, SIO, imprimante, cassette, carte mémoire, disquette — voir la liste des en-têtes en `03-memoire-et-systeme-pc-e500s.md` §7) ; le détail complet des ~10 drivers dépasse le cadre de ce résumé et reste consultable dans le manuel original (`Docs/Doc technique/TechnicalReferenceManualPC-E500.pdf` et sa version mise en forme dans `Mise en forme documents SHARP/`).

### 2.5 Le device 9 — Function Driver, la bibliothèque mathématique

*Manuel technique pp. 79-82 du livre (PDF pp. 83-86 de `TechnicalReferenceManualPC-E500.pdf`, le bon exemplaire). Le manuel le dit « Device unable to use » : pas de lecteur, pas de fichier — les commandes `08H`-`28H` sont des erreurs, et tout passe par `41H`-`7FH`.*

`(cl)` = `9` choisit le device et **`(ch)` choisit la famille**. Les trois familles réutilisent les **mêmes** numéros de commande : `mvw (cl),00009h` écrit les deux octets en une instruction pour la famille 0, `mvw (cl),00109h` pour la famille 1.

| `(ch)` | Famille | Opérandes | Retenue au retour | État |
|---|---|---|---|---|
| `0` | **numérique et chaîne** — comparaisons `41H`-`46H`, arithmétique `47H`-`4BH`, fonctions `4CH`-`5BH`, comparaisons de chaînes `70H`-`75H`, `ASC`/`CHR$`/`STR$`/`VAL` `76H`-`79H`, conversions `7EH`/`7FH` | `X` en `(bp+0)`..`(bp+14)`, `Y` en `(bp+15)`..`(bp+29)` ; sens **« `Y op X` → `X` »** ; une chaîne vit sur la pile `U` et la commande la **consomme** | ⛔ **n'indique RIEN** : la sortie d'erreur du guichet (`0EF076h`) fait `rc` / `retf` | **mesuré** — `SQR`, `ADDTEST`, `DIVTEST` (2-6 septembre 2026) |
| `1` | **matrices** — `41H`-`56H` : opérations sur X et Y, inverse, transposée, déterminant, scalaires, X↔M, X↔MA-MZ, systèmes d'équations | les **tableaux BASIC** `X(,)`, `Y(,)`, `M(,)`, `MA(,)`-`MZ(,)` en simple précision ; le scalaire est **lu** dans la variable simple `X`, le déterminant y est **écrit** ; la lettre A-Z de `52H`/`53H` en `[BFE03H]` | **armée = erreur**, code BASIC dans `A` (60, 21, 20 ou 22) ; `A` n'a pas de sens si elle est claire | **mesuré** sur `4BH` et `4CH` — `MATTEST` (14 septembre 2026) |
| `2` | **statistiques et régressions** — `41H`-`47H` | paramètres en `[BFE03H]`/`[BFE04H]` (« 0-8 X sequence », « 0-8 Y sequence ») ; l'alimentation des données n'est pas établie | armée = erreur | **lu, non mesuré** ; le manuel **barre** `43H`-`47H`, et la ROM ne laisse passer que `41H` |

**Contraintes des familles 1 et 2**, lues dans le code (`0E7CE9h` et `0E8F74h`) :

- **`BP` doit valoir au moins `0A0h`**, sinon erreur 60. Le calcul travaille en RAM interne `080h`-`0BFh` avec `BP` = `080h`, et ne sauve lui-même que `0A0h`-`0BFh` (en `BFFD8H`, et `BP` en `BFFD0H`). Un appelant dont le cadre vit sous `0A0h` doit donc sauver `080h`-`09Fh` lui-même.
- Mesuré : `BP` vaut **`0BEh`** (190) au moment d'un `CALL` depuis le BASIC, et **`096h`** (150) à l'entrée d'une fonction d'extension appelée pendant l'évaluation d'une expression.

⛔ **Famille 0 : la division par zéro déplace `BP` de +30, en silence**, et cela s'accumule. Qui appelle `div` teste son diviseur lui-même, avant l'appel.

Détail, mesures et pièges : `12-extensions-basic.md` §13 (famille 0) et §17 (matrices). Recette d'appel commentée : `SC62015Disassembler/Docs/Routines-ROM-PC-E500S.asm`, section « APPELER LE DEVICE 9 ».

### 2.6 Trois portes vers l'IOCS

| Porte | Convention | Ce que fait la ROM |
|---|---|---|
| `callf iocs_call` (`FFFE8H`) | depuis le code machine : `(cl)`, `(ch)`, `IL`, puis les registres de la commande | répartition vers le driver par la chaîne d'en-têtes |
| `CALL &FFFDC` (`iocs_call3`) | **depuis le BASIC** : `POKE &BFE00,cl,ch,il` | `0EF01Ah` : `mv il,[0BFE02h]` / `mvw (cl),[0BFE00h]` / `callf iocs_call` |
| `CALL &FFFD8` (`secure_work_call`) | depuis le BASIC : `POKE &BFE03,` adresse (3 o) `,` valeur (3 o) | `0F964Fh` : relit les six octets, puis **IOCS `042H` `secure_work`** du device 8 |

> ⛔ **Depuis le BASIC, `(bl)` ne se pose pas par `POKE &D4`** — mesuré le 2026-10-10 : `POKE &BFE00,…:POKE &D4,30,0,0,0:CALL &BFE00` sur **une seule ligne**, la commande `52H` a vu `(bl)` = 0. `&D5` (`bh`) et `&D6`/`&D7` (`cx`) ont tenu dans les mêmes conditions. Charger `bl`/`bh` **dans le code** appelé (`30 CD D4 lo hi` = `mvw (bl),hilo`), comme le font `*IOCS`, `*MNU` et `*PSTR` de `ROUTINES.BAS`.
>
> ⚠️ Le manuel (p. 31) ne documente que `BFE00H`-`BFE02H`. Au-delà, c'est le brouillon de la ROM (`03` §4, ligne `BFE00`) : un petit code d'appel peut y être posé et exécuté sur la même ligne, mais pas des données qui doivent survivre ; au-delà de `BFE0DH`, les placer dans la zone langage machine.

> ⚠️ `CALL &FFFDC` ne transmet que trois registres : il ne convient qu'aux commandes qui n'attendent rien d'autre, ou dont les opérandes sont déjà en place — c'est le cas des matrices, rangées dans des tableaux BASIC. **Qu'il suffise effectivement pour la famille 1 n'est pas encore mesuré** (`12-extensions-basic.md` §16).

## 3. Voir aussi

- `03-memoire-et-systeme-pc-e500s.md` §7 — chaîne des en-têtes IOCS installés par défaut et leurs points d'entrée.
- `12-extensions-basic.md` §13, §14, §17 — le device 9 employé depuis une extension du BASIC, la réservation de zone par `CALL &FFFD8`, et les matrices.
- `06-ecosysteme-outils.md` — `PLINKC162` est un exemple concret de driver IOCS tiers (lecteur `L:`) installé selon ce mécanisme ; `BASEXT-DRV` (device `BEXT:`) en est un second, mesuré en 2026.
- `03-memoire-et-systeme-pc-e500s.md` §7bis — où insérer le bloc d'un pilote dans `S1:` pour qu'il ne bouge pas.
- `SC62015Disassembler/Data/FCSFunctions.json` — sous-ensemble machine-readable utilisé par le désassembleur pour annoter les appels `CALLF FFFE4H`/`FFFE8H` dans les listings.
