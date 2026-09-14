# Écosystème d'outils déjà réalisés (dans `C:\Claude`)

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier répond directement à la demande initiale : « compléter ce que nous avons déjà réalisé » — il cartographie les projets existants, leur rôle, et comment ils s'articulent entre eux et avec le contenu des autres fichiers de ce référentiel.

## 1. Vue d'ensemble de la chaîne

```
BASIC texte (.BAS)  <—renum—>  BASIC texte renuméroté
      |  bacon / sharpbasic-c / Sharp Basic Converter (tokeniseur ⇄ détokeniseur)
      v
BASIC tokenisé (.BAC / .BSA)  ——— transfert série ———>  Sharp PC-E500S
      ^
      |  (langage machine, indépendant du chemin BASIC ci-dessus)
Source XASM (.S/.ASM)  --xasm-->  Objet (.obj, .uu, .hex, .s19...)  --désassembleur-->  Listing (.lst/.asm)
                                          |
                                          v
                                   PLINKC162 (driver IOCS L:, exemple de programme machine language complet)
```

Deux familles d'outils cohabitent : la chaîne **BASIC** (texte ↔ tokenisé ↔ transfert) et la chaîne **langage machine SC62015** (assemblage ↔ désassemblage), qui se rejoignent au niveau du transfert vers la machine réelle et de l'exécution (un programme BASIC peut faire `CALL` vers du code assemblé par XASM).

## 2. Chaîne langage machine (SC62015 / ESR-L)

| Projet | Rôle | État | Entrée → Sortie |
|---|---|---|---|
| **`XASM Origine`** | Assembleur croisé original (E. Kako/N. Kon, DOS, 1990s), source C + `.EXE` d'époque. | Archive de référence. | `.ASM`/`.S` → `.obj`/`.lst` (format historique). |
| **`xasm2026-1-2`** | Port C11 moderne de XASM (GCC/CMake), sorties étendues. | Fonctionnel, non-régression octet-par-octet vérifiée vs original. | Idem + `.hex`/`.s19`/`.map`/`.d`/`.uu`/dump HxD. |
| **`xasm2026-4`** | Réécriture C# (.NET 8) du même assembleur, **génération maintenue**. | Fonctionnel, non-régression **bit-exacte** vs `xasm2026-1`. | Même CLI/formats, plus `-U` (références croisées), `-K` (listing réduit aux constantes d'include utilisées) et le dialecte A62 de N. Kon (`05-format-fichiers-et-xasm.md` §1 et §4). |
| **`SC62015Disassembler`** (+ mirroir `-github`) | Désassembleur C#/.NET 9, lit les objets produits par XASM (tous formats, + images ROM, dont la ROM d'extension du PC-E500S) ; extrait aussi les fichiers des lecteurs `S1:`/`S2:`/`S3:`. | **Complet** — 972 tests en septembre 2026, vérifiés à trois niveaux : réencodage de chaque instruction à l'octet, reproduction de 25 listings XASM (18 764 instructions, 0 divergence), et **34 programmes réassemblés par XASM identiques à l'octet**. | `.obj`/ROM → `.lst` (listing annoté) / `.asm` (source réassemblable) / `.json`/`.csv`. |
| **`SC62015Disassembler/Samples/BASEXT` et `DEVICE9`** | Extensions du BASIC (`LPEEK`, `WPEEK`, `LPOKE`, `MOD`) et **sondes** du Function Driver (`SQR`, `ADDTEST`, `DIVTEST`, `MATTEST`), chacune avec son programme BASIC d'essai. | Validés sur PC-E500S et PockEmul, septembre 2026 ; les résultats sont consignés dans l'en-tête de chaque source. | Source `.ASM` → `.OBJ` / `.UU` + `.BAS` d'essai ; journal dans `12-extensions-basic.md`. |
| **`PLINKC162`** | Driver IOCS résident (« Pocket Link Cache »), lecteur virtuel `L:` relié par liaison série à un serveur PC (`APLINKS`). Exemple réel et complet de programme en langage machine + protocole. | Archive historique (1999), source assembleur A62/XASM + serveur Win32/DOS. | Source assembleur `PLINKC.asm`/`.BAS` ; protocole documenté dans `Anleitung.txt`. |

