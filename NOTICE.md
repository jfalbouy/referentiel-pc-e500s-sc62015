# NOTICE — droits d'auteur, licences et crédits

*Rédigé le 2026-09-25 — mis à jour le 2026-09-25*

Ce dépôt est une **œuvre originale** : treize chapitres rédigés en français. Mais il **repose sur
des sources qui ne sont pas les siennes** — manuels du constructeur, logiciels tiers des années
1990, photos, articles de forums et de magazines. Ce fichier dit lesquelles, et ce que le dépôt
redistribue ou non. En cas de doute, la règle la plus restrictive s'applique : **usage non
commercial uniquement**.

---

## 1. Ce que ce dépôt contient, et sous quelle licence

Les treize fichiers `00-index.md` à `12-extensions-basic.md`, écrits par **Jean-François Albouy**
(2026), sous **PolyForm Noncommercial License 1.0.0** ([`LICENSE`](LICENSE)).

Ce sont une **synthèse et une traduction** : les manuels Sharp lus et recoupés, la ROM
désassemblée, des mesures faites sur des PC-E500/PC-E500S réels et sur émulateur, et des sources
communautaires japonaises et allemandes mises en cohérence. Les adresses, les valeurs d'opcode et
les noms de registres qui y figurent sont des **observations sur une machine**, non du code ou du
texte emprunté.

## 2. Ce que ce dépôt ne redistribue PAS, délibérément

Ces documents ont servi à écrire les chapitres, qui les citent précisément — page par page quand
c'est possible — mais ils **ne sont pas versés dans le dépôt** : ils appartiennent à leurs auteurs.

| Source citée | Détenteur | Où elle est citée |
|---|---|---|
| *ESR-L CPU Instruction Manual*, *Technical Reference Manual PC-E500* | **Sharp Corporation** | `01`, `02`, `03`, `04` |
| Manuels utilisateur PC-E500/PC-E500S (anglais, allemand) | Sharp Corporation | `03` §5, `12` §17 |
| Manuels du lecteur de disquettes **CE-140F** (utilisation, service) | Sharp Corporation | `07` §1bis |
| Notice du fabricant de la carte mémoire 1024 Ko | son fabricant (M. Kemper / Dynatech) | `08` §3.3 |
| Photos d'annonces de vente (cartes mère et cartes mémoire) | leurs auteurs | `08`, `09` |
| Corpus d'archives japonaises : DELTA, EXTSLOT, INSTd/INSTt, KNJSC, utilitaires de disque, guide de modification matérielle | leurs auteurs (TORO, « Lycanthrophy nomi », et autres), diffusés sur NIFTY-Serve / POCKET通信, 1992-1996 | `03` §7bis, `07` §3bis, `09` §4bis |

⚠️ **Le cas de `KNJSC` dit pourquoi cette règle vaut mieux qu'un scrupule de principe** : sa propre
notice précise que les sources de *KAKO's tools* dont il dérive ne sont **pas librement
redistribuables**, et qu'il est pour cette raison diffusé sous forme de *fichiers de différences*.

Les photos prises par l'auteur de ce dépôt (`IMG_*.jpg`) restent sa propriété ; elles sont, elles
aussi, gardées hors du dépôt, pour n'y laisser que du texte.

## 3. Sources communautaires citées

Chacune est nommée avec son lien dans [`07-sources-et-bibliographie.md`](07-sources-et-bibliographie.md) :
kako.com (E./Seiji Kako), la bibliothèque de Takayuki Mizuno, **TORO's Library** (高橋良和),
wizforest.com, Electrelic (H. Asano), `andrewwoods3d.com` (Andrew Woods, 1995),
`gikonekos/sc62015-opcode-reference`, **Forth500** (Robert van Engelen), le forum `silicium.org`,
les pages allemandes d'Arno Welzel, `tmfg.jp` et Octopart. Les propos rapportés de forums le sont
en tant que **témoignages**, attribués à leur auteur ou à son pseudonyme.

## 4. Les autres dépôts de l'auteur cités par les chapitres

| Dépôt | Licence | Remarque |
|---|---|---|
| `sharp-pce500s-basext` | PolyForm Noncommercial 1.0.0 | œuvre originale |
| `sharp-pce500s-basext-drv` | PolyForm Noncommercial 1.0.0 | son installateur dérive du `DRIVER_TEMPLATE` et de PLINKC : voir son propre `NOTICE.md` |
| `sharp-pce500-plinkc-` | voir son `NOTICE` | archive de 1999 conservée intacte dans `original/`, à côté d'une version modernisée |
| `sharp-pce500s-keyboard_history` | **privé** | il porte le TSR de *History 1.11* (TORO, 1994) |
| `sharp-pce500s-uuencode` | **privé** | il conserve les binaires MS-DOS de 1993 (R. Marks) comme témoins |

## 5. Marques

**Sharp**, **PC-E500**, **PC-E500S**, **PC-E550**, **CE-140F** sont des marques de Sharp
Corporation. Ce dépôt n'a aucun lien avec Sharp Corporation, ni avec les auteurs des logiciels
qu'il cite. **PockEmul** est l'œuvre de Rémy Rouvin.

## 6. Signaler un problème

Si vous êtes l'auteur d'une source citée ici et que quelque chose vous paraît mal attribué, ou
cité plus largement qu'il ne convient, ouvrez une *issue* : ce sera corrigé.
