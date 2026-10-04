# Metroid Prime Trilogy (USA) on a PAL Wii — Priiloader hack

Play the **NTSC-U Metroid Prime Trilogy disc (`R3ME01`)** on a **European Wii (System Menu 4.3E)**, launched normally from the Disc Channel.

Without this hack (and even with Priiloader's *Region Free EVERYTHING*), the trilogy menu loads, but selecting **Metroid Prime** or **Metroid Prime 2** fades to black and the console resets. **Metroid Prime 3** works.

Made possible with Claude.

## Requirements

- System Menu **4.3E (v514)**
- Priiloader with **Region Free EVERYTHING** enabled (so the disc can boot at all)
- Wii set to **60 Hz (EuRGB60)** in Wii Settings → Screen
- A display that accepts NTSC 480i/480p: HDMI adapter, component, or RGB SCART. Composite on a PAL-only TV shows black and white.

## Install

1. Append the entry from `mpt_ntscu_fix_hack.ini` to the **end** of the `hacks_hash.ini` that shipped with your Priiloader version. Priiloader stores which hacks are on by their position in the list, so appending keeps your existing settings.
2. Copy the file to `SD:/apps/priiloader/hacks_hash.ini`.
3. In Priiloader → *System Menu Hacks*, enable **MPT (USA) fix for PAL Wii (60Hz)** and save.

## How it works

### 1. Why it crashes

The trilogy is four DOLs: a front-end (`rs5fe_p.dol`) and one per game (`rs5mp1_p.dol`, `rs5mp2_p.dol`, `rs5mp3_p.dol`). When you pick a game, the front-end calls the SDK's `OSExec()`. That reloads the IOS the disc requires, re-runs the apploader and boots the game's DOL, without going back through the System Menu.

That launch path has no region check. The problem is the **video mode**:

- The System Menu writes the console's TV format to low memory `0x800000CC`: `1` (PAL) or `5` (EuRGB60) on a European Wii. Region Free EVERYTHING doesn't change this; its patches only bypass region comparisons.
- The SDK's `VIInit()` in each DOL derives the TV format from the VI hardware and that word, and `OSExec` leaves it untouched.
- The front-end and MP3 handle PAL and EuRGB60. The NTSC builds of **MP1 and MP2** don't: their render-mode selection (MP1 `0x80351b84`, MP2 `0x803706cc`) prints *"PAL TV Format not compatible with NTSC build"* when `VIGetTvFormat()` returns 1 or 5. It then continues with a NULL render-mode pointer and crashes.

### 2. The fix

Before the disc boots, set `0x800000CC = 0` (NTSC), but only for `R3ME`. With the Wii at 60 Hz, the VI hardware is already in NTSC timing (EuRGB60 uses NTSC timing), so every DOL in the trilogy then reports NTSC.

The value stays in low memory through every switch between the trilogy menu and the games, so the fix only needs to run once, at launch. This is also what Priiloader's own disc launcher does for NTSC discs, which is why booting the disc from Priiloader's menu already worked.

Launching through the System Menu instead of Priiloader keeps the genuine Disc Channel launch, and the System Menu still writes the play record, so the session appears on the Message Board.

### 3. Where the patch goes

In the System Menu's `BS2StartGame` (4.3E, `0x8137bec8`), just after the disc ID is in memory at `0x80000000` and just before IOS is reloaded and the apploader runs, the menu copies boot info to low memory and flushes it. Those 22 instructions are rewritten so they do the same work in fewer instructions, freeing room for the check. No extra code is loaded and no free memory is needed.

| Original | Patched |
|---|---|
| `lis r28,0x8000` | `lis r28,0x8000` |
| `li r0,0x80` | `lwz r4,0(r28)` — disc ID |
| `lwz r4,0(r28)` | `stw r4,0x3180(r28)` |
| `addi r3,r29,0x9c0` | `li r0,0x80` |
| `stw r4,0x3180(r28)` | `stb r0,0x3184(r28)` |
| `stb r0,0x3184(r28)` | `addis r5,r4,-0x5233` — subtract `'R3'` |
| `lwz r0,0x3a0(r3)` | `addic. r5,r5,-0x4D45` — subtract `'ME'`; result is 0 only for `R3ME` |
| `cmpwi r0,0` | `bne +8` |
| `beq +0xc` | `stw r5,0xCC(r28)` — **TV format = NTSC (0)** |
| `li r0,0x80` | `lwz r0,0xd60(r29)` — same field as `0x3a0(r29+0x9c0)` |
| `b +8` | `neg r5,r0` |
| `li r0,0` | `or r0,r5,r0` — top bit set if the field is non-zero |
| `lis r6,0x8000` | `rlwinm r0,r0,8,24,24` — gives 0x80 or 0, branch-free |
| `li r4,0x100` | `stb r0,0x3187(r28)` |
| `stb r0,0x3187(r6)` | `lwz r5,-0x55b0(r13)` |
| `addi r3,r6,0x3100` | `lwz r0,4(r5)` |
| `lwz r5,-0x55b0(r13)` | `stw r0,0x3194(r28)` |
| `lwz r0,4(r5)` | `lwz r0,0(r5)` |
| `stw r0,0x3194(r6)` | `stw r0,0x3198(r28)` |
| `lwz r5,-0x55b0(r13)` — reload | `addi r3,r28,0xC0` |
| `lwz r0,0(r5)` | `li r4,0x3140` — flush `0x800000C0`–`0x80003200` |
| `stw r0,0x3198(r6)` | `mr r6,r28` |

The next instruction is the original `bl DCFlushRange`. Its range is widened from `0x80003100`–`0x80003200` to `0x800000C0`–`0x80003200` so the new value is written to RAM before IOS is reloaded. The other low-memory writes, the flag at `0x80003187`, and registers `r0`, `r5`, `r6` and `r28` end up with the same values as the original code. Only `r3`/`r4` (the flush arguments) differ, deliberately.

### 4. Scope and safety

- It only triggers for disc ID `R3ME`. PAL MPT (`R3MP`) and every other disc are untouched.
- `minversion = maxversion = 514`, so it only applies to 4.3E. The patch is located by matching all 22 original instructions, and that pattern appears once in 4.3E.
- Nothing is written to NAND beyond Priiloader's own hack settings.

## Limitations

- At **50 Hz** the VI hardware stays in PAL timing and MP1/MP2 still crash. Set the Wii to 60 Hz.
- The output is NTSC rather than PAL60. A **PAL60** version for composite users would need MP1 and MP2 themselves patched after they load. That's possible but not implemented.
- Only tested on 4.3E v514.
