# Référentiel Sharp PC-E500S / SC62015

*Rédigé le 2026-09-25 — mis à jour le 2026-09-25*

Un référentiel technique **en français** sur le **Sharp PC-E500 / PC-E500S / PC-E550** et son CPU
**SC62015 (ESR-L)** : architecture, jeu d'instructions, carte mémoire système, appels FCS/IOCS,
formats de fichiers et assembleur XASM, cartes mémoire, et l'extension du BASIC par ses deux
crochets.

**➡️ Entrez par [`00-index.md`](00-index.md)**, qui présente les treize chapitres et dit lequel lire
selon la question posée.

## Ce que ce référentiel est

Une **synthèse**, pas une compilation : les manuels Sharp retrouvés (*ESR-L CPU Instruction
Manual*, *Technical Reference Manual PC-E500*) lus et recoupés, la ROM désassemblée, des mesures
faites sur des machines réelles et sur émulateur, et des sources communautaires japonaises et
allemandes mises en cohérence — le tout en français, avec la provenance de chaque affirmation.

Deux conventions gouvernent le texte, et elles expliquent sa forme :

- **une convention de preuve** : ✅ mesuré sur machine ou par un outil · 📖 lu dans la ROM ou dans
  une documentation, non éprouvé · ⚠️ piège ou point incertain · ⛔ **erreur corrigée, gardée écrite
  avec ce qui l'a démentie**. Les erreurs ne sont pas effacées : savoir ce qui a été cru à tort, et
  pourquoi, coûte moins cher que de le recroire ;
- **la non-duplication** : quand une table exhaustive existe déjà ailleurs, le chapitre y renvoie
  plutôt que de la recopier.

## Ce qu'il ne contient pas

Ni manuels Sharp, ni logiciels tiers, ni photos : ces sources sont **citées précisément** mais
**pas redistribuées** — elles appartiennent à leurs auteurs. Voir [`NOTICE.md`](NOTICE.md).

## Les projets qui l'accompagnent

Le référentiel documente un écosystème d'outils (cartographié au chapitre
[`06`](06-ecosysteme-outils.md)), dont ces dépôts publics :

- [`sharp-pce500s-basext`](https://github.com/jfalbouy/sharp-pce500s-basext) — quatorze mots-clés
  ajoutés au BASIC de la machine (`LPEEK`, `MOD`, `TRIM$`, `XCONSOLE`…) ;
- [`sharp-pce500s-basext-drv`](https://github.com/jfalbouy/sharp-pce500s-basext-drv) — les mêmes,
  résidents dans un pilote de `S1:` ;
- [`sharp-pce500-plinkc-`](https://github.com/jfalbouy/sharp-pce500-plinkc-) — le lecteur virtuel
  `L:` de 1999, archive intacte et version modernisée ;
- [`XASM2026_CSharp`](https://github.com/jfalbouy/XASM2026_CSharp) et
  [`XASM-SHARP_PC-E500S`](https://github.com/jfalbouy/XASM-SHARP_PC-E500S) — l'assembleur croisé ;
- [`renum-basic-sharp`](https://github.com/jfalbouy/renum-basic-sharp),
  [`sharp-transfert-serial`](https://github.com/jfalbouy/sharp-transfert-serial) — outils BASIC et
  transfert série.

## Licence

**PolyForm Noncommercial License 1.0.0** — usage non commercial ([`LICENSE`](LICENSE)).
© 2026 Jean-François Albouy.

Une correction, une source à ajouter, une affirmation à démentir : ouvrez une *issue*. Une mesure
qui contredit ce qui est écrit ici est la bienvenue — c'est ainsi que ce document s'est construit.
