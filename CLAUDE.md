# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This is not a source repository. It is the **shipped build of "Scampton the Great"**, a Deltarune-style bullet-hell fangame made in **GameMaker Studio 2** (non-YYC, bytecode version 17, project name `Scampton_The_Great`). There is no source tree, build system, linter or test suite. The user's goal is to modify the game completely, so every change is made by editing the compiled `data.win` with UndertaleModTool.

Files in this folder:
- `data.win` (~150 MB) holds all game content: code, sprites, objects, rooms and 102 embedded sounds.
- `data.win.bak` is the pristine original. It was made before the first audio replacement. Never overwrite it, and build every modification from it (or from a deliberately chosen working copy).
- `audiogroup1.dat` and `audiogroup2.dat` are external audio groups. `AG_Music` holds `TS_Intro` and `TS_End`. `AG_Sounds` holds `snd_PhoneLeft`, `snd_PhoneRight` and `snd_Blast`.
- `scamptonthegreat.exe` is the GameMaker runner. It needs no changes and loads `data.win` at startup.
- `options.ini` and `creep.mp4` are runtime settings and a video asset.

## Tooling (lives outside this folder)

UndertaleModTool CLI v0.9.2.0 is unpacked at `C:\Users\misap\Desktop\utmt\cli\UndertaleModCli.exe`. The helper C# scripts are in `C:\Users\misap\Desktop\utmt\`: `list.csx` (list sounds), `exp.csx` (export audio), `rep.csx` (replace the `BIG` sound), `arch.csx` (list rooms, objects and scripts). `utmt\export\` holds exported audio. The GUI build (UndertaleModTool_v0.9.2.0-Windows.zip from the GitHub releases) can be downloaded if visual editing is needed.

Useful commands (run from bash):

```bash
CLI=C:/Users/misap/Desktop/utmt/cli/UndertaleModCli.exe
G=C:/Users/misap/Desktop/scampton-the-great

# Quick info (fast)
$CLI info $G/data.win < /dev/null

# Dump ALL decompiled GML code to a folder
$CLI dump $G/data.win -c UMT_DUMP_ALL -o C:/Users/misap/Desktop/utmt/code < /dev/null

# Dump specific code entries, strings or sprites
$CLI dump $G/data.win -c gml_Script_scrDIA -s --sprites -o <dir> < /dev/null

# Run a C# script against the data file and write the result to a NEW file
$CLI load $G/data.win.bak -s script.csx -o C:/Users/misap/Desktop/utmt/data_new.win -f < /dev/null
cp C:/Users/misap/Desktop/utmt/data_new.win $G/data.win
```

Pitfalls:
- **Always redirect stdin with `< /dev/null`** (in PowerShell, use bash instead). Without it the CLI waits for input and hangs with near-zero CPU.
- **Always pass `-o <new file> -f`** when using `load`. Don't let it write back over `data.win` implicitly, and always test the result before copying it into place.
- Scripts are C# (`.csx`) run with the UTMT scripting API (`Data.Sounds`, `Data.GameObjects`, `Data.Code`, ...). `System.Linq` is not imported by default, so add `using System.Linq;` for `.Select`/`.Take`. Use `ScriptMessage(...)` to print.
- Loading and saving the 150 MB file takes about 20–60 s, so run it with a timeout of at least 120 s.
- There is no way to run automated checks. To verify a change, launch `scamptonthegreat.exe` and play the relevant part.

## Game structure (from `arch.csx`)

- **Rooms (9):** `rMainu` (menu), `rIntro`, `rIntro_2`, `rBattle_2` (the main fight), `rLibrary`, `rNoHit`, `rGameOver`, `rCredits`, `rOutro`.
- **Objects (50):** The `o*` prefix is used. Battle flow and the combat framework: `oStartBattle1`, `oBattleBC`, `oBox`, `oSoul`, `oPattern`, `oMakeUI`, `oActionSET`, `oFightDraw`, `oFightStick`, `oBullet`, `oBigBullet`, `oBlueAttack`, `oWhiteAttack`, `oParryAttack`, `oRudeBuster`. Party and enemies: `oKris`, `oSusie`, `oRalsei`, `oSpamtonRig`, `oMarselo`, `oPep`, `oPepStick`, `oGoldenPipis`. Others: `oTextEngine`, `oTextBrick`, `oGameOver_NoHit`, `oGameOverSET`.
- **Scripts (98 entries):** Gameplay helpers are named `scr*` / `gml_Script_*` (the pair is a wrapper plus its function). Notable ones are `scrMakeAttack`, `scrEndPattern`, `scrParry`, `scrDIA` (dialogue), `scrTextSpez`, `scrFlavourBub`, `scrDice` and `scrSkipIntro`. Save handling is in `saveGame`, `loadGame` and `deleteSave`.
- **Audio:** 108 sounds across the audio groups `audiogroup_default` (embedded), `AG_Music` and `AG_Sounds` (external `.dat`).
- **Embedded audio is always raw PCM WAV, even for sounds whose original file name ends in `.ogg` or `.mp3`.** The main battle theme is the sound `BIG`. The other long tracks are `Bigger_bang_fin`, `junkilpede_music_full`, `circus` and `mus_lib`.

