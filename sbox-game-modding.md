# Modding a published s&box game locally: from cache to editable project

A field guide for turning a game you have installed from sbox.game into a local, editable s&box
project, so you can tweak it for **your own private use**. Everything here was worked out and
verified on a real published game and is written generically so you can skip the discovery phase.
Names, paths and numbers below are illustrative placeholders; yours will differ.

> **Read the "Legal and ethics" section before anything else.** Short version: this is for private,
> local modding. The code and assets belong to the author. Do not publish, upload, share or
> redistribute any of it, or any build made from it, without the author's written permission.

---

## 1. Legal and ethics

This is not legal advice. It is the practical position, and the safe way to behave.

### What license does the result fall under?

**The same one the original work is under, which for most games means "all rights reserved".
Your modification does not get a new license of its own.**

- A decompiled `.cs` file is the author's code. A converted `.scene` or `.item` is the author's
  content in a different file format. A modded build is a derivative of both. Changing the
  format or adding your own edits does not move the copyright to you.
- **Check what the author declared.** The package listing carries two fields,
  `AssetLicense` and `SoftwareLicense` (see section 3). `0` means *None*, i.e. the author granted
  no license, so the default applies: you may run the game, and you have no right to copy,
  modify-and-share or redistribute it. Only a non-zero value, or a license the author states on
  their page or repository, grants more, and then only on that license's terms.
  The asset license values the platform knows are CC0, CC BY, CC BY-SA and CC BY-NC-ND
  (`engine/Sandbox.Services/Api/Models/Licensing.cs`). Even CC BY-NC-ND forbids sharing modified
  versions.
- **The s&box engine's MIT license does not apply to games built on it.** MIT covers Facepunch's
  engine source (`sbox-public`), not the content of packages published on the platform.
  Facepunch's own docs say that if you are "repackaging someone else's work with no real
  creativity of your own", you should not publish or monetize it
  (`sbox-docs/docs/getting-started/monetization.md`).
- Third-party content inside the game (fonts, music, sounds, textures, models, logos) has its own
  licenses and owners, and the game author may only hold limited rights to it.

### Rules to follow

1. **Private and local only.** Do not upload the decompiled code, extracted assets, converted
   scenes, or a modified build anywhere: not sbox.game, not GitHub, not Discord, not a mod site,
   not a torrent. A private git repo is fine for yourself; a public one is a publication.
2. **Never publish assets or other IP without permission.** Textures, models, sounds, music,
   shaders, fonts, voice lines, logos, names and characters are all covered. Ask the author. If
   they say yes, get it in writing and follow whatever terms they set.
3. **No standalone export.** Project > Export needs Valve approval and a Facepunch standalone
   license (`sbox-docs/docs/exporting/index.md`). It is not a route for someone else's game.
4. **Do not pass off your edits as the author's work, or the author's work as yours.**
5. **If you want to share an idea, share the idea.** Write your own code and make your own assets
   in a clean project. Describing a mechanic is fine. Copying the implementation is not.
6. **If the author asks you to stop or take something down, do.** The better route is to ask them
   first: many authors are happy to open a modding channel or release source.
7. **Keep `.gitignore` honest.** Add the extraction folders and compiled art to it so you cannot
   push them by accident (see section 9).
8. **Work on copies.** Never write into the s&box install or its download cache. Everything below
   only reads from them.

Respecting the author is also practical: s&box has no mechanism for publishing a mod of someone
else's package, so there is nowhere legitimate to put the result anyway.

---

## 2. The pipeline at a glance

```
 game installed (played once)
        |
        v
 s&box cache  --- .source2/cache/services/*.json   (what the game IS: manifest, dll list, listing)
        |    \-- download/assets/<name>.<crc>.<ext> (the bytes, indexed by CRC)
        v
 1. identify the right JSON files
 2. extract assets to a dump folder         (match manifest Path + Crc to cache files)
 3. fetch what the cache lacks              (cdn.sbox.game only)
 4. decompile the DLL                       (ilspycmd, pinned version)
 5. convert compiled assets to sources      (scenes, prefabs, GameResources: JSON slicing)
 6. stage compiled art as-is                (textures, models, materials, sounds, shaders)
 7. prepare the C# so it recompiles         (strip generator output)
 8. write a .sbproj and open it in the editor
 9. mod
```

What makes this tractable:

- **The code is not obfuscated and keeps its source layout.** s&box injects
  `[SourceLocation( "file.cs", line )]` on every type and member, so you can recover the author's
  original file split.