**Comment ils se répondent :** `SC62015Disassembler` utilise les mêmes tables de référence que XASM (`Data/OpcodeTable.json` dérive de la même documentation que la table d'opcodes XASM) et se valide contre des `.lst` **produits par XASM lui-même** sur des programmes réels (`register`, `tmap`, `vogue` — voir `Docs/Exemples/`). `xasm2026-4` se valide à son tour contre `xasm2026-1` en réassemblant les mêmes sources. `PLINKC162` est un cas d'usage réel qui mobilise à la fois l'assembleur (écrire le driver) et la structure IOCS documentée en `04-fcs-iocs.md` (installer un en-tête de driver, cf. `03-memoire-et-systeme-pc-e500s.md` §7).

## 3. Chaîne BASIC (tokenisation, édition, transfert)

| Projet | Rôle | État |
|---|---|---|
| **`disbacon`** | Outils MS-DOS 1995 (© E. Kako) : `bacon` (tokeniseur `.BAS`→`.bac`) et `disbacon` (détokeniseur `.bac`→`.BAS`). Round-trip quasi parfait, sauf l'espace après `:` (information perdue par le format lui-même). | Archive historique, sources C d'origine. |
| **`Sharp Basic Converter`** | Réécriture C#/.NET moderne du même problème : décompilateur `.BSA`→`.BAS` et compilateur `.BAS`→`.BSA`, CLI complète (`compress`/`decompress`/`convert`/`roundtrip`/`inspect`). | Décompilateur 30/31 fichiers octet-exact ; compilateur round-trip texte complet, 24/32 octet-exact (8 cas de résolution d'adresse interne non convergents identifiés). |
| **`sharpbasic-c`** | Portage/fusion en C17 de `disbacon`/`bacon`, aligné sur `Sharp Basic Converter` (double implémentation mono-fichier + modulaire, sortie byte-identique). | Compile sans warning, testé contre le même corpus. |
| **`Renum Basic Sharp`** | Renumérote un programme BASIC texte (`RENUM new,old,pas`), en C, documentation et tests en français. | Fonctionnel, corpus de test `Exemples/*.BAS`. |
| **`Sharp transfert serial`** | Transfert de programmes BASIC (texte) entre PC et PC-E500S par liaison série RS-232 : implémentation PowerShell de référence + port C Win32 comportementalement identique, plus `INIT.BAS` (bootstrap côté Sharp). | Validé sur matériel réel. |

Ces cinq projets couvrent ensemble le cycle complet d'un programme BASIC : écriture/édition en texte, renumérotation, tokenisation pour stockage natif sur la machine (le format interne du PC-E500S n'est pas du texte brut), et transfert physique vers/depuis un PC actuel.

## 4. Documentation transverse

| Élément | Contenu |
|---|---|
| **`Mise en forme documents SHARP`** | Versions nettoyées/mises en page (docx + pdf) des deux manuels Sharp retrouvés : *ESR-L CPU Instruction Manual* et *Technical Reference Manual PC-E500* — sources primaires de ce référentiel (voir `07-sources-et-bibliographie.md`). |
| **`SC62015Disassembler/Docs/Doc technique`** | Contient déjà une documentation substantielle : table d'instructions détaillée (`README - PC-E500 Instruction Table.md`), documentation XASM complète (`Documentation_XASM_PC-E500S.md`), et une **carte mémoire PC-E500S déjà dépouillée en français** (`PCE500 Description mémoire.xls.xlsx`) — c'est la source principale de `03-memoire-et-systeme-pc-e500s.md`. |
| **`Renum Basic Sharp/Documentation`** | Table des codes/tokens BASIC (`Codes_BASIC_PC-E500S.md` + `.csv`), table de caractères CP437, manuels PC-E500/PC-E500S en anglais et allemand. Pertinent surtout pour la chaîne BASIC (§3), hors du périmètre strict SC62015 de ce référentiel. |

## 5. Ce que ce référentiel ajoute (par rapport à l'existant)

- Une **synthèse unique et en français** de l'architecture CPU (`01`) et du jeu d'instructions (`02`), qui n'existaient jusqu'ici qu'en anglais dans `Doc technique`, ou éparpillées entre plusieurs `CLAUDE.md` de projets différents.
- Une **mise en forme structurée** de la carte mémoire système (`03`) et du catalogue FCS/IOCS (`04`), qui n'existaient que sous forme de tableur brut (`.xlsx`) ou de manuel scanné (PDF), avec en prime le **comblement de 8 fonctions FCS** absentes de `Data/FCSFunctions.json` (`00H`, `08H`, `0AH`-`0FH`) retrouvées dans le manuel officiel.
- Une **cartographie explicite** de l'écosystème logiciel (`06`, ce fichier), qui n'existait nulle part sous forme consolidée — chaque projet documentait son propre périmètre sans vue d'ensemble.
- Des **sources internationales** (Japon, Allemagne) non exploitées jusqu'ici dans la documentation du projet : historique de XASM/DISASM12 par leur auteur d'origine, communauté de rétro-ingénierie japonaise actuelle (2024-2026), collections et notes techniques allemandes (`07-sources-et-bibliographie.md`).

## 6. Voir aussi

- `05-format-fichiers-et-xasm.md` — détail du format objet XASM et des formats de sortie, commun à toute la chaîne langage machine.
- `07-sources-et-bibliographie.md` — provenance de chaque source, y compris les pages japonaises et allemandes.
