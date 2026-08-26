# Référentiel Sharp PC-E500S / SC62015

Référentiel technique en français sur le Sharp PC-E500S et son CPU **SC62015 (ESR-L)**, constitué pour compléter l'écosystème d'outils déjà réalisé dans `C:\Claude` (cross-assembleur XASM, désassembleur, outils BASIC, driver IOCS...). Il consolide les manuels Sharp d'origine, la rétro-ingénierie déjà menée dans ces projets, et des sources communautaires japonaises et allemandes non exploitées jusqu'ici.

## Sommaire

1. [**Architecture du CPU SC62015**](./01-architecture-cpu-sc62015.md) — identité du CPU, registres, espaces mémoire interne/externe, pagination, modes d'adressage, pile système/utilisateur, interruptions, HALT/OFF/RESET.
2. [**Jeu d'instructions**](./02-jeu-instructions.md) — table des opcodes 16×16, encodage des registres, table des PRE-bytes, sommaire des 65 mnémoniques (256 opcodes).
3. [**Mémoire et système du PC-E500S**](./03-memoire-et-systeme-pc-e500s.md) — registres d'E/S `F0h`-`FFh` détaillés bit à bit, adresses système internes et externes, cartes mémoire (`S1`/`S2`/`B`, FRAM), vecteurs d'interruption, chaîne des drivers IOCS.
4. [**FCS et IOCS**](./04-fcs-iocs.md) — catalogue des appels système (niveaux 0/1/2), fonctions FCS (`00H`-`10H`), structure d'en-tête de driver, plages de commandes IOCS, codes d'erreur.
5. [**Formats de fichiers et XASM**](./05-format-fichiers-et-xasm.md) — syntaxe et directives XASM, formats objet (en-tête 16 octets, `-TB`/`-TH`/ROM/brut), sorties (`.hex`/`.s19`/`.map`/`.uu`...), modes d'adressage en syntaxe assembleur, diagnostics.
6. [**Écosystème d'outils**](./06-ecosysteme-outils.md) — cartographie de tous les projets déjà réalisés (XASM ×3 générations, désassembleur, PLINKC162, outils BASIC, transfert série) et de la façon dont ils s'articulent.
7. [**Sources et bibliographie**](./07-sources-et-bibliographie.md) — provenance détaillée de chaque information : manuels Sharp, rétro-ingénierie interne, sources japonaises et allemandes.
8. [**Cartes mémoire**](./08-cartes-memoire.md) — cartes officielles Sharp, inventaire technique des cinq cartes tierces photographiées (CC Sharp Card 128 Ko, carte M. Kemper 256 Ko [1993], carte Böttcher/M.K. v4.1 256 Ko [1998, seconde génération du même concepteur], carte Becker & Partner 256 Ko [fabricant indépendant, Aachen], M. Kemper/Dynatech 1024 Ko), et pistes concrètes (puces FRAM `MB85R`) pour fabriquer de nouvelles cartes 256/512/1024 Ko.
9. [**Cartes mère — RAM interne**](./09-cartes-meres-ram-interne.md) — diagnostic comparatif de deux cartes mère (32 Ko stock vs 256 Ko), confirmation communautaire du schéma 2×128 Ko, carte mémoire externe précise par ligne `CE`, limites réelles observées en combinant RAM interne et carte.
10. [**Fabrication carte FRAM 1024 Ko**](./10-fabrication-carte-1024ko-fram.md) — nomenclature chiffrée (composants, fournisseurs, coût total estimé), schéma fonctionnel proposé (décodeur, registre de banque, sélection des puces) et proposition de PCB (format, empilage, placement), avec hypothèses de conception clairement signalées.
11. [**Fabrication carte FRAM 256 Ko**](./11-fabrication-carte-256ko-fram.md) — même exercice pour une carte 256 Ko (sans registre de banque, plus simple et moins chère), avec prix de référence du produit commercial japonais équivalent (`tmfg.jp`, ~80 €) et découverte annexe d'un service commercial de transformation RAM interne 32K→256K.

## Comment utiliser ce référentiel

- Pour écrire ou relire du code assembleur SC62015 : commencer par `01` (registres/adressage) et `02` (instructions), puis `05` pour la syntaxe XASM concrète.
- Pour comprendre ou étendre les annotations du désassembleur (appels système, adresses connues) : `03` et `04`.
- Pour situer un projet existant dans l'ensemble ou décider où ajouter un nouvel outil : `06`.
- Pour vérifier ou sourcer une affirmation : `07`.

## Principe de non-duplication

Ce référentiel **synthétise et traduit** plutôt que de dupliquer intégralement les tables déjà exhaustives présentes dans le projet (notamment `SC62015Disassembler/Docs/Doc technique/README - PC-E500 Instruction Table.md`, 417 lignes anglaises, et `Data/OpcodeTable.json`, 256 entrées). Chaque fichier renvoie explicitement vers ces sources pour le détail octet-par-octet (encodage exact, cycles) plutôt que de le recopier une seconde fois. La valeur ajoutée est : la synthèse en français, la mise en cohérence entre sources parfois divergentes, le comblement de lacunes identifiées (fonctions FCS manquantes, carte mémoire mise en forme, sources internationales), et la cartographie de l'ensemble.

## Statut

Version initiale — juillet 2026. À enrichir au fil de l'eau (nouvelles adresses système identifiées, nouveaux outils, corrections). Les emplacements exacts des fichiers sources cités sont donnés dans `07-sources-et-bibliographie.md` pour faciliter les mises à jour futures.