- **Scenes, prefabs and custom GameResources are plain JSON** inside the compiled file. No Valve
  tooling needed.
- **Art does not need to be decompiled.** The editor loads compiled `_c` art files with no source
  beside them. Re-source only what you want to edit.

---

## 3. Starting point: the JSON files that describe a game

Everything starts in the s&box cache. On Windows with Steam the paths look like this (adjust the
drive and library):

| What | Path |
|---|---|
| JSON metadata | `<sbox>\.source2\cache\services\` |
| Downloaded bytes | `<sbox>\download\assets\` |
| Managed reference DLLs | `<sbox>\bin\managed\` |

where `<sbox>` is e.g. `S:\SteamLibrary\steamapps\common\sbox`.

**Play the game once first.** The cache only contains games your client has downloaded.

Inside `services\` you will find several JSON files. For a game `org.ident` they are:

| File | What it is | Use it for |
|---|---|---|
| `package.<org>.<ident>.json` | The store listing: title, description, tags, `Version`, `AssetLicense`, `SoftwareLicense`, `Meta` | Project identity, **license fields**, startup scene, player counts |
| `manifest.<versionId>.json` | `{"Schema":2,"Files":[...]}`: every file in the game with `Url`, `Crc`, `Path`, `Size` | The list of all assets to extract |
| `dlls.<org>.<ident>.<versionId>.<engineVersion>.json` | A one-entry array pointing at the compiled game code, e.g. `Path: ".bin/package.<org>.<ident>.dll"` | Finding the DLL to decompile |

Gotchas when picking files:

- **There are other manifests in the folder.** Each `manifest.<n>.json` is a different package.
  Several may belong to unrelated packages such as cosmetics. Match the
  manifest to your game via `package.<org>.<ident>.json` -> `Version.Id` (that is `<versionId>`),
  and double check by looking for your game's scene or script names inside it.
- **The package JSON tells you where the manifest came from** (`Version.ManifestUrl`) and the
  startup scene, in `Version.Meta`.
- **Version drift:** when the author updates the game, new `manifest`/`dlls` files appear. Make
  sure the manifest and DLL ids you use belong to the same version.
- The JSON may start with a BOM. Strip it before parsing (`replace(/^\uFEFF/, "")`).

### Read the license fields now

```bash
python -c "import json;d=json.load(open('package.org.ident.json',encoding='utf8'));print(d['AssetLicense'],d['SoftwareLicense'])"
```

`0` / `0` means no license granted. See section 1 before you go further.

### What a manifest entry looks like

```json
{
  "Url":  "https://cdn.sbox.game/asset/<assetId>.<crc>.vtex_c",
  "Crc":  "<crc>",
  "Path": "materials/some/folder/texture_name.generated.vtex_c",
  "Size": 123456
}
```

`Path` is the virtual path inside the game. `Crc` is how you find the file in the cache. The game
DLL entry has the same shape with `Path: ".bin/package.<org>.<ident>.dll"`.

---

## 4. Tools you need

| Tool | Why | Notes |
|---|---|---|
| s&box (Steam) with the editor | Opens and compiles the project | `sbox-dev.exe` or `sbox-editor.exe` |
| **.NET 10 SDK** | Syntax-checking builds from the command line | `winget install Microsoft.DotNet.SDK.10` |
| **`ilspycmd` 9.0.0.7889** | Decompiles the DLL | `dotnet tool install -g ilspycmd --version 9.0.0.7889`. **Pin it.** The latest (11.1.x) ships a broken NuGet package ("DotnetToolSettings.xml not found") |
| Node.js | Runs the helper scripts in `tools/` | Any recent version |
| ValveResourceFormat / Source2Viewer | Only if you must see art (models, textures) | Optional; not needed to mod logic |

---

## 5. Step by step

All commands assume you run them from the project root. Replace paths with yours. **Quote any path
containing `&` or spaces.**

### Step 1: extract the assets from the cache

```
node tools/extract-sbox-package.js "<manifest.json>" "<sbox>\download\assets" .file-dump
```

It copies each manifest entry out of the cache to `.file-dump/<Path>`. The cache is only read.

The cache names files `<name>.<crc>.<ext>`. The extractor indexes them by CRC.

**Trap: CRCs are not always 16 characters.** They are unpadded hex; in one game with about 1,100
files, 47 were 15 characters and 3 were 14. A matcher written for exactly 16 will report those files as "not in
cache", and a wrong diagnosis follows ("I must play more to download them"). Use
`/\.([0-9a-f]{8,16})\.[^.]+$/i`. Before concluding a file is missing, search the cache for its
basename.

The DLL ends up at `.file-dump/.bin/package.<org>.<ident>.dll`.

### Step 2: fetch what the cache does not have

Textures are the usual gap. s&box streams texture data at runtime, so many `.generated.vtex_c`
files were never written to the cache. In one game, about a fifth of the files (hundreds of MB) were missing,
almost all generated `vtex_c`. Without them, most surfaces render as the purple-black
missing-material checkerboard.

**Playing more does not fix it.** Check the cache timestamps if you doubt that.

```
node tools/fetch-missing.js "<manifest.json>" "<sbox>\download\assets" .file-dump --concurrency 8
```

The manifest lists a CDN URL per file. The script only downloads from `https://cdn.sbox.game`
(treat manifest URLs as untrusted data), skips files already present at the manifest size, and
retries. It reproduces what the game client itself downloads. It is a sizeable download, so run it
deliberately.

