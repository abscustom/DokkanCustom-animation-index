# DokkanCustom live animation asset plan

This repository is the **small public animation index**, not the place to put every LWF or texture. The website in `DokkanCustom` remains the GitHub Pages site. When someone presses a Super Attack button, their browser reads a small JSON file here, then downloads only the character rig, effects, script, background, and audio used by that one animation.

## What the final flow looks like

```text
DokkanCustom GitHub Pages
  -> animation index JSON in this repository
  -> direct LWF, PNG, Lua, USM, and audio URLs in public asset repositories
  -> existing ActionBank/LWF player renders the attack in the visitor's browser
```

There is no VPS, ngrok, Cloudflare R2, GitHub Release download, or MP4 library in that flow.

## Why the assets need to be split

The current local source is:

```text
C:\Users\Ruffy\Desktop\CardHub Console\local-hosting\assets
```

Measured live-animation source sizes:

| Source folder | Size |
| --- | ---: |
| `ingame/battle/sp_effect` | 8.73 GiB |
| `ingame/battle/effect` | 1.15 GiB |
| `ingame/battle/character` | 0.92 GiB |
| `movie` | 1.11 GiB |
| `lua` | 0.11 GiB |
| `ingame/battle/bg` | 0.06 GiB |
| `se` and `voice` | 0.23 GiB |

Do not place all of that in this repository or add it to `DokkanCustom`. `sp_effect` alone is already 8.73 GiB. Keep every public Git repository below about 5 GiB, use ordinary Git files under 100 MiB, and do not use Git LFS for browser-loaded files.

Create these three public asset repositories beside this index repository:

| Repository | Content | Estimated size |
| --- | --- | ---: |
| `DokkanCustom-animation-spfx-a` | `sp_effect_a0_*` through `sp_effect_a4_*` | 3.70 GiB |
| `DokkanCustom-animation-spfx-b` | `sp_effect_b1_*` through `sp_effect_b4_*` | 4.05 GiB |
| `DokkanCustom-animation-core` | `sp_effect_a5_*` through `sp_effect_a9_*`, `effect`, `character` except `idle`, `bg`, `lua`, `movie`, `se`, and `voice` | about 4.57 GiB |

The current idle packs stay in the existing DokkanCustom asset setup. They are not copied into these Super Attack repositories.

## Folder layout in every asset repository

Keep the original game-relative names. Do **not** use the old `super-attacks/battle` publish layout, because the player and the static index need the native paths.

```text
assets/
  ingame/battle/
    character/00001/battle/...
    character/00001/sp01/...
    effect/battle_150000/...
    sp_effect/sp_effect_a1_00001/...
    bg/battle_bg_00001/...
  lua/ab_script/attack_sp/sp0001.lua
  lua/ab_script/active_skill/as0001.lua
  movie/en/ingame/battle/sp_effect/...
  se/...
  voice/...
```

The `character/<id>/idle` folders stay out of this new asset set. Keep `battle` and `spXX` folders because Super Attacks need both the battle rig and the special-motion rig.

## What to copy from CardHub Console

The existing extractor already downloads and verifies the right source folders. It does not need to be replaced.

1. Run the normal importer in CardHub Console for `effects`, `motions`, `backgrounds`, `movies`, and `lua` when updating game files.
2. Copy the resulting native folders from `local-hosting/assets` into the three repositories above. Split only `sp_effect` by its pack prefix; do not rename any file or folder.
3. Leave `.cpk` archives, updater working folders, `cache`, and `staging` out of all asset repositories. Only publish the extracted playable files: `.lwf`, `.png`, `.jpg`, `.webp`, `.lua`, `.usm`, `.acb`, and `.awb` where required.
4. Keep card-art texture assets in the existing general-assets repository. The static card metadata will continue to point at that repository for cut-ins, names, phrases, and card art.

`tools/sync-mumu.js` is the importer to keep using. Its groups already match the needed sources:

```text
effects      -> ingame/battle/effect and ingame/battle/sp_effect
motions      -> ingame/battle/character
backgrounds  -> ingame/battle/bg
movies       -> movie/.../battle/effect and movie/.../battle/sp_effect
lua          -> lua
```

