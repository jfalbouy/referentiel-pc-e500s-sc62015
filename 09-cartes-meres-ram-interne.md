# Cartes mère — RAM interne 32 Ko vs 256 Ko

*Rédigé le 2026-08-26 — mis à jour le 2026-09-25*

> Voir `00-index.md` pour la vue d'ensemble. Ce fichier documente les deux cartes mère photographiées dans `Photos carte mère Sharp/` (« Sharp 32K » et « Sharp 256K »), pour comprendre comment Sharp a physiquement implémenté les deux configurations de RAM interne, en complément de `08-cartes-memoire.md` (cartes mémoire amovibles).

## 1. Carte mère « 32 Ko » (configuration standard)

*Photos : `Sharp 32K/IMG_2426.jpg` à `IMG_2432.jpg` — carte complète, plusieurs angles rapprochés.*

Carte mère complète (dos du clavier, boîtier ouvert), marquage PCB **« N1105 ECZA »**, connecteurs flat-cable **« JAE 15S »** de part et d'autre (clavier/afficheur). Composants identifiés :

| Réf. | Marquage | Fonction |
|---|---|---|
| CPU | `SC62015C01 2J 13` | Processeur ESR-L, boîtier QFP. |
| ROM | `LH534HF9 LSI LOGIC JAPAN D331 02 B` | ROM système Sharp (masque « LH534HF9 »), 256 Ko. |
| RAM interne | `MITSUBISHI 32811B M5M5256BFP-12L` | SRAM CMOS **32K×8** (256 Kbit), 120 ns — **seule puce RAM interne, 32 Ko au total**, occupant la fenêtre haute `B8000H`-`BFFFFH` de `CE0` (voir `03-memoire-et-systeme-pc-e500s.md` §1bis-1ter). |
| Pilotes LCD | 2× `HD61202 3G43 JAPAN` | Circuits pilotes de segments LCD Hitachi (chacun gère une partie de la matrice d'affichage). |
| Petit CI près de la RAM | `3V01F 3B` | Non identifié avec certitude — probablement un circuit de supervision de tension/gestion de la pile de sauvegarde. |
| Pile principale | Bouton lithium (CR2016 attendu d'après les spécifications officielles) | Sauvegarde RAM interne + horloge hors alimentation principale. |
| Molette | Potentiomètre rotatif | Réglage du contraste LCD. |

## 2. Carte mère « 256 Ko » (variante haut de gamme / import)

*Photos : `Sharp 256K/s-l1600-4.jpg`, `s-l1600-5.jpg` — photos de type petites annonces (nommage `s-l1600`), probablement une référence externe plutôt qu'une carte possédée physiquement par l'utilisateur.*

Même famille de carte (marquage PCB **« N1105 ECZA »** identique, mêmes connecteurs `JAE 15S`, mêmes 2× `HD61202`), mais :

| Réf. | Marquage | Fonction |
|---|---|---|
| CPU | `SC62015...` (même famille, lecture partielle) | Processeur ESR-L. |
| ROM | **`LH53471K SHARP JAPAN 0012 D`** | ROM système — **masque différent** de la carte 32 Ko (`LH534HF9`). Voir §3. |
| RAM interne | **2× `V62C5181024L-70W` / `V62C5181024LL-70W`** | SRAM CMOS **128K×8 chacune** (1 Mbit), 70 ns — **2 puces → 256 Ko au total**, occupant la fenêtre `CE0` complète `80000H`-`BFFFFH`. |
| Fils de raccordement | Nombreux fils rouges soudés directement sur les broches des puces RAM et vers d'autres zones du PCB | **Indice de modification manuelle** (voir §3). |

Une carte mère PC-E500S 256 Ko *différente* (import allemand, décrite par un utilisateur du forum silicium.org en 2015, cf. §4) porte deux puces **Sony `CXK581000AM`** (également 128K×8) à la place des `V62C5181024L` — confirmation indépendante que le schéma « 2×128 Ko » est bien la solution employée par Sharp pour cette configuration, avec au moins deux fournisseurs de SRAM différents selon le lot de production.

## 3. Lecture technique : simple repeuplement ou modification manuelle ?

Deux hypothèses, non tranchées par les seules photos :

1. **SKU Sharp d'origine** : Sharp aurait produit une variante « 256 Ko » du PC-E500S avec une ROM légèrement différente (`LH53471K`) intégrant le support du double espace de RAM interne — cohérent avec le témoignage du forum silicium.org (§4), où l'utilisateur reçoit une machine 256 Ko *neuve d'origine* (pas modifiée par un tiers) en provenance d'Allemagne, et où un autre intervenant confirme : « les deux machines sont fonctionnellement identiques à la ROM près, qui ajoute quelques fonctions [...] pas 128 Ko de ROM en plus, juste quelques 16 Ko utiles ».
2. **Modification manuelle a posteriori** : sur la carte photographiée ici (`s-l1600-4/5.jpg`), la présence de nombreux **fils de raccordement rouges soudés à la main** sur/autour des puces RAM suggère fortement qu'il s'agit d'une carte **modifiée par un particulier** (remplacement de la RAM d'origine par des puces plus grandes + câblage additionnel des lignes d'adresse/sélection manquantes), et non d'un exemplaire d'usine tel quel.

**Ces deux hypothèses ne s'excluent pas** : Sharp a bien commercialisé une variante 256 Ko d'origine (§4), ce qui rend une modification artisanale plausible et techniquement réalisable en s'inspirant du même schéma (2 puces 128 Ko + câblage des lignes d'adresse supplémentaires) sur une carte 32 Ko de base.

📖 **La seconde hypothèse n'est plus une conjecture** : une source d'époque documente la
modification pas à pas, brochages et schémas compris, et va jusqu'à 1 Mio interne (§4bis). Des fils
soudés à la main sur les broches des RAM sont donc la signature d'une pratique courante, et non
l'indice d'un bricolage isolé.

## 4. Confirmation communautaire (forum silicium.org, 2015)

Fil « **PC-E500S 256K** » (`forum.silicium.org/viewtopic.php?t=39164`, juillet 2015) — témoignage détaillé et directement exploitable :

- L'auteur reçoit un PC-E500S 256 Ko « équipé d'une carte 256K sauvegardée par pile » (carte mère, pas carte amovible) en provenance d'Allemagne. Machine et clavier légèrement plus grands que le PC-E500S standard.
- Confirmation par un autre intervenant : différence entre les deux machines = ROM uniquement, « pas 128 Ko de ROM en plus, juste quelques 16 Ko utiles » — cohérent avec l'observation `LH534HF9` (32 Ko) vs `LH53471K` (256 Ko) de ce référentiel.
- **Ouverture de la machine** : confirmation physique de **2 puces SRAM 128 Ko `Sony CXK581000AM`** sur la carte mère.
- **Comportement `FRE0` mesuré** :
  - `FILES "S1:"` retourne toujours un répertoire à 4 entrées (`DATA.BAS`, `TEXT.BAS`, `FUNCKEY`, `AER`), carte ou pas — c'est la RAM interne elle-même qui joue le rôle de `S1:`.
  - PC-E500 (32 Ko) + carte 32 Ko, `MEM$="B"` → `FRE0=61368` (fusion RAM interne + carte, fonctionne).
  - PC-E500S (256 Ko) + carte, `MEM$="S2"` → `FRE0` passe de 257935 à 32116 (carte utilisée **seule**, fonctionne).
  - PC-E500S (256 Ko) + carte, `MEM$="B"` → **`Out of memory`** (la fusion échoue, alors que la même manipulation réussit à plus petite échelle avec 32 Ko).
  - PC-E500 (32 Ko) + carte 256 Ko, `MEM$="B"` → **`Out of memory`** également.
  - Carte 256 Ko dans le slot `S1:` d'un **PC-1360** → non reconnue (`*`). Expliqué par un intervenant (Rom1500, spécialiste des cartes PC-1500/1600) : la carte utilise les lignes d'adresse **`A15`, `A16`, `A17`** sans résistances de tirage (*pull-up*) intégrées à la carte elle-même — ces lignes sont non connectées (`NC`) sur le PC-1360, qui ne peut donc pas les lire correctement. **Ceci confirme qu'au moins `A15`-`A17` sont disponibles sur le connecteur de carte**, et que les cartes de grande capacité s'appuient sur ces lignes hautes pour sélectionner leur banc — cohérent avec l'analyse de `08-cartes-memoire.md` §1/§3.
- La carte mémoire externe par ligne `CE` citée par un intervenant (reproduite en `03-memoire-et-systeme-pc-e500s.md` §1bis) provient de ce fil.

## 4bis. Une source d'époque : le « Guide de modification de la série E500 » (1996)

*`Zou/Guide_modification_serie_E500_2e_edition_FR.md` — 2ᵉ édition, 1996, « #4041 Lycanthrophy
nomi », traduit du japonais le 2026-09-10. Corpus local, hors dépôt (`07` §3bis). 📖 **Lu, non
éprouvé** : rien n'a été soudé ni mesuré ici.*

Ce document tranche la question laissée ouverte au §3 : **la modification artisanale de la RAM
interne était une pratique documentée**, avec son guide, ses brochages et ses schémas. Il donne :

- les **brochages** des trois SRAM qui comptent pour ces machines : `HM62256` (32 Kio, 28 broches),
  `HM628128` (128 Kio, 32 broches), `HM628512` (512 Kio, 32 broches) — la deuxième est exactement
  la classe de puce observée sur la carte mère 256 Ko du §2 (`V62C5181024L`, `CXK581000AM`) ;
- le **brochage du CPU**, 100 broches, avec ses groupes (données, adresses, entrées clavier, port E,
  horloge) ;
- les circuits de décodage employés (`74HC00`, `74HC02`, `74HC138`, `74HC139`, `74HC157`,
  `74HC158`/`74HC258`) et les précautions de dessoudage/empilage ;
- un **schéma complet** portant la RAM interne d'un E500/E550 à **1 Mio** avec deux puces de
  4 Mbit, réparti en `S1:` 256 Kio + `S2:` 256 Kio + **`D:` 512 Kio**.

Quatre points valent d'être retenus, parce qu'ils ne se déduisent pas des photos :

1. **`D:` n'est pas un lecteur de la ROM** : c'est le disque RAM du pilote **DELTA** (version 3.4 ou
   3.5, `07` §3bis), « comparable à une extension EMS ». La capacité de 512 Kio n'existe donc que
   par un pilote résident, et suppose la mémoire **bancaire** que le schéma câble.
2. **`S2:` devient interne** : la modification intègre les 256 Kio de la carte, ce qui rend la carte
   amovible inutile et **libère le connecteur** pour autre chose (l'auteur cite un disque dur).
3. ⛔ **La modification ne s'applique pas au E650** : sa zone ROM, plus grande, entre en conflit avec
   la zone employée par DELTA. Une limite d'implantation mémoire, pas de soudure.
4. Sur **E500** les deux puces doivent être **empilées** ; sur **E550** elles tiennent côte à côte.

Le schéma est repris de deux articles de la même communauté, cités par leur numéro, leur date et
leur taille : **Daris**, « DELTA avec une RAM 4 Mbits ! » (1995-09-21) et **ganze**, « RAM 4 Mbits
`S1:`256 `D:`256 ou `S2:`256 interne » (1996-02-26) — ce dernier permettant de **choisir** entre
`S2:` et `D:` pour la seconde moitié. L'existence de ces articles donne une piste de recherche
précise, et une datation : la pratique était établie dès 1995.

## 5. Conséquences pour un projet de carte mémoire (256/512/1024 Ko)

1. **Le schéma « 2 puces 128 Ko » observé ici pour la RAM interne 256 Ko est distinct de celui des cartes amovibles** (`08-cartes-memoire.md`) mais partage le même principe : combiner deux puces standard plutôt que chercher une puce unique de grande capacité — argument supplémentaire en faveur de l'approche « 2× FRAM 128 Ko + décodeur » proposée pour une carte FRAM 256 Ko amovible.
2. **Résistances de tirage sur `A15`-`A17`** : à prévoir explicitement sur toute nouvelle carte visant une compatibilité large (le PC-1360 les laisse flottantes) — point concret à ajouter aux vérifications de `08-cartes-memoire.md` §6.
3. **Ne pas promettre la fusion `MEM$="B"` au-delà de ~256-288 Ko** dans la documentation d'une carte DIY : le comportement réel observé montre une limite non documentée par Sharp, indépendante de la limite théorique de 512 Ko du bus (`01-architecture-cpu-sc62015.md` §3.2). Les modes `S1`/`S2` (carte utilisée seule) restent en revanche fiables à toute capacité testée.

## 6. Voir aussi

- `03-memoire-et-systeme-pc-e500s.md` §1bis-1ter — carte mémoire externe complète par ligne `CE`, fenêtre de RAM interne par modèle.
- `08-cartes-memoire.md` — cartes mémoire amovibles (SRAM/FRAM), même logique de décodage par puces combinées.
- `07-sources-et-bibliographie.md` — détail de la source forum silicium.org, et §3bis pour le corpus japonais dont vient le guide du §4bis.
- Dossier `Photos carte mère Sharp/` — photos sources de ce fichier, **conservées hors du dépôt** (`NOTICE.md`).
