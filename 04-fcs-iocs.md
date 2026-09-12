# FCS et IOCS — appels système du PC-E500S

> Voir `00-index.md` pour la vue d'ensemble. Source principale : *Technical Reference Manual PC-E500* (chapitres 1 « File Control System », 2 « Outline of IOCS », 3 « How to use each device »), recoupé avec `SC62015Disassembler/Data/FCSFunctions.json`. Ce fichier **complète** ce JSON : le manuel documente 17 fonctions FCS (`00H`-`10H`) alors que le JSON du désassembleur n'en couvrait que 9 (reconstruites depuis des listings) ; les codes manquants sont ajoutés ici avec leur source.

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
| `40H` | Initialisation, commande commune à tous les drivers. |
| `41H`-`7FH` | Fonctions propres à chaque périphérique (cf. `Data/FCSFunctions.json`, ex. `0x22`, `0x47` = `compress_memory`). |
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
| `42H` | Sortie d'une chaîne à une position arbitraire. |
| `44H` | Positionnement du curseur. |
| `45H` | Type d'affichage du curseur. |
| `46H` | Affichage d'un symbole. |
| `47H`/`48H` | Défilement de `n` lignes vers le haut/bas. |
| `49H` | Effacement d'une ligne. |
| `4AH` | Affichage d'un motif 8 points. |

Le manuel documente de la même façon chaque autre driver standard (clavier, SIO, imprimante, cassette, carte mémoire, disquette — voir la liste des en-têtes en `03-memoire-et-systeme-pc-e500s.md` §7) ; le détail complet des ~10 drivers dépasse le cadre de ce résumé et reste consultable dans le manuel original (`Docs/Doc technique/TechnicalReferenceManualPC-E500.pdf` et sa version mise en forme dans `Mise en forme documents SHARP/`).

## 3. Voir aussi

- `03-memoire-et-systeme-pc-e500s.md` §7 — chaîne des en-têtes IOCS installés par défaut et leurs points d'entrée.
- `06-ecosysteme-outils.md` — `PLINKC162` est un exemple concret de driver IOCS tiers (lecteur `L:`) installé selon ce mécanisme.
- `SC62015Disassembler/Data/FCSFunctions.json` — sous-ensemble machine-readable utilisé par le désassembleur pour annoter les appels `CALLF FFFE4H`/`FFFE8H` dans les listings.