`tools/sync-idle-assets.mjs` remains the separate idle sync; it should not publish idle files into these Super Attack repositories.

## The missing piece: static metadata

The current local player asks the CardHub server for endpoints such as these:

```text
/api/card/<cardId>
/api/effect/<effectId>
/api/animation/<scriptName>
/api/level-bg/<backgroundId>
/api/se?cue=<cueId>
/api/voice?cue=<cueId>
```

GitHub raw files cannot generate those endpoints. The fix is to export their JSON responses once during asset publishing and store them in this repository:

```text
api/card/4031731.json
api/effect/1507.json
api/animation/sp2910.json
api/level-bg/79.json
manifest/asset-roots.json
manifest/version.json
```

Each JSON response must have the same fields the local server returns today, but every `url` must be a direct raw URL to the correct asset repository. For example:

```text
https://raw.githubusercontent.com/abscustom/DokkanCustom-animation-spfx-a/main/assets/ingame/battle/sp_effect/sp_effect_a1_00001/sp_effect_a1_00001.lwf
```

The local server already has the logic to export. The relevant source is `C:\Users\Ruffy\Desktop\CardHub Console\tools\dokkan-animation-server.mjs`:

- `cardPayload()` builds card-to-character and card-texture metadata.
- `effectPayload()` builds effect-pack metadata.
- `animationPayload()` builds the Lua source and referenced effects.
- `levelBgPayload()` builds battle-background metadata.
- `listPackFiles()` and `listCharacterLwfFiles()` enumerate each LWF and texture file.

Add a new CardHub Console script named `tools/export-static-animation-index.mjs`. It should reuse those functions or their extracted shared helpers, write the JSON files above, and route every source file through `asset-roots.json` according to its path or effect pack prefix. This exporter replaces the old release-zip packaging path for live playback.

## DokkanCustom code changes

The current production player still has old localtunnel/ngrok defaults, and its live players request server endpoints. Change the website in this order:

1. Add one static index base constant, for example:

   ```text
   https://raw.githubusercontent.com/abscustom/DokkanCustom-animation-index/main
   ```

2. Replace the remote animation bridge default in `js-card-details/card-animation-player.js` with that index base. Keep `http://127.0.0.1:3137` as the local development override.

3. Update `js-card-details/action-bank-runner.js`, `js-card-details/chara-layer.js`, and `js-card-details/battle-bg-layer.js` so production metadata requests load static files with the `.json` extension:

   ```text
   /api/card/<id>.json
   /api/effect/<id>.json
   /api/animation/<name>.json
   /api/level-bg/<id>.json
   ```

   Local development must keep using the current extensionless endpoints.

4. Keep every LWF, PNG, Lua, USM, and audio file URL absolute in the exported JSON. Then the existing `LwfPackPlayer`, `ActionBankRunner`, `CharaLayer`, and `BattleBgLayer` can fetch the right files without knowing which asset repository owns them.

5. Export direct URLs for audio and USM files. Do not depend on `/api/movie`, `/api/se`, or `/api/voice`, because those are server-only routes today.

6. Remove the production ngrok/localtunnel fallback only after the static index works in a normal GitHub Pages visit.

## Validation before moving the full library

Use one known Super Attack as the first proof: card `1000011`, script `sp0001`.

1. Publish only the required packs and metadata for that attack.
2. Open the deployed DokkanCustom page in a normal browser window, with no local CardHub server running.
3. Confirm the player loads the attacker and enemy rigs, effects, Lua, background, sound, and KO result from raw GitHub URLs.
4. Check a modern Super Attack, an Active Skill, a tall Intro/Standby animation, and a KO screen.
5. Only then publish the remaining packs and remove the duplicate Super Attack files from `DokkanCustom`.

## What stays and what changes

Keep the MP4 converter as a local test and optional export tool. Do not use its generated video library as the deployed asset library.

Keep this repository small. Its job is versioned metadata and routing. The three asset repositories hold the actual files. This avoids the 1 GiB GitHub Pages size limit and stops the site from needing a private local server.