## Replacing audio (verified workflow)

Convert the source to the same format the game stores it in, then replace `AudioFile.Data` in a script:

```bash
ffmpeg -y -i input.mp4 -vn -c:a pcm_s16le -ar 44100 -ac 2 -f wav new_big.wav
```

```csharp
foreach (var s in Data.Sounds)
    if (s.Name.Content == "BIG") s.AudioFile.Data = File.ReadAllBytes(@"C:\...\new_big.wav");
```

This works only for **embedded** sounds (`s.AudioFile != null`). Sounds in `AG_Music` / `AG_Sounds` live in `audiogroup*.dat`, so open that file with `load` instead of `data.win`. This approach was tested on `BIG`. Embedding a real `.ogg` in `data.win` has not been tried.

## Mechanics found while modding (story attacks, menu)

Build history: `data.win.before_v3` is the state before the story/menu update. The build script is `utmt\story_menu_v3.csx`. It is applied to the current `data.win`, not to `.bak`.

- **Heart attacks `B1`/`B2`/`B3`** (`oPattern_Step_0`): The `Scams` head (`oWhiteAttack`, flag `Scams`) flies in. At Tyme 84 the hearts appear (`oParryAttack` with `ScamsHeart1@HeartN`, logic in `oParryAttack_Step_0`). A heart is parried with Z/Enter (`oSoul.ParryTime`, a 10-frame window). Each parry subtracts `k2` (which grows to `oPattern.i2`) from the heart's `k1=100`. A broken heart does `oPattern.Var1--`/`Var2--`. When `Var1<=0` the head flies off and sets `oPattern.i3=1`, and 50 frames later the pattern ends (`Pattern = PastPT`). In B3 without `LoopGoop`, `Var1==1` spawns the `Heart4` cutscene. The head shoots from Tyme 275 every 96 frames.
- **N1**: The anvil (`Anvil`) and the Spamton with scissors (`AnvilSpamton`). The anvil drop time is written to `oPattern.i1` (150–200). The anvil does not reach the box; it is a gag with a shake. The pattern ends at Tyme 275.
- **L1**: The big Spamton (`LastP`, drawn in `oWhiteAttack_Draw_0`) shoots from his plush hands at (150,340) and (490,340). Every 5th shot is a bomb (`oParryAttack` `BOMB`). A parried bomb flies up, and at `y<180` it does `oPattern.k1--` (3 hits). At `k1==0`, 140 frames later it sets `LoopGoop=true` and ends.
- Box sizes come from `scrBoxStats(x,y,xs,ys,depth,..)` in `oActionSET_Step_0`. `sBox` is 12×12 with a centred origin, so the box is `12*xs` × `12*ys`. B1–B3: x=220, 240×120, depth −12000. N1: 320,220, 180×180. L1: 320,340, 192×144, depth −12000. **In patterns with depth −12000 the box covers bullets at −7000, so create add-on bullets at depth −15000.**
- Sprite origins: `sString2` (1,0) is top-centre and good for lines. `sOrb` (12,8). `sMoney` (6,6). `sFont_Det` (0,0): the glyph is 6×9, and the frame is the index in `BET = " ABC…xyz0123456789*:.,-!'[]()¤%?/><$="` minus 1. The menu uses this font at scale 2 with an advance of 12 px. `sSoul` frame 1 is a white heart; tint it red.
- Comparing a string with a number (`Pattern == "B1"` when Pattern is 3) is safe in this runtime. Still, the new triggers use numbers (`oP.ForceStory` 1..5, `oP.PracticePat` 1..11).
- The add-on layers for story attacks are in the add-on block at the end of `oPattern_Step_0` (vars `Sx1..Sx3`, `Hang`, `Hsh`, `Sh0`, `SLim`, bullet flags `HShield`, `SAdd`, `SStr`). `SStr` is a telegraphed string: red, harmless, then a damaging white line for 20 frames.
- Hotkeys in battle: 1–7 pick a regular attack; 8/9/0 = B1/B2/B3; `-` = N1; `=` = L1 (the next turn; L1 shifts the HUD and turns on LoopGoop, so it is for testing only). Shift skips the current attack.
- Menu (`rMainu`): `oCC2` `MenuPris` draws the menu (`gml_Object_oCC2_Draw_0`), and the items are `oTextEngine` with `Cur2`. Navigation is `scrMMcursor(cur, up, down, left, right)` in `oAlways_Step_0`. Extras now has a new item, `s_MMP` "Practice". Left/right choose `oP.PracIdx` and Z starts the battle with `oP.PracticePat = idx+1`. `oActionSET` forces that attack every turn. Entering `rMainu` resets practice.
- Credits (`rCredits`): `oCC2` `CredPlace` in `oCC2_Draw_0`. Entries are `cred(name, role)` with `£` as the line break in roles. The last two entries must stay the blank spacer and `Return_i5`, because navigation skips `length-2`. Icons are `sCredCon` frames by index. v4 (`utmt\menu_credits_v4.csx`, backup `data.win.before_v4`) puts Claude first with a procedural 12-ray spark (`draw_triangle`, colour 217,119,87), removes Special Thanks and remaps icon frames (`k1-1`, and `k1` for index ≥15).