When done, the manifest entry count should equal files on disk with no size mismatches.

### Step 3: decompile the game code

```bash
ilspycmd -p -o decompiled \
  -r "<sbox>/bin/managed" \
  --nested-directories \
  .file-dump/.bin/package.<org>.<ident>.dll
```

- `-r` points at s&box's managed DLLs. Without it `Sandbox.*` types do not resolve.
- `-p` makes a project. `--nested-directories` mirrors namespaces as folders.
- Expect a project of one file per type, tens of thousands of lines at most for a small game.

**Recover the original file layout.** Before you strip anything (step 5), grep the output for
`[SourceLocation("File.cs", line)]`. ILSpy merges partial classes, so you get fewer files than the author had.
Save the mapping (for example to `.docs/source-location-map.json`) if you want to restore the
author's layout later.

If the package ships a `.xml` next to the DLL, check it for doc comments. It is often empty, so do
not count on it.

### Step 4: convert compiled assets back into editable sources

```
node tools/unpack-assets.js .file-dump Assets
```

Every `_c` file is a Source 2 resource container. Header, then a block table:

```
uint32 fileSize
uint16 headerVersion      // 12
uint16 version            // 0
uint32 blockOffset        // relative to offset 8
uint32 blockCount
// blockCount entries of: char[4] name, uint32 offset (relative to this field), uint32 size
```

Blocks you will see: `GUID`, `RERL` (referenced paths as strings), `REDI`/`RED2`, `DATA`
(payload), `DBLO` (extra data on scenes).

**The key finding: for scenes, prefabs and GameResources, `DATA` is plain UTF-8 JSON, identical to
the source file the editor writes.** The conversion is:

| Compiled block | Source file |
|---|---|
| `DATA` (strip trailing NULs) | `name.ext` |
| `DBLO` | `name.ext_d` (blob sidecar, byte for byte) |
| `GUID` | `name.ext.meta` as `{ "guid": "<guid>" }` |

What converts, and what stays compiled:

| JSON, convert to source | Binary, keep as `_c` |
|---|---|
| scene, prefab, decal, sound (`SoundEvent`), plus every custom GameResource extension the game defines (check the DLL for `[AssetType(... Extension = ...)]`) | vtex, vmat, vmdl, vsnd, shader |

**Probe, do not guess.** Try parsing the `DATA` block of every compiled extension in the manifest
as JSON. This found dozens of `.sound_c` files (`SoundEvent`) that were JSON, which a "known types" list
would have missed. Repeat the probe when the game updates.

Details worth knowing:

- **GUIDs:** scenes and resources reference assets by `Id` *and* `Path`. Keep the original GUID
  (from the `GUID` block) when writing sources, or every reference to the asset breaks. Only some
  assets carry a `GUID` block; the rest are referenced by path and need no `.meta`.
- **`$blob`:** `{"$blob": "<guid>"}` entries point into the `_d` sidecar. Mesh geometry from the
  editor's block tools and clutter data live there; most of a map's bytes are blobs. Compiled files
  also contain an extra `$refs` array next to `$blob` that sources do not have. It is harmless.
- **Compiler-inlined data:** a compiled scene can differ from its source by data the compiler
  inlined (for example an SVG texture reference expanded to base64). It adds resolved data and
  drops nothing.
- **Verify the conversion.** `node tools/verify-exact.js .file-dump Assets` minifies each source and
  compares it byte for byte with the original `DATA` block. Run it after changing the converter.
