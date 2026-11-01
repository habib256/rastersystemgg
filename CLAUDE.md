# CLAUDE.md

Hub d'orientation pour Claude Code. **Ces instructions priment.**

Sesame — émulateur Sega Master System pédagogique. C++17, GLFW3 + OpenGL
(immediate mode), aucune dépendance dans le cœur. Développé sur macOS Silicon.
**Commentaires et documentation en français ; interface et journaux en ANGLAIS.**

Architecture en deux mots : **le `Bus` *est* le plan mémoire** (route
read8/write8 et in/out vers les puces) et **`sesame_core` ne dépend pas du GUI**.

## Où trouver quoi

| Doc | Contenu |
|-----|---------|
| [`README.md`](README.md) | Présentation et usage (en anglais, public). |
| [`docs/IMPLEMENTED.md`](docs/IMPLEMENTED.md) | Ce qui est fait, par puce. |
| [`TODO.md`](TODO.md) | Uniquement l'ouvert : plan en phases (audit 2026-08) + moyen terme. |
| [`CHANGELOG.md`](CHANGELOG.md) | Historique des chantiers terminés, par sous-système. |
| [`docs/RECHERCHE-EMULATEURS.md`](docs/RECHERCHE-EMULATEURS.md) | État de l'art des émulateurs SMS/GG open-source, licences, pistes roadmap. |

## Build & run

```sh
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j    # cibles : sesame (GUI), sesame-headless, sesame_core
./build/sesame <rom.sms>
```

Aucun sous-module : le cœur est autonome ; le GUI requiert GLFW (brew/pacman).

⚠ Ne PAS faire `rm -rf build` ; `cmake -B build` reconfigure.

## Tester = le headless (outil n°1)

Pas de framework de test : validation par `sesame-headless` (déterministe,
traces + captures PPM) et les ROM de test générées par `tools/`.

```sh
python3 tools/make_test_rom.py            # génère roms/selftest.sms
./build/sesame-headless roms/selftest.sms --frames 120 --screenshot s.ppm
./build/sesame-headless <rom> --frames N --trace t.txt
./build/sesame-headless <rom> --frames N --sdsc    # console de debug homebrew
python3 tools/run_selftests.py            # la suite complète
```

## Conventions non négociables

- Manettes : bits **actifs à l'état bas** sur les ports 0xDC/0xDD ; l'API
  `Io::setPad` prend elle des bits actifs à 1.
- Le premier Ko de ROM (0x0000-0x03FF) n'est **jamais paginé** par le mapper
  **Sega standard** (Codemasters, lui, pagine tout).
- Écrire 0xFFFC-0xFFFF touche **à la fois** le miroir RAM et le mapper
  (Sega seulement — les autres mappers ignorent ces adresses).
- /INT du VDP est un **niveau** (tenu tant que le drapeau et son enable sont
  actifs) ; le bouton Pause est un **front** NMI.
- Interfaces publiques des `.hpp` de `src/core/` = contrat : ne changer une
  signature qu'en mettant à jour TOUS les appelants dans le même commit.

## Références matérielles — et leur grade

**Le silicium est la référence.** Toute affirmation sur le matériel porte **son
grade et sa source**, dans un crochet en tête de commentaire, pour qu'un
lecteur et un `grep` s'accordent sur ce qui soutient le code :

| Grade | Preuve | Pour RasterSystemGG |
|---|---|---|
| **A** | Mesure sur **machine réelle** | Oscilloscope ou capture sur une vraie Master System / Game Gear. |
| **B** | **Document primaire** | Fiches techniques (TMS9918/315-5124, SN76489), documentation Sega, SMS Power! quand il cite le matériel. |
| **C** | **Désassemblage de ROM** | BIOS, jeux. Preuve du *contrat* attendu par le logiciel, pas du silicium. |
| **D** | **Un autre émulateur** | MAME (`sms.cpp`, `sn76496.cpp`, `315_5124.cpp`), Emulicious, Meka — à égalité. Lecture du matériel faite par un tiers : utile, faillible. |
| **E** | **Affirmé sans source.** | |
| **F** | **Inconnu assumé** | deviné, ajusté, approximatif, non vérifié. |

Trois règles en découlent. **Une citation de grade D nomme sa révision** (tag
ou commit) : les numéros de ligne amont bougent à chaque commit, et une plage
sans révision ne désigne rien. **Un grade inférieur ne l'emporte jamais sur un
grade supérieur** — là où RasterSystemGG s'écarte de MAME parce qu'une fiche
technique, une ROM ou une mesure dit autre chose, c'est correct, et le
commentaire le dit ; cela protège le code d'un futur « alignement sur MAME »
qui serait une régression. **Un F se garde, il ne se cache pas** : écris ce qui
le trancherait.

```
// [B] Fiche 315-5124 § 3 — le drapeau de collision tombe au sprite 9.
// [D] MAME 315_5124.cpp:812-830 @ mame0287 — idem, tel que MAME le lit.
// [C] BIOS US $0283 — le système attend le bit déjà effacé.
// [A] Mesuré sur une vraie SMS : 228 cycles par ligne.
// [F] Inconnu : ajusté jusqu'à ce que le jeu passe. Tranché par : une capture.
```

Cette hiérarchie est commune à tous les émulateurs du même auteur (POM2, POM1,
POMIIGS, POM68K, POMPPC, NeoST, RasterSystemGG). Convention SDSC pour la console de debug
(ports 0xFC/0xFD) — utilisée par nos ROM de test.
