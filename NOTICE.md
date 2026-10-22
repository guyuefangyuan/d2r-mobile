# Scope and copyright notice

[中文版 / Chinese](NOTICE.zh-CN.md)

## What this repository is

`d2r-mobile` is a **compatibility framework**: it translates the graphics and system calls that
the Nintendo Switch build makes (NVN, HID, filesystem, …) onto Android (Vulkan + JNI), so a game
you own can run on an Android device — **provided you supply your own game files**.

This repository publishes **framework source only**: C / Objective-C / Java / Python sources,
build scripts, tests and documentation.

## What is *not* here

None of the following is in this repository, and none will be provided. Please do not ask for it
in issues or pull requests:

- any game data or game code: `game.romfs`, the `main` / `sdk` / `subsdk*` NSO images, shader
  packages, textures, audio, models, UI layouts, fonts, `*.nsp` / `*.xci`, `prod.keys` /
  `title.keys`;
- anything derived from the above: extracted assets, HUD atlases, rewritten game data packages
  (for example `data/00-00000007.data`), patched module images;
- packaged builds: no APK, no IPA, no distributable binary;
- any tool, link, torrent or key that helps you obtain the game.

The only game-related artefacts in the tree are a few TSV tables (`dm/gfx/nvn/shader_map.tsv`,
`shader_facts.tsv`, `pso_manifest.tsv`). They record **factual observations** such as
"shader hash → which resource types it binds". They are produced by the scripts under
`dm/gfx/tools/` from the author's own copy of the game and contain no game code and no artwork.
You can regenerate them from your own copy.

## What you have to bring

- a legitimately purchased Nintendo Switch copy of *Diablo II: Resurrected*, including the update
  patch, plus the keys you dumped from your own console;
- the extraction and workspace preparation steps live in `setup.sh` and
  `docs/DEPENDENCIES.zh-CN.md`.

This repository does **not** provide, and does not advise on, any way to obtain the game files.

## Trademarks and affiliation

*Diablo II: Resurrected* and its names, artwork and audio are trademarks and/or copyrighted
property of Blizzard Entertainment, Inc. *Nintendo Switch* is a trademark of Nintendo.
This project is **not affiliated with, sponsored by or endorsed by** any of them. The game name
is used only to describe what the framework is compatible with (nominative use).

This repository contains **no** Blizzard or Nintendo artwork. The launcher icon (Android
`mipmap-*`, `drawable-nodpi/ic_launcher_foreground.png`, iOS `AppIcon1024.png`) is the upstream
project's own AI-generated original emblem; the prompt it was made from is recorded in
`dm/android/artwork/README.md` and explicitly asks for "not a copy of an existing game's icon".
Replace it with your own branding freely.

## Licence

- This repository is based on [dodola/d2r-mobile](https://github.com/dodola/d2r-mobile) (MIT);
  its `LICENSE` and copyright notices are kept.
- The changes here are released under **MIT** as well; see `LICENSE`.
- The third-party components under `dm/third_party/` keep their own licences (Ryujinx: MIT;
  libdeflate: MIT; tinyexr / stb: MIT or Public Domain). See the licence files inside those
  directories.

## Disclaimer

This software is provided "as is", without warranty of any kind. You are responsible for checking
whether using it is legal where you live, and you use it at your own risk. Do not use this
repository to distribute the game or any other copyrighted content.