- **Integrity audit.** Resolve every `$blob` against its sidecar and every asset path reference
  against the manifest. Expect every blob to resolve, and a few "missing" paths that are
  engine content shipped in `<sbox>\core\` (light cookies, particle textures, `models/dev/plane`).
  Check there before deciding something is missing. When scraping paths out of `RERL`, allow `+`
  in file names or you will clip entries like `+name_png_1a2b3c4d.generated.vtex_c`.

### Step 5: stage the compiled art

```
node tools/stage-art.js .file-dump Assets
```

Copies `vtex_c`, `vmat_c`, `vmdl_c`, `vsnd_c`, `shader_c`, and raw files the game ships as-is
(`.png`, `.ttf`, custom formats) into `Assets/` at their virtual paths. s&box keeps compiled `_c`
beside sources in `Assets/`, and the engine loads them with no source present. Hundreds of art
files can be staged this way without a recompile.

Games can declare custom raw file types in their package (`Resources: "*.ext"` in the package
`Meta`). Carry that setting into the `.sbproj`, and add the extension to the staging script.

### Step 6: make the code recompile

```
node tools/prepare-code.js decompiled Code
```

ILSpy faithfully reproduces what s&box's source generators emitted into the DLL. Left in place,
those members collide with the generator running again at build time. The script is idempotent and
removes:

| Generator output | Fix |
|---|---|
| `[SourceLocation]` attributes (one per type and member) | strip; save the mapping first |
| `using Sandbox.Internal;` noise | strip |
| `IUpdateSubscriber` / `IFixedUpdateSubscriber` / `IPreRenderSubscriber` in base lists | delete them. The generator adds them to any `Component` that overrides `OnUpdate` / `OnFixedUpdate` / `OnPreRender`. Do not add a `using` |
| `[Rpc.*]` methods whose real body is inside a `Resume` delegate | promote the body out of the delegate and dedent by **two** levels |
| `[Sync]` properties expanded into manual backing logic | restore the auto-property |
| `__*__Attrs`, `__*_Cached*` fields, orphan `k__BackingField`, mangled `base._002Ector()` | delete |
| `[DefaultValue]` on a property that already has an initializer | delete (the generator derives it; non-"easy" types like `Vector2Int` and `TimeSince` get a duplicate attribute and fail) |

Also:

- `using` fixes: ILSpy may print engine-generated members as fully qualified calls, e.g.
  `StringToken.Literal("Name", 1234u)`. That is `StringTokenUpgrader` output. Write
  `Attributes.Set("Name", value)`.
- If you restore the original file layout, a class that was `partial` in the source may have lost
  the keyword when ILSpy merged the parts. Re-add `partial` before splitting it.

### Step 7: write the project files

Layout (modelled on any working s&box project):

```
mygame.sbproj
Code/            C# sources (+ csproj, which the editor regenerates)
Assets/          JSON sources + compiled art, same virtual tree as the manifest
ProjectSettings/ *.config
Libraries/
Editor/
```

- **ProjectSettings:** the package ships them at the virtual path `projectsettings/*.config`
  (lowercase). Move them to `ProjectSettings/` at the project root. They are JSON with a
  `__schema` field.
- **`.sbproj`:** build it from `Version.Meta` in `package.<org>.<ident>.json`: `StartupScene`, min
  and max players, `Resources`, `GameNetworkType`. Set `"Type": "game"` and a `CsProjName`.
- **Use `"Org": "local"`, not the author's org.** Scene and resource JSON references components by
  bare type name (`"__type": "MyNamespace.MyComponent"`), never assembly-qualified, so the assembly name does
  not need to match the original. Using someone else's org ident on a local project would also be
  wrong.
- **The csproj:** the editor regenerates `Code/<name>.csproj` from the `.sbproj` every time the
  project opens. Hand edits there do not survive. If you build outside the editor, send
  `OutputPath` somewhere inside your project, never into `sbox/.vs/output` or the install.

### Step 8: open it

```powershell
& "<sbox>\sbox-dev.exe" -project "<path to>\mygame.sbproj"
```

On cmd.exe, drop the leading `&`. **Quote the project path.** Pasting the cmd line into
PowerShell fails in the parser before s&box starts. Close any running editor first.

The editor is the real acceptance test. A clean result is every C# file and every converted asset
compiling with zero errors, and the startup scene loading.

---

## 6. Verifying it works

| Check | How | Good result |
|---|---|---|
| Round trip | Editor rebuilds `_c` beside each source; compare `DATA` with your source and compare sizes | identical, same size as shipped |
| Code compiles | Open in the editor and read the console | 0 errors |
| References | Audit blobs and asset paths (step 4) | 0 broken in the game's own content |
| Behaviour | Play every map, compare with the retail build | matched |

You can launch the retail game for comparison with
`"<sbox>\sbox.exe" -rungame <org>.<ident>`. That opens the game's page in the menu; it does not
start it directly.

---

## 7. Gotchas and traps

**A green `dotnet build` proves little.** The `Sandbox.Generator` analyzer targets a different
Roslyn version than the SDK ships, so the SDK silently skips it (warning `CS9057`). No generator
means no duplicate members and no errors, even when the code will fail in the editor with
`Ambiguity between 'X.__m_..__Attrs' and ...`. Treat `dotnet build` as a syntax check only.

**Do not name a recovered shader like the original.** Every s&box `shader_c` embeds its
preprocessed source as JSON (`{"Programs":{"VFX_PROGRAM_VS":...,"VFX_PROGRAM_PS":...}}`). You can
read exactly what an effect does:

```bash
grep -a -o '{"Programs":.*' Assets/shaders/some.shader_c
```

Includes are inlined and macros expanded, so it is readable but not pretty. To change an effect,
author a new `.shader` under a **new name** and switch to it from code. A `.shader` placed beside
the original `.shader_c` with the same name makes the editor recompile over the shipped bytecode.

**Dangling references that were never yours to fix.** Packages can ship with references to files
they did not publish (for example baked-lighting volumes under a dev scene path, or dev
leftovers). The retail game has the same errors. Do not chase them.

**Models and materials do not round-trip nicely.** `vmdl_c` -> editable is manual and lossy for
skinning and attachments, `vmat_c` is usually fine through ValveResourceFormat. Leave them
compiled unless you truly need to edit them.

**Leaderboards and other platform services** may throw inside the engine for a local package. It is
harmless.

**Engine drift.** The package records the engine version it was built for. Usually the installed
client runs it fine, but if you hit API errors, compare against the
public engine source (`sbox-public`).

**Never write into the cache or install.** If the cache looks wrong, re-run the game or the fetch
step; do not edit it.

---

## 8. Where to read code once you have it

Start with the namespaces and conventions, not line by line:

- `Component` subclasses are the building blocks; look for a `Manager` per subsystem and a
  singleton entry point.
- Custom data types derive from `GameResource` with `[AssetType( ..., Extension = "..." )]`. Their
  JSON files are your tuning knobs (weapons, enemies, dialogue).
- Search for `[Property]`, `[Sync]`, `[Rpc.*]`, `[Create]`-style attributes to find what the
  scenes configure.
- Hot-reload works in the editor: change C# or JSON and the project rebuilds.

Good first mods, ordered by effort: tuning numbers in a GameResource, a new weapon file or enemy
definition cloned from an existing one, a code-side toggle (effects, crosshair, auto-reload), a
new scene built from existing prefabs, new movement code.

---

## 9. Housekeeping

Add this to `.gitignore` the moment you create the project, so nothing sensitive is committed:

```
.file-dump/
Assets/**/*.vtex_c
Assets/**/*.vmat_c
Assets/**/*.vmdl_c
Assets/**/*.vsnd_c
Assets/**/*.shader_c
Assets/**/*.png
Assets/**/*.ttf
bin/
obj/
.sbox/
.vs/
```

The converted JSON sources and the decompiled code are still the author's work. Keep the whole
repo private. If you want a reproducible setup, commit **your scripts and your own changes**
(patches, new files you wrote) and have each person regenerate the rest from their own install.

---

## 10. Quick reference

| I want to... | Do |
|---|---|
| Find the right manifest | `package.<org>.<ident>.json` -> `Version.Id` -> `manifest.<id>.json` |
| Find the DLL | `dlls.<org>.<ident>.<id>.<engine>.json` -> `Path` -> extracted to `.file-dump/.bin/` |
| Check the license | `AssetLicense` / `SoftwareLicense` in the package JSON (0 = none granted) |
| Get every file | `extract-sbox-package.js`, then `fetch-missing.js` for the gaps |
| Decompile | `ilspycmd` 9.0.0.7889 with `-r <sbox>/bin/managed` |
| Get editable scenes/items | `unpack-assets.js` (slice `DATA`, `DBLO`, `GUID`) |
| Keep art working | `stage-art.js` (leave it compiled) |
| Make code build | `prepare-code.js`, then confirm in the editor |
| Open | `sbox-dev.exe -project "<sbproj>"` |

And once more: **private, local, for modding only. Do not publish code, assets or builds, and ask
the author before sharing anything derived from their game.**
